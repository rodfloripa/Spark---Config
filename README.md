

## Otimização do Spark

Aqui estão algumas dicas para otimizar o Spark e economizar recursos:

O "botão" que 90% dos usuários Spark usa errado: spark.sql.shuffle.partitions.

(Salve ♻️ porque o default "200" está custando caro para o seu cluster).

Quando o Spark faz um shuffle (um join, groupBy ou sort), ele precisa decidir em quantos "pedaços" (partições) ele vai quebrar o resultado.

Esse número é controlado pelo spark.sql.shuffle.partitions.

O valor padrão? 200.

E aqui mora o problema.

"200" é um chute. É um número genérico que não faz ideia se você está processando 10MB ou 10TB.
<p style="text-align: justify;">
    Cenário A: "Small Data" (Ex: 50MB)
    Você faz um groupBy. O Spark, obediente, cria 200 partições.
    Resultado: 195 partições vazias.
</p>

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

<p style="text-align: justify;">
    Se você ler 10 GB de dados com maxPartitionBytes em 128 MB, terá inicialmente cerca de 80 partições.
    Se você não ajustar o shuffle, o Spark usará o padrão de 200, o que pode ser excessivo para esse volume, gerando tarefas vazias.
</p>

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



### 5. Monitore e ajuste do paralelismo e garbage collection (GC)
Ajuste o `spark.default.parallelism` e a memória do executor para evitar falhas e lentidão no processamento.

| Tamanho dos dados | spark.default.parallelism | spark.executor.memory |
| :--- | :--- | :--- |
| **Pequeno** (< 100 MB) | 2-4 | 1-2 GB |
| **Médio** (100 MB a 1 GB) | 4-8 | 2-4 GB |
| **Grande** (1 GB a 10 GB) | 8-16 | 4-8 GB |
| **Muito grande** (> 10 GB) | 16-32 | 8-16 GB |


*Tamanho dos dados pequeno (< 100 MB)*

- `spark.default.parallelism`: 2-4
- `spark.sql.files.openCostInBytes`: 1-4 MB

Exemplo:
spark.conf.set("spark.default.parallelism", 2)

spark.conf.set("spark.sql.files.openCostInBytes", 1 * 1024 * 1024) # 1 MB


*Tamanho dos dados médio (100 MB a 1 GB)*

- `spark.default.parallelism`: 4-8
- `spark.sql.files.openCostInBytes`: 4-16 MB

Exemplo:
spark.conf.set("spark.default.parallelism", 4)

spark.conf.set("spark.sql.files.openCostInBytes", 4 * 1024 * 1024) # 4 MB


*Tamanho dos dados grande (1 GB a 10 GB)*

- `spark.default.parallelism`: 8-16
- `spark.sql.files.openCostInBytes`: 16-64 MB

Exemplo:
spark.conf.set("spark.default.parallelism", 8)

spark.conf.set("spark.sql.files.openCostInBytes", 16 * 1024 * 1024) # 16 MB


*Tamanho dos dados muito grande (> 10 GB)*

- `spark.default.parallelism`: 16-32
- `spark.sql.files.openCostInBytes`: 64-128 MB

Exemplo:
spark.conf.set("spark.default.parallelism", 16)

spark.conf.set("spark.sql.files.openCostInBytes", 64 * 1024 * 1024) # 64 MB

Lembre-se de que esses são apenas exemplos e que o ajuste desses parâmetros depende do seu ambiente de execução e do tamanho dos dados.

*Regra geral*

- `spark.default.parallelism`: 2-4 vezes o número de núcleos de CPU disponíveis.
- `spark.sql.files.openCostInBytes`: 1-10% do tamanho do arquivo.



Há várias configurações de memória RAM do executor que você pode ajustar no Spark:

1. spark.executor.memory: define a memória RAM total disponível para cada executor
2. spark.executor.memoryOverhead: define a memória adicional para o executor (por exemplo, para o sistema operacional e outros processos)
3. spark.memory.fraction: define a fração de memória RAM usada para armazenamento de dados (padrão: 0,6)
4. spark.memory.storageFraction: define a fração de memória RAM usada para armazenamento de dados em cache (padrão: 0,5)
5. spark.executor.pyspark.memory: define a memória RAM disponível para o Python worker (somente para PySpark)
6. spark.executor.pyspark.memoryOverhead: define a memória adicional para o Python worker (somente para PySpark)

Exemplo:
*  spark.conf.set("spark.executor.memory", "4g") - 4 GB de memória RAM
*  spark.conf.set("spark.executor.memoryOverhead", "1g") - 1 GB de memória adicional
*  spark.conf.set("spark.memory.fraction", 0.6) - 60% da memória RAM para armazenamento de dados
*  spark.conf.set("spark.memory.storageFraction", 0.5) - 50% da memória RAM para armazenamento de dados em cache

