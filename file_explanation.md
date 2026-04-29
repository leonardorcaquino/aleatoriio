# Explicacao Detalhada da Estrutura do Projeto

Este documento consolida a explicacao da estrutura base do template
`kedro_Template`, criado para uma arquitetura de pipeline factory usando
Databricks, Kedro, Delta Lake, ADLS e Unity Catalog.

## Versoes Base

O projeto foi ajustado para considerar as seguintes versoes:

```text
Python >= 3.11
Kedro >= 1.3.0
kedro-datasets >= 3.0.0
PySpark >= 3.5.0
Delta Spark >= 3.0.0
```

Essas versoes estao declaradas no arquivo `pyproject.toml`.

## Ideia Central da Solucao

A solucao usa tabelas Delta parametrizadas como fonte de metadados para
descrever pipelines, nodes, datasets, mapeamentos de colunas, regras de
qualidade e estrategias de carga.

O fluxo conceitual e:

```text
Tabelas cfg_* no Unity Catalog
        |
        v
ConfigLoader
        |
        v
MetadataBundle
        |
        v
PipelineBuilder / NodeBuilder
        |
        v
Pipelines e nodes Kedro dinamicos
        |
        v
Execucao Spark no Databricks
        |
        v
Tabelas Delta finais + auditoria
```

Em outras palavras: as tabelas `cfg_*` dizem o que deve acontecer; o Kedro
monta a DAG dinamicamente; o Databricks executa com Spark; e o Delta Lake
armazena os dados finais e os registros de auditoria.

## Estrutura Geral

```text
kedro_Template/
  conf/
    base/
      catalog/
      parameters/
      logging/
    local/
  docs/
    architecture/
    metadata_tables/
  notebooks/
  src/
    kedro_template/
      pipeline_factory/
        load_strategies/
        quality/
      pipelines/
  tests/
    pipeline_factory/
```

## Arquivos da Raiz

### README.md

Arquivo principal de apresentacao do projeto.

Serve para explicar rapidamente:

- o objetivo do template;
- as tecnologias usadas;
- a estrutura de diretorios;
- as versoes base do projeto;
- o status atual da implementacao.

Este arquivo e normalmente o primeiro ponto de contato para qualquer pessoa
que abrir o repositorio.

### pyproject.toml

Arquivo principal de configuracao Python.

Ele define:

- nome do projeto;
- versao do projeto;
- versao minima do Python;
- dependencias principais;
- dependencias opcionais de desenvolvimento;
- configuracoes do Kedro;
- configuracoes do Ruff.

As principais dependencias declaradas sao:

```toml
kedro >= 1.3.0
kedro-datasets >= 3.0.0
pyspark >= 3.5.0
delta-spark >= 3.0.0
pydantic >= 2.0.0
pyyaml >= 6.0.0
```

O `pyproject.toml` e importante porque centraliza a definicao do ambiente
Python e ajuda a manter o projeto reproduzivel.

### .gitignore

Define arquivos e pastas que nao devem ser versionados.

Exemplos:

- caches Python;
- ambientes virtuais;
- logs;
- dados locais;
- configuracoes locais sensiveis;
- arquivos de IDE;
- configuracoes Databricks locais.

Isso evita que arquivos temporarios, credenciais ou artefatos grandes sejam
enviados para o Git.

## Pasta conf

A pasta `conf` segue o padrao do Kedro para separar configuracoes do projeto.

### conf/base

Contem configuracoes versionaveis e comuns entre ambientes.

Tudo que esta em `conf/base` deve representar uma configuracao padrao do
projeto.

### conf/local

Contem configuracoes locais, especificas de uma maquina ou ambiente.

Por padrao, o conteudo dessa pasta e ignorado pelo Git, exceto o arquivo
`.gitkeep`.

Essa pasta deve ser usada para configuracoes sensiveis ou especificas, como:

- credenciais locais;
- endpoints temporarios;
- parametros de teste;
- caminhos locais.

### conf/local/.gitkeep

Arquivo vazio usado apenas para garantir que a pasta `conf/local` exista no
repositorio.

Sem ele, uma pasta vazia nao seria versionada pelo Git.

