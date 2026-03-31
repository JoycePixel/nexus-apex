# Nexus Apex
## A Inteligência em Movimento do Mercado Financeiro

```
 ╔════════════════════════════════════════════════════════════════╗
 ║  Pipeline de Dados | Múltiplos Mercados | Arquitetura Kappa    ║
 ║  Versão 1.0 • Brasil · EUA · Global (Cripto, Forex, Macro)     ║
 ╚════════════════════════════════════════════════════════════════╝
```

### Visão Geral

Nexus Apex é uma infraestrutura de processamento de dados financeiros que transforma fluxos brutos de múltiplas fontes em inteligência estruturada, disponibilizada em tempo quase real para análise e decisão.

A arquitetura **Kappa** adopta um modelo revolucionário: um único pipeline, dois cenários. Os mesmos transformadores que processam dados ao vivo reprocessam históricos, eliminando duplicação lógica e reduzindo pontos de falha.

**Diferencial:** O sistema não apenas coleta dados — orquestra, valida, enriquece e serve com garantias de consistência, auditoria completa e capacidade de reprocessamento total.

---

## Fontes de Dados: A Rede de Captura

Oito fluxos independentes alimentam o sistema com dados de múltiplos horizontes temporais e geográficos:

| Fonte | Escopo | Frequência | Cobertura |
|-------|--------|-----------|----------|
| **Yahoo Finance** | Ações, ETFs, Índices | 15 min | Brasil + EUA |
| **Alpha Vantage** | Forex, Criptomoedas | 15 min | Global |
| **B3** | Fechamento oficial | Diário (18h) | Brasil |
| **CVM** | Documentação corporativa | Daily scan | Brasil (empresas abertas) |
| **Bacen** | Indicadores macroeconômicos | Diário (meio-dia) | Brasil |
| **CoinGecko** | Preços de 200+ criptomoedas | 15 min | 24/7 Global |
| **FRED** | Macro EUA (CPI, Fed Funds, Treasuries) | Diário | EUA |
| **ECB** | Forex EUR/USD, Indicadores europeus | Diário | Europa |

---

## Anatomia do Pipeline: Do Caos à Ordem

```
COLETA                 DISTRIBUIÇÃO              TRANSFORMAÇÃO             PERSISTÊNCIA
════════════════════════════════════════════════════════════════════════════════════════

Fonte 1 ─┐
Fonte 2 ─┼─→ Eventos Brutos ─→ Pub/Sub ─┬─→ Processador Spark ─→ Validação ─┐
Fonte 3 ─┤                     (barramento) │   (Dedup, Norm,      (QA)       ├─→ BigQuery
Fonte 4 ─┤                                 │    Agregação,                    │
Fonte 5 ─┼─────────────────────────────────┼─────Enriquecimento)─────────────┘
Fonte 6 ─┤                                 │
Fonte 7 ─┤                                 └─→ Cloud Storage (Histórico 30d)
Fonte 8 ─┘
```

### Estágio 1: Publicação de Eventos
Cada fonte, em sua própria cadência, publica eventos estruturados no barramento central. Cada mensagem carrega: timestamp, identificador único, tipo de ativo, mercado e valor.

### Estágio 2: Distribuição Dual
O Pub/Sub bifurca o fluxo:
- **Via rápida** → Processador em tempo real (latência < 2 minutos)
- **Via arquivo** → Cloud Storage para reprocessamento futuro (retenção: 30 dias)

### Estágio 3: Transformação e Enriquecimento
O Apache Spark aplica as operações core do pipeline:

- **Deduplicação** — Eventos duplicados no transporte são descartados (garante idempotência)
- **Normalização** — Tipos de dados, tickers e timestamps são padronizados; valores inválidos removidos
- **Agregação Temporal** — Cálculo de OHLCV (Open, High, Low, Close, Volume) em janelas de 15 minutos e 1 dia
- **Enriquecimento** — Acréscimo de metadados: nome do ativo, setor, bolsa, moeda base

### Estágio 4: Persistência e Serving
Dados processados são gravados no BigQuery com três dimensões de particionamento:
- Data (otimiza varreduras históricas)
- Mercado (Brasil, EUA, Cripto, etc.)
- Ativo (acelera consultas por título específico)

---

## Camadas de Armazenamento

### Camada 1: Barramento de Eventos (Pub/Sub)
Núcleo do sistema. Retém mensagens por 7 dias em tópicos segregados por tipo de dado. Permite reprocessamento de dados recentes e garante nenhuma mensagem seja perdida durante falhas transitórias.

### Camada 2: Arquivo Histórico (Cloud Storage)
Cópia imutável de todos os eventos brutos. Organizados em hierarquia de partições: `topic/year/month/day/hour/*.parquet`. Retém 30 dias, serve como fonte de verdade para reprocessamentos maiores.

### Camada 3: Banco de Dados Transacional (BigQuery)
Destino final otimizado para leitura. Tabelas desnormalizadas por mercado/ativo, índices em timestamps e tickers. Tempo de resposta: < 5 segundos para consultas ad-hoc, < 100ms para dashboards em cache.

---

## Consumidores dos Dados

### Dashboards Analíticos
Ferramentas de visualização (Looker / Metabase) conectadas diretamente ao BigQuery com controle de acesso por perfil. Atualização em tempo real de cotações, volumes e indicadores macroeconômicos.