Lembre-se de que o ajuste dessas configurações depende do seu ambiente de execução e do tamanho dos dados.



#### Configurações de RAM do Executor:
* `spark.executor.memoryOverhead`: Memória para o SO e processos externos.
* `spark.memory.fraction`: Fração da RAM para armazenamento (padrão 0.6).
* `spark.memory.storageFraction`: Fração da RAM para cache (padrão 0.5).

Monitoramento do GC

    1. Acesse o Spark UI em `http://<driver-node>:4040`
    2. Clique em "Executors"
    3. Verifique a coluna "GC Time" para cada executor
    4. Se o tempo de GC for alto (> 10%), ajuste a memória do executor

### 6. Use o Spark SQL
O Spark SQL (DataFrames e Datasets) é mais eficiente que a RDD API devido ao otimizador Catalyst.


Lembre-se de monitorar o desempenho do seu aplicativo Spark e ajustar as configurações conforme necessário! 😊

## Python

1. Estruturas de Dados: Tuplas vs. Listas
As tuplas são imutáveis e possuem um tamanho fixo, o que torna sua alocação de memória muito mais rápida que a das listas, que precisam de espaço extra para redimensionamento dinâmico.
   
Exemplo:
   
    Python
   
    Lento: Lista (mutável)
   
    minha_lista = [1, 2, 3, 4, 5] 
    
    Rápido: Tupla (imutável)
   
    minha_tupla = (1, 2, 3, 4, 5) 


    Resultado: Em testes, a criação de uma tupla pode ser cerca de 6 vezes mais rápida que a de uma lista2.


3. Buscas com Sets e Dicionários
Dicionários e conjuntos (sets) utilizam tabelas de hash, permitindo que o Python encontre um item diretamente sem percorrer toda a estrutura3333. Isso resulta em uma busca de tempo constante, denotada como $O(1)$4.

Exemplo:

    
    Lento em listas grandes: O Python olha item por item
    if 999999 in lista_de_um_milhao: 
        pass
    
    Instantâneo em Sets/Dicts: O Python vai direto ao endereço
    if 999999 in set_de_um_milhao: 
        pass


    Performance: Enquanto a busca em uma lista grande pode levar milissegundos, em um set ou dicionário o tempo é virtualmente zero5.


4. Variáveis Locais vs. Globais
O Python utiliza a regra LEGB para buscar variáveis, começando sempre pelo escopo local6666. Como o escopo local é menor, a busca é muito mais ágil do que no escopo global.

    Exemplo:
    Python
    Menos eficiente
    contador_global = 0
    def teste_global():
        global contador_global
        for i in range(1000000):
            contador_global += 1
    
    Mais eficiente
    def teste_local():
        contador_local = 0
        for i in range(1000000):
            contador_local += 1


    Nota: O uso de variáveis locais pode reduzir o tempo de execução em cerca de 35% em loops intensivos8.


5. Encapsulamento em Classes
Manter variáveis restritas a funções e classes ajuda o interpretador a gerenciar menos nomes simultaneamente, melhorando a performance e a gestão de memória.

    Exemplo:
    Python
    class RetanguloEncapsulado:
        def __init__(self, largura, altura):
            self._largura = largura # Atributo protegido
            self._altura = altura
    
        def area(self):
            return self._largura * self._altura


    Benefício: Além da performance, evita conflitos de nomes e garante que os dados não sejam modificados acidentalmente por código externo.


6. List Comprehensions e Geradores
As compreensões de lista são otimizadas internamente, sendo mais rápidas que o uso do método .append() dentro de um loop for tradicional.

    Exemplo de List Comprehension:
    Python
    Rápido
    quadrados = [x**2 for x in range(10)]


    Exemplo de Expressão Geradora:
    Python
    Economiza memória (não cria a lista inteira de uma vez)
    soma_quadrados = sum(x**2 for x in range(1000000))


    Comparação: Expressões geradoras são mais rápidas e consomem muito menos memória ao lidar com grandes volumes de dados12.


7. Funções Built-in e NumPy
Sempre prefira as funções nativas do Python (escritas em C) ou bibliotecas especializadas como o NumPy para processamento numérico.

    Exemplo (Ordenação):
    Python
     Lento: Algoritmo manual (Bubble Sort)
    def bubble_sort(arr): ... 
    
    Instantâneo: Função nativa
    sorted(meu_array)


    Exemplo (Soma com NumPy):
    Python
    import numpy as np
    Muito mais rápido que sum() do Python para arrays gigantes
    total = np.sum(array_numpy) 


    Diferença: Em arrays grandes, o NumPy pode realizar operações em 0.008 segundos, enquanto uma função Python customizada levaria mais de 1 segundo.




