# AbIntel API Overview

AbIntel delivers plug-and-play micro-services that convert raw transactions, invoices, tickets, sensor feeds, or free-text into predictions in under a minute, helping teams launch projects quickly without dedicated ML expertise.

## Platform Snapshot
* Kubernetes autoscaling up to 32× T4s with optional H100 support
* SOC 2 Type 2 audit in progress with ISO 27001 certified data centers
* Zero-downtime canary releases
* Semantic versioning via `/v1`

API base URL: `https://api-prod.abintel.app`

## Authentication
Every request must include the headers:
- `X-Customer-Api-Id`: tenant UUID
- `X-Secret`: 64‑character secret rotated every 90 days
- `Accept-Language`: response language (`en`, `es`, `pt`)

## Queue a Task
`POST /api/v1/ai/{route}` enqueues an AI job. For example:
```
POST /api/v1/ai/forecast_revenue
```
The body can contain any JSON payload and is stored exactly as provided. The service saves the task in Cosmos DB, debits a `UsageRecord`, and pushes the job to Azure Service Bus. Workers run the model and update the task status when complete.

## Check Task Status
- `GET /api/v1/taskadmin/tasks?status_filter=completed&limit=20`
- `GET /api/v1/taskadmin/tasks/{task_id}?part=response`

Both endpoints use the same authentication headers and return either the model output or an error message.

## Dashboards and Usage
- `GET /api/v1/taskadmin/queue/stats`
- `GET /api/v1/taskadmin/queue/queued?queue=medium`
- `GET /api/v1/taskadmin/usage/month`
- `GET /api/v1/taskadmin/usage/history`

These endpoints read from Cosmos DB to show queue health and track monthly call counts and costs.

## Typical Lifecycle
1. Client posts payload to `/ai/...`
2. Proxy stores task and enqueues it
3. Worker processes the job and saves the response
4. Client retrieves the result via `/taskadmin/tasks/{id}?part=response`

Common statuses are `queued`, `completed`, and `failed`.

## Route Catalogue
- **Customer Intelligence**: purchasing segmentation, loyalty scoring, churn risk, and more
- **Propensity & Recommendation**: buy/upgrade propensity, item recommendations, cross-sell, uplift modeling
- **Forecasting & Planning**: revenue and unit forecasts, cost projections
- **Inventory & Supply-Chain**: optimization, excess inventory analysis, NLP reports
- **Cost & Risk Analytics**: credit risk, channel attribution, anomaly detection, sentiment analysis

