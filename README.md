# Spark Lecture

This repository contains a minimal content to an Apache Spark lecture / tutorial.

## Running pyspark-notebook

You need to have Docker installed in your system.

To run the container, use `docker-compose up` in the root folder.

## Otimização do Spark

Aqui estão algumas dicas para otimizar o Spark e economizar recursos:

1. Ajuste o tamanho dos blocos (block size)
- `spark.sql.files.maxPartitionBytes`: ajuste o tamanho máximo dos blocos de dados lidos do disco.
- Isso pode ajudar a reduzir o número de tarefas e melhorar a eficiência.

*Ajuste do tamanho dos blocos (block size) no Spark*
Tamanho do arquivo	spark.sql.files.maxPartitionBytes	spark.sql.files.openCostInBytes
Pequeno (< 100 MB)	32 MB a 64 MB	1 MB a 4 MB
Médio (100 MB a 1 GB)	64 MB a 128 MB	4 MB a 16 MB
Grande (1 GB a 10 GB)	128 MB a 256 MB	16 MB a 64 MB
Muito grande (> 10 GB)	256 MB a 512 MB	64 MB a 128 MB

- Regra geral:
    - O tamanho dos blocos deve ser entre 1/10 e 1/5 do tamanho do arquivo.
    - O custo de abertura de arquivo deve ser entre 1/100 e 1/50 do tamanho do bloco.

2. Use o cache de dados
- `spark.cache`: armazene dados frequentemente acessados em memória para reduzir a leitura do disco.
- Use `cache()` ou `persist()` para armazenar dados em memória.

3. Otimize as junções (joins)
- `spark.sql.autoBroadcastJoinThreshold`: ajuste o limite para broadcast de tabelas pequenas em junções.
- Use `broadcast()` para forçar o broadcast de tabelas pequenas.

*O que é broadcast?*

- Quando uma tabela é transmitida, o Spark a envia para todos os nós do cluster, permitindo que a junção seja realizada localmente em cada nó.

Como ajustar o `spark.sql.autoBroadcastJoinThreshold`?

- O valor padrão é 10 MB.
- Ajuste de acordo com o tamanho das tabelas:
    - Pequena (< 10 MB): provavelmente será transmitida automaticamente.
    - Média (10 MB a 100 MB): aumente para 50 MB ou 100 MB.
    - Grande (> 100 MB): provavelmente não será transmitida automaticamente.

4. Ajuste o paralelismo
- `spark.default.parallelism`: ajuste o número de tarefas paralelas para operações como `map()` e `reduce()`.
- `spark.sql.files.openCostInBytes`: ajuste o custo de abertura de arquivos para otimizar o paralelismo.

*Ajuste do paralelismo no Spark*
Tamanho dos dados	spark.default.parallelism	spark.sql.files.openCostInBytes
Pequeno (< 100 MB)	2-4	1-4 MB
Médio (100 MB a 1 GB)	4-8	4-16 MB
Grande (1 GB a 10 GB)	8-16	16-64 MB
Muito grande (> 10 GB)	16-32	64-128 MB

5. Monitore e ajuste o garbage collection (GC)
- `spark.executor.memory`: ajuste a memória do executor para evitar GC excessivo.
- Monitore o GC com ferramentas como o Spark UI ou o Ganglia.

*Ajuste da memória do executor*
Tamanho dos dados	spark.executor.memory
Pequeno (< 100 MB)	1-2 GB
Médio (100 MB a 1 GB)	2-4 GB
Grande (1 GB a 10 GB)	4-8 GB
Muito grande (> 10 GB)	8-16 GB

*Configurações de memória RAM do executor*

- `spark.executor.memory`: define a memória RAM total disponível para cada executor
- `spark.executor.memoryOverhead`: define a memória adicional para o executor (por exemplo, para o sistema operacional e outros processos)
- `spark.memory.fraction`: define a fração de memória RAM usada para armazenamento de dados (padrão: 0,6)
- `spark.memory.storageFraction`: define a fração de memória RAM usada para armazenamento de dados em cache (padrão: 0,5)

Exemplo:
spark.conf.set("spark.executor.memory", "4g")  
4 GB de memória RAM
spark.conf.set("spark.executor.memoryOverhead", "1g")  
1 GB de memória adicional
spark.conf.set("spark.memory.fraction", 0.6)  
60% da memória RAM para armazenamento de dados
spark.conf.set("spark.memory.storageFraction", 0.5)  # 50% da memória RAM para armazenamento de dados em cache

6. Use o Spark SQL
- O Spark SQL é mais eficiente do que o RDD API para operações de dados estruturados.
- Use `DataFrame` e `Dataset` para aproveitar as otimizações do Spark SQL.

Lembre-se de monitorar o desempenho do seu aplicativo Spark e ajustar as configurações conforme necessário! 😊
