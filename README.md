
---

## Otimização do Spark

<p align="justify">Aqui estão algumas dicas para otimizar o Spark e economizar recursos:</p>

<p align="justify">O "botão" que 90% dos usuários Spark usa errado: spark.sql.shuffle.partitions.</p>

<p align="justify">(Salve ♻️ porque o default "200" está custando caro para o seu cluster).</p>

<p align="justify">Quando o Spark faz um shuffle (um join, groupBy ou sort), ele precisa decidir em quantos "pedaços" (partições) ele vai quebrar o resultado. Esse número é controlado pelo spark.sql.shuffle.partitions. O valor padrão? 200. E aqui mora o problema. "200" é um chute. É um número genérico que não faz ideia se você está processando 10MB ou 10TB.</p>

<p align="justify">Cenário A: "Small Data" (Ex: 50MB) Você faz um groupBy. O Spark, obediente, cria 200 partições. Resultado: 195 partições vazias. Você gastou overhead de CPU e agendador para orquestrar 200 tarefas quando 5 seriam suficientes.</p>

### 1. Otimize o Shuffle Partitions

<p align="justify">O parâmetro <code>spark.sql.shuffle.partitions</code> controla o número de partições usadas durante as trocas de dados (joins e agregações). A recomendação é manter cada partição entre 100 MB e 200 MB.</p>

| Tamanho Total dos Dados | `spark.sql.shuffle.partitions` | Justificativa |
| --- | --- | --- |
| **Pequeno** (< 1 GB) | 10 a 50 | Evita o overhead de muitas tarefas pequenas. |
| **Médio** (1 GB a 10 GB) | 50 a 200 | Mantém o paralelismo alinhado com executores médios. |
| **Grande** (10 GB a 100 GB) | 200 a 1000 | Evita sobrecarga no Garbage Collection (GC). |
| **Muito Grande** (> 100 GB) | 1000+ | Necessário para distribuir carga em clusters massivos. |

<p align="justify">Relação com o maxPartitionBytes: Enquanto o spark.sql.shuffle.partitions controla os dados durante as trocas (joins/agregados), o seu guia menciona o spark.sql.files.maxPartitionBytes. Este último controla a leitura inicial do disco.</p>

<p align="justify">Se você ler 10 GB de dados com maxPartitionBytes em 128 MB, terá inicialmente cerca de 80 partições. Se você não ajustar o shuffle, o Spark usará o padrão de 200, o que pode ser excessivo para esse volume, gerando tarefas vazias.</p>

<p align="justify">Dica Extra: Sempre monitore a aba SQL no Spark UI. Se você notar que o "Shuffle Read Size" por tarefa está muito alto (ex: > 500 MB), aumente o número de partições para evitar o uso excessivo de memória do executor (spark.executor.memory).</p>

> **Dica:** No Spark 3.0+, habilite o AQE (`spark.sql.adaptive.enabled`) para que o Spark ajuste esse número automaticamente.

### 2. Ajuste o tamanho dos blocos (block size)

<p align="justify">O parâmetro <code>spark.sql.files.maxPartitionBytes</code> define o tamanho máximo dos blocos lidos do disco, ajudando a reduzir o número de tarefas iniciais.</p>

| Tamanho do arquivo | spark.sql.files.maxPartitionBytes | spark.sql.files.openCostInBytes |
| --- | --- | --- |
| **Pequeno** (< 100 MB) | 32 MB a 64 MB | 1 MB a 4 MB |
| **Médio** (100 MB a 1 GB) | 64 MB a 128 MB | 4 MB a 16 MB |
| **Grande** (1 GB a 10 GB) | 128 MB a 256 MB | 16 MB a 64 MB |
| **Muito grande** (> 10 GB) | 256 MB a 512 MB | 64 MB a 128 MB |

* <p align="justify"><b>Regra geral:</b> O tamanho dos blocos deve ser entre 1/10 e 1/5 do tamanho do arquivo.</p>
* <p align="justify"><b>Custo de abertura:</b> Deve ser entre 1/100 e 1/50 do tamanho do bloco.</p>

### 3. Use o cache de dados

* <p align="justify">Use <code>spark.cache</code> para armazenar dados acessados frequentemente em memória.</p>
* <p align="justify">Utilize <code>cache()</code> ou <code>persist()</code> para evitar reprocessamento e reduzir leitura de disco.</p>

### 4. Otimize as junções (joins)

<p align="justify">O <b>broadcast</b> envia tabelas pequenas para todos os nós, permitindo junções locais sem shuffle.</p>

