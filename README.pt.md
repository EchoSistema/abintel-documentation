# Visão Geral da API AbIntel

A AbIntel fornece microsserviços plug-and-play que transformam transações, faturas, tickets, feeds de sensores ou texto livre em previsões em menos de um minuto, permitindo lançamentos rápidos sem equipe de ML.

## Visão da Plataforma
* Escalonamento automático em Kubernetes até 32 T4s, com suporte opcional a H100
* Auditoria SOC 2 Tipo 2 em andamento e datacenters certificados ISO 27001
* Lançamentos canário sem downtime
* Versionamento semântico via `/v1`

URL base da API: `https://api-prod.abintel.app`

## Autenticação
Todas as requisições devem incluir:
- `X-Customer-Api-Id`: UUID do cliente
- `X-Secret`: segredo de 64 caracteres rotacionado a cada 90 dias
- `Accept-Language`: idioma da resposta (`en`, `es`, `pt`)

## Enfileirar uma Tarefa
`POST /api/v1/ai/{route}` enfileira um job de IA. Exemplo:
```
POST /api/v1/ai/forecast_revenue
```
O corpo pode conter qualquer JSON e é armazenado como enviado. O serviço salva a tarefa no Cosmos DB, debita um `UsageRecord` e envia o job para o Azure Service Bus. Os workers executam o modelo e atualizam o status ao concluir.

## Consultar o Status
- `GET /api/v1/taskadmin/tasks?status_filter=completed&limit=20`
- `GET /api/v1/taskadmin/tasks/{task_id}?part=response`

Ambos usam os mesmos cabeçalhos de autenticação e retornam a saída do modelo ou mensagem de erro.

## Dashboards e Uso
- `GET /api/v1/taskadmin/queue/stats`
- `GET /api/v1/taskadmin/queue/queued?queue=medium`
- `GET /api/v1/taskadmin/usage/month`
- `GET /api/v1/taskadmin/usage/history`

Esses endpoints consultam o Cosmos DB para mostrar a saúde das filas e acompanhar chamadas e custos mensais.

## Ciclo Típico
1. Cliente envia payload para `/ai/...`
2. Proxy salva e enfileira a tarefa
3. Worker processa o job e grava a resposta
4. Cliente busca o resultado via `/taskadmin/tasks/{id}?part=response`

Os status comuns são `queued`, `completed` e `failed`.

## Catálogo de Rotas
- **Customer Intelligence**: segmentação de compras, fidelidade, risco de churn e mais
- **Propensity & Recommendation**: propensão de compra/upgrade, recomendações, cross-sell, uplift modeling
- **Forecasting & Planning**: previsões de receita e unidades, projeções de custo
- **Inventory & Supply-Chain**: otimização, análise de excesso de estoque, relatórios NLP
- **Cost & Risk Analytics**: risco de crédito, atribuição de canal, detecção de anomalias, análise de sentimento