### API REST
Servidor FastAPI no Cloud Run expõe três categorias de endpoints:
- `/quotes/{ticker}` — Última cotação + histórico intraday
- `/ohlcv/{ticker}/{period}` — Barras de 15 min, horária, diária
- `/macroeconomics/{indicator}` — Séries macroeconômicas (Selic, IPCA, CPI, etc.)

Cache em memória (5 minutos) acelera consultas repetidas. Tempo de resposta típico: 200-500ms.

### Notebooks Científicos
Ambiente de análise exploratória (Vertex AI Workbench) com acesso SQL direto ao BigQuery. Usado para backtesting, desenvolvimento de modelos e pesquisa quantitativa.

---

## Reprocessamento Histórico (Replay): O Poder da Kappa

Um dos maiores diferenciais é a capacidade de reprocessar dados usando exatamente o mesmo código de transformação usado em tempo real.

**Cenários de uso:**
- Bug corrigido → Recalcular dados dos últimos 15 dias
- Nova feature (coluna) → Preencher retroativamente com lógica atual
- Outage ou falha → Recuperar dados perdidos até 30 dias atrás
- Mudança na lógica de enriquecimento → Atualizar todos os históricos

**Fluxo:** `Cloud Storage (arquivo) → Spark Job (mesmo código) → BigQuery (sobrescreve) → Validação automática`

Tempo típico: 10-30 minutos para reprocessar 7 dias de dados.

---

## Orquestração e Confiabilidade

**Cloud Composer 2** (Airflow gerenciado pelo GCP) orquestra a sinfonia completa:

```
EveryDay 00:00 ─── Scheduler ─── Coleta B3 ─┐
                                              ├─→ Validação QA ─→ Alert via Slack
EveryDay 12:00 ─── Scheduler ─── Coleta Bacen ┤
                                              │
Every 15min    ─── Scheduler ─── Coleta Forex ┘
```

**Responsabilidades:**
- Agendar coletores de dados em suas cadências
- Provisionar/desprovisionar clusters Spark sob demanda (reduz custo em 70%)
- Monitorar latência de eventos (alertar se fila > 30 minutos)
- Validar qualidade diária: volume mínimo, nulidade, duplicatas
- Disparar reprocessamentos automáticos em caso de anomalia

---

## Stack Tecnológico

| Layer | Ferramenta | Função |
|-------|-----------|--------|
| **Orquestração** | Cloud Composer 2 (Airflow) | Orquestra workflows, agendamento, retries |
| **Pub/Sub** | Google Cloud Pub/Sub | Barramento de eventos, 7d retenção |
| **Processamento** | Apache Spark + Dataproc | ETL, transformações, agregações |
| **Armazenamento Bruto** | Cloud Storage (Parquet) | Arquivo imutável, 30d |
| **Data Warehouse** | BigQuery | Queries ad-hoc, dashboards, análise |
| **API** | Cloud Run + FastAPI | REST endpoints, low-latency serving |
| **Segredos** | Secret Manager | Chaves de API, credenciais com rotação |
| **Monitoramento** | Cloud Logging + Cloud Monitoring | Traces distribuídos, alertas, SLOs |
| **Análise** | Vertex AI Workbench (Jupyter) | Environment para ciência de dados |
| **Visualização** | Looker + Metabase | Dashboards executivos |

---

## Status de Implementação

```
Fase 1: Infraestrutura
  [████████████░░░░░░░░] 60%
  ✓ Pub/Sub configurado
  ✓ BigQuery schemas definidos
  ○ Cloud Composer DAGs em refinamento

Fase 2: Coletores de Dados
  [░░░░░░░░░░░░░░░░░░░░] 0%
  ○ Yahoo Finance connector
  ○ B3 parser (CrawlerBot)
  ○ Alpha Vantage sync
  ○ CVM document ingestion

Fase 3: Motor de Transformação
  [░░░░░░░░░░░░░░░░░░░░] 0%
  ○ Spark transformers (dedup, norm)
  ○ Agregações OHLCV
  ○ Enriquecimento (metadados)
  ○ Tests unitários + integração

Fase 4: Serving e Qualidade
  [░░░░░░░░░░░░░░░░░░░░] 0%
  ○ API FastAPI endpoints
  ○ Cache em memória (Redis)
  ○ Dashboards Looker
  ○ SLO monitoring
```

---

## Princípios de Design

1. **Event-First** — Tudo é evento; imutabilidade é lei. Replay é garantido.
2. **Kappa sobre Lambda** — Um pipeline, dois cenários (tempo real + histórico).
3. **Data Quality como Feature** — Validação integrada, não uma etapa. Alertas automáticos.
4. **Infrastructure-as-Code** — Toda a infra versionada, reproduzível, idempotente.
5. **Observabilidade Extrema** — Logs distribuídos, traces, métricas em cada ponto.

---

## Como Contribuir

Nexus Apex é uma plataforma em desenvolvimento. Ideias, issues e PRs são bem-vindas.

**Roadmap futuro:**
- Machine learning pipeline para forecasting de preços
- Alertas inteligentes baseados em anomalias
- Suporte a blockchain e DeFi
- Integração com APIs de brokers para order routing

---

```
╔════════════════════════════════════════════════════════════════╗
║                    NEXUS APEX                                  ║
║         Onde os dados se elevam ao ápice da clareza            ║
║                                                                ║
║  _Ex chao ordo, in nexu veritas_                              ║
║  (Do caos nasce a ordem, na conexão habita a verdade)         ║
╚════════════════════════════════════════════════════════════════╝
```