| Categoria da Tabela | Tamanho | Ajuste do `spark.sql.autoBroadcastJoinThreshold` |
| --- | --- | --- |
| **Pequena** | < 10 MB | Transmitida automaticamente (padrão). |
| **Média** | 10 MB a 100 MB | Aumente para 50 MB ou 100 MB. |
| **Grande** | > 100 MB | Geralmente não é transmitida automaticamente. |

### 5. Monitore e ajuste do paralelismo e garbage collection (GC)

<p align="justify">Ajuste o <code>spark.default.parallelism</code> e a memória do executor para evitar falhas e lentidão no processamento.</p>

| Tamanho dos dados | spark.default.parallelism | spark.executor.memory |
| --- | --- | --- |
| **Pequeno** (< 100 MB) | 2-4 | 1-2 GB |
| **Médio** (100 MB a 1 GB) | 4-8 | 2-4 GB |
| **Grande** (1 GB a 10 GB) | 8-16 | 4-8 GB |
| **Muito grande** (> 10 GB) | 16-32 | 8-16 GB |

<p align="justify"><i>Tamanho dos dados pequeno (< 100 MB)</i></p>

```python
spark.conf.set("spark.default.parallelism", 2)
spark.conf.set("spark.sql.files.openCostInBytes", 1 * 1024 * 1024) # 1 MB

```

<p align="justify"><i>Tamanho dos dados médio (100 MB a 1 GB)</i></p>

```python
spark.conf.set("spark.default.parallelism", 4)
spark.conf.set("spark.sql.files.openCostInBytes", 4 * 1024 * 1024) # 4 MB

```

<p align="justify"><i>Tamanho dos dados grande (1 GB a 10 GB)</i></p>

```python
spark.conf.set("spark.default.parallelism", 8)
spark.conf.set("spark.sql.files.openCostInBytes", 16 * 1024 * 1024) # 16 MB

```

<p align="justify"><i>Tamanho dos dados muito grande (> 10 GB)</i></p>

```python
spark.conf.set("spark.default.parallelism", 16)
spark.conf.set("spark.sql.files.openCostInBytes", 64 * 1024 * 1024) # 64 MB

```

<p align="justify">Lembre-se de que esses são apenas exemplos e que o ajuste desses parâmetros depende do seu ambiente de execução e do tamanho dos dados.</p>

<p align="justify"><b>Regra geral:</b></p>

* <p align="justify"><code>spark.default.parallelism</code>: 2-4 vezes o número de núcleos de CPU disponíveis.</p>
* <p align="justify"><code>spark.sql.files.openCostInBytes</code>: 1-10% do tamanho do arquivo.</p>

<p align="justify">Há várias configurações de memória RAM do executor que você pode ajustar no Spark:</p>


<p align="justify">1. spark.executor.memory: define a memória RAM total disponível para cada executor</p>



<p align="justify">2. spark.executor.memoryOverhead: define a memória adicional para o executor (por exemplo, para o sistema operacional e outros processos)</p>



<p align="justify">3. spark.memory.fraction: define a fração de memória RAM usada para armazenamento de dados (padrão: 0,6)</p>



<p align="justify">4. spark.memory.storageFraction: define a fração de memória RAM usada para armazenamento de dados em cache (padrão: 0,5)</p>



<p align="justify">5. spark.executor.pyspark.memory: define a memória RAM disponível para o Python worker (somente para PySpark)</p>



<p align="justify">6. spark.executor.pyspark.memoryOverhead: define a memória adicional para o Python worker (somente para PySpark)</p>

```python
spark.conf.set("spark.executor.memory", "4g") # 4 GB de memória RAM
spark.conf.set("spark.executor.memoryOverhead", "1g") # 1 GB de memória adicional
spark.conf.set("spark.memory.fraction", 0.6) # 60% da memória RAM para armazenamento de dados
spark.conf.set("spark.memory.storageFraction", 0.5) # 50% da memória RAM para armazenamento de dados em cache

```

<p align="justify">Lembre-se de que o ajuste dessas configurações depende do seu ambiente de execução e do tamanho dos dados.</p>

#### Configurações de RAM do Executor:

* <p align="justify"><code>spark.executor.memoryOverhead</code>: Memória para o SO e processos externos.</p>
* <p align="justify"><code>spark.memory.fraction</code>: Fração da RAM para armazenamento (padrão 0.6).</p>
* <p align="justify"><code>spark.memory.storageFraction</code>: Fração da RAM para cache (padrão 0.5).</p>

