# 4.4 Inventario y Cadena de Suministro
+> Nota: traducción generada automáticamente desde el inglés; revisar antes de publicar.

**Inventory & Supply-Chain**

**Endpoints:**
- `inventory_history_improved`
- `inventory_optimization`
- `nlp_openai_excess_inventory_report`
- `excess_inventory_nlp`
- `nlp_analisys`
- `nlp_analisys_en`

---

## 4.4.1 /api/v1/ai/inventory_history_improved
**Daily stock ledger + 360-degree inventory analytics**

**Purpose** A single endpoint that converts raw purchases & sales into a rich inventory ledger, full KPI deck and aging dashboard.

### Headers `X-Customer-Api-Id`, `X-Secret`

```yml
POST https://api.abintel.ai/api/v1/ai/inventory_history_improved
Headers:
```json
X-Customer-Api-Id: <uuid>
X-Secret: <secret>
Content-Type: application/json
```

---

### What pain-point it fixes

| Operational headache                       | How the endpoint helps                                                                |
| ------------------------------------------ | ------------------------------------------------------------------------------------- |
| Need a day-by-day ledger for audit & BI    | `daily_inventory[]` rolls every purchase & sale into a running balance with %-changes |
| Finance wants turnover, DOS, volatility…   | `inventory_analysis{}` delivers 30+ KPIs with plain-English explanations              |
| Struggle to spot aging / obsolescence risk | `inventory_aging{}` splits lots into sold vs unsold with aging & risk buckets         |
| Need an all-in margin view                 | Price, cost & daily carrying cost feed margin %, EOQ, price sensitivity               |

---

### Payload reference

| Path                             | Type            | Req? | Purpose                   | Notes                       |
| -------------------------------- | --------------- | ---- | ------------------------- | --------------------------- |
| product_code                     | string / number | ✔︎   | SKU / article code        | returned verbatim           |
| sales_unity_cost                 | number          | ✔︎   | Unit COGS                 | currency-agnostic           |
| sales_unity_price                | number          | ✔︎   | Unit sell price           | used for margin, elasticity |
| inventory_daily_cost             | number          | ✔︎   | Holding cost / unit / day | for EOQ & efficiency flags  |
| lead_time_days                   | number          | ✔︎   | Replenishment lead time   | safety-stock calc           |
| inventory_initial_situation[]    | array           | ✔︎   | Opening stock snapshot    | exactly one row             |
| purchases[]                      | array           | ✔︎   | GRNs during the horizon   | any order                   |
| ↳ purchase_date                  | YYYY-MM-DD      | ✔︎   |                           |                             |
| ↳ purchase_quantity              | number          | ✔︎   | units                     |                             |
| sales[]                          | array           | ✔︎   | Shipments / invoices      | any order                   |
| ↳ invoice_date                   | YYYY-MM-DD      | ✔︎   |                           |                             |
| ↳ sales_quantity                 | number          | ✔︎   | units                     |                             |

---

### Minimal working example

```json
{
  "product_code": 38788,
  "sales_unity_cost": 25.50,
  "sales_unity_price": 60.73,
  "inventory_daily_cost": 0.15,
  "lead_time_days": 7.0,
  "inventory_initial_situation": [
    { "inventory_data": "2023-12-31", "inventory_quantity": 1050.0 }
  ],
  "purchases": [
    { "purchase_date": "2024-01-31", "purchase_quantity": 280.0 },
    { "purchase_date": "2024-02-28", "purchase_quantity": 240.0 },
    { "purchase_date": "2024-03-29", "purchase_quantity": 320.0 }
  ],
  "sales": [
    { "invoice_date": "2024-01-02", "sales_quantity":  5.0 },
    ⋯
    { "invoice_date": "2024-03-15", "sales_quantity": 31.0 }
  ]
}
```

---

### How the engine works (under the hood)

* Calendar scaffold – build a dense date index from min() to max(); back-fill zeros.

* Ledger build – on each day: net_change = purchase – sales, closing_inventory = prior + net_change.

* Activity tagging – No Activity, Only Purchase, Only Sales, Purchase & Sales.

* Stats & classifications – turnover, DOS, volatility, safety-stock, ABC, lifecycle, seasonality.

* Lot ageing – FIFO consume purchases, track days until sold; flag unsold lots >90 days.

* Elasticity & price sensitivity – simple OLS on price vs qty (if price series varied).

---

### Canonical response (excerpt)

```json
{
  "daily_inventory": [
    {
      "date": "2024-01-31",
      "purchase_quantity": 280,
      "sales_quantity": 0,
      "net_change": 280,
      "closing_inventory": 1216,
      "activity": "Only Purchase",
      "pct_change": 29.91
    },
    ⋯
  ],
  "inventory_analysis": {
    "initial_inventory": { "value": 1050, "explanation": "…" },
    "cumulative_purchases": { "value": 840, "explanation": "…" },
    "cumulative_sales": { "value": 721, "explanation": "…" },
    "turnover_ratio_initial": { "value": 0.687, "turnover_description": "Very Low Turnover" },
    "inventory_days_of_supply": { "value": 123.97, "days_supply_description": "Very Sluggish Inventory" },
    ⋯
  },
  "inventory_aging": {
    "sold_vs_unsold": {
      "sold_lots": {
        "summary": {
          "average_aging_days": 22,
          "std_aging_days": 12.73
        }
      },
      "unsold_lots": {
        "summary": { "weighted_average_current_age": 0 }
      }
    }
  },
  "product_details": {
    "product_code": 38788,
    "sales_unity_cost": 25.5,
    "sales_unity_price": 60.73,
    "inventory_daily_cost": 0.15
  },
  "processing_info": {
    "processing_time_seconds": 0.026,
    "calculation_date": "2025-05-21"
  }
}
```

---

### Reading the numbers — quick insights for this sample

| Insight                | Metric                  | Why it matters                                      |
| ---------------------- | ----------------------- | --------------------------------------------------- |
| Excess-inventory alert | DOS = 124 days          | Capital locked; consider promo / PO deferral.       |
| Safety stock needed    | 42.7 units              | Cover 7-day lead time with demand σ ≈ 9.8.          |
| Low turnover           | 0.69× initial stock     | SKU is slow-moving → review range or price.         |
| Lot ageing healthy     | All sold within 31 days | No obsolescence yet; monitor the 320-unit lot.      |
| Price sensitivity      | CV margin ≈ 0 → Low     | Room for small price rises with minimal demand hit. |

---

### Core flow metrics
(See full list of KPIs in your source.)

---

### Next step

Feed `daily_inventory` into your BI layer for real-time stock dashboards.

Trigger alerts:
if `inventory_days_of_supply > 90` and `excess_inventory_risk = High` → auto-suggest clearance promo.

Loop with forecasting: pair this ledger with `/api/forecast_units` to simulate future stock-outs & reorder points.

---

