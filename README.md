# Nexus Apex - Pipeline de Dados de Mercado Financeiro
### Documentação do Projeto

> **Versão:** 1.0  
> **Mercados cobertos:** Brasil (B3, CVM, Bacen) · EUA (Yahoo Finance, Alpha Vantage) · Global (Cripto, Forex, Macro)

---

## O que é este projeto

Este projeto é um pipeline de dados financeiros que coleta, processa e disponibiliza informações de múltiplos mercados em tempo quase real. A ideia central é simples: os dados chegam de diversas fontes, passam por um processo de limpeza e organização, e ficam disponíveis para consulta por dashboards, APIs e ferramentas de análise.

A arquitetura adotada é chamada de **Kappa**  um modelo que trata todos os dados como um fluxo contínuo de eventos, usando um único pipeline para dados em tempo real e para reprocessamentos históricos. Isso significa menos complexidade e um único código que serve a todos os casos.

---

## De onde vêm os dados

O pipeline coleta informações de oito fontes distintas:

| Fonte | O que fornece | Quando coleta |
|---|---|---|
| **Yahoo Finance** | Cotações de ações brasileiras e americanas, ETFs e  índices | A cada 15 minutos, durante o horário de mercado |
| **Alpha Vantage** | Câmbio (Forex) e criptomoedas | A cada 15 minutos |
| **B3 (Bovespa)** | Cotações oficiais de fechamento do mercado brasileiro | Diariamente, após o fechamento (18h) |
| **CVM** | Documentos de empresas abertas: balanços, ITRs e fatos relevantes | Varredura diária |
| **Bacen (SGS)** | Indicadores econômicos: Selic, IPCA, CDI e câmbio oficial | Diariamente ao meio-dia |
| **CoinGecko** | Preços das 200 principais criptomoedas | A cada 15 minutos, 24 horas por dia |
| **FRED (Federal Reserve)** | Indicadores macro dos EUA: CPI, Fed Funds e Treasuries | Diariamente |
| **ECB (Banco Central Europeu)** | Câmbio EUR/USD e indicadores europeus | Diariamente |

---

## Como o pipeline funciona

O fluxo de dados percorre quatro etapas principais:

### 1. Coleta e publicação
Processos automatizados coletam dados de cada fonte e os publicam como eventos em um barramento central. Cada evento carrega informações padronizadas: identificador único, horário, fonte, ativo e o dado em si.

### 2. Distribuição dos eventos
O barramento central (Pub/Sub) recebe todos os eventos e os distribui simultaneamente para dois destinos: o processador em tempo real e o arquivo histórico no armazenamento em nuvem.

### 3. Processamento
Um motor de processamento recebe os eventos e aplica as seguintes transformações:

- **Deduplicação**  elimina eventos duplicados que possam ter chegado mais de uma vez
- **Normalização**  padroniza tipos de dados, formatos de ticker e remove valores inválidos
- **Agregação em janelas temporais**  calcula abertura, máxima, mínima, fechamento e volume (OHLCV) em janelas de 15 minutos e diárias
- **Enriquecimento**  adiciona informações cadastrais dos ativos (nome, setor, bolsa)

### 4. Disponibilização
Os dados processados são gravados em tabelas organizadas por data, ativo e mercado, prontas para consulta eficiente.

---

## Onde os dados ficam armazenados

Os dados percorrem três tipos de armazenamento com funções distintas:

**Barramento de eventos (Pub/Sub)**
Ponto central por onde todos os eventos transitam. Mantém as mensagens disponíveis por 7 dias, o que permite reprocessar dados recentes em caso de falha.

**Arquivo histórico (Cloud Storage)**
Cópia de todos os eventos brutos, organizada por tópico, ano, mês, dia e hora. Retém os dados por 30 dias e serve como fonte para reprocessamentos históricos.

**Camada de serviço (BigQuery)**
Destino final dos dados processados. As tabelas são otimizadas para consulta rápida, particionadas por data e indexadas por ativo e mercado. É daqui que dashboards, APIs e notebooks lêem os dados.

---

## Como os dados são consumidos

Três tipos de consumidores acessam os dados processados:

**Dashboards**
Ferramentas de visualização (Looker ou Metabase) conectadas diretamente ao BigQuery, com acesso restrito à camada de serviço. Usadas para acompanhar cotações, indicadores macro e volumes de mercado em painéis visuais.

**API REST**
Uma API web (hospedada no Cloud Run) expõe endpoints para consulta de cotações, séries históricas OHLCV e indicadores macroeconômicos. Responde em segundos graças a um cache que armazena resultados frequentes por até 5 minutos.

**Notebooks analíticos**
Ambiente de análise exploratória (Vertex AI Workbench) com acesso direto ao BigQuery para análises quantitativas, backtesting e desenvolvimento de modelos.

---

## Reprocessamento histórico (Replay)

Um dos diferenciais da arquitetura Kappa é a capacidade de reprocessar dados históricos usando exatamente o mesmo código do processamento em tempo real. Isso é útil quando:

- Um bug é corrigido e os dados dos últimos dias precisam ser recalculados
- Uma nova coluna é adicionada e precisa ser preenchida retroativamente
- O pipeline ficou inativo por algum período e os dados precisam ser recuperados
- A lógica de enriquecimento muda e os dados históricos precisam ser atualizados

O reprocessamento pode cobrir qualquer janela dentro dos últimos 30 dias, lendo do arquivo histórico no Cloud Storage.

---

## Orquestração e monitoramento

Todo o ciclo de vida do pipeline é gerenciado pelo **Cloud Composer** (Airflow), que é responsável por:

- Agendar e executar os coletores de cada fonte de dados
- Criar os ambientes de processamento sob demanda e destruí-los após o uso
- Monitorar o atraso na fila de eventos e alertar quando ele ultrapassar 30 minutos
- Verificar diariamente a qualidade dos dados (volume mínimo, ausência de nulos, sem duplicatas)
- Acionar reprocessamentos quando necessário

Alertas de falha são enviados automaticamente via Slack.

---

## Ferramentas utilizadas

| Categoria | Ferramenta | Função |
|---|---|---|
| **Orquestração** | Cloud Composer 2 (Airflow) | Agenda e monitora todo o pipeline |
| **Barramento de eventos** | Cloud Pub/Sub | Distribui eventos entre os componentes |
| **Processamento** | Apache Spark (Dataproc) | Transforma e agrega os dados |
| **Armazenamento histórico** | Cloud Storage (GCS) | Arquivo de eventos brutos por 30 dias |
| **Camada de serviço** | BigQuery | Armazena e serve os dados processados |
| **API** | Cloud Run + FastAPI | Expõe endpoints REST para consulta |
| **Dashboards** | Looker / Metabase | Visualização e análise |
| **Segredos** | Secret Manager | Gerencia chaves de API com segurança |
| **Monitoramento** | Cloud Monitoring + Logging | Alertas e rastreabilidade |

---

## Status de implementação

**Fase 1  Infraestrutura** *(concluída parcialmente)*
- [x]

**Fase 2  Coletores** *(pendente)*
- [ ]

**Fase 3  Processamento** *(pendente)*
- [ ] 

**Fase 4  Serving e qualidade** *(pendente)*
- [ ]

---

>**Nexus Apex**, A conexão que eleva o pulso do mercado ao ápice da inteligência  
>_Ex chao ordo, in nexu veritas_