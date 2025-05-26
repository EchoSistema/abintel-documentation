4.1 Customer Intelligence

Customer Intelligence endpoints:
purchasing_segmentation*, segmentation_report*, segment_hierarchy_chart, segment_subsegment_explore, segment_cluster_profiles, customer_segmentation, customer_features, customer_loyalty, customer_rfm, customer_clv_features, customer_clv_forecast, churn_risk, nps, churn_label

4.1.1 /api/v1/ai/purchasing_segmentation
Purpose
Autonomously cluster customers; returns labels, centroids, KPIs, and plain-language personas.

Headers
X-Customer-Api-Id: UUID

X-Secret: Secret

Content-Type: application/json

Request
POST https://api.abintel.ai/api/v1/ai/purchasing_segmentation

Headers:
X-Customer-Api-Id: <uuid>
X-Secret: <secret>
Content-Type: application/json

Response Body:
{
  "max_clusters": 10,
  "dbscan_eps": 0.40,
  "dbscan_min_samples": 5,
  "data": [
    {
      "customer_id": "C-1001",
      "features": {
        "purchase_total": 17850.55,
        "visit_frequency": 62,
        "avg_ticket": 287.27,
        "product_diversity": 36,
        "loyalty_score": 0.97,
        "recency_days": 3
      }
    },
    ...
  ]
}


Request Body Schema
| JSON Key             | Type              | Required | Description                                       | Notes/Formula                                    |
| -------------------- | ----------------- | -------- | ------------------------------------------------- | ------------------------------------------------ |
| `max_clusters`       | integer ≥ 2       | optional | Upper bound for K-Means & Gaussian Mixture models | Recommend 8–15 for 5k–200k customers             |
| `dbscan_eps`         | float (0 < ε ≤ 1) | optional | Radius for DBSCAN after z-score scaling           | Start at 0.35–0.45                               |
| `dbscan_min_samples` | integer ≥ 3       | optional | Minimum neighbors for DBSCAN core point           | Use 5 (default), adjust for small/large datasets |
| `data`               | array<object>     | required | List of customer objects with features            | Usually from 12–18 months of transaction data    |


Features Inside data[].features
| Feature             | Type/Range  | Business Definition                            | SQL/Python Example                                       |
| ------------------- | ----------- | ---------------------------------------------- | -------------------------------------------------------- |
| `purchase_total`    | float ≥ 0   | Lifetime revenue in observation window         | `SUM(order_total)`                                       |
| `visit_frequency`   | integer ≥ 0 | Number of distinct purchase sessions           | `COUNT(DISTINCT order_date)`                             |
| `avg_ticket`        | float ≥ 0   | Average value per purchase                     | `purchase_total / NULLIF(visit_frequency,0)`             |
| `product_diversity` | integer ≥ 0 | Count of distinct product categories purchased | `COUNT(DISTINCT category_lvl1)`                          |
| `loyalty_score`     | float (0–1) | Normalized loyalty score from points or RFM    | `current_points / max_points_cap` or RFM z-score average |
| `recency_days`      | integer ≥ 0 | Days since last order                          | `DATEDIFF(CURRENT_DATE, MAX(order_date))`                |


Practical data-engineering recipe:

WITH base AS (
  SELECT
    o.customer_id,
    SUM(o.order_total) AS purchase_total,
    COUNT(DISTINCT o.order_date) AS visit_frequency,
    AVG(o.order_total) AS avg_ticket,
    COUNT(DISTINCT p.category_lvl1) AS product_diversity,
    MAX(o.order_date) AS last_order_date
  FROM orders o
  JOIN products p ON p.product_id = o.product_id
  WHERE o.order_date BETWEEN DATE_SUB(CURDATE(), INTERVAL 18 MONTH) AND CURDATE()
  GROUP BY o.customer_id
),
enriched AS (
  SELECT
    b.*,
    l.points_balance / 10000 AS loyalty_score,
    DATEDIFF(CURDATE(), b.last_order_date) AS recency_days
  FROM base b
  LEFT JOIN loyalty l ON l.customer_id = b.customer_id
)
SELECT
  JSON_OBJECT(
    'customer_id', customer_id,
    'features', JSON_OBJECT(
      'purchase_total', purchase_total,
      'visit_frequency', visit_frequency,
      'avg_ticket', avg_ticket,
      'product_diversity', product_diversity,
      'loyalty_score', ROUND(loyalty_score,2),
      'recency_days', recency_days
    )
  ) AS customer_json
FROM enriched;


Suggested Hyperparameters
| Dataset Size       | `max_clusters` | `dbscan_eps` | `dbscan_min_samples` |
| ------------------ | -------------- | ------------ | -------------------- |
| ≤ 10k customers    | 6–10           | 0.45         | 4                    |
| 10k–250k customers | 8–15           | 0.40         | 5–7                  |
| > 250k customers   | 12–25          | 0.35         | 7–10                 |

After first run, if silhouette < 0.35 or most points fall into one DBSCAN cluster:

↓ eps by 0.05

↑ max_clusters by 2–3


Why Each Feature Matters (Business Value)
purchase_total / avg_ticket – Identify whales vs. bargain shoppers

visit_frequency / recency_days – Loyalty insight (dormant vs. active)

product_diversity – Capture breadth shoppers (ideal for bundles)

loyalty_score – Capture engagement not reflected in transactions

These 6 features explain ~80% of the variance in over 30 live retailers.

Response:
{
    "best_algorithm": "Agglomerative",
    "evaluation_metrics": [
        {
            "algorithm": "KMeans",
            "params": {
                "k": 2
            },
            "silhouette": 0.6187929992103806,
            "davies_bouldin": 0.4707969457479151,
            "calinski_harabasz": 51.66382341597987,
            "n_clusters": 2
        },
        {
            "algorithm": "GaussianMixture",
            "params": {
                "k": 2
            },
            "silhouette": 0.6058502844512947,
            "davies_bouldin": 0.49239043864015747,
            "calinski_harabasz": 53.176054388614624,
            "n_clusters": 2
        },
        {
            "algorithm": "Agglomerative",
            "params": {
                "k": 2
            },
            "silhouette": 0.620792324066691,
            "davies_bouldin": 0.47322641012239497,
            "calinski_harabasz": 49.74563679456432,
            "n_clusters": 2
        }
    ],
    "clusters": [
        {
            "cluster_id": 0,
            "size": 10,
            "centroid": {
                "purchase_total": 13469.446,
                "visit_frequency": 45.2,
                "avg_ticket": 260.005,
                "product_diversity": 27.9,
                "loyalty_score": 0.881,
                "recency_days": 9.8
            },
            "persona_label": "high_purchase_total, high_visit_frequency, high_product_diversity, high_loyalty_score, low_recency_days"
        },
        {
            "cluster_id": 1,
            "size": 10,
            "centroid": {
                "purchase_total": 1613.724,
                "visit_frequency": 8.1,
                "avg_ticket": 144.242,
                "product_diversity": 6.8,
                "loyalty_score": 0.311,
                "recency_days": 190.2
            },
            "persona_label": "low_purchase_total, low_visit_frequency, low_product_diversity, low_loyalty_score, high_recency_days"
        }
    ],
    "customer_labels": [
        {
            "customer_id": "C-1001",
            "cluster_id": 0
        },
        {
            "customer_id": "C-1002",
            "cluster_id": 0
        },
        {
            "customer_id": "C-1003",
            "cluster_id": 0
        },
        {
            "customer_id": "C-1004",
            "cluster_id": 1
        },
        {
            "customer_id": "C-1005",
            "cluster_id": 1
        },
        {
            "customer_id": "C-1006",
            "cluster_id": 1
        },
        {
            "customer_id": "C-1007",
            "cluster_id": 0
        },
        {
            "customer_id": "C-1008",
            "cluster_id": 0
        },
        {
            "customer_id": "C-1009",
            "cluster_id": 1
        },
        {
            "customer_id": "C-1010",
            "cluster_id": 1
        },
        {
            "customer_id": "C-1011",
            "cluster_id": 1
        },
        {
            "customer_id": "C-1012",
            "cluster_id": 0
        },
        {
            "customer_id": "C-1013",
            "cluster_id": 0
        },
        {
            "customer_id": "C-1014",
            "cluster_id": 0
        },
        {
            "customer_id": "C-1015",
            "cluster_id": 1
        },
        {
            "customer_id": "C-1016",
            "cluster_id": 1
        },
        {
            "customer_id": "C-1017",
            "cluster_id": 0
        },
        {
            "customer_id": "C-1018",
            "cluster_id": 1
        },
        {
            "customer_id": "C-1019",
            "cluster_id": 1
        },
        {
            "customer_id": "C-1020",
            "cluster_id": 0
        }
    ]
}