## Catalogo de Metadados

### conf/base/catalog/metadata_catalog.yml

Declara os datasets logicos das tabelas de configuracao da factory.

As tabelas configuradas sao:

```text
cfg_pipeline
cfg_dataset
cfg_node
cfg_column_mapping
cfg_load_strategy
cfg_quality_rule
cfg_dependency
cfg_runtime_parameter
cfg_audit_log
```

Essas tabelas representam o contrato de metadados da solucao.

Exemplo conceitual:

```yaml
cfg_pipeline:
  type: spark.SparkDataset
  table_name: ${globals:metadata_catalog}.${globals:metadata_schema}.cfg_pipeline
  file_format: delta
```

Na pratica, em Databricks, a implementacao atual tambem permite ler essas
tabelas diretamente via `spark.table(...)`, sem depender exclusivamente do
catalogo Kedro.

Isso e util porque, em uma execucao Databricks, pode ser mais simples e direto
resolver as tabelas pelo Unity Catalog usando SparkSession.

## Parametros

### conf/base/parameters/globals.yml

Define parametros globais do projeto.

Campos atuais:

```yaml
metadata_catalog: main
metadata_schema: metadata
execution_environment: dev
default_timezone: America/Sao_Paulo
```

Esses parametros servem para montar os nomes completos das tabelas de
metadados, como:

```text
main.metadata.cfg_pipeline
main.metadata.cfg_dataset
main.metadata.cfg_node
```

### conf/base/parameters/pipeline_factory.yml

Define parametros especificos da factory.

Exemplos:

```yaml
pipeline_factory:
  active_only: true
  default_layer_order:
    - bronze
    - silver
    - gold
  default_write_options:
    mergeSchema: "true"
  audit:
    enabled: true
    table_name: cfg_audit_log
  quality:
    enabled: true
    fail_on_critical: true
```

Esse arquivo controla comportamentos gerais da factory, como:

- executar apenas pipelines ativos;
- ordem logica das camadas;
- opcoes padrao de escrita;
- ativacao de auditoria;
- ativacao de validacoes de qualidade.

## Pasta docs

A pasta `docs` contem documentacao tecnica da arquitetura.

### docs/architecture/solution_outline.md

Documento com o desenho conceitual da solucao.

Ele explica:

- o papel do Kedro;
- o papel do Databricks;
- o papel do Unity Catalog;
- o uso das tabelas Delta parametrizadas;
- o fluxo geral de execucao;
- os componentes principais da factory.

Esse documento e util para alinhamento tecnico e arquitetural.

### docs/architecture/file_explanation.md

Este arquivo.

Serve como uma documentacao detalhada da estrutura do projeto, explicando a
funcao de cada arquivo e modulo.

### docs/metadata_tables/table_specification.md

Documento com a especificacao inicial das tabelas de metadados.

Ele descreve campos, tipos esperados, obrigatoriedade e finalidade das tabelas:

- `cfg_pipeline`;
- `cfg_dataset`;
- `cfg_node`;
- `cfg_column_mapping`;
- `cfg_load_strategy`;
- `cfg_audit_log`.

Esse documento funciona como contrato inicial para criar as tabelas Delta no
Unity Catalog.

## Pasta notebooks

Pasta reservada para notebooks exploratorios.

Pode ser usada para:

- testar leitura das tabelas `cfg_*`;
- prototipar transformacoes Spark;
- validar merges Delta;
- testar regras de qualidade;
- gerar dados de exemplo.

No template atual, a pasta existe como estrutura, mas ainda nao possui
notebooks.

## Pacote Python Principal

### src/kedro_template/__init__.py

Marca `kedro_template` como um pacote Python.

Sem esse arquivo, o Python nao trataria a pasta como um pacote importavel em
alguns contextos.

### src/kedro_template/settings.py

Arquivo reservado para configuracoes avancadas do Kedro.

Pode ser usado futuramente para:

- hooks do Kedro;
- configuracoes customizadas de sessao;
- resolvers de configuracao;
- datasets customizados;
- plugins;
- integracoes adicionais.

No estado atual, ele e um placeholder.

