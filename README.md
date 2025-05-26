# AbIntel Release v1.8

## 1. Executive Summary — “Why ABIntel?”

### Challenge vs Outcome

| **Challenge**                      | **ABIntel’s Outcome (Proven in Production)**                                                                                                            |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Disconnected silo data, no ML team | ✅ Plug-and-play micro-services that ingest raw transactions, invoices, tickets, sensor feeds, or free-text and deliver actionable predictions in < 60s. |
| Slow time-to-value                 | ✅ First campaign or cost project live in ≤ 2 hours; typical 6–14% EBIT uplift inside a fiscal quarter.                                                  |
| GPU expertise lacking              | ✅ Route switch to GPU or safe CPU fallback is automatic.                                                                                                |
| Hand-offs between Data & Business  | ✅ Every response contains English, Spanish, and Portuguese narratives so marketers, planners, and finance teams understand the “so what” instantly.     |

---

## 1.1 Platform Architecture at a Glance

* 🧩 Kubernetes autoscaling up to **32× T4s** (or **H100s in Q4 2025**)
* 🔐 SOC 2 Type 2 audit in progress; **ISO 27001** certified DCs
* 🚀 Zero-downtime **canary releases**
* 🏷️ Semantic versioning via `/v1`

---

## 1.2 API Server

```
https://api-prod.abintel.app
```

---

## 1.3 Queue a Task → `POST /ai/{downstream_route}`

### Example

```http
POST /api/v1/ai/forecast_revenue
```

#### Headers

```http
X-Customer-Api-Id: 4d09ffe5-33d7-4a63-b4fd-fc1decf4fa8f
X-Secret: AfIqb2F9JNET_…HzHUa0w
```

#### Body

```json
{
  "region": "EU",
  "horizon": 12,
  "include_promotions": true
}
```

### Under the Hood

1. Proxy checks your plan → queue mapping (`Business → tasks-medium`)
2. Verifies if downstream path is allowed
3. Task saved in **Cosmos DB** with `status: "queued"`
4. One `UsageRecord` debited
5. Job is enqueued to **Azure Service Bus**
6. Worker runs AI model → writes `status: "completed"` in Cosmos DB

### Response (HTTP 202 - Queued)

```json
{
  "task_id": "c7d1b5e76c67485eab1c9a407f7e6a97",
  "queue": "tasks-medium",
  "status": "queued"
}
```

> 💡 Tip: Everything you `POST` is stored exactly as given. You can send large, nested objects.

---

## 1.4 Check Task Status / Download Results

### 1.4.1 List Recent Jobs

```http
GET /api/v1/taskadmin/tasks?status_filter=completed&limit=20
```

Same authentication headers.

### 1.4.2 Fetch One Job

```http
GET /api/v1/taskadmin/tasks/{task_id}?part=response
```

#### Example – Success

```json
{
  "auc": 0.912,
  "predictions": [ ... ],
  "interpretation": "Model XGBoost – good separation."
}
```

#### Example – Failure

```json
{
  "error": "HTTP 500 Server Error: …",
  "status_code": 500
}
```

---

## 1.5 Live Queue & Worker Dashboards

| **Endpoint**                  | **Use-case**                                                        | **Example**                                       |
| ----------------------------- | ------------------------------------------------------------------- | ------------------------------------------------- |
| `GET /taskadmin/queue/stats`  | Summary – queued / completed / failed counts per queue + oldest job | `GET /api/v1/taskadmin/queue/stats`               |
| `GET /taskadmin/queue/queued` | Paginated dump for custom dashboards                                | `GET /api/v1/taskadmin/queue/queued?queue=medium` |

> 🧠 Both endpoints query Cosmos DB – no worker load.

---

## 1.6 Usage / Billing APIs

| Endpoint                       | Description                        |
| ------------------------------ | ---------------------------------- |
| `GET /taskadmin/usage/month`   | Current month call count & cost    |
| `GET /taskadmin/usage/history` | Historical usage and cost by month |

#### Example Response

```json
{
  "month": "2025-05",
  "calls": 842,
  "cost": 42.10
}
```

> 💰 Costs are derived from `UsageRecord.cost`. You can define per-route pricing.

---

## 1.7 Typical Lifecycle

```mermaid
sequenceDiagram
    Customer ->> Proxy /ai/... : POST payload
    Proxy ->> Cosmos DB : Save task (status="queued")
    Proxy ->> Azure Service Bus : Enqueue task
    Celery Worker ->> Azure Bus : Poll
    Worker ->> AI Backend : POST for processing
    Worker ->> Cosmos DB : Save (status="completed" + response)
    Customer ->> /taskadmin/tasks/{id}?part=response : Fetch result
```

---

## 1.8 Error Handling Cheatsheet

| **Status**  | **Meaning**            | **Next Step**                             |
| ----------- | ---------------------- | ----------------------------------------- |
| `queued`    | Awaiting a free worker | Check `/queue/stats` for backlog          |
| `completed` | Result ready           | Download it                               |
| `failed`    | Error or timeout       | Inspect `error` field; retry if transient |

---

## 1.9 FAQ

* **How long do results stay?**
  ⏳ 30 days (default)

* **Can I cancel a job?**
  ❌ Not yet – open a support ticket

* **Timeouts?**
  ⏱️ 60s hard timeout; chunk large jobs

* **Security?**
  🔐 TLS 1.2 for all data in transit
  🔒 Secrets hashed at rest

---