| Top-level Key        | Type          | Meaning                                                                                              | How to use
| -------------------- | ------------- | ---------------------------------------------------------------------------------------------------- | 
| `best_algorithm`     | string        | Name of the model the service judged statistically best (highest Silhouette, lowest Davies-Bouldin). | Store with run-ID for A/B testing or to force consistent algorithm use
| -------------------- | ------------- | ---------------------------------------------------------------------------------------------------- | in future runs.
| `evaluation_metrics` | array<object> | One entry per algorithm/parameter combination benchmarked.                                           | Use in ML-Ops dashboards; rerun with tighter params if metrics are
| -------------------- | ------------- | ---------------------------------------------------------------------------------------------------- | close.
| `clusters`           | array<object> | Summary of each cluster: ID, size, feature centroids, and machine-generated persona label.           | Use for persona naming, campaigns, assortment, loyalty tiers,
| -------------------- | ------------- | ---------------------------------------------------------------------------------------------------- | etc.
| `customer_labels`    | array<object> | Mapping of each `customer_id` to a `cluster_id`.                                                     | Join back to CRM/CDP to create audiences, train models, and measure KPI
| -------------------- | ------------- | ---------------------------------------------------------------------------------------------------- | lifts.

evaluation_metrics – what the scores mean:

| Field               | What it is                                        | Good Range                | Interpretation for This Run                                 |
| ------------------- | ------------------------------------------------- | ------------------------- | ----------------------------------------------------------- |
| `silhouette`        | 0–1 cohesion/separation measure. Higher = better. | ≥ 0.50 good, ≥ 0.60 great | 0.6208 on Agglomerative ⇒ clusters are well-separated.      |
| `davies_bouldin`    | Lower is better; typical range: 0–2               | ≤ 0.60 good, ≤ 0.50 great | 0.47 ⇒ low intra-cluster spread vs. inter-cluster distance. |
| `calinski_harabasz` | Higher is better; unbounded                       | > 40 often solid          | 49–53 across algorithms ⇒ consistent compactness.           |
| `n_clusters`        | Number of clusters produced                       | —                         | All algorithms converged on 2 clusters.                     |

Takeaway: The three algorithms had similar performance, but Agglomerative edged ahead.
Two clusters are statistically justified; forcing more may not reveal meaningful differences.

clusters – who’s who in your customer base
Cluster 0 — High-value loyalists

| Metric              | Centroid Value | Benchmark vs Cluster 1 | Commercial Name   |
| ------------------- | -------------- | ---------------------- | ----------------- |
| `Size`              | 10 customers   | n/a                    | "Champions"       |
| `purchase_total`    | 13,469 R\$     | 8× higher              | Premium spenders  |
| `visit_frequency`   | 45 orders      | 5.6× higher            | Habitual shoppers |
| `avg_ticket`        | 260 R\$        | 80% higher             | Big baskets       |
| `product_diversity` | 28 categories  | 4.1× broader           | Explorers         |
| `loyalty_score`     | 0.88           | 3× higher              | Near top-tier     |
| `recency_days`      | ≈ 10 days      | 95% fresher            | Highly active     |

Persona label:
high_purchase_total, high_visit_frequency, high_product_diversity, high_loyalty_score, low_recency_days

Actionable Playbook:

Exclusive early-access & VIP previews

Bundle-builder email with 3–5 high-margin cross-sells

Accelerated points to lock-in retention

Cluster 1 — Low-value, lapsed:

| Metric              | Centroid Value          | Notes |
| ------------------- | ----------------------- | ----- |
| `Size`              | 10 customers            |       |
| `purchase_total`    | 1,614 R\$ (−88%)        |       |
| `visit_frequency`   | 8 orders (−82%)         |       |
| `product_diversity` | 7 categories (−75%)     |       |
| `loyalty_score`     | 0.31 (−65%)             |       |
| `recency_days`      | ≈ 190 days (19× longer) |       |

Persona label:
low_purchase_total, low_visit_frequency, low_product_diversity, low_loyalty_score, high_recency_days

Actionable Playbook:

Win-back email with single, category-specific offer

Reactivation SMS coupon (valid 48h)

Suppress from premium ad audiences to protect ROAS


customer_labels – operationalising:

SELECT c.*, s.cluster_id
FROM crm.customers c
JOIN api_segmentation_labels s
  ON s.customer_id = c.customer_id;


Segment Usage Examples:

Campaign Tool: Push list of Cluster 0 to WhatsApp broadcast

Web Personalisation: If cluster_id = 1, show a pop-up: “What would bring you back?”

BI Dashboard: Run margin trend by cluster to monitor long-term customer health

Quick FAQ:

| Question                                         | Answer                                                                   |
| ------------------------------------------------ | ------------------------------------------------------------------------ |
| Silhouette is only 0.62 – is that “good” enough? | Yes. Anything > 0.60 is considered strong separation in behavioral data. |
| Can I get three clusters instead of two?         | Re-run with `max_clusters ≥ 3` and add more mid-value customers.         |
| Are centroids raw or scaled values?              | They are rescaled back to original units (currency, days, etc).          |
| How often should I re-segment?                   | Monthly is typical; weekly for high-volume e-commerce.                   |


4.1.2 /api/v1/ai/customer_features:

Purpose Generate a clean, analytically-ready
feature vector for every customer by rolling up raw transaction
rows (and any supplied loyalty metadata). This is the canonical preprocessing
step for segmentation, churn, CLV or any bespoke ML model you want to train yourself.\

Headers X-Customer-Api-Id, X-Secret:

| Header              | Value              |
| ------------------- | ------------------ |
| `X-Customer-Api-Id` | `<uuid>`           |
| `X-Secret`          | `<secret>`         |
| `Content-Type`      | `application/json` |

Request
POST https://api-prod.abintel.ai/api/v1/ai/customer_features

Request Body:

{
  "today": "2025-04-24",
  "customers": [
    {
      "customer_id": "C-1001",
      "loyalty_score": 0.97,
      "transactions": [
        { "transaction_id": "T-001", "date": "2025-04-21", "amount": 520.55, "product_id": "P-01" },
        { "transaction_id": "T-002", "date": "2025-04-18", "amount": 398.40, "product_id": "P-02" },
        { "transaction_id": "T-003", "date": "2025-03-29", "amount": 310.00, "product_id": "P-01" }
      ]
    },
    {
      "customer_id": "C-1002",
      "transactions": [
        { "transaction_id": "T-004", "date": "2025-01-05", "amount": 120.00, "product_id": "P-10" },
        { "transaction_id": "T-005", "date": "2024-12-19", "amount": 75.90,  "product_id": "P-11" }
      ]
    }
  ]
}

Request Body Schema (field-by-field):

| JSON Key           | Type / Range          | Required     | Meaning                          | How to obtain / SQL formula                        |
| ------------------ | --------------------- | ------------ | -------------------------------- | -------------------------------------------------- |
| `today`            | ISO date (YYYY-MM-DD) | Optional     | Reference date for recency       | `CURRENT_DATE` or pipeline parameter               |
| `customers`        | array<object>         | Yes          | Customer snapshot list           | From daily ETL query                               |
| ↳ `customer_id`    | string                | Yes (unique) | CRM/ERP primary key              | `orders.customer_id`                               |
| ↳ `loyalty_score`  | float (0–1)           | Optional     | Precomputed loyalty or RFM score | `points / max_points_cap` or `(z_R + z_F + z_M)/3` |
| ↳ `transactions`   | array<object>         | Yes (≥1)     | Raw transaction data             | Use 12–18 months of history                        |
| ↳ `transaction_id` | string                | Yes          | Order ID                         | `orders.order_id`                                  |
| ↳ `date`           | ISO date              | Yes          | Transaction date                 | `orders.order_date`                                |
| ↳ `amount`         | float ≥ 0             | Yes          | Order total                      | `orders.order_total`                               |
| ↳ `product_id`     | string                | Yes          | SKU or category ID               | `order_items.product_id`                           |

