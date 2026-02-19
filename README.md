# Painel Produtividade Operação

Este repositório contém a solução completa do dashboard **Produtividade Operação** desenvolvido em Power BI, com fonte de dados no Databricks.

---

## 1. Objetivo de Negócios

O painel fornece indicadores de produtividade dos aprovadores de ordens de serviço (OS) no processo de manutenção de frotas. Ele calcula quantas OSs são aprovadas por dia, ajustando por tipo de manutenção, valores orçados, interações com parceiros, feriados e regras de SLA.

**Usuários:** coordenadores, supervisores, aprovadores e analistas de operações.

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart LR
  A[Aprovador recebe OS] --> B{Tipo de manutenção}
  B -->|Corretiva| C[Calcula peso_os]
  B -->|Preventiva| C
  B -->|Sinistro| C
  C --> D[Aplica fatores de correção]
  D --> E[Produtividade = peso_os / dia útil]
  E --> F[Dashboard Power BI]
```

---

## 2. Arquitetura

| Camada | Tecnologia | Detalhes |
|--------|-----------|----------|
| **Dados** | Azure Databricks | Catálogo `hive_metastore`, schemas `gold` e `bronze` |
| **ETL/Query** | Spark SQL | Query self-contained com CTEs (sem tabela materializada) |
| **UDF** | Databricks | `dev_charles_barros.fn_calcular_sla_formatado` |
| **Modelagem** | Power BI (TMDL) | Modelo tabular importado |
| **Conexão** | Databricks.Query | Host: `adb-7941093640821140.0.azuredatabricks.net` |
| **Visualização** | Power BI Report | Páginas de visão geral, detalhes e auditoria |

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart TD
  subgraph Databricks["Azure Databricks"]
    G[gold.fact_maintenanceservices]
    H[gold.dim_maintenance*]
    I[gold.dim_dates]
    J[bronze.tlbagda..._historico]
    K[gold.Dim_MaintenanceParameterLogValue]
  end
  subgraph PowerBI["Power BI"]
    L["Azure - Databricks.Query"]
    M[M Transform Steps]
    N[Modelo Tabular]
    O[Dashboard]
  end
  G --> L
  H --> L
  I --> L
  J --> L
  K --> L
  L --> M
  M --> N
  N --> O
```

---

## 3. Estrutura da Modelagem

A tabela principal é **FactAprovacao**, carregada por uma query SQL complexa com múltiplas CTEs. Relacionamentos com dimensões de calendário, aprovadores, supervisores, coordenadores e metas.

```mermaid
%%{init: {'theme': 'dark'}}%%
erDiagram
  FactAprovacao ||--o{ DimCalendario : "data_aprovacao"
  FactAprovacao ||--o{ dim_aprovadores : "_aprovador"
  FactAprovacao ||--o{ dim_supervisores : "_aprovador"
  FactAprovacao ||--o{ dim_coordenadores : "_aprovador"
  FactAprovacao ||--o{ dim_metas : "CustomerId"
  FactAprovacao ||--o{ FactAuditoria : "OrderServiceCode"
  FactAprovacao ||--o{ glosas : "OrderServiceCode"
```

---

## 4. Tabelas Utilizadas

| Tabela | Schema | Descrição |
|--------|--------|-----------|
| `fact_maintenanceservices` | gold | OSs e timestamps de aprovação |
| `fact_maintenanceitems` | gold | Itens para cálculo de aderência |
| `dim_maintenancevehicles` | gold | Veículos (família, cliente) |
| `dim_maintenancetypes` | gold | Tipo de manutenção |
| `dim_webusers` | gold | Usuários aprovadores (4 joins) |
| `dim_dates` | gold | Calendário com feriados |
| `dim_Maintenanceprotocols` | gold | Protocolos de manutenção |
| `dim_maintenancemerchants` | gold | Estabelecimentos (filtro concessionária) |
| `dim_maintenancelabors` | gold | Tipos de mão de obra |
| `Dim_MaintenanceParameterLogValue` | gold | Log de parâmetros (preço parceiro) |
| `tlbagda_fuel_manutencao_oficina_historico` | bronze | Histórico de interações (revisão/cotação) |

