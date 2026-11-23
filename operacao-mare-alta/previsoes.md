## A1 — Painel do pátio

Eu acho que, sem criar nenhum índice, o MongoDB vai usar COLLSCAN e SORT, porque vai precisar procurar os dados e depois colocar eles em ordem. Acredito que vai examinar aproximadamente 400 mil documentos e nenhuma chave de índice, porque ainda não tem índice criado. Também acho que vai precisar usar SORT em memória e no final vai retornar 20 documentos.

Depois de criar o primeiro índice, acho que vai usar IXSCAN e FETCH, mas talvez ainda precise de SORT para ordenar os resultados. Já com o índice ESR, acredito que vai usar IXSCAN e FETCH, só que sem precisar de SORT separado, porque os dados já vão estar na ordem certa.

## A2 — Incidente no pátio

Eu acho que a consulta vai dar erro no começo, porque ainda não foi criado o índice 2dsphere, que é necessário para fazer a busca pela localização.

Depois de criar esse índice, acredito que o MongoDB vai usar GEO_NEAR_2DSPHERE para encontrar os registros próximos do local do incidente. Acho que vai examinar algumas dezenas ou centenas de documentos e chaves, dependendo de quantos registros tiverem naquela região. Também acredito que não vai precisar de um SORT separado em memória e que vai retornar no máximo 10 documentos.

## A3 — TTL que não apaga

Eu acho que os alertas antigos não estão sendo apagados porque o campo registrado_em foi salvo como texto em vez de data. Pelo que entendi, o TTL precisa que esse campo esteja no formato de data do MongoDB para conseguir identificar quais registros já passaram do prazo.

Minha previsão é que, quando verificar o tipo desse campo, ele vai aparecer como string. Para resolver isso, acredito que seria necessário salvar as datas como Date, assim o MongoDB conseguiria apagar automaticamente os alertas antigos usando o índice TTL.
## A4 — Consulta coberta

Eu acho que o MongoDB vai usar IXSCAN e FETCH mesmo com o índice criado, porque ele ainda vai precisar procurar os documentos para mostrar os resultados. Acredito que vai examinar uns 16 mil documentos e 16 mil chaves, sem precisar usar SORT. Deve retornar aproximadamente 16 mil registros. Se tirar o _id: 0, acho que vai usar COLLSCAN e examinar os 400 mil documentos.

## B1 — Relatório gerencial

Acho que o MongoDB vai usar COLLSCAN porque precisa analisar todos os registros para calcular os pesos e as médias. Minha previsão é que ele examine aproximadamente 400 mil documentos e nenhuma chave de índice. Também acho que vai precisar usar SORT para organizar os resultados e no final vai retornar 3 terminais.

## B2 — Debug arquitetural

Acredito que o código original vai funcionar normalmente, porque já tem os comandos de agrupar e ordenar os resultados. Acho que vai usar COLLSCAN, examinando os 400 mil documentos e nenhuma chave de índice. Não acredito que vai precisar de SORT em memória e espero que retorne os 3 terminais com maior peso total.

## C2 — Investigação societária

Eu acho que a consulta vai precisar seguir as setas que saem da empresa sancionada para encontrar as outras empresas relacionadas a ela. Acredito que usando até 3 níveis de relacionamento já vai conseguir encontrar todas as empresas necessárias. Minha previsão é que apareçam aproximadamente 90 registros e que não tenha resultados repetidos, porque cada contêiner pertence a uma empresa.