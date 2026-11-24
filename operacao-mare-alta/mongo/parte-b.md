# Parte B — Aggregation Pipeline

## B1 — Relatório gerencial

### Índice

```javascript
db.movimentacoes.createIndex({
  classe: 1,
  ts: 1
})
```

### Pipeline correto

```javascript
db.movimentacoes.aggregate([
  {
    $match: {
      classe: "Refrigerado",
      ts: { $gte: ISODate("2026-04-01T00:00:00Z") }
    }
  },
  {
    $group: {
      _id: "$terminal",
      total_movimentacoes: { $sum: 1 },
      peso_total: { $sum: "$peso_kg" },
      peso_medio: { $avg: "$peso_kg" },
      maior_peso: { $max: "$peso_kg" }
    }
  },
  { $sort: { peso_total: -1 } },
  { $limit: 3 },
  {
    $project: {
      _id: 0,
      terminal: "$_id",
      total_movimentacoes: 1,
      peso_total: 1,
      peso_medio: 1,
      maior_peso: 1
    }
  }
])
```

### Explain

```javascript
db.movimentacoes.explain("executionStats").aggregate([
  {
    $match: {
      classe: "Refrigerado",
      ts: { $gte: ISODate("2026-04-01T00:00:00Z") }
    }
  },
  {
    $group: {
      _id: "$terminal",
      total_movimentacoes: { $sum: 1 },
      peso_total: { $sum: "$peso_kg" },
      peso_medio: { $avg: "$peso_kg" },
      maior_peso: { $max: "$peso_kg" }
    }
  },
  { $sort: { peso_total: -1 } },
  { $limit: 3 },
  {
    $project: {
      _id: 0,
      terminal: "$_id",
      total_movimentacoes: 1,
      peso_total: 1,
      peso_medio: 1,
      maior_peso: 1
    }
  }
])
```

O plano utilizou `IXSCAN`, `FETCH` e `PROJECTION_SIMPLE`. Foram examinados 70.405 documentos e 70.405 chaves. O estágio `$group` recebeu 70.405 documentos e criou 5 grupos. A ordenação não precisou utilizar o disco e limitou o relatório aos três terminais de maior peso total.

## B2 — Debug arquitetural

### a) Código original executado sem alterações

```javascript
db.movimentacoes.aggregate([
  { $project: { terminal: 1, classe: 1 } },
  {
    $group: {
      _id: { terminal: "classe" },
      kg_total: { $sum: "peso_kg" },
      mov: { $sum: 1 }
    }
  },
  { $limit: 3 },
  { $sort: { kg_total: -1 } },
  { $match: { "_id.classe": "Refrigerado" } }
])
```

A execução não apresentou erro, mas retornou apenas T2 com `20.119` movimentações e `kg_total: 0`. Isso não corresponde ao resultado esperado dos três terminais com maior peso total.

### b) Defeitos encontrados

| Defeito | Classificação |
|---|---|
| `$project` removeu `peso_kg` antes do agrupamento | Erro de lógica |
| `_id` agrupou o literal `classe` como chave, em vez de usar `$terminal` | Erro de lógica |
| `{ $sum: "peso_kg" }` não referencia o campo corretamente | Erro de lógica |
| `$limit` ocorreu antes do `$sort` | Erro de lógica |
| `$match` da classe foi aplicado somente no final | Erro de lógica |
| `$project` antes do `$match` processou documentos desnecessários | Erro de desempenho |

### c) Pipeline corrigido

```javascript
db.movimentacoes.aggregate([
  { $match: { classe: "Refrigerado" } },
  {
    $group: {
      _id: "$terminal",
      kg_total: { $sum: "$peso_kg" },
      mov: { $sum: 1 }
    }
  },
  { $sort: { kg_total: -1 } },
  { $limit: 3 },
  {
    $project: {
      _id: 0,
      terminal: "$_id",
      kg_total: 1,
      mov: 1
    }
  }
])
```

O pipeline corrigido filtra a classe no início, agrupa por terminal e soma `peso_kg` corretamente. Em seguida, ordena pelo peso total e limita aos três maiores antes da projeção final.

### d) Impacto da refatoração

O `$match` inicial reduz a quantidade de documentos enviados às etapas seguintes, diminuindo uso de RAM no `$group`, CPU e ordenação. Além disso, permite aproveitar melhor um índice apropriado em `{ classe: 1, ts: 1 }` e evita processamento desnecessário. A alteração também garante que a ordenação seja feita sobre os pesos calculados, em vez de limitar registros antes de classificá-los.