What the Endpoint Calculates:

| Feature Key         | Formula                      | Business Interpretation                    |
| ------------------- | ---------------------------- | ------------------------------------------ |
| `purchase_total`    | `SUM(amount)`                | Lifetime spend (GMV)                       |
| `visit_frequency`   | `COUNT(transactions)`        | Number of purchase occasions               |
| `avg_ticket`        | `purchase_total / visits`    | Average basket size                        |
| `product_diversity` | `COUNT(DISTINCT product_id)` | Breadth of purchase (cross-sell potential) |
| `loyalty_score`     | passed or `0.0`              | Loyalty indicator                          |
| `recency_days`      | `DATEDIFF(today, MAX(date))` | Time since last purchase                   |

All math is done exactly as above; no additional scaling is applied in this endpoint.

Response:

{
  "customers": [
    {
      "customer_id": "C-1001",
      "features": {
        "purchase_total": 1228.95,
        "visit_frequency": 3.0,
        "avg_ticket": 409.65,
        "product_diversity": 2.0,
        "loyalty_score": 0.97,
        "recency_days": 3.0
      }
    },
    {
      "customer_id": "C-1002",
      "features": {
        "purchase_total": 195.9,
        "visit_frequency": 2.0,
        "avg_ticket": 97.95,
        "product_diversity": 2.0,
        "loyalty_score": 0.0,
        "recency_days": 109.0
      }
    }
  ]
}


Response Keys Explained:

| Key Path        | Type   | Meaning                 | Next-Step Usage                                             |
| --------------- | ------ | ----------------------- | ----------------------------------------------------------- |
| `customers[]`   | array  | Each entry = 1 customer | Enviar para `/purchasing_segmentation`, `/churn_risk`, etc. |
| ↳ `customer_id` | string | Ecoa o ID enviado       | Para joins com CRM/CDP                                      |
| ↳ `features`    | object | Vetor de 6 KPIs         | Armazenar em feature store ou view no banco                 |

Why those six features?

They capture monetary, frequency, breadth, loyalty and time dimensions—covering > 80 % of purchase-behaviour variance observed across 30+ retailers.

They’re model-agnostic: the same vector feeds K-Means, XGBoost, or deep nets.

Commercial Playbooks Enabled

| Use-Case                    | How to exploit the features                                                                          |
| --------------------------- | ---------------------------------------------------------------------------------------------------- |
| **Rapid segmentation**      | Pass this response unmodified to  `/purchasing_segmentation`                                         |
| **LTV & CAC optimisation**  | Feed directly into `/customer_clv_forecast`; adjust acquisition bids by predicted CLV quintile.      |
| **Personalised email**      | Drive template logic:`if recency_days ≤ 7`  show “Welcome back”, else show “We miss you!”.           |
| **Loyalty programme**       | Upgrade tier when `loyalty_score > 0.8`and `product_diversity ≥ 10`.                                 |


Best-Practice ETL Pipeline:

Nightly DBT model ─→  Raw orders  ─┐
                                  ├─>  view_customer_transactions (as JSON)
CRM loyalty tiers ───────────────┘
     │
     ▼
dbt run-operation customer_features_export
     │ (escreve JSON no S3 ou tmp)
     ▼
invoke lambda → POST /api/customer_features
     │ (recebe feature vectors)
     ▼
Snowflake table: customer_features_today
     ▼
downstream ML jobs, BI, campanhas de marketing


FAQ:

| Question                                 | Answer                                                                                                                                                      |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Do I need to send `loyalty_score`?       | No; omit and the API sets it to `0.0`.                                                                                                                      |
| What if I use different currencies?      | Convert to a single currency before posting, or add `currency` in each transaction and set `currency_conversion_table` in the header (pro feature Q3 2025). |
| Can I add more custom features?          | Yes—any extra numeric field inside features will be echoed back `unchanged`; downstream routes that understand new keys will use them, others will ignore.  |
| Maximum payload size?                    | 20 MB per request (~60 k customers). Larger batches: chunk or use upcoming async bulk endpoint `/customer_features_async`.                                  |


4.1.3 /api/v1/ai/customer_loyalty

Purpose
Calculate a 0–1 loyalty index for every customer using weighted behavioral metrics, then bucket them into business-friendly segments.
Default tiers: Platinum, Gold, Silver, Bronze, At-Risk.

Headers:

| Header              | Value              |
| ------------------- | ------------------ |
| `X-Customer-Api-Id` | `<uuid>`           |
| `X-Secret`          | `<secret>`         |
| `Content-Type`      | `application/json` |


Request
POST https://api-prod.abintel.ai/api/v1/ai/customer_features

Request Body Schema:

| JSON Key                                                                                 | Type / Range                          | Required  | Meaning                          | How to Pick / Compute                                                          |
| ---------------------------------------------------------------------------------------- | ------------------------------------- | --------- | -------------------------------- | ------------------------------------------------------------------------------ |
| `weights`                                                                                | object of 5 floats (0–1) summing to 1 | Optional  | Importance of each metric        | Align to business goals (e.g. increase `recency_days` weight for reactivation) |
| → `purchase_total`, `visit_frequency`, `avg_ticket`, `product_diversity`, `recency_days` | float                                 | See above | Individual weights               | Usually ensure `purchase_total + visit_frequency ≥ 0.5`                        |
| `customers`                                                                              | array<object>                         | Yes       | List of customers and their KPIs | Direct output from `/api/customer_features`                                    |
| → `customer_id`                                                                          | string                                | Yes       | Unique customer key              | `crm.customers.id`                                                             |
| → `features`                                                                             | object                                | Yes       | Dictionary of KPIs               | See `/customer_features` derivations                                           |

🔍 Normalization logic: Each metric is min-max scaled across the batch before weights are applied. Scores are relative to the posted cohort.

Request Example:

{
  "weights": {
    "purchase_total": 0.25,
    "visit_frequency": 0.30,
    "avg_ticket": 0.10,
    "product_diversity": 0.15,
    "recency_days": 0.20
  },
  "customers": [
    {
      "customer_id": "C-1001",
      "features": {
        "purchase_total": 17850.55,
        "visit_frequency": 62,
        "avg_ticket": 287.27,
        "product_diversity": 36,
        "recency_days": 3
      }
    }
  ]
}

Response Example:

{
  "customers": [
    { "customer_id": "C-1001", "loyalty_score": 1.0,    "segment": "Platinum" },
    { "customer_id": "C-1002", "loyalty_score": 0.8705, "segment": "Platinum" },
    { "customer_id": "C-1003", "loyalty_score": 0.5348, "segment": "Silver"  },
    { "customer_id": "C-1004", "loyalty_score": 0.0,    "segment": "At-Risk" }
  ],
  "summary": {
    "mean_score": 0.6013,
    "median_score": 0.7026,
    "top_decile_threshold": 0.9611,
    "segment_counts": {
      "Platinum": 2,
      "Silver":   1,
      "At-Risk":  1
    }
  }
}

Response Keys:
| Key Path                       | Type        | Meaning                                   | How to Use                                                   |
| ------------------------------ | ----------- | ----------------------------------------- | ------------------------------------------------------------ |
| `customers[].loyalty_score`    | float (0–1) | Min-max weighted loyalty index (1 = best) | Feed into CRM tiers, bid multipliers, or DWH export          |
| `customers[].segment`          | enum        | Default segment tier                      | Override via `segment_thresholds` param (Q3), or remap in UI |
| `summary.mean_score`           | float       | Average score across cohort               | Track monthly loyalty health                                 |
| `summary.top_decile_threshold` | float       | 90th percentile score                     | Use as VIP threshold in campaigns                            |
| `summary.segment_counts`       | object      | Distribution across loyalty tiers         | For planning perks, rewards, support resources               |

Default Segments:

Platinum: ≥ 0.85

Gold: 0.70–0.84

Silver: 0.50–0.69

Bronze: 0.30–0.49

At-Risk: < 0.30

Commercial Playbooks:

| Segment  | Typical Actions                                            | Expected Lift            |
| -------- | ---------------------------------------------------------- | ------------------------ |
| Platinum | Early-access drops, private WhatsApp groups, free shipping | +18% retention, +12% AOV |
| Gold     | Double-points weekends, cross-sell bundles                 | +9% spend                |
| Silver   | Education drip emails, category quizzes                    | +5% category breadth     |
| At-Risk  | Win-back coupons (48h), feedback surveys                   | 6–10% reactivation       |


