# Case Técnico — Engenharia de Dados

Solução de engenharia de dados desenvolvida no Databricks com Python, PySpark e Delta Lake.

O projeto transforma nove fontes brutas, com formatos e estruturas diferentes, em um modelo analítico dimensional preparado para consumo por profissionais de BI e Dados.

## Objetivo

Construir uma base organizada, confiável, rastreável e pronta para responder análises relacionadas a:

- receita e volume de pedidos;
- ticket médio;
- cancelamentos;
- produtos, categorias e clientes;
- desempenho de vendedores, canais e regiões;
- entregas e atrasos;
- ocorrências operacionais;
- qualidade dos dados.

## Tecnologias utilizadas

- Databricks;
- Python;
- PySpark;
- Spark SQL;
- Delta Lake;
- Unity Catalog;
- Git e GitHub;
- Pandas e OpenPyXL para leitura dos arquivos Excel.

## Arquitetura

A solução utiliza uma arquitetura em camadas:

```text
Fontes brutas
    ↓
Landing Volume
    ↓
Bronze
    ↓
Silver
    ↓
Gold
    ↓
Consumo analítico / BI
```

### Bronze

Preserva os dados recebidos das fontes e adiciona metadados de rastreabilidade.

### Silver

Normaliza, converte, valida e consolida os registros, mantendo indicadores de qualidade `dq_*`.

### Gold

Disponibiliza dimensões, fatos e indicadores mensais com granularidades definidas e nomes adequados para consumo analítico.

Mais detalhes estão disponíveis em [docs/arquitetura.md](docs/arquitetura.md).

## Fontes processadas

O projeto processa nove arquivos:

| Fonte | Formato | Conteúdo |
|---|---|---|
| `atendimento_ocorrencias.ndjson` | NDJSON | Ocorrências operacionais |
| `cadastro_produtos_api_dump.json` | JSON | Produtos e preços |
| `comercial_canais.xlsx` | Excel | Canais comerciais |
| `crm_clientes_export.xlsx` | Excel | Clientes |
| `erp_pedidos_cabecalho_2025.csv` | CSV | Cabeçalho dos pedidos |
| `erp_pedidos_itens_2025.csv` | CSV | Itens dos pedidos |
| `legado_regioes_pipe.txt` | TXT delimitado | Regiões |
| `logistica_entregas.json` | JSON | Entregas |
| `vendedores.csv` | CSV | Vendedores |

## Organização dos notebooks

Os notebooks devem ser executados na seguinte ordem:

### 1. `00_exploracao_fontes`

Executa a investigação inicial:

- descoberta dinâmica dos arquivos;
- leitura dos diferentes formatos;
- inventário de schemas;
- análise de valores ausentes;
- identificação de duplicidades;
- validação de datas e números;
- integridade referencial;
- consistência financeira e temporal.

### 2. `01_ingestao_bronze`

Realiza:

- cópia das fontes para o Volume de landing;
- ingestão das nove fontes;
- criação das tabelas Delta Bronze;
- geração de hash por registro;
- inclusão de metadados técnicos;
- registro da auditoria de ingestão.

### 3. `02_transformacao_silver_dimensoes`

Cria as entidades dimensionais consolidadas:

- `silver_regioes`;
- `silver_canais`;
- `silver_clientes`;
- `silver_vendedores`;
- `silver_produtos`.

### 4. `03_transformacao_silver_fatos`

Cria as entidades transacionais:

- `silver_pedidos`;
- `silver_pedidos_itens`;
- `silver_entregas`;
- `silver_ocorrencias`.

### 5. `04_modelagem_gold`

Cria o modelo analítico final:

#### Dimensões

- `dim_data`;
- `dim_clientes`;
- `dim_produtos`;
- `dim_vendedores`;
- `dim_canais`;
- `dim_regioes`.

#### Fatos

- `fct_pedidos`;
- `fct_pedidos_itens`;
- `fct_entregas`;
- `fct_ocorrencias`.

#### Agregação

- `agg_indicadores_mensais`.

## Instruções de execução

1. Importe ou clone este repositório em um Git Folder do Databricks.
2. Mantenha os arquivos brutos disponíveis na pasta `datasources`.
3. Execute os notebooks na ordem numérica, de `00` até `04`.
4. Confirme que as auditorias de cada etapa apresentam status de sucesso.
5. Consulte as tabelas no catálogo identificado dinamicamente pelos notebooks.

A solução não depende de um e-mail ou catálogo fixo. O catálogo e os caminhos são localizados dinamicamente durante a execução.

## Auditoria e rastreabilidade

Foram criadas as seguintes tabelas de auditoria:

| Tabela | Responsabilidade |
|---|---|
| `bronze_ingestion_audit` | Auditoria da ingestão das fontes |
| `silver_dimension_audit` | Auditoria das dimensões Silver |
| `silver_fact_audit` | Auditoria dos fatos Silver |
| `gold_model_audit` | Auditoria estrutural da Gold |

As tabelas finais também possuem identificadores de execução, timestamps de processamento e indicadores de qualidade.

## Principais problemas encontrados

Durante a exploração foram identificados, entre outros:

- chaves de negócio duplicadas;
- variações de caixa, acentuação e espaços;
- datas em formatos diferentes;
- datas impossíveis;
- valores numéricos como `N/A` e `unknown`;
- clientes, produtos, pedidos e canais órfãos;
- divergências entre valores informados e calculados;
- conflitos entre status logístico e datas;
- campos obrigatórios ausentes;
- diferentes versões do mesmo registro.

Os registros foram preservados sempre que possível. Problemas não estruturais foram sinalizados por colunas `dq_*`, evitando perda silenciosa de informação.

## Decisões principais

- uso de Delta Lake em todas as camadas;
- separação entre dimensões e fatos;
- granularidades independentes para pedidos, itens, entregas e ocorrências;
- deduplicação determinística;
- preservação dos valores originais na Silver;
- criação de membro técnico `NAO_IDENTIFICADO` nas dimensões Gold;
- geração determinística de chaves técnicas com SHA-256;
- agregação prévia de entregas e ocorrências antes da criação dos indicadores mensais;
- tratamento de campos inválidos com conversões seguras, sem interromper o pipeline.

## Documentação

- [Arquitetura](docs/arquitetura.md)
- [Documentação técnica](docs/documentacao_tecnica.md)
- [Resumo executivo](docs/resumo_executivo.md)
