# Parte A — MongoDB

Parâmetros: classe `Refrigerado`, terminal `T4`, data de corte `2026-04-01T00:00:00Z` e incidente `-25.4680, -48.5100`.

## A1 — Índices compostos e painel do pátio

### Consulta base

```javascript
db.movimentacoes.find({
  terminal: "T4",
  classe: "Refrigerado",
  ts: { $gte: ISODate("2026-04-01T00:00:00Z") }
})
.sort({ peso_kg: -1 })
.limit(20)
.explain("executionStats")
```

### a) Sem índice

O plano apresentou `COLLSCAN` e `SORT`, examinou 400.000 documentos, 0 chaves e retornou 20 documentos em 532 ms.

### b) Índice na ordem do filtro

```javascript
db.movimentacoes.createIndex({
  terminal: 1,
  classe: 1,
  ts: 1,
  peso_kg: -1
})
```

```javascript
db.movimentacoes.find({
  terminal: "T4",
  classe: "Refrigerado",
  ts: { $gte: ISODate("2026-04-01T00:00:00Z") }
})
.sort({ peso_kg: -1 })
.limit(20)
.explain("executionStats")
```

O plano apresentou `IXSCAN`, `SORT` e `FETCH`. Foram examinadas 14.119 chaves e 20 documentos; o comando retornou 20 registros em 23 ms.

### c) Índice ESR

Primeiro, remova o índice da letra b:

```javascript
db.movimentacoes.dropIndex({
  terminal: 1,
  classe: 1,
  ts: 1,
  peso_kg: -1
})
```

Depois crie o índice ESR:

```javascript
db.movimentacoes.createIndex({
  terminal: 1,
  classe: 1,
  peso_kg: -1,
  ts: 1
})
```

Execute novamente a consulta da letra a. O plano apresentou `LIMIT`, `FETCH` e `IXSCAN`, sem estágio `SORT` separado. Foram examinadas 28 chaves e 20 documentos; foram retornados 20 registros em 8 ms.

### d) Comparação

| Cenário | Stage principal | Docs examinados | Chaves examinadas | nReturned | Tempo |
|---|---|---:|---:|---:|---:|
| Sem índice | `COLLSCAN`, `SORT` | 400.000 | 0 | 20 | 532 ms |
| Índice filtro | `IXSCAN`, `SORT`, `FETCH` | 20 | 14.119 | 20 | 23 ms |
| Índice ESR | `IXSCAN`, `FETCH`, `LIMIT` | 20 | 28 | 20 | 8 ms |

Na B-Tree, os campos usados por igualdade podem selecionar uma faixa contínua de valores. Porém, ao colocar `ts` antes de `peso_kg`, o campo de intervalo impede que a B-Tree já entregue os resultados na ordem exigida por `peso_kg`; portanto, o MongoDB executa `SORT`. No índice ESR, terminal e classe são campos de igualdade, peso é colocado antes do campo de intervalo `ts` e mantém a ordem do resultado. Assim, o índice já segue a ordenação exigida e não precisa de etapa separada de ordenação.

## A2 — Incidente no pátio

### Consulta sem o índice

```javascript
db.movimentacoes.find({
  localizacao: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [-48.5100, -25.4680]
      },
      $maxDistance: 300
    }
  }
}).limit(10)
```

### Índice geoespacial

```javascript
db.movimentacoes.createIndex({ localizacao: "2dsphere" })
```

### Consulta com explain

```javascript
db.movimentacoes.find({
  localizacao: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [-48.5100, -25.4680]
      },
      $maxDistance: 300
    }
  }
}).limit(10).explain("executionStats")
```

Após criar o índice, o plano apresentou `LIMIT`, `GEO_NEAR_2DSPHERE`, `FETCH` e `IXSCAN`. Foram observados 0 documentos examinados, 0 chaves examinadas, 0 retornos e tempo de 3 ms. Portanto, não havia movimentações dentro do raio de 300 metros informado.

## A3 — TTL que não apaga

### Comprovação do tipo

```javascript
db.alertas.aggregate([
  {
    $project: {
      registrado_em: 1,
      tipo_campo: { $type: "$registrado_em" }
    }
  },
  { $limit: 5 }
])
```

### Índices existentes

```javascript
db.alertas.getIndexes()
```

O campo `registrado_em` foi encontrado como string, não como `Date`, e não havia um índice TTL criado na coleção. O TTL do MongoDB exige que o campo indexado contenha valores BSON `Date` ou um array de datas. Texto não é convertido pelo monitor TTL.

### Índice TTL correto

```javascript
db.alertas.createIndex(
  { registrado_em: 1 },
  { expireAfterSeconds: 2592000 }
)
```

Para novos alertas, a aplicação deve gravar:

```javascript
{
  registrado_em: new Date()
}
```

## A4 — Consulta coberta

### Índice para cobertura

```javascript
db.movimentacoes.createIndex({
  tipo: 1,
  terminal: 1,
  container: 1
})
```

### Consulta coberta

```javascript
db.movimentacoes.find(
  { tipo: "inspecao", terminal: "T4" },
  { _id: 0, tipo: 1, terminal: 1, container: 1 }
).explain("executionStats")
```

Com `_id: 0` e todos os campos necessários presentes no índice, o plano pode utilizar somente `IXSCAN`/projeção coberta. Os registros apresentados no teste foram 16.219, com 16.219 chaves examinadas e 0 documentos examinados.

### Consulta sem `_id: 0`

```javascript
db.movimentacoes.find(
  { tipo: "inspecao", terminal: "T4" },
  { tipo: 1, terminal: 1, container: 1 }
).explain("executionStats")
```

Ao projetar o campo `_id` implicitamente, a consulta deixa de ser coberta pelo índice criado, porque o `_id` não faz parte dele. O plano observado utilizou `PROJECTION_SIMPLE`, `FETCH` e `IXSCAN`, com 16.219 documentos e 16.219 chaves examinadas, em 24 ms.

### Auditoria de índices

```javascript
db.movimentacoes.getIndexes()
```

No final das partes A e B, os índices de teste não usados devem ser removidos para evitar penalização por índice excedente.