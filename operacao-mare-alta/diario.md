# Diário

## Hipóteses testadas

### Parte A — MongoDB


A1: Sem índice, o plano deve utilizar `COLLSCAN` seguido de `SORT`; o índice no mesmo sentido do filtro pode eliminar parte do `COLLSCAN`, mas não necessariamente o `SORT`; o índice ESR deve combinar igualdade, intervalo e ordenação sem estágio separado de `SORT`.

A2: Sem o índice geoespacial, a consulta `$near` não será executada corretamente; após criar o índice `2dsphere`, o estágio deve utilizar `GEO_NEAR_2DSPHERE` e respeitar o raio de 300 metros.

A3: O campo `registrado_em` provavelmente está armazenado como texto, incompatível com o monitoramento TTL.

A4: A consulta pode ser coberta pelo índice composto `{ tipo: 1, terminal: 1, container: 1 }`, desde que a projeção exclua `_id`.

### Parte B — MongoDB


B1: Um índice em `{ classe: 1, ts: 1 }` deve apoiar o estágio `$match` inicial e reduzir a quantidade de documentos enviados ao `$group`.

B2: O pipeline original possui erros de campo, etapa de ordenação, filtro tardio e leitura desnecessária de documentos.

### Parte C — Neo4j


C1: O grafo deve representar empresas, contêineres e terminais como nós, sem tabelas associativas ou chaves estrangeiras.

C2: As relações `SOCIA_DE` devem ser percorridas até três níveis em ambos os sentidos para encontrar empresas conectadas à sancionada, justificando a direção segundo o sentido real do grafo.

C3: Os sintomas são causados por nós duplicados, recorrência ilimitada, direção incorreta da pilha e contagem que inclui duplicatas.

## Erros encontrados


A1: No índice { terminal: 1, classe: 1, ts: 1, peso_kg: -1 }, o campo de intervalo ts aparece antes do campo de ordenação peso_kg, portanto o MongoDB ainda precisou usar SORT.

A3: O campo registrado_em foi identificado como string. Além disso, não havia índice TTL configurado.

A4: Ao remover _id: 0, o MongoDB passou a usar FETCH e deixou de atender à consulta exclusivamente pelas entradas do índice.

B2: _id ficou com o formato incorreto no agrupamento, peso_kg não foi referenciado com $, o $limit precedeu o $sort, o filtro ocorreu tarde demais e o $project inicial descartou o campo necessário.

C3-i: A criação de uma empresa sem procurar o nó existente gera nós duplicados.

C3-ii: SOCIA_DE* sem limite pode percorrer ciclos ilimitados e consumir memória.

C3-iii: A direção usada em EMPILHADO_SOBRE deve ser considerada no sentido “topo aponta para base”.

C3-iv: count(k) conta registros retornados e pode incluir repetição de contêineres se houver mais de uma passagem pelo mesmo terminal.

## Correções


A1: Remover o índice anterior e criar o índice ESR { terminal: 1, classe: 1, peso_kg: -1, ts: 1 }.

A3: Converter novos registros para Date e criar o índice TTL { registrado_em: 1 } com expireAfterSeconds: 2592000.

A4: Criar { tipo: 1, terminal: 1, container: 1 } e manter _id: 0 na projeção da consulta coberta.

B1: Usar $match na primeira etapa e agrupar os registros por $terminal.

B2: Filtrar primeiro, agrupar por terminal, somar peso_kg, ordenar pelo peso total e limitar a três resultados.

C1/C2: Usar nós com propriedades reais e relacionamentos direcionados, com DISTINCT para eliminar duplicatas.

 C3: Corrigir todas as consultas para obter nós existentes, limitar caminhos, respeitar direções e usar count(DISTINCT k).

## Evidências e resultados

Os comandos, planos de execução e explicações técnicos estão em:

mongo/parte-a.md

mongo/parte-b.md

 neo4j/parte-c.md

## Conclusão

Os testes confirmaram a importância de colocar campos de igualdade antes dos campos de intervalo, colocar o campo de ordenação antes do intervalo em índices compatíveis com SORT, usar tipos BSON corretos para TTL e preparar índices que contenham todos os campos necessários a uma consulta coberta. Na Parte B, o filtro inicial e a ordem correta das etapas foram fundamentais para a correção arquitetural. Na Parte C, o modelo exigiu identidade por propriedades e consultas com recorrência limitada.