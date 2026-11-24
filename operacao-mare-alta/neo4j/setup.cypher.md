# Carga do grafo Neo4j

Execute as consultas nesta ordem: Terminais, Empresas, Sociedades, Contêineres, Passagens e Pilha.

## 1. Terminais

```cypher
UNWIND [{nome:"T1"},{nome:"T2"},{nome:"T3"},{nome:"T4"},{nome:"T5"}] AS t
CREATE (:Terminal {nome: t.nome});
```

## 2. Empresas

```cypher
UNWIND [
  {cnpj:"E0",nome:"Empresa 0",sancionada:false},
  {cnpj:"E1",nome:"Empresa 1",sancionada:false},
  {cnpj:"E2",nome:"Empresa 2",sancionada:true},
  {cnpj:"E3",nome:"Empresa 3",sancionada:false},
  {cnpj:"E4",nome:"Empresa 4",sancionada:false},
  {cnpj:"E5",nome:"Empresa 5",sancionada:false},
  {cnpj:"E6",nome:"Empresa 6",sancionada:false},
  {cnpj:"E7",nome:"Empresa 7",sancionada:false},
  {cnpj:"E8",nome:"Empresa 8",sancionada:false},
  {cnpj:"E9",nome:"Empresa 9",sancionada:false},
  {cnpj:"E10",nome:"Empresa 10",sancionada:false},
  {cnpj:"E11",nome:"Empresa 11",sancionada:false},
  {cnpj:"E12",nome:"Empresa 12",sancionada:false},
  {cnpj:"E13",nome:"Empresa 13",sancionada:false},
  {cnpj:"E14",nome:"Empresa 14",sancionada:false},
  {cnpj:"E15",nome:"Empresa 15",sancionada:false},
  {cnpj:"E16",nome:"Empresa 16",sancionada:false},
  {cnpj:"E17",nome:"Empresa 17",sancionada:false},
  {cnpj:"E18",nome:"Empresa 18",sancionada:false},
  {cnpj:"E19",nome:"Empresa 19",sancionada:false},
  {cnpj:"E20",nome:"Empresa 20",sancionada:false},
  {cnpj:"E21",nome:"Empresa 21",sancionada:false},
  {cnpj:"E22",nome:"Empresa 22",sancionada:false},
  {cnpj:"E23",nome:"Empresa 23",sancionada:false},
  {cnpj:"E24",nome:"Empresa 24",sancionada:false},
  {cnpj:"E25",nome:"Empresa 25",sancionada:false},
  {cnpj:"E26",nome:"Empresa 26",sancionada:false},
  {cnpj:"E27",nome:"Empresa 27",sancionada:false},
  {cnpj:"E28",nome:"Empresa 28",sancionada:false},
  {cnpj:"E29",nome:"Empresa 29",sancionada:false}
] AS e
CREATE (:Empresa {cnpj: e.cnpj, nome: e.nome, sancionada: e.sancionada});
```

## 3. Relações societárias

Use a lista completa de relações `SOCIA_DE` fornecida no enunciado. Exemplo:

```cypher
UNWIND [{de:"E23",para:"E10",pct:8},{de:"E2",para:"E19",pct:77},{de:"E14",para:"E2",pct:31}] AS s
MATCH (a:Empresa {cnpj:s.de}), (b:Empresa {cnpj:s.para})
CREATE (a)-[:SOCIA_DE {pct:s.pct}]->(b);
```

Para a carga completa, substitua o array pelo array integral fornecido no enunciado.

## 4. Contêineres e propriedade

```cypher
UNWIND [{id:"K0",classe:"Perigoso",dono:"E2"},{id:"K1",classe:"Granel",dono:"E2"},{id:"K89",classe:"Granel",dono:"E9"}] AS c
MATCH (e:Empresa {cnpj:c.dono})
CREATE (:Container {id:c.id, classe:c.classe})-[:PERTENCE_A]->(e);
```

Repita o método completo com os 90 contêineres `K0` a `K89` fornecidos.

## 5. Passagens

```cypher
UNWIND [{id:"K0",term:"T4",data:"2026-10-29"},{id:"K89",term:"T5",data:"2026-10-03"}] AS p
MATCH (k:Container {id:p.id}), (t:Terminal {nome:p.term})
CREATE (k)-[:PASSOU_POR {data:p.data}]->(t);
```

Use a lista completa de passagens fornecida no enunciado.

## 6. Pilha

```cypher
UNWIND [{topo:"K1",base:"K0"},{topo:"K2",base:"K1"},{topo:"K59",base:"K58"}] AS p
MATCH (a:Container {id:p.topo}), (b:Container {id:p.base})
CREATE (a)-[:EMPILHADO_SOBRE]->(b);
```

Use a lista completa fornecida no enunciado para criar todas as pilhas.

## 7. Verificação

```cypher
MATCH (n) RETURN labels(n) AS tipo, count(n) AS quantidade;
MATCH ()-[r]->() RETURN type(r) AS relacionamento, count(r) AS quantidade;
```