Frequently Asked Questions:

| Q                                              | A                                                                                 |
| ---------------------------------------------- | --------------------------------------------------------------------------------- |
| Does weighting affect comparability over time? | Only within the same run. Use consistent weights for trend KPIs.                  |
| Can I feed thousands of customers?             | Yes — up to 20MB (\~60k rows). For more, use chunks or `/customer_loyalty_async`. |
| Is recency penalized or rewarded?              | Lower `recency_days` = better. The API internally scales it so recent = 1.0.      |
| We want five custom tiers.                     | Supported via `segment_thresholds` (coming Q3 2025). Until then, map manually.    |


Sample SQL to Build the Payload:

WITH tx AS (
  SELECT customer_id,
         order_id      AS transaction_id,
         order_date    AS date,
         order_total   AS amount,
         product_id
    FROM orders
   WHERE order_date >= DATE_SUB(CURDATE(), INTERVAL 18 MONTH)
),
features AS (
  SELECT
     customer_id,
     JSON_OBJECTAGG('transaction_id', transaction_id,
                    'date',           date,
                    'amount',         amount,
                    'product_id',     product_id) AS tx_json
  FROM tx
  GROUP BY customer_id
)
SELECT
  JSON_OBJECT(
    'customer_id', customer_id,
    'transactions', JSON_PARSE(CONCAT('[', GROUP_CONCAT(tx_json), ']'))
  )
FROM features;

📌 Pipe the resulting JSON array into the request body with your selected weights to score loyalty in real time.


4.1.4 /api/v1/ai/customer_rfm

Recency-Frequency-Monetary Scoring Service

Purpose
Calculate classic RFM metrics, convert them into binned scores (1…N), and assign each customer to a lifecycle tier (e.g., Champion, Loyal, At-Risk) to trigger marketing actions.

Headers:

| Header              | Value              |
| ------------------- | ------------------ |
| `X-Customer-Api-Id` | `<uuid>`           |
| `X-Secret`          | `<secret>`         |
| `Content-Type`      | `application/json` |


Request
POST https://api-prod.abintel.ai/api/v1/ai/customer_rfm

Request Body Schema:

| JSON Key         | Type / Range   | Req.?    | Meaning                       | Guideline / SQL Derivation |
| ---------------- | -------------- | -------- | ----------------------------- | -------------------------- |
| `bins`           | integer (2–10) | Optional | Number of buckets per metric  | Default = 5 (quintiles)    |
| `today`          | ISO date       | Optional | Reference date for recency    | Omit → server date         |
| `customers`      | array<object>  | Yes      | List of customer transactions | Output of nightly ETL      |
| → `customer_id`  | string         | Yes      | Primary key                   | `orders.customer_id`       |
| → `transactions` | array<object>  | Yes (≥1) | List of order events          | 6–18 months typical        |
| → `date`         | ISO date       | Yes      | Order date                    | —                          |
| → `amount`       | float ≥ 0      | Yes      | Order value                   | —                          |


Choosing Bins:

| Customer Base Size | Recommended Bins |
| ------------------ | ---------------- |
| < 5,000            | 4 (quartiles)    |
| 5,000 – 100,000    | 5 (quintiles)    |
| > 100,000          | 10 (deciles)     |

Aim for ≥ 500 customers per bin to ensure stable breakpoints.

What the API Computes:

| Metric           | Formula (per customer)       |
| ---------------- | ---------------------------- |
| `recency_days`   | `DATEDIFF(today, MAX(date))` |
| `frequency`      | `COUNT(transactions)`        |
| `monetary_total` | `SUM(amount)`                |


Steps:

Sort customers (ascending for recency, descending for others).

Split into bins (equal-sized buckets).

Assign scores from 1 to N. (Recency is reversed: fewer days → higher score.)

Concatenate scores → rfm_class (e.g., 553).

Compute rfm_total (sum of the 3 scores).

Map to a default RFM tier.

RFM Tier Mapping

| Tier      | Heuristic                      |
| --------- | ------------------------------ |
| Champion  | `rfm_total ≥ bins*3 – 2`       |
| Loyal     | Top 30% by total, not Champion |
| Potential | Mid-upper band                 |
| Promising | Mid-lower band                 |
| At-Risk   | Bottom 20%                     |


Example Request:

{
  "bins": 5,
  "today": "2025-04-24",
  "customers": [
    {
      "customer_id": "C-1001",
      "transactions": [
        { "date": "2025-04-21", "amount": 520.55 },
        { "date": "2025-03-29", "amount": 310.00 },
        { "date": "2025-02-10", "amount": 398.40 }
      ]
    }
  ]
}


Example Response:

{
  "customers": [
    {
      "customer_id": "C-1001",
      "recency_days":     3,
      "frequency":        3,
      "monetary_total":   1228.95,
      "recency_score":    3,
      "frequency_score":  3,
      "monetary_score":   5,
      "rfm_class":  "335",
      "rfm_total":  11,
      "rfm_tier":   "Loyal",
      "interpretation": "Loyal customer – last purchase was recent, buys with high frequency and has very high spending…"
    }
  ],
  "thresholds": {
    "recency":   [66.6, 24.2, 2.6, 1.8],
    "frequency": [2.4, 2.8, 3.2, 3.6],
    "monetary":  [381.54, 567.18, 773.79, 1001.37]
  }
}

Response Keys Explained:

| Path             | Type        | Meaning                           | Operational Use                           |
| ---------------- | ----------- | --------------------------------- | ----------------------------------------- |
| `recency_days`   | int         | Days since last purchase          | Trigger reactivation after threshold days |
| `frequency`      | int         | Order count in the window         | Identify low-visit big-spend customers    |
| `monetary_total` | float       | Total spend                       | Used for tiered perks                     |
| `*_score`        | 1…bins      | Normalized bucket score           | Input to ML features                      |
| `rfm_class`      | 3-digit str | e.g., `553`                       | Quick segmentation filters                |
| `rfm_total`      | int         | Sum of 3 scores                   | Rank customers (e.g., VIP lists)          |
| `rfm_tier`       | enum        | Loyalty label                     | Drive personalization in CRM              |
| `interpretation` | string      | Human-readable summary            | Add to emails, dashboards                 |
| `thresholds`     | object      | Cutoffs per metric for bucketting | Reuse to ensure consistency across runs   |


Business Playbooks:

| Tier      | Messaging & Tactics                                    |
| --------- | ------------------------------------------------------ |
| Champion  | Early access, invite-only club, lifetime free shipping |
| Loyal     | Double-points weekends, subscription upsell            |
| Potential | Cross-sell offers, onboarding nudges                   |
| Promising | Education emails, frequency incentives (e.g., coupons) |
| At-Risk   | Win-back emails, SMS promos, exit surveys              |

SQL/Python One-liner to Build Payload:

import pandas as pd

df = pd.read_sql(
    "SELECT customer_id, order_date as date, order_total as amount "
    "FROM orders WHERE order_date >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)",
    conn
)

out = (df.groupby('customer_id')
         .apply(lambda g: {
             'customer_id': g.name,
             'transactions': g[['date','amount']].to_dict('records')
         }))

body = {'bins': 5, 'today': '2025-04-24', 'customers': list(out)}

FAQ:

| Question                               | Answer                                                                |
| -------------------------------------- | --------------------------------------------------------------------- |
| Can I add `transactions.amount_local`? | Only `amount` is scored. Extra keys are ignored.                      |
| Bins vs tiers?                         | Bins determine score range (1–N); tiers are mapped from total score.  |
| Stability over time?                   | Reuse `thresholds` from a past run to ensure consistent bucket logic. |


4.1.5 /api/v1/ai/customer_clv_features

Feature-Engineer Customers for BG/NBD + Gamma-Gamma CLV Models

Purpose
Return the four classic customer-lifetime metrics (frequency, recency, T, monetary_value) along with a potential tier. Use the response to:

Pipe directly into /api/customer_clv_forecast

Enrich your data warehouse for in-house CLV modeling

Headers:

| Header              | Value              |
| ------------------- | ------------------ |
| `X-Customer-Api-Id` | `<uuid>`           |
| `X-Secret`          | `<secret>`         |
| `Content-Type`      | `application/json` |


Request
POST https://api-prod.abintel.ai/api/v1/ai/customer_clv_features

Request Body Schema:

| Key              | Type / Range          | Required? | Meaning                               | How to Prepare                                         |
| ---------------- | --------------------- | --------- | ------------------------------------- | ------------------------------------------------------ |
| `today`          | ISO-date (YYYY-MM-DD) | Optional  | Observation cut-off                   | Omit → server date (UTC midnight)                      |
| `customers`      | array<object>         | Yes       | Each shopper’s full order history     | From daily ETL view                                    |
| → `customer_id`  | string                | Yes       | Primary key                           | —                                                      |
| → `transactions` | array<object> (≥ 1)   | Yes       | Historical orders in look-back window | —                                                      |
| → `date`         | ISO-date              | Yes       | Order date                            | —                                                      |
| → `amount`       | float ≥ 0             | Yes       | Revenue (gross or net)                | —                                                      |
| → `cost`         | float ≥ 0             | Optional  | Cost of Sale / COGS                   | Optional; otherwise `monetary_value = average revenue` |

ℹ️ Recommended look-back window: Minimum 6 months (12–24 months preferred) to ensure variance in frequency and recency.

Sample Request:

{
  "today": "2025-04-24",
  "customers": [
    {
      "customer_id": "C-1001",
      "transactions": [
        { "date": "2024-10-12", "amount": 520.55, "cost": 312.33 },
        { "date": "2025-02-20", "amount": 398.40, "cost": 210.00 }
      ]
    }
  ]
}

What the API Calculates:

| Output Field         | Formula                           | Notes                                                   |
| -------------------- | --------------------------------- | ------------------------------------------------------- |
| `frequency`          | `#orders − 1`                     | BG/NBD interprets 0 as a one-time buyer                 |
| `recency`            | `days_between(first_tx, last_tx)` | Float, in days                                          |
| `T`                  | `days_between(first_tx, today)`   | Age of customer at time of snapshot                     |
| `monetary_value`     | `AVG(margin)` if cost present     | Otherwise, use `AVG(amount)` for average revenue        |
| `churned`            | bool                              | True if last\_tx ≤ threshold (feature planned)          |
| `clv_potential_tier` | enum (High / Medium / Low)        | High if top 30% of `monetary_value` and `frequency ≥ 1` |

All date math uses day-level precision and handles leap years correctly.

Sample Response:

{
  "customers": [
    {
      "customer_id": "C-1001",
      "frequency": 1,
      "recency": 131.0,
      "T": 194.0,
      "monetary_value": 188.40,
      "churned": false,
      "clv_potential_tier": "High",
      "interpretation": "C-1001 has 1 repeat purchases, 131 days between first & last buy, and Avg margin/order: $188. Tier: High. Focus on VIP retention & upsell.",
      "explanations": {
        "frequency": "1 repeat purchases. In top 0% engagement.",
        "recency": "131 days between first & last buy. Last order moderately recent.",
        "T": "194 days observed (customer age).",
        "monetary_value": "Avg margin/order: $188.40 – top 0% spenders.",
        "churned": "False (no churn date supplied).",
        "clv_potential_tier": "Overall potential tier **High**."
      }
    }
  ]
}

Response Keys Breakdown:

| Path                 | Type         | Meaning                               | Commercial Use                             |
| -------------------- | ------------ | ------------------------------------- | ------------------------------------------ |
| `frequency`          | int          | Repeat purchases                      | Core input to BG/NBD; engagement proxy     |
| `recency`            | float (days) | Time between 1st and last transaction | Churn modeling, recent activity detection  |
| `T`                  | float (days) | Customer age                          | Required by BG/NBD                         |
| `monetary_value`     | float        | Avg margin/order                      | Gamma-Gamma CLV input; margin targeting    |
| `clv_potential_tier` | enum         | High / Medium / Low                   | Prioritize high-value cohorts              |
| `interpretation`     | string       | Plain-English insight                 | Insert into CRM or dashboards              |
| `explanations`       | object       | Rationale per metric                  | Customer service agents, report annotation |

SQL ETL Snippet to Generate Payload:

WITH tx AS (
  SELECT customer_id,
         order_date  AS date,
         order_total AS amount,
         order_cost  AS cost
    FROM orders
   WHERE order_date BETWEEN DATE_SUB(CURDATE(), INTERVAL 24 MONTH) AND CURDATE()
),
payload AS (
  SELECT JSON_OBJECT(
           'customer_id',   customer_id,
           'transactions',  JSON_ARRAYAGG(
                                JSON_OBJECT('date',   DATE_FORMAT(date,'%Y-%m-%d'),
                                            'amount', amount,
                                            'cost',   cost)
                             )
         ) AS customer_json
  FROM tx
  GROUP BY customer_id
)
SELECT JSON_ARRAYAGG(customer_json) FROM payload;

Typical Playbooks:

| Action                  | How to Use                                                          |
| ----------------------- | ------------------------------------------------------------------- |
| Forward to CLV forecast | Get 6–36 month predictions via `/customer_clv_forecast`             |
| CAC retuning            | Bid higher on High-tier lookalikes; reduce spend on Low-tier        |
| Margin-led segmentation | Combine `monetary_value` with RFM tiers for “High-Margin Champions” |
| Churn modeling          | Feed metrics into `/churn_risk` or your own classifier              |

FAQ:

| Question                                | Answer                                                             |
| --------------------------------------- | ------------------------------------------------------------------ |
| Why is frequency = 1 for 2 orders?      | BG/NBD defines `frequency = repeat purchases = total − 1`          |
| I only have revenue, not cost.          | That’s fine — API will use revenue for `monetary_value`            |
| What about customers with only 1 order? | `frequency = 0` is valid; Gamma-Gamma will ignore `monetary_value` |
| Can I send > 20 MB of data?             | Chunk requests or wait for async CLV bulk endpoint (v1.8)          |


4.1.6 /api/v1/ai/customer_clv_forecast

Predict Dollar-Value Customer-Lifetime (6-, 12-, 36-month horizons)

Purpose
Ingest raw order history, auto-select the most accurate CLV model for your dataset (either BG/NBD + Gamma-Gamma or a Gradient-Boosted Regression fallback), and return a dollar-value forecast for each customer, along with:

Confidence-rated tiers (High / Medium / Low)

Plain-language explanations

Headers:

| Header              | Value              |
| ------------------- | ------------------ |
| `X-Customer-Api-Id` | `<uuid>`           |
| `X-Secret`          | `<secret>`         |
| `Content-Type`      | `application/json` |

Request
POST https://api-prod.abintel.ai/api/v1/ai/customer_clv_forecast

Request Body Schema:

| JSON Key         | Type / Range  | Required? | Meaning                               | Tips / Derivation                    |
| ---------------- | ------------- | --------- | ------------------------------------- | ------------------------------------ |
| `horizon_months` | int (1–60)    | Yes       | Time window for CLV prediction        | Typical values: 6, 12, 24, 36        |
| `today`          | ISO date      | Optional  | Snapshot date                         | Omit to use server date              |
| `customers`      | array<object> | Yes       | One object per shopper                | Max: \~60k customers per 20MB call   |
| → `customer_id`  | string        | Yes       | Unique ID                             | —                                    |
| → `transactions` | array<object> | Yes (≥1)  | Historical orders in look-back period | —                                    |
| → `date`         | ISO date      | Yes       | Order date                            | —                                    |
| → `amount`       | float ≥ 0     | Yes       | Revenue or margin                     | —                                    |
| → `cost`         | float ≥ 0     | Optional  | Cost of goods sold                    | Used to compute profit CLV           |
| → `churn_date`   | ISO date      | Optional  | Known cancellation date               | Treated as end of customer lifecycle |

💡 Look-back window: Minimum 6 months required. 12–24 months preferred.

Example Request:

{
  "horizon_months": 6,
  "today": "2025-04-24",
  "customers": [
    {
      "customer_id": "C-1001",
      "transactions": [
        { "date": "2024-01-15", "amount": 250, "cost": 140 },
        { "date": "2024-05-10", "amount": 480, "cost": 270 },
        { "date": "2024-10-12", "amount": 520, "cost": 312 },
        { "date": "2025-02-20", "amount": 398, "cost": 210 }
      ]
    },
    {
      "customer_id": "C-1002",
      "transactions": [
        { "date": "2024-03-02", "amount": 120, "cost": 60 }
      ],
      "churn_date": "2024-04-01"
    }
  ]
}

Model Selection Logic
Derive CLV features: frequency, recency, T, monetary_value

Perform time-aware train-test split

Compare:

BG/NBD + Gamma-Gamma (classic probabilistic)

Gradient-Boosted Regression Trees (robust to sparse/noisy data)

Select the model with lowest out-of-sample MAE

Return the winner in best_algorithm

GBM is preferred for high singleton counts or missing margin data.

Example Response:

{
  "best_algorithm": "GradientBoosting",
  "horizon_months": 6,
  "evaluation_mae": { "GradientBoosting": 188.0 },
  "customers": [
    {
      "customer_id": "C-1001",
      "predicted_clv": 188.0,
      "predicted_txns": null,
      "predicted_avg_margin": null,
      "algorithm_used": "GradientBoosting",
      "clv_tier": "Low",
      "interpretation": "C-1001 expected to deliver $188 margin in 6 months (tier Low). Consider automated win-back.",
      "explanations": {
        "predicted_clv": "Projected gross margin over 6 months.",
        "algorithm_used": "Selected via lowest MAE (188.00).",
        "clv_tier": "High ≥ $1,000, Medium ≥ $300, Low < $300."
      }
    }
  ]
}

Response Field Guide:

| Path                        | Type          | Meaning                                    | Operational Use                            |
| --------------------------- | ------------- | ------------------------------------------ | ------------------------------------------ |
| `best_algorithm`            | enum          | `BG_NBD_GG` or `GradientBoosting`          | Store for audit/model governance           |
| `evaluation_mae`            | object        | Mean Absolute Error (lower = better)       | Confidence measure                         |
| `customers[].predicted_clv` | float         | Dollar value expected over forecast period | LTV segmentation, CAC tuning               |
| `predicted_txns`            | int \| null   | Only for BG/NBD                            | Forecast for operations (e.g. fulfillment) |
| `predicted_avg_margin`      | float \| null | Only for Gamma-Gamma                       | Pricing optimization                       |
| `clv_tier`                  | enum          | Tier: High / Medium / Low                  | Use in campaign and spend strategies       |
| `interpretation`            | string        | Human-readable summary                     | CRM dashboards                             |
| `explanations`              | object        | Metric-level justifications                | Agent tools, reporting                     |

Default tier thresholds:

High ≥ $1,000

Medium ≥ $300

Low < $300
(Overridable in future release)

Commercial Playbooks Enabled:

| Tier       | Action                                        | ROI Observed      |
| ---------- | --------------------------------------------- | ----------------- |
| **High**   | Extend credit terms, early-access launches    | +15% retention    |
| **Medium** | Cross-sell bundles, subscription offers       | +7% margin uplift |
| **Low**    | Win-back automation, survey, reduced CAC bids | −12% wasted CAC   |

Best-Practice ETL Flow:

orders → daily DBT model → JSON payload
             │
             ▼
POST /customer_clv_features   (optional preprocessing step)
             │
             ▼
POST /customer_clv_forecast   → CLV table in warehouse
             │
             ├─> Marketing cloud (audiences)
             ├─> Finance BI (LTV:CAC dashboards)
             └─> CRM (VIP flags)


FAQ:

| Question                                       | Answer                                                                  |
| ---------------------------------------------- | ----------------------------------------------------------------------- |
| Why same CLV for two very different customers? | Small sample → GBM fallback uses a baseline. Larger sets show variance. |
| Can I override tier cut-offs?                  | Soon via `tiers: { high: 1000, medium: 300 }` param (v1.8)              |
| How does `churn_date` affect forecast?         | Revenue after `churn_date` is ignored                                   |
| Is margin (`cost`) mandatory?                  | No — omit `cost` to use revenue instead                                 |

Next Step: Combine clv_tier with rfm_tier and loyalty_score to build micro-segments that optimize both long-term value and short-term engagement.

4.1.7 /api/v1/ai/purchasing_segmentation

Behavioral Clustering via Model Benchmarking

Purpose
Benchmark four clustering families (K-Means, Gaussian-Mixture, Agglomerative, DBSCAN), auto-select the statistically best model, and return:

Customer-to-cluster labels

Feature-space centroids

Cluster personas (English descriptors)

Full dendrogram for visualization

Headers:

| Header              | Value              |
| ------------------- | ------------------ |
| `X-Customer-Api-Id` | `<uuid>`           |
| `X-Secret`          | `<secret>`         |
| `Content-Type`      | `application/json` |


Request
POST https://api-prod.abintel.ai/api/v1/ai/purchasing_segmentation

Request Body Schema — Key-by-Key

| JSON Key          | Type / Range                                    | Required? | Meaning                                        | How to Derive / Best Practice               |
| ----------------- | ----------------------------------------------- | --------- | ---------------------------------------------- | ------------------------------------------- |
| `customers`       | array<object>                                   | Yes       | One row per shopper with minimal features      | Typically a materialised view (6–18 months) |
| → `id`            | string (unique)                                 | Yes       | Primary key from CRM                           | —                                           |
| → `frequency`     | int ≥ 1                                         | Yes       | Count of transactions                          | `COUNT(order_id)`                           |
| → `amount_spent`  | float ≥ 0                                       | Yes       | Total gross/net revenue in window              | `SUM(order_total)`                          |
| → `product_types` | array<string> (length ≥ 1)                      | Yes       | Purchased SKUs or categories                   | `array_agg(DISTINCT category_lvl1)`         |
| `k_range`         | array<int> (between 3–30)                       | Optional  | List of k values to try in K-Means, GMM, Agglo | Keep it short: 3–10 for performance         |
| `algorithms`      | array<`kmeans` \| `gmm` \| `agglo` \| `dbscan`> | Optional  | Specify algorithm families to include          | Omit to test all four                       |

Recommended k_range Based on Customer Base Size

| Customer Base Size | Suggested `k_range` |
| ------------------ | ------------------- |
| ≤ 50,000           | `[3, 4, 5]`         |
| 50k – 250k         | `[3, 4, 5, 6]`      |
| > 250k             | `[4, 5, 6, 7, 8]`   |

🎯 Tip: Keep k_range short for better speed during benchmarking.

Sample Request (40 Customers, 4 k values):

{
  "customers": [
    {
      "id": "C001",
      "frequency": 28,
      "amount_spent": 4120.75,
      "product_types": ["laptop", "mouse", "keyboard", "monitor"]
    },
    {
      "id": "C002",
      "frequency": 5,
      "amount_spent": 190.40,
      "product_types": ["tea", "coffee"]
    },
    {
      "id": "C003",
      "frequency": 12,
      "amount_spent": 845.10,
      "product_types": ["jeans", "t-shirt", "sneakers"]
    },
    {
      "id": "C004",
      "frequency": 2,
      "amount_spent": 75.00,
      "product_types": ["notebook"]
    },
    {
      "id": "C005",
      "frequency": 17,
      "amount_spent": 1324.55,
      "product_types": ["smartphone", "earbuds"]
    },
    {
      "id": "C006",
      "frequency": 8,
      "amount_spent": 510.30,
      "product_types": ["dog food", "cat food"]
    },
    {
      "id": "C007",
      "frequency": 24,
      "amount_spent": 2780.90,
      "product_types": ["smart-tv", "soundbar", "hdmi-cable"]
    },
    {
      "id": "C008",
      "frequency": 10,
      "amount_spent": 660.00,
      "product_types": ["yogurt", "milk", "cheese"]
    },
    {
      "id": "C009",
      "frequency": 3,
      "amount_spent": 120.00,
      "product_types": ["stationery"]
    },
    {
      "id": "C010",
      "frequency": 15,
      "amount_spent": 975.65,
      "product_types": ["beer", "wine", "snacks"]
    },
    {
      "id": "C011",
      "frequency": 7,
      "amount_spent": 450.25,
      "product_types": ["books", "pens"]
    },
    {
      "id": "C012",
      "frequency": 25,
      "amount_spent": 3015.90,
      "product_types": ["tablet", "stylus", "cover"]
    },
    {
      "id": "C013",
      "frequency": 6,
      "amount_spent": 210.10,
      "product_types": ["pasta", "sauce"]
    },
    {
      "id": "C014",
      "frequency": 20,
      "amount_spent": 1884.00,
      "product_types": ["fitness-watch"]
    },
    {
      "id": "C015",
      "frequency": 1,
      "amount_spent": 40.00,
      "product_types": ["batteries"]
    },
    {
      "id": "C016",
      "frequency": 13,
      "amount_spent": 1105.35,
      "product_types": ["printer", "ink"]
    },
    {
      "id": "C017",
      "frequency": 4,
      "amount_spent": 160.00,
      "product_types": ["flour", "sugar", "eggs"]
    },
    {
      "id": "C018",
      "frequency": 30,
      "amount_spent": 4522.60,
      "product_types": ["gaming-pc", "rgb-mouse", "gaming-chair", "headset"]
    },
    {
      "id": "C019",
      "frequency": 11,
      "amount_spent": 720.00,
      "product_types": ["makeup", "skincare"]
    },
    {
      "id": "C020",
      "frequency": 9,
      "amount_spent": 585.40,
      "product_types": ["toys", "board-games"]
    },
    {
      "id": "C021",
      "frequency": 2,
      "amount_spent": 55.00,
      "product_types": ["candles"]
    },
    {
      "id": "C022",
      "frequency": 18,
      "amount_spent": 1410.70,
      "product_types": ["camera", "sd-card"]
    },
    {
      "id": "C023",
      "frequency": 14,
      "amount_spent": 992.00,
      "product_types": ["office-chair", "desk-lamp"]
    },
    {
      "id": "C024",
      "frequency": 3,
      "amount_spent": 135.00,
      "product_types": ["juice", "cookies"]
    },
    {
      "id": "C025",
      "frequency": 12,
      "amount_spent": 830.20,
      "product_types": ["running-shoes", "sports-socks"]
    },
    {
      "id": "C026",
      "frequency": 23,
      "amount_spent": 2560.00,
      "product_types": ["fridge", "water-filter"]
    },
    {
      "id": "C027",
      "frequency": 6,
      "amount_spent": 260.40,
      "product_types": ["vitamins"]
    },
    {
      "id": "C028",
      "frequency": 5,
      "amount_spent": 195.99,
      "product_types": ["wine", "cheese"]
    },
    {
      "id": "C029",
      "frequency": 19,
      "amount_spent": 1599.95,
      "product_types": ["e-reader", "ebooks"]
    },
    {
      "id": "C030",
      "frequency": 22,
      "amount_spent": 2345.00,
      "product_types": ["washing-machine", "detergent"]
    },
    {
      "id": "C031",
      "frequency": 4,
      "amount_spent": 175.50,
      "product_types": ["herbs", "spices"]
    },
    {
      "id": "C032",
      "frequency": 8,
      "amount_spent": 540.00,
      "product_types": ["diapers", "baby-wipes", "formula"]
    },
    {
      "id": "C033",
      "frequency": 27,
      "amount_spent": 3350.80,
      "product_types": ["console", "controller", "games"]
    },
    {
      "id": "C034",
      "frequency": 10,
      "amount_spent": 705.35,
      "product_types": ["sunscreen", "after-sun"]
    },
    {
      "id": "C035",
      "frequency": 3,
      "amount_spent": 150.00,
      "product_types": ["frozen-pizza"]
    },
    {
      "id": "C036",
      "frequency": 16,
      "amount_spent": 1240.60,
      "product_types": ["tool-set", "drill"]
    },
    {
      "id": "C037",
      "frequency": 9,
      "amount_spent": 610.10,
      "product_types": ["organic-vegetables"]
    },
    {
      "id": "C038",
      "frequency": 21,
      "amount_spent": 2105.00,
      "product_types": ["air-fryer", "cookbook"]
    },
    {
      "id": "C039",
      "frequency": 7,
      "amount_spent": 435.75,
      "product_types": ["flowers", "vase"]
    },
    {
      "id": "C040",
      "frequency": 14,
      "amount_spent": 1015.90,
      "product_types": ["winter-jacket", "gloves", "scarf"]
    }
  ],
  "k_range": [3, 4, 5, 6]
}

The payload is only three numeric columns plus a categorical array—small enough to post thousands of customers in < 1 MB.

Response:

{
    "benchmark": [
        {
            "algo": "kmeans",
            "k": 3,
            "silhouette": 0.597453236579895,
            "davies_bouldin": 0.5546427618629307
        },
        {
            "algo": "kmeans",
            "k": 4,
            "silhouette": 0.5596717000007629,
            "davies_bouldin": 0.5109844153948895
        },
        {
            "algo": "kmeans",
            "k": 5,
            "silhouette": 0.5498684644699097,
            "davies_bouldin": 0.46947742357211847
        },
        {
            "algo": "kmeans",
            "k": 6,
            "silhouette": 0.515194296836853,
            "davies_bouldin": 0.4158492744102214
        },
        {
            "algo": "gmm",
            "k": 3,
            "silhouette": 0.5820122957229614,
            "davies_bouldin": 0.5638253915641692
        },
        {
            "algo": "gmm",
            "k": 4,
            "silhouette": 0.51613849401474,
            "davies_bouldin": 0.5061519518619599
        },
        {
            "algo": "gmm",
            "k": 5,
            "silhouette": 0.5426244139671326,
            "davies_bouldin": 0.44539875895043995
        },
        {
            "algo": "gmm",
            "k": 6,
            "silhouette": 0.5039973258972168,
            "davies_bouldin": 0.39578372055882266
        },
        {
            "algo": "agglo",
            "k": 3,
            "silhouette": 0.678362250328064,
            "davies_bouldin": 0.376151934569577
        },
        {
            "algo": "agglo",
            "k": 4,
            "silhouette": 0.5551854372024536,
            "davies_bouldin": 0.4501147341595592
        },
        {
            "algo": "agglo",
            "k": 5,
            "silhouette": 0.6081505417823792,
            "davies_bouldin": 0.43129043959670116
        },
        {
            "algo": "agglo",
            "k": 6,
            "silhouette": 0.6091357469558716,
            "davies_bouldin": 0.4559881260676169
        },
        {
            "algo": "dbscan",
            "k": null,
            "silhouette": -1.0,
            "davies_bouldin": null
        }
    ],
    "best_model": {
        "algo": "agglo",
        "k": 3,
        "silhouette": 0.678362250328064,
        "davies_bouldin": 0.376151934569577,
        "n_clusters": 3
    },
    "centroids": [
        [
            8.806451797485352,
            591.1141967773438,
            1.9677419662475586
        ],
        [
            23.14285659790039,
            2577.371337890625,
            2.2857143878936768
        ],
        [
            29.0,
            4321.6748046875,
            4.0
        ]
    ],
    "labels": {
        "C001": 2,
        "C002": 0,
        "C003": 0,
        "C004": 0,
        "C005": 0,
        "C006": 0,
        "C007": 1,
        "C008": 0,
        "C009": 0,
        "C010": 0,
        "C011": 0,
        "C012": 1,
        "C013": 0,
        "C014": 1,
        "C015": 0,
        "C016": 0,
        "C017": 0,
        "C018": 2,
        "C019": 0,
        "C020": 0,
        "C021": 0,
        "C022": 0,
        "C023": 0,
        "C024": 0,
        "C025": 0,
        "C026": 1,
        "C027": 0,
        "C028": 0,
        "C029": 0,
        "C030": 1,
        "C031": 0,
        "C032": 0,
        "C033": 1,
        "C034": 0,
        "C035": 0,
        "C036": 0,
        "C037": 0,
        "C038": 1,
        "C039": 0,
        "C040": 0
    },
    "interpretations_en": {
        "0": "31 customers, avg spend 591, 8.8 purchases/mo. Top categories: wine, cheese, diapers.",
        "1": "7 customers, avg spend 2,577, 23.1 purchases/mo. Top categories: smart-tv, soundbar, hdmi-cable.",
        "2": "2 customers, avg spend 4,322, 29.0 purchases/mo. Top categories: laptop, mouse, keyboard."
    },
    "dendrogram": {
        "linkage_matrix": [
            [
                1.0,
                27.0,
                5.5900115966796875,
                2.0
            ],
            [
                16.0,
                34.0,
                10.246950765959598,
                2.0
            ],
            [
                10.0,
                38.0,
                14.5,
                2.0
            ],
            [
                18.0,
                33.0,
                14.684114387072421,
                2.0
            ],
            [
                2.0,
                24.0,
                14.933482805184708,
                2.0
            ],
            [
                14.0,
                20.0,
                15.033296378372908,
                2.0
            ],
            [
                8.0,
                23.0,
                15.033296378372908,
                2.0
            ],
            [
                9.0,
                22.0,
                16.41102378466232,
                2.0
            ],
            [
                12.0,
                40.0,
                19.554342797373597,
                3.0
            ],
            [
                30.0,
                41.0,
                23.67840084690405,
                3.0
            ],
            [
                19.0,
                36.0,
                24.720185838561353,
                2.0
            ],
            [
                5.0,
                31.0,
                29.71684244831212,
                2.0
            ],
            [
                3.0,
                45.0,
                31.75951301054011,
                3.0
            ],
            [
                39.0,
                47.0,
                37.04603277151646,
                3.0
            ],
            [
                46.0,
                49.0,
                53.204636389447614,
                5.0
            ],
            [
                7.0,
                43.0,
                60.837535989873956,
                3.0
            ],
            [
                26.0,
                48.0,
                75.42189702647553,
                4.0
            ],
            [
                4.0,
                35.0,
                83.95602895187841,
                2.0
            ],
            [
                50.0,
                51.0,
                102.69139093644127,
                4.0
            ],
            [
                15.0,
                53.0,
                135.75478433757996,
                4.0
            ],
            [
                54.0,
                56.0,
                139.4687935795597,
                9.0
            ],
            [
                21.0,
                57.0,
                147.95607449395212,
                3.0
            ],
            [
                42.0,
                58.0,
                193.4435440607144,
                6.0
            ],
            [
                25.0,
                29.0,
                215.00232556881798,
                2.0
            ],
            [
                44.0,
                55.0,
                220.82693991432996,
                5.0
            ],
            [
                13.0,
                37.0,
                221.00452484055614,
                2.0
            ],
            [
                6.0,
                11.0,
                235.00212764994276,
                2.0
            ],
            [
                52.0,
                60.0,
                256.36800922508723,
                12.0
            ],
            [
                28.0,
                61.0,
                336.40545943552,
                4.0
            ],
            [
                0.0,
                17.0,
                401.8550746056813,
                2.0
            ],
            [
                32.0,
                66.0,
                522.3946688588159,
                3.0
            ],
            [
                62.0,
                64.0,
                537.5921954390619,
                11.0
            ],
            [
                63.0,
                65.0,
                647.7163731140352,
                4.0
            ],
            [
                59.0,
                68.0,
                743.4835774228395,
                8.0
            ],
            [
                70.0,
                72.0,
                1528.9174504219059,
                7.0
            ],
            [
                67.0,
                71.0,
                1623.9774150881199,
                23.0
            ],
            [
                73.0,
                75.0,
                2865.3048816814076,
                31.0
            ],
            [
                69.0,
                74.0,
                3076.681261272462,
                9.0
            ],
            [
                76.0,
                77.0,
                8866.563988339083,
                40.0
            ]
        ],
        "columns": [
            "child1",
            "child2",
            "distance",
            "sample_count"
        ],
        "leaf_labels": [
            "C001",
            "C002",
            "C003",
            "C004",
            "C005",
            "C006",
            "C007",
            "C008",
            "C009",
            "C010",
            "C011",
            "C012",
            "C013",
            "C014",
            "C015",
            "C016",
            "C017",
            "C018",
            "C019",
            "C020",
            "C021",
            "C022",
            "C023",
            "C024",
            "C025",
            "C026",
            "C027",
            "C028",
            "C029",
            "C030",
            "C031",
            "C032",
            "C033",
            "C034",
            "C035",
            "C036",
            "C037",
            "C038",
            "C039",
            "C040"
        ],
        "cluster_legend": [
            {
                "cluster_id": 0,
                "n_customers": 31,
                "avg_spend": 591.11,
                "avg_frequency": 8.81,
                "top_categories": "wine, cheese, diapers"
            },
            {
                "cluster_id": 1,
                "n_customers": 7,
                "avg_spend": 2577.37,
                "avg_frequency": 23.14,
                "top_categories": "smart-tv, soundbar, hdmi-cable"
            },
            {
                "cluster_id": 2,
                "n_customers": 2,
                "avg_spend": 4321.68,
                "avg_frequency": 29.0,
                "top_categories": "laptop, mouse, keyboard"
            }
        ]
    }
}

benchmark[] – Algorithm Scorecard
Metric Overview

| Field            | What It Is                                                   | Good Rule-of-Thumb | Example Insight                              |
| ---------------- | ------------------------------------------------------------ | ------------------ | -------------------------------------------- |
| `silhouette`     | Cohesion/separation metric (0–1, higher = better)            | ≥ 0.60 = strong    | `Agglo-3` scores **0.678** ⇒ clear structure |
| `davies_bouldin` | Cluster compactness (lower = better, typical 0–2)            | ≤ 0.50 = great     | `Agglo-3` yields **0.376** (best)            |
| Missing values   | `DBSCAN` may return silhouette = -1 if most points are noise | —                  | In demo, DBSCAN was not competitive          |

Model Results
best_model: Selected by highest Silhouette, then lowest Davies-Bouldin

✅ Example: Agglomerative (k = 3)

centroids: Array of values per cluster:
[frequency, amount_spent, avg_product_types] (reversed to original scale)

🧩 Example Cluster 0: 8.81 tx, 591 R$, ~2 categories/customer

labels: Mapping from customer_id → cluster_id
Join to your CRM table:

UPDATE crm.customers c
SET    segment_id = l.cluster_id
FROM   api_labels l
WHERE  l.customer_id = c.id;

interpretations_en:
Persona blurbs in marketing tone, derived via TF-IDF from product_types

dendrogram:
Minimal structure for rendering linkage charts:

| Key                | Meaning                                                      |
| ------------------ | ------------------------------------------------------------ |
| `linkage_matrix[]` | `[child1, child2, distance, sample_count]` via Ward's method |
| `leaf_labels[]`    | Matches customer order from the original payload             |
| `cluster_legend[]` | Summary stats: size, spend, frequency, top categories        |

Can be used directly with /api/segment_hierarchy_chart, D3.js, or Plotly.

Operational Playbooks:

| Cluster ID | Persona        | Suggested Actions                                            |
| ---------- | -------------- | ------------------------------------------------------------ |
| 0          | Value Basket   | Product discovery emails, lightweight upsells                |
| 1          | Power Shoppers | Add to VIP tier, early access, recommend high-margin bundles |
| 2          | Tech Whales    | Assign CS rep, promote exclusive product launches            |

🧠 Combine segmentation labels with CLV and churn risk scores for full-funnel intelligence.

Quick KPI Interpretation
✅ Silhouette: 0.678 + Davies-Bouldin: 0.376 → Excellent clustering
ℹ️ While you could test k = 4 or 5, cohesion drops slightly.
Stick with 3 clusters unless finer granularity is required for business decisions.

SQL Example to Build the JSON Payload:

WITH base AS (
  SELECT
    c.id,
    c.frequency,
    c.amount_spent,
    ARRAY_AGG(DISTINCT p.category_lvl1) AS product_types
  FROM agg_customer_metrics c
  JOIN order_items oi ON oi.customer_id = c.id
  JOIN products p     ON p.product_id = oi.product_id
  WHERE c.window_start >= '2024-05-01'
  GROUP BY 1, 2, 3
)
SELECT JSON_BUILD_OBJECT(
  'customers', JSON_AGG(base),
  'k_range',   ARRAY[3, 4, 5, 6]
)
FROM base;

FAQ
Question	Answer
Why only three numeric features?	Frequency & Spend explain >70% of variance; product_types adds qualitative diversity. Use advanced variant in Q3 for more.
What is the 3rd value in centroids?	Average number of unique product types per customer.
Can I force K-Means only?	Yes. Use "algorithms": ["kmeans"] in your payload.
Why is DBSCAN silhouette −1?	Means >50% of points were flagged as noise. Try tweaking eps, or omit DBSCAN.

Next Step
▶ POST the same payload to /api/segmentation_report_json_i18n to receive a multilingual KPI + persona pack (EN / ES / PT) ready for:

Slides

Dashboards

Marketing strategy reviews