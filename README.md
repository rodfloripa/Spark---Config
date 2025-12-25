

## Otimização do Spark

Aqui estão algumas dicas para otimizar o Spark e economizar recursos:

O "botão" que 90% dos usuários Spark usa errado: spark.sql.shuffle.partitions.

(Salve ♻️ porque o default "200" está custando caro para o seu cluster).

Quando o Spark faz um shuffle (um join, groupBy ou sort), ele precisa decidir em quantos "pedaços" (partições) ele vai quebrar o resultado.

Esse número é controlado pelo spark.sql.shuffle.partitions.

O valor padrão? 200.

E aqui mora o problema.

"200" é um chute. É um número genérico que não faz ideia se você está processando 10MB ou 10TB.

    Cenário A: "Small Data" (Ex: 50MB)

    Você faz um groupBy. O Spark, obediente, cria 200 partições.

    Resultado: 195 partições vazias.

Você gastou overhead de CPU e agendador para orquestrar 200 tarefas quando 5 seriam suficientes

### 1. Otimize o Shuffle Partitions
O parâmetro `spark.sql.shuffle.partitions` controla o número de partições usadas durante as trocas de dados (joins e agregações). A recomendação é manter cada partição entre 100 MB e 200 MB.

| Tamanho Total dos Dados | `spark.sql.shuffle.partitions` | Justificativa |
| :--- | :--- | :--- |
| **Pequeno** (< 1 GB) | 10 a 50 | Evita o overhead de muitas tarefas pequenas. |
| **Médio** (1 GB a 10 GB) | 50 a 200 | Mantém o paralelismo alinhado com executores médios. |
| **Grande** (10 GB a 100 GB) | 200 a 1000 | Evita sobrecarga no Garbage Collection (GC). |
| **Muito Grande** (> 100 GB) | 1000+ | Necessário para distribuir carga em clusters massivos. |

Relação com o maxPartitionBytes

Enquanto o spark.sql.shuffle.partitions controla os dados durante as trocas (joins/agregados), o seu guia menciona o spark.sql.files.maxPartitionBytes. Este último controla a leitura inicial do disco.

    Se você ler 10 GB de dados com maxPartitionBytes em 128 MB, terá inicialmente cerca de 80 partições.

    Se você não ajustar o shuffle, o Spark usará o padrão de 200, o que pode ser excessivo para esse volume, gerando tarefas vazias.

Dica Extra: Sempre monitore a aba SQL no Spark UI. Se você notar que o "Shuffle Read Size" por tarefa está muito alto (ex: > 500 MB), aumente o número de partições para evitar o uso excessivo de memória do executor (spark.executor.memory).

> **Dica:** No Spark 3.0+, habilite o AQE (`spark.sql.adaptive.enabled`) para que o Spark ajuste esse número automaticamente.

### 2. Ajuste o tamanho dos blocos (block size)
O parâmetro `spark.sql.files.maxPartitionBytes` define o tamanho máximo dos blocos lidos do disco, ajudando a reduzir o número de tarefas iniciais.

| Tamanho do arquivo | spark.sql.files.maxPartitionBytes | spark.sql.files.openCostInBytes |
| :--- | :--- | :--- |
| **Pequeno** (< 100 MB) | 32 MB a 64 MB | 1 MB a 4 MB |
| **Médio** (100 MB a 1 GB) | 64 MB a 128 MB | 4 MB a 16 MB |
| **Grande** (1 GB a 10 GB) | 128 MB a 256 MB | 16 MB a 64 MB |
| **Muito grande** (> 10 GB) | 256 MB a 512 MB | 64 MB a 128 MB |

* **Regra geral:** O tamanho dos blocos deve ser entre 1/10 e 1/5 do tamanho do arquivo.
* **Custo de abertura:** Deve ser entre 1/100 e 1/50 do tamanho do bloco.


### 3. Use o cache de dados
* Use `spark.cache` para armazenar dados acessados frequentemente em memória.
* Utilize `cache()` ou `persist()` para evitar reprocessamento e reduzir leitura de disco.

### 4. Otimize as junções (joins)
O **broadcast** envia tabelas pequenas para todos os nós, permitindo junções locais sem shuffle.

| Categoria da Tabela | Tamanho | Ajuste do `spark.sql.autoBroadcastJoinThreshold` |
| :--- | :--- | :--- |
| **Pequena** | < 10 MB | Transmitida automaticamente (padrão). |
| **Média** | 10 MB a 100 MB | Aumente para 50 MB ou 100 MB. |
| **Grande** | > 100 MB | Geralmente não é transmitida automaticamente. |

### 5. Ajuste o paralelismo e memória
Ajuste o `spark.default.parallelism` e a memória do executor para evitar falhas e lentidão no processamento.

| Tamanho dos dados | spark.default.parallelism | spark.executor.memory |
| :--- | :--- | :--- |
| **Pequeno** (< 100 MB) | 2-4 | 1-2 GB |
| **Médio** (100 MB a 1 GB) | 4-8 | 2-4 GB |
| **Grande** (1 GB a 10 GB) | 8-16 | 4-8 GB |
| **Muito grande** (> 10 GB) | 16-32 | 8-16 GB |

#### Configurações de RAM do Executor:
* `spark.executor.memoryOverhead`: Memória para o SO e processos externos.
* `spark.memory.fraction`: Fração da RAM para armazenamento (padrão 0.6).
* `spark.memory.storageFraction`: Fração da RAM para cache (padrão 0.5).

### 6. Use o Spark SQL
O Spark SQL (DataFrames e Datasets) é mais eficiente que a RDD API devido ao otimizador Catalyst.

---
Lembre-se de monitorar o desempenho do seu aplicativo Spark e ajustar as configurações conforme necessário! 😊