<p align="justify">Monitoramento do GC:</p>



<p align="justify">1. Acesse o Spark UI em <code>http://<driver-node>:4040</code></p>



<p align="justify">2. Clique em "Executors"</p>



<p align="justify">3. Verifique a coluna "GC Time" para cada executor</p>



<p align="justify">4. Se o tempo de GC for alto (> 10%), ajuste a memória do executor</p>

### 6. Use o Spark SQL

<p align="justify">O Spark SQL (DataFrames e Datasets) é mais eficiente que a RDD API devido ao otimizador Catalyst.</p>

<p align="justify">Lembre-se de monitorar o desempenho do seu aplicativo Spark e ajustar as configurações conforme necessário! 😊</p>

---

## Python

### 1. Estruturas de Dados: Tuplas vs. Listas

<p align="justify">As tuplas são imutáveis e possuem um tamanho fixo, o que torna sua alocação de memória muito mais rápida que a das listas, que precisam de espaço extra para redimensionamento dinâmico.</p>

```python
# Lento: Lista (mutável)
minha_lista = [1, 2, 3, 4, 5] 

# Rápido: Tupla (imutável)
minha_tupla = (1, 2, 3, 4, 5) 

```

<p align="justify">Resultado: Em testes, a criação de uma tupla pode ser cerca de 6 vezes mais rápida que a de uma lista.</p>

### 3. Buscas com Sets e Dicionários

<p align="justify">Dicionários e conjuntos (sets) utilizam tabelas de hash, permitindo que o Python encontre um item diretamente sem percorrer toda a estrutura. Isso resulta em uma busca de tempo constante, denotada como .</p>

```python
# Lento em listas grandes: O Python olha item por item
if 999999 in lista_de_um_milhao: 
    pass

# Instantâneo em Sets/Dicts: O Python vai direto ao endereço
if 999999 in set_de_um_milhao: 
    pass

```

<p align="justify">Performance: Enquanto a busca em uma lista grande pode levar milissegundos, em um set ou dicionário o tempo é virtualmente zero.</p>

### 4. Variáveis Locais vs. Globais

<p align="justify">O Python utiliza a regra LEGB para buscar variáveis, começando sempre pelo escopo local. Como o escopo local é menor, a busca é muito mais ágil do que no escopo global.</p>

```python
# Menos eficiente
contador_global = 0
def teste_global():
    global contador_global
    for i in range(1000000):
        contador_global += 1

# Mais eficiente
def teste_local():
    contador_local = 0
    for i in range(1000000):
        contador_local += 1

```

<p align="justify">Nota: O uso de variáveis locais pode reduzir o tempo de execução em cerca de 35% em loops intensivos.</p>

### 5. Encapsulamento em Classes

<p align="justify">Manter variáveis restritas a funções e classes ajuda o interpretador a gerenciar menos nomes simultaneamente, melhorando a performance e a gestão de memória.</p>

```python
class RetanguloEncapsulado:
    def __init__(self, largura, altura):
        self._largura = largura # Atributo protegido
        self._altura = altura

    def area(self):
        return self._largura * self._altura

```

<p align="justify">Benefício: Além da performance, evita conflitos de nomes e garante que os dados não sejam modificados acidentalmente por código externo.</p>

### 6. List Comprehensions e Geradores

<p align="justify">As compreensões de lista são otimizadas internamente, sendo mais rápidas que o uso do método .append() dentro de um loop for tradicional.</p>

```python
# Rápido (List Comprehension)
quadrados = [x**2 for x in range(10)]

# Economiza memória (Expressão Geradora)
soma_quadrados = sum(x**2 for x in range(1000000))

```

<p align="justify">Comparação: Expressões geradoras são mais rápidas e consomem muito menos memória ao lidar com grandes volumes de dados.</p>

### 7. Funções Built-in e NumPy

<p align="justify">Sempre prefira as funções nativas do Python (escritas em C) ou bibliotecas especializadas como o NumPy para processamento numérico.</p>

```python
# Lento: Algoritmo manual (Bubble Sort)
def bubble_sort(arr): ... 

# Instantâneo: Função nativa
sorted(meu_array)

# Soma com NumPy
import numpy as np
total = np.sum(array_numpy) # Muito mais rápido que sum() do Python para arrays gigantes

```

<p align="justify">Diferença: Em arrays grandes, o NumPy pode realizar operações em 0.008 segundos, enquanto uma função Python customizada levaria mais de 1 segundo.</p>

---