### src/kedro_template/pipeline_registry.py

Arquivo padrao do Kedro para registrar pipelines.

Ele contem duas funcoes:

```python
def register_pipelines() -> dict[str, Pipeline]:
    return {"__default__": Pipeline([])}
```

Essa funcao mantem o contrato esperado pelo Kedro. Como a proposta da solucao
e montar pipelines dinamicamente, o registro estatico fica vazio por enquanto.

Tambem existe:

```python
def build_dynamic_pipelines(...)
```

Essa funcao e o ponto de entrada para uma execucao dinamica no Databricks. Ela:

1. carrega os metadados usando `ConfigLoader`;
2. valida as configuracoes;
3. cria um `PipelineBuilder`;
4. retorna um dicionario de pipelines Kedro dinamicas.

Uso conceitual:

```python
pipelines = build_dynamic_pipelines(
    spark=spark,
    active_only=True,
    write_options={"mergeSchema": "true"},
)
```

## Pasta pipeline_factory

A pasta `pipeline_factory` contem o nucleo da solucao.

Ela e responsavel por:

- carregar metadados;
- validar referencias;
- criar contratos internos;
- montar nodes;
- montar pipelines;
- resolver datasets;
- aplicar regras de qualidade;
- executar estrategias de carga;
- registrar auditoria.

### src/kedro_template/pipeline_factory/__init__.py

Arquivo de inicializacao do pacote `pipeline_factory`.

Atualmente exporta:

```python
ConfigLoader
MetadataValidationError
```

Ele foi mantido propositalmente leve para evitar que importar o pacote inteiro
exija Kedro, Spark ou Delta imediatamente.

Isso ajuda em testes unitarios e validacoes de metadados sem cluster.

## Modelos de Metadados

### src/kedro_template/pipeline_factory/models.py

Define os contratos internos da factory usando `dataclass`.

Esse arquivo e importante porque transforma linhas de tabelas `cfg_*` em
objetos Python tipados e mais seguros.

#### DatasetSpec

Representa uma linha da `cfg_dataset`.

Campos principais:

- `dataset_id`;
- `dataset_name`;
- `catalog_name`;
- `schema_name`;
- `table_name`;
- `full_table_name`;
- `format`;
- `layer`;
- `read_mode`;
- `write_mode`;
- `storage_path`;
- `partition_columns`;
- `primary_key`;
- `watermark_column`;
- `is_active`.

Esse objeto descreve uma fonte ou destino de dados.

Exemplo conceitual:

```python
DatasetSpec(
    dataset_id="ds_customer_bronze",
    catalog_name="main",
    schema_name="bronze",
    table_name="customer",
    full_table_name="main.bronze.customer",
    format="delta",
    read_mode="table",
)
```

#### PipelineSpec

Representa uma linha da `cfg_pipeline`.

Campos principais:

- `pipeline_id`;
- `pipeline_name`;
- `domain`;
- `pipeline_type`;
- `layer_from`;
- `layer_to`;
- `is_active`;
- `schedule_cron`;
- `owner_team`;
- `criticality`;
- `description`.

Esse objeto representa uma pipeline logica, como:

```text
customer_bronze_to_silver
```

#### NodeSpec

Representa uma linha da `cfg_node`.

Campos principais:

- `node_id`;
- `pipeline_id`;
- `node_name`;
- `node_type`;
- `function_name`;
- `execution_order`;
- `input_dataset_id`;
- `output_dataset_id`;
- `parameters`;
- `is_active`.

Esse objeto define um node Kedro dinamico.

Tipos suportados inicialmente:

```text
extract
transform
validate
load
```

#### ColumnMappingSpec

Representa uma linha da `cfg_column_mapping`.

Controla:

- coluna origem;
- coluna destino;
- tipo destino;
- expressao de transformacao;
- valor default;
- se e chave primaria;
- se e coluna de particao;
- se aceita nulo.

Esse contrato permite criar transformacoes simples por metadados, sem precisar
codificar uma funcao nova para cada tabela.

#### LoadStrategySpec

Representa uma linha da `cfg_load_strategy`.

Define como os dados serao gravados no destino.

Campos importantes:

- `load_type`;
- `merge_keys`;
- `watermark_column`;
- `delete_not_matched`;
- `deduplication_keys`;
- `optimize_after_write`;
- `zorder_columns`;
- `vacuum_retention_hours`.

Exemplos de `load_type`:

```text
full_overwrite
incremental_append
merge_scd1
```

#### QualityRuleSpec

Representa uma linha da `cfg_quality_rule`.

Define uma regra de qualidade.

Tipos suportados:

```text
not_null
unique
accepted_values
regex
range
custom_sql
```

Campos importantes:

- `rule_id`;
- `pipeline_id`;
- `dataset_id`;
- `column_name`;
- `rule_type`;
- `rule_expression`;
- `severity`;
- `threshold`;
- `on_failure`.

#### AuditEvent

Representa um evento de auditoria.

Pode registrar:

- pipeline executada;
- node executado;
- status;
- inicio;
- fim;
- contagem de origem;
- contagem de destino;
- rejeitados;
- watermark;
- erro;
- job/run do Databricks.

#### MetadataBundle

Agrupa todos os metadados carregados.

Ele contem:

- pipelines indexadas por `pipeline_id`;
- datasets indexados por `dataset_id`;
- lista de nodes;
- mapeamentos;
- estrategias de carga;
- regras de qualidade;
- dependencias;
- parametros de runtime.

Tambem possui metodos auxiliares:

```python
nodes_for_pipeline(...)
mappings_for_pipeline(...)
quality_rules_for(...)
load_strategy_for(...)
```

Esses metodos facilitam a montagem dinamica da pipeline.

## Utilitarios de Metadados

### src/kedro_template/pipeline_factory/metadata_utils.py

Contem funcoes auxiliares para normalizar valores vindos das tabelas Delta.

#### parse_bool

Converte valores comuns para booleano.

Aceita exemplos como:

```text
true
false
1
0
sim
nao
yes
no
```

Isso e util porque metadados podem vir como string, booleano ou numero.

#### parse_json_object

Converte um campo JSON em dicionario Python.

Usado principalmente para `parameters_json` da `cfg_node`.

Exemplo:

```json
{"where": "dt_ref >= current_date()"}
```

vira:

```python
{"where": "dt_ref >= current_date()"}
```

#### parse_string_list

Converte listas armazenadas em formatos diferentes.

Aceita:

```text
customer_id,document_id
```

ou:

```json
["customer_id", "document_id"]
```

e retorna:

```python
["customer_id", "document_id"]
```

#### rows_to_records

Converte dados vindos de Spark, Pandas ou listas Python para uma lista de
dicionarios.

Isso permite que o `ConfigLoader` funcione tanto em testes locais quanto em
ambiente Databricks.

#### dataclass_from_record

Cria uma dataclass a partir de um dicionario, ignorando colunas extras.

Isso permite que as tabelas `cfg_*` tenham campos adicionais sem quebrar o
codigo imediatamente.

## Carregamento e Validacao de Configuracao

### src/kedro_template/pipeline_factory/config_loader.py

Este arquivo contem o `ConfigLoader`, responsavel por ler e validar os
metadados.

Ele pode carregar metadados de duas formas:

1. via catalogo Kedro;
2. via SparkSession.

Uso com Spark:

```python
bundle = ConfigLoader(spark=spark).load(active_only=True)
```

Uso com catalogo Kedro:

```python
bundle = ConfigLoader(catalog=catalog).load(active_only=True)
```

O loader le as tabelas:

```text
cfg_pipeline
cfg_dataset
cfg_node
cfg_column_mapping
cfg_load_strategy
cfg_quality_rule
cfg_dependency
cfg_runtime_parameter
```

Depois transforma essas linhas em:

```text
PipelineSpec
DatasetSpec
NodeSpec
ColumnMappingSpec
LoadStrategySpec
QualityRuleSpec
MetadataBundle
```

### MetadataValidationError

Excecao disparada quando existe inconsistencia nos metadados.

Exemplos:

- node aponta para uma pipeline inexistente;
- node aponta para dataset inexistente;
- mapping aponta para dataset inexistente;
- estrategia de carga aponta para dataset destino inexistente.