---

## 5. Medidas DAX Principais

| Medida | Fórmula/Descrição |
|--------|-------------------|
| `Tempo Estimado Aprovação` | `7.48 * FactAprovacao[peso_os]` |
| `Dentro SLA` | `sla_aprovocao <= dim_metas.SLA (h)` (calculada em M) |
| `produtividade_semana_aprovador` | soma_peso_os / dias_úteis (calculada em SQL) |
| `peso_os` | `1 / meta_os_dia_corrigido` (calculada em SQL) |

> As demais medidas estão no arquivo `_Medidas.tmdl` do modelo semântico.

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart TD
  A[Tipo Manutencao + Faixa Qtde Itens] --> B[tempo_padrao]
  C[Interacoes revisao/cotacao] --> D[tempo_extra_interacao]
  E[Cliente mercado publico] --> F[tempo_extra_mercadopublico]
  G["Familia modelo + Valor OS"] --> H["_monta / fator_correcao_monta"]
  I[Preco parceiro] --> J["fator_correcao_precoparceiro 1.0 ou 1.1"]
  B --> K["meta_os_dia = 7.48h x 60 / tempo_padrao + extras"]
  D --> K
  F --> K
  K --> L["meta_corrigida = meta / fator_monta x fator_preco"]
  H --> L
  J --> L
  L --> M["peso_os = 1 / meta_corrigida"]
  M --> N["produtividade = soma peso_os / dias_uteis"]
```

---

## 6. Código SQL

A query está disponível em dois formatos:

| Arquivo | Descrição |
|---------|-----------|
| `FactAprovacao.sql` | Versão original com `CREATE TABLE` no schema `dev_matheus_bertoti` |
| `FactAprovacao_DirectQuery.sql` | **Versão atual** - query self-contained sem dependência de schema dev |

### Estrutura das CTEs

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart TD
  subgraph bloco1["Bloco 1: Preco Parceiro"]
    P1[pi_logs] --> P2[pi_sets]
    P2 --> P3[pi_adds]
    P2 --> P4[pi_removes]
    P3 --> P5[pi_events]
    P4 --> P5
    P5 --> P6[pi_ordered]
    P6 --> P7[param_intervals]
  end
  subgraph bloco2["Bloco 2: Query Principal"]
    Q1[CTE_BASE] --> Q2[CTE_FLAG]
    P7 --> Q2
    Q2 --> Q3[CTE_CALC]
    Q3 --> Q4["CTE2 - dias uteis"]
    Q3 --> Q5["CTE3 - aderencia"]
    Q5 --> Q6["CTE4 - totais"]
    Q3 --> Q7[SELECT FINAL]
    Q4 --> Q7
    Q6 --> Q7
  end
```

---

## 7. Modificações Recentes

| Data | Modificação |
|------|-------------|
| 2026-02-19 | Eliminado schema `dev_matheus_bertoti` e tabela materializada |
| 2026-02-19 | Criada query self-contained `FactAprovacao_DirectQuery.sql` |
| 2026-02-19 | Atualizada partition source no TMDL para usar query direta |
| 2026-02-19 | Versionamento inicial no GitHub |

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart LR
  subgraph antes["Antes"]
    A1["Databricks: CREATE TABLE dev_matheus_bertoti.produtividadeoperacao"]
    A2["Power BI: SELECT * FROM dev_matheus_bertoti..."]
    A1 --> A2
  end
  subgraph depois["Depois"]
    B1["Power BI: WITH ... SELECT query completa inline"]
    B2["Databricks executa sob demanda"]
    B1 --> B2
  end
  antes -.->|migracao| depois
```

---

## 8. Notas Adicionais

- O painel **não retorna valores em feriados** ou dias não úteis por design (filtro `dd.HolidayOrBridge`).
- Aprovadores do sábado seguem regra diferente (`WeekDayNumber <= 5` vs `<= 4`).
- Caso o UDF `dev_charles_barros.fn_calcular_sla_formatado` seja removido, será necessário replicar sua lógica inline.

---

> **Repositório:** [EntregaResultados/Painel-Produtividade-Operacao](https://github.com/EntregaResultados/Painel-Produtividade-Operacao)
