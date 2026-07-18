# Arquitetura da solução

## Visão geral

A solução foi estruturada em uma arquitetura de dados em camadas, separando ingestão, tratamento e consumo analítico.

    A[Arquivos CSV, Excel, JSON, NDJSON e TXT]
    B[Volume de Landing]
    C[Camada Bronze]
    D[Silver Dimensões]
    E[Silver Fatos]
    F[Gold Dimensões]
    G[Gold Fatos]
    H[Indicadores mensais]
    I[Analista de BI / Dashboards]

### Fontes
A entrada é composta por nove arquivos de diferentes sistemas e formatos.

Os arquivos possuem diferenças de:

- delimitadores;
- encoding;
- estrutura;
- níveis de aninhamento;
- formatos de data;
- nomenclatura;
- qualidade.

### Landing
Os arquivos são localizados dinamicamente no Git Folder e copiados.

Objetivos:

- separar código e arquivos de processamento;
- estabelecer um ponto controlado de entrada;
- validar a presença das nove fontes;
- facilitar rastreabilidade.

### Bronze

A Bronze mantém o conteúdo das fontes com alterações mínimas.

Cada registro recebe:

- `_source_record_hash`;
- `_source_file`;
- `_source_path`;
- `_ingestion_batch_id`;
- `_ingestion_timestamp_utc`.

As tabelas são persistidas em formato Delta.

### Silver

A Silver é dividida em entidades dimensionais e transacionais.

#### Dimensões Silver

```text
silver_regioes
silver_canais
silver_clientes
silver_vendedores
silver_produtos
```

#### Fatos Silver

```text
silver_pedidos
silver_pedidos_itens
silver_entregas
silver_ocorrencias
```

Responsabilidades da Silver:

- normalização de identificadores;
- padronização de domínios;
- remoção de espaços laterais;
- tratamento de caixa e acentuação;
- conversão segura de datas e números;
- resolução de duplicidades;
- validação de relacionamentos;
- preservação dos valores originais;
- criação de indicadores `dq_*`.

### Gold

A Gold disponibiliza um modelo dimensional orientado ao consumo.

    DIM_CLIENTES {
        string customer_key PK
        string customer_id
        string customer_name
        string segment
        string customer_size
        string state_code
    }

    DIM_PRODUTOS {
        string product_key PK
        string product_id
        string product_name
        string category
        decimal list_price
    }

    DIM_VENDEDORES {
        string seller_key PK
        string seller_id
        string seller_name
        string channel_key FK
        string region_key FK
    }

    FCT_PEDIDOS {
        string order_key PK
        string customer_key FK
        string seller_key FK
        string channel_key FK
        string region_key FK
        int order_date_key FK
        decimal net_amount
        long order_count
    }

    FCT_PEDIDOS_ITENS {
        string order_item_key PK
        string order_key FK
        string product_key FK
        decimal quantity
        decimal unit_price
        decimal total_item
    }

    FCT_ENTREGAS {
        string delivery_key PK
        string order_key FK
        int shipped_date_key FK
        int delivered_date_key FK
        decimal delivery_cost
        long late_delivery_count
    }

    FCT_OCORRENCIAS {
        string occurrence_key PK
        string order_key FK
        int opened_date_key FK
        int closed_date_key FK
        double resolution_hours
    }



## Prevenção de duplicação de métricas

Pedidos, itens, entregas e ocorrências possuem granularidades diferentes.

Por esse motivo, eles não foram consolidados em uma única tabela ampla. Uma tabela única poderia multiplicar:

- receita;
- descontos;
- quantidade de pedidos;
- custos logísticos;
- quantidade de ocorrências.

Para a criação dos indicadores mensais, entregas e ocorrências são agregadas por pedido antes do relacionamento com `fct_pedidos`.

## Chaves técnicas

As dimensões utilizam chaves técnicas determinísticas geradas com SHA-256.

Benefícios:

- estabilidade entre execuções;
- independência da ordem de processamento;
- redução de colisões;
- facilidade de relacionamentos.

A chave `0` é reservada ao membro `NAO_IDENTIFICADO`.

## Auditoria

As auditorias registram:

- quantidade de entrada;
- quantidade de saída;
- registros consolidados;
- chaves duplicadas;
- chaves nulas;
- registros órfãos;
- status da execução.

## Estratégia de reprocessamento
A solução atual utiliza sobrescrita controlada nas tabelas de dados e `append` nas tabelas de auditoria.