Essa validacao e importante porque falhar cedo e melhor do que iniciar uma
execucao Spark longa com configuracao quebrada.

## Resolucao de Datasets

### src/kedro_template/pipeline_factory/dataset_resolver.py

Contem a classe `DatasetResolver`.

Ela centraliza a leitura e escrita dos datasets descritos em `cfg_dataset`.

#### read

Le um dataset como DataFrame Spark.

Suporta tres modos:

```text
table
path
query
```

Se `read_mode = table`, usa:

```python
spark.table(dataset.full_table_name)
```

Se `read_mode = path`, usa:

```python
spark.read.format(dataset.format).load(dataset.storage_path)
```

Se `read_mode = query`, usa:

```python
spark.sql(query)
```

#### write_append

Grava dados em modo append.

Uso esperado para:

- logs;
- eventos;
- dados incrementais imutaveis.

#### write_overwrite

Grava dados em modo overwrite.

Uso esperado para:

- cargas full;
- recriacao de tabelas;
- primeira carga de uma tabela destino.

#### optimize

Executa `OPTIMIZE` em uma tabela Delta no Databricks.

Tambem suporta `ZORDER BY`.

#### vacuum

Executa `VACUUM` em uma tabela Delta.

Deve ser usado com cuidado em ambientes produtivos, respeitando politicas de
retencao.

## Montagem de Nodes

### src/kedro_template/pipeline_factory/node_builder.py

Contem a classe `NodeBuilder`.

Ela converte cada `NodeSpec` em um node Kedro.

Tipos de node suportados:

```text
extract
transform
validate
load
```

### Node extract

Le um dataset usando `DatasetResolver`.

Usa `input_dataset_id` ou `output_dataset_id` para descobrir qual tabela ler.

Tambem aceita filtro via parametro:

```json
{"where": "dt_ref >= current_date()"}
```

### Node transform

Aplica transformacoes genericas.

Pode fazer:

- filtro por `where`;
- selecao de colunas;
- renomeacao;
- cast de tipos;
- expressao Spark SQL;
- default values;
- deduplicacao;
- inclusao de colunas tecnicas.

As colunas tecnicas adicionadas por padrao sao:

```text
_processing_timestamp
_pipeline_id
_node_id
```

### Node validate

Executa regras de qualidade usando `QualityRuleRunner`.

Se uma regra falhar e estiver configurada com:

```text
on_failure = fail_pipeline
```

a execucao levanta erro.

### Node load

Grava o DataFrame no destino.

Ele procura a estrategia de carga em `cfg_load_strategy`. Se nao encontrar,
usa o `write_mode` do dataset destino como fallback.

Exemplo:

```text
write_mode = merge_scd1
```

### custom_functions

O `NodeBuilder` aceita funcoes customizadas.

Isso permite que transformacoes muito especificas sejam implementadas em
Python/Spark e apenas referenciadas pelos metadados.

Esse ponto e importante porque evita transformar as tabelas de configuracao em
uma linguagem de programacao escondida.

## Montagem de Pipelines

### src/kedro_template/pipeline_factory/pipeline_builder.py

Contem a classe `PipelineBuilder`.

Responsavel por criar objetos `Pipeline` do Kedro a partir dos metadados
validados.

### build_pipeline

Monta uma pipeline especifica.

Fluxo:

1. recebe um `pipeline_id`;
2. busca os nodes ativos dessa pipeline;
3. ordena por `execution_order`;
4. cria os nodes Kedro;
5. retorna um objeto `Pipeline`.

### build_all

Monta todas as pipelines ativas.

Tambem cria uma pipeline `__default__`, combinando todas as pipelines
disponiveis.

## Auditoria

### src/kedro_template/pipeline_factory/audit.py

Contem a classe `AuditLogger`.

Ela escreve eventos em uma tabela Delta de auditoria.

Eventos suportados:

```text
running
success
failed
```

Tambem existem duas funcoes auxiliares:

```python
new_execution_id()
utc_now()
```

### new_execution_id

Gera um UUID para identificar uma execucao.

### utc_now

Retorna timestamp UTC com timezone.

### AuditLogger.running

Registra o inicio de uma execucao.

### AuditLogger.success

Registra sucesso, incluindo metricas como:

- registros lidos;
- registros gravados;
- registros rejeitados;
- watermark inicial;
- watermark final.

### AuditLogger.failed

Registra falha e mensagem de erro.

## Estrategias de Carga

A pasta `load_strategies` contem as formas de gravacao suportadas.

### src/kedro_template/pipeline_factory/load_strategies/base.py

Define a classe abstrata `LoadStrategy`.

Toda estrategia de carga deve implementar:

```python
execute(dataframe, target_dataset, strategy)
```

Tambem contem `_post_write`, que executa operacoes opcionais depois da escrita:

- `OPTIMIZE`;
- `VACUUM`.

### src/kedro_template/pipeline_factory/load_strategies/full_overwrite.py

Implementa `FullOverwriteStrategy`.

Ela:

1. conta os registros de entrada;
2. grava o DataFrame em modo overwrite;
3. executa pos-processamento opcional;
4. retorna metricas.

Uso recomendado:

- cargas pequenas;
- reconstrucao completa;
- tabelas derivadas que podem ser recriadas.

### src/kedro_template/pipeline_factory/load_strategies/incremental_append.py

Implementa `IncrementalAppendStrategy`.

Ela:

1. conta registros;
2. grava em modo append;
3. executa pos-processamento opcional;
4. retorna metricas.

Uso recomendado:

- eventos;
- logs;
- dados imutaveis;
- cargas incrementais sem atualizacao de registros antigos.

### src/kedro_template/pipeline_factory/load_strategies/merge_scd1.py

Implementa `MergeScd1Strategy`.

Ela usa Delta Lake merge para fazer upsert.

Fluxo:

1. valida se `merge_keys` foi informado;
2. conta registros de origem;
3. verifica se a tabela destino existe;
4. se nao existir, cria por overwrite;
5. se existir, executa merge;
6. atualiza registros encontrados;
7. insere registros novos;
8. opcionalmente remove registros ausentes na origem;
9. executa pos-processamento opcional.

Uso recomendado:

- dimensoes sem historico;
- tabelas cadastrais;
- cargas com atualizacao do estado atual.

### src/kedro_template/pipeline_factory/load_strategies/__init__.py

Expoe a funcao:

```python
create_load_strategy(load_type, resolver, write_options)
```

Ela funciona como uma factory interna das estrategias de carga.

Mapeamentos suportados:

```text
full -> FullOverwriteStrategy
full_overwrite -> FullOverwriteStrategy
overwrite -> FullOverwriteStrategy
append -> IncrementalAppendStrategy
incremental_append -> IncrementalAppendStrategy
merge -> MergeScd1Strategy
scd1 -> MergeScd1Strategy
merge_scd1 -> MergeScd1Strategy
```

## Qualidade de Dados

### src/kedro_template/pipeline_factory/quality/rule_runner.py

Contem:

```python
QualityRuleRunner
QualityResult
DataQualityError
```

### QualityRuleRunner

Executa regras de qualidade sobre DataFrames Spark.

Regras suportadas:

```text
not_null
unique
accepted_values
regex
range
custom_sql
```

### not_null

Falha linhas onde a coluna esta nula.

### unique

Falha linhas duplicadas com base na coluna informada.

### accepted_values

Valida se a coluna contem apenas valores permitidos.

Exemplo de `rule_expression`:

```text
ATIVO,INATIVO,CANCELADO
```

### regex

Valida a coluna usando expressao regular.

### range

Valida minimo e maximo numerico.

Exemplo:

```text
0,100
```

### custom_sql

Permite uma expressao SQL customizada.

Exemplo:

```sql
amount >= 0 and customer_id is not null
```

O runner aplica:

```python
dataframe.where(f"NOT ({rule_expression})")
```

para identificar linhas com falha.

### QualityResult

Representa o resultado de uma regra:

- `rule_id`;
- `status`;
- `failed_count`;
- `total_count`;
- `failure_ratio`;
- `severity`;
- `message`.

### DataQualityError

Erro levantado quando uma regra falha e a configuracao manda interromper a
pipeline.

## Testes

### tests/pipeline_factory/test_config_loader.py

Testa o `ConfigLoader`.

Cenarios cobertos:

- carregamento de pipelines;
- normalizacao de datasets;
- criacao automatica de `full_table_name`;
- parsing de listas;
- parsing de JSON em `parameters_json`;
- validacao de referencia invalida.

Esse teste usa um `FakeCatalog`, evitando depender de Kedro, Spark ou
Databricks.

### tests/pipeline_factory/test_metadata_utils.py

Testa os utilitarios de metadados.

Cenarios cobertos:

- conversao de booleanos;
- listas em JSON;
- listas separadas por virgula;
- tratamento de valores nulos.

## Como Validar Localmente

Como os testes atuais nao dependem de Spark, eles podem ser executados com:

```powershell
$env:PYTHONPATH='src'
python -m unittest discover -s tests\pipeline_factory
```

Resultado esperado:

```text
Ran 4 tests
OK
```

## Como a Execucao Deve Funcionar no Databricks

Um entrypoint Databricks futuro deve fazer algo como:

```python
from kedro_template.pipeline_registry import build_dynamic_pipelines

pipelines = build_dynamic_pipelines(
    spark=spark,
    active_only=True,
    write_options={"mergeSchema": "true"},
)

pipeline = pipelines["pipe_customer"]
```

Depois disso, a execucao pode ser integrada ao mecanismo de runner escolhido
para o projeto Kedro.

## Fluxo Esperado de uma Pipeline

Exemplo de pipeline com quatro nodes:

```text
extract_customer
        |
        v
transform_customer
        |
        v
validate_customer
        |
        v
load_customer
```

Cada node vem da tabela `cfg_node`.

A ordem vem de:

```text
execution_order
```

Os datasets vem de:

```text
cfg_dataset
```

Os mapeamentos vem de:

```text
cfg_column_mapping
```

A estrategia de gravacao vem de:

```text
cfg_load_strategy
```

As regras de qualidade vem de:

```text
cfg_quality_rule
```

## Observacoes Importantes

### Metadados nao devem virar linguagem de programacao

As tabelas `cfg_*` devem parametrizar padroes conhecidos.

Exemplos bons para metadados:

- selecionar colunas;
- renomear colunas;
- aplicar casts;
- aplicar filtros simples;
- configurar merge keys;
- configurar regras de qualidade;
- escolher estrategia de carga.

Exemplos que devem virar codigo Python/Spark:

- regras de negocio muito especificas;
- transformacoes com muitas etapas;
- joins complexos;
- algoritmos;
- calculos que mudam bastante por dominio.

Para esses casos, o campo `function_name` deve apontar para uma funcao
customizada registrada no projeto.

### Unity Catalog deve ser o padrao

Como as fontes estao no ADLS governado pelo Unity Catalog, o padrao preferido
de identificacao de tabelas deve ser:

```text
catalog.schema.table
```

O uso de path ADLS deve ficar para casos especificos.

### SCD2 ainda nao foi implementado

O template implementa:

- full overwrite;
- incremental append;
- merge SCD1.

SCD2 deve ser uma proxima estrategia, provavelmente em:

```text
src/kedro_template/pipeline_factory/load_strategies/merge_scd2.py
```

Ela exigiria campos adicionais como:

- chave de negocio;
- hash de comparacao;
- `effective_from`;
- `effective_to`;
- `is_current`;
- regra de fechamento de vigencia.

## Proximos Passos Recomendados

1. Criar scripts SQL para gerar as tabelas `cfg_*` no Unity Catalog.
2. Criar dados de exemplo para uma pipeline piloto.
3. Criar um entrypoint Databricks para executar uma pipeline por parametro.
4. Implementar hooks de auditoria integrados ao ciclo de execucao.
5. Validar a factory em um cluster Databricks real.
6. Implementar `merge_scd2` se houver necessidade de historico.
7. Adicionar suporte a dependencias entre pipelines usando `cfg_dependency`.
8. Adicionar quarantine table para registros rejeitados em qualidade.
