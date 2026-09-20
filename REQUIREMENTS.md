# Requirements — Dubai Logistics & Last-Mile Delivery Data Platform

> Derived from the project-scoping conversation captured in [PROMPT.md](PROMPT.md).
> Source: https://chatgpt.com/share/6aae49c8-abfc-83e8-9b09-4a9ad7a7481b

## 1. Project Overview

A portfolio-grade Data Engineering project that simulates a centralized operational
data platform for a Dubai-based logistics and last-mile delivery company operating
multiple warehouses, delivery fleets, drivers, and third-party delivery partners.

**Positioning:** this must be presented honestly as an independent/portfolio project
built around a fictional Dubai logistics client — not as a real client engagement
unless client work is actually performed. See [DECISIONS.md](DECISIONS.md#d1).

## 2. Business Problem

Operational data is scattered across ERP, WMS, driver apps, GPS providers, fleet
systems, and delivery-partner APIs. The platform must centralize this data to answer:

- Which orders are at risk of missing SLA?
- Which drivers/routes have poor delivery performance?
- Which warehouses are creating bottlenecks?
- Why are deliveries failing?
- How efficiently are vehicles being utilized?
- Where are delivery routes causing excessive distance/time?
- What happened operationally today?

## 3. Data Sources

| Source | Data Provided |
|---|---|
| ERP | Orders, customers, invoices, order status |
| WMS | Picking, packing, dispatch, inventory |
| Driver App | Delivery status, proof of delivery, driver status, GPS |
| GPS Feeds | Latitude, longitude, speed, heading, ignition status |
| Fleet Management | Vehicle, fuel, maintenance, telemetry |
| Delivery Partner APIs | Third-party tracking, delivery status, ETA |
| HR / Driver System | Driver information |
| Route System | Planned routes and stops |

## 4. Functional Requirements

### 4.1 Dashboards & Reports (must build)
- FR-1: Delivery SLA dashboard
- FR-2: Driver performance dashboard
- FR-3: Warehouse throughput dashboard
- FR-4: Failed delivery analysis
- FR-5: Vehicle utilization dashboard
- FR-6: Route analytics (planned vs. actual)
- FR-7: Daily operational report (automated, scheduled)

### 4.2 SLA Dashboard KPIs
Total orders, delivered orders, pending orders, SLA %, average delivery time,
failed deliveries, first-attempt delivery rate, average distance, orders per driver,
active vehicles — broken down by warehouse and by hour.

### 4.3 Driver Performance Metrics
Orders assigned, orders delivered, failed deliveries, SLA %, average delivery time,
average distance, idle time, route deviation, customer attempts.

### 4.4 Warehouse Analytics
Track the inbound → picking → packing → ready-for-dispatch → dispatched flow.
Metrics: orders received/hour, picking/packing throughput, average processing time,
dock utilization, dispatch volume, backlog, order aging.

### 4.5 Failed Delivery Analysis
Standardized failure categories: `CUSTOMER_UNAVAILABLE`, `WRONG_ADDRESS`,
`CUSTOMER_CANCELLED`, `VEHICLE_BREAKDOWN`, `DRIVER_UNAVAILABLE`, `TRAFFIC_DELAY`,
`PACKAGE_DAMAGED`, `PAYMENT_FAILURE`, `OTHER`.
Analyze failure rate by warehouse, driver, route, time of day, location, delivery
partner, and failure reason.

### 4.6 Vehicle Utilization
`Utilization % = Active Vehicle Hours / Available Vehicle Hours`.
Also track kilometers driven, deliveries per vehicle, idle time, fuel consumption,
maintenance events, utilization by vehicle type.

### 4.7 Route Analytics
Compare planned vs. actual route: distance, duration, deviation, stops completed,
average stop duration, traffic delay, delivery density. Optionally derive a
route efficiency business score.

### 4.8 Daily Operations Report
Automatically generated daily summary (orders, SLA, fleet, warehouses, top failure
reasons), scheduled via Airflow.

## 5. Non-Functional / Architecture Requirements

- NFR-1: Platform must be **hybrid batch + real-time** (see [DECISIONS.md](DECISIONS.md#d2)).
- NFR-2: Historical/batch data must flow through a **medallion lakehouse**
  (Bronze → Silver → Gold).
- NFR-3: Real-time driver GPS and delivery events must flow through
  **Kafka → Spark Structured Streaming → operational store (Redis/PostgreSQL) → live dashboard**.
- NFR-4: Core analytics must be served through a **star-schema dimensional model**
  in a cloud data warehouse.
- NFR-5: Data quality checks must be enforced at ingestion and transformation.
- NFR-6: Pipelines must support schema evolution without breaking downstream consumers.
- NFR-7: Full data lineage must be traceable from source system to BI layer.
- NFR-8: Pipeline health must be observable (Kafka lag, Airflow failures, Spark job
  status, data freshness, latency, record counts, data-quality failures).
- NFR-9: Architecture must be demonstrably scalable from dev-scale to
  production-scale data volumes (see §8).

## 6. Data Model

### 6.1 Bronze (raw)
`orders/`, `gps_events/`, `drivers/`, `vehicles/`, `warehouses/`, `deliveries/`,
`partner_events/` — no major transformation.

### 6.2 Silver (cleaned/standardized)
`orders_clean`, `delivery_events`, `gps_events_clean`, `driver_activity`,
`vehicle_activity`, `warehouse_operations`, `route_events`.
Must handle: duplicate events, malformed GPS coordinates, missing timestamps,
timezone normalization, schema changes, late-arriving events, invalid
driver/vehicle IDs.

### 6.3 Gold (business-ready)
`delivery_sla_daily`, `driver_performance`, `warehouse_throughput`,
`vehicle_utilization`, `route_performance`, `failed_delivery_analysis`,
`daily_operations_summary`.

### 6.4 Fact Tables
`fact_orders`, `fact_deliveries`, `fact_delivery_events`, `fact_gps_events`,
`fact_route_execution`, `fact_vehicle_usage`, `fact_warehouse_operations`.

### 6.5 Dimension Tables
`dim_customer`, `dim_driver`, `dim_vehicle`, `dim_warehouse`, `dim_route`,
`dim_delivery_partner`, `dim_date`, `dim_location`.

### 6.6 Source Table Schemas (synthetic)
- **customers**: `customer_id, customer_type, customer_zone, latitude, longitude, created_at`
- **orders**: `order_id, customer_id, warehouse_id, order_created_at, promised_delivery_at, priority, package_type, package_weight, delivery_mode, order_status`
- **drivers**: `driver_id, driver_type, home_zone, hire_date, employment_status`
- **vehicles**: `vehicle_id, vehicle_type, capacity_kg, fuel_type, warehouse_id, vehicle_status`
- **deliveries**: `delivery_id, order_id, driver_id, vehicle_id, route_id, assigned_at, pickup_at, out_for_delivery_at, delivered_at, delivery_status, failure_reason, attempt_number`
- **gps_events**: `event_id, driver_id, vehicle_id, route_id, event_timestamp, latitude, longitude, speed, heading, ignition_status`

## 7. Streaming Requirements

- Driver app emits GPS events as JSON: `driver_id, vehicle_id, latitude, longitude, speed, heading, timestamp`.
- Kafka topics: `driver-gps-events` (or `driver-gps`), `delivery-events`, `order-events`,
  `vehicle-events`, `warehouse-events`, `partner-events`.
- Spark Structured Streaming must perform: validation, deduplication, window
  aggregation, and write to Redis/PostgreSQL for the live dashboard.
- Real-time analytics to compute: current driver location, speed violations
  (vs. configured threshold), delivery delay (expected vs. current ETA), route
  deviation (actual GPS trajectory vs. planned route), driver idle time
  (stationary + activity signal + time threshold), warehouse backlog.
- Must be able to discuss/demonstrate: partitions, offsets, consumer groups,
  retention, event ordering, late/duplicate events, checkpointing, and
  exactly-once vs. at-least-once semantics.

## 8. Data Requirements & Volumes

No real client data is available or required. Data strategy: **public/open data +
a public reference logistics dataset + synthetic business data + a simulated
real-time GPS stream.** Full rationale in [DECISIONS.md](DECISIONS.md#d3).

| Component | Source |
|---|---|
| Dubai geography (roads, locations) | OpenStreetMap |
| Dubai public reference data | Dubai Open Data / Dubai Pulse (dubai.ae/open-data) |
| Delivery-distribution realism | Public logistics datasets (e.g. Kaggle 25K delivery-logistics dataset; Kaggle GPS/telemetry-style dataset) — used only for schema/distribution ideas, never relabeled as Dubai client data |
| Customers, orders, drivers, vehicles, warehouses, deliveries | Synthetic generator |
| Routes | Real road network + routing engine (OSRM) |
| Traffic | Synthetic/reference |
| Weather | Public historical weather data/API |
| Delivery partner API | Self-built FastAPI simulator |
| Streaming events | Self-built Kafka event/GPS generator |

**Volumes:**

| Scale | Customers | Drivers | Vehicles | Orders | GPS Events |
|---|---:|---:|---:|---:|---:|
| Development | 10,000 | 500 | 300 | 100,000 | 1,000,000 |
| Production simulation | 500,000 | 5,000 | 3,000 | 10,000,000+ | 100,000,000+ |

Production-scale data does not need to run locally end-to-end; the architecture
and configuration must demonstrate it can scale to that volume.

## 9. Data Quality Requirements

Enforced via Great Expectations or Soda:
- `order_id` → NOT NULL
- `driver_id` → valid FK
- `vehicle_id` → valid FK
- GPS latitude → within [-90, 90]
- GPS longitude → within [-180, 180]
- `event_time` → valid timestamp

## 10. Technology Stack Requirements

- **Processing:** Python, PySpark, Apache Kafka, Spark Structured Streaming, Apache Airflow
- **Storage:** AWS S3, Delta Lake, PostgreSQL
- **Warehouse:** Snowflake, BigQuery, or Redshift (pick one)
- **Serving layer:** Redis, PostgreSQL
- **BI:** Power BI or Apache Superset
- **Infrastructure:** Docker, Terraform, AWS, GitHub Actions
- **Data quality:** Great Expectations or Soda

## 11. Deployment Architecture (target)

Batch: `ERP/WMS/Fleet/Partner APIs → Airflow → S3 → PySpark → Delta Lake → Warehouse → Power BI`
Streaming: `GPS/Driver App/Vehicle/Delivery Events → Kafka (MSK) → Spark Streaming → Redis/Postgres → Live Dashboard`
Batch compute on AWS runs on EMR/Spark; orchestration via Airflow.

## 12. Repository Structure (target)

```
dubai-logistics-data-platform/
├── data-generator/
│   ├── customers.py
│   ├── orders.py
│   ├── drivers.py
│   ├── vehicles.py
│   ├── warehouses.py
│   ├── deliveries.py
│   ├── routes.py
│   ├── delivery_events.py
│   └── gps_simulator.py
├── data/
│   ├── raw/
│   └── reference/
├── streaming/
├── spark/
├── airflow/
├── warehouse/
├── dashboard/
├── infrastructure/
├── genai/              # (extended scope, §13) RAG, text-to-SQL, agents
├── api/                # (extended scope, §13) FastAPI backend
└── frontend/           # (extended scope, §13) Next.js copilot UI
```

## 13. GenAI Operations Copilot Layer (extended scope)

A GenAI layer sits **on top of** the core Data Engineering platform (§1–§12) —
it does not replace any of it. If built, the project may be renamed
**"AI-Powered Real-Time Logistics Data Platform."** Rationale in
[DECISIONS.md](DECISIONS.md#d11).

```text
                 LOGISTICS DATA PLATFORM
                         │
        ┌────────────────┼─────────────────┐
        │                │                 │
      Batch           Streaming          APIs
        │                │                 │
        ▼                ▼                 ▼
   S3/Delta Lake       Kafka          ERP/WMS/Partners
        │                │
        ▼                ▼
     Spark          Spark Streaming
        │                │
        └────────┬───────┘
                 ▼
          Gold Data Layer
                 │
        ┌────────┴─────────┐
        │                  │
     BI Layer          GenAI Layer
        │                  │
    Power BI        LLM + RAG + SQL Agent + Tool Calling
                           │
                           ▼
                Logistics AI Copilot
```

### 13.1 Functional Requirements

- **GFR-1 — Logistics AI Copilot (chat interface):** Operations managers ask
  questions in plain English (e.g. *"Why did SLA performance drop yesterday?"*)
  and receive answers grounded in actual Gold-layer data, not invented by the
  LLM. Flow: `User → LLM → Intent Detection → SQL Agent → Gold Warehouse → Results → LLM → Explanation`.
- **GFR-2 — Text-to-SQL agent:** Translates natural-language questions (e.g.
  *"Show me the 10 drivers with the highest failed-delivery rate this month"*)
  into SQL against the Gold fact/dimension tables (§6), executes it, and has
  the LLM explain the result in plain language.
- **GFR-3 — RAG over operational documentation:** A knowledge base of
  fictional-but-realistic company documents (`delivery-sla-policy`,
  `driver-handbook`, `warehouse-sop`, `failed-delivery-policy`,
  `vehicle-maintenance-policy`, `escalation-policy`,
  `partner-api-documentation`) is chunked, embedded, and stored in a vector DB
  (Qdrant/Pinecone) so questions like *"What should the operations team do
  when a customer is unavailable during the first delivery attempt?"* are
  answered from actual policy text.
- **GFR-4 — Automated daily operations brief:** Airflow triggers an LLM each
  evening to turn Gold-layer daily metrics into a narrative summary (key
  observations, areas requiring attention), deliverable via email/Slack in a
  real deployment.
- **GFR-5 — AI root-cause analysis:** Combines orders, GPS, routes, traffic,
  warehouse, and weather/driver data to explain *why* a metric moved (e.g. why
  deliveries are delayed in a specific zone). The LLM narrates correlations
  and figures that are computed in SQL/Spark — it does not calculate them
  itself.
- **GFR-6 — AI incident detection:** Anomaly detection on the Spark Streaming
  output (e.g. a warehouse backlog spike well above its normal baseline)
  triggers an LLM-generated incident summary with likely contributing causes.
- **GFR-7 — Data-dictionary RAG:** A second vector index over table
  descriptions and business/metric definitions (e.g. *"What exactly does
  SLA % mean in this system?"*) so the copilot and SQL agent use approved
  definitions instead of guessing.
- **GFR-8 — (Optional/advanced) AI Data Engineer Copilot:** A tool-calling
  agent that checks Airflow, Kafka, Spark, data-quality, and warehouse
  freshness status, and reports overall pipeline health in natural language
  (e.g. `Pipeline Health: DEGRADED — GPS pipeline 12-minute processing delay`).

### 13.2 Non-Functional Requirements

- **GNFR-1:** The LLM must never compute business metrics itself — every
  numeric answer must come from a tool call into SQL/Spark/the warehouse; the
  LLM only summarizes and explains retrieved results.
- **GNFR-2:** The text-to-SQL agent must be constrained to read-only queries
  against an approved schema/table allow-list (guardrails).
- **GNFR-3:** RAG answers must be traceable back to source documents
  (citations) for auditability.

### 13.3 GenAI Technology Stack

- **LLM:** OpenAI or Anthropic API
- **Agent orchestration:** LangGraph / LangChain
- **Vector DB:** Qdrant (or Pinecone)
- **Application layer:** FastAPI backend; Next.js/React/Tailwind frontend

### 13.4 Extended Module List

```text
01 — Data Ingestion              07 — Data Quality
02 — Batch ETL                   08 — GenAI RAG
03 — Lakehouse                   09 — Text-to-SQL
04 — Real-Time Streaming         10 — AI Operations Copilot
05 — Data Warehouse              11 — Automated Reports
06 — Operational Analytics       12 — Incident Analysis
```

## 14. Out of Scope

- Any real customer, order, or driver data from an actual company.
- Claiming Dubai government open data as the source of fictional order/driver records.
- Presenting this as a completed real freelance client engagement unless client work
  actually occurs.

## 15. Success Criteria

- End-to-end batch pipeline (Bronze → Silver → Gold → Warehouse → BI) runs on
  synthetic data at development scale.
- End-to-end streaming pipeline (GPS simulator → Kafka → Spark Streaming → Redis/Postgres → live dashboard) runs and updates in near real time.
- All seven dashboards/reports in §4.1 are functional against Gold-layer data.
- Data quality checks, schema-evolution handling, and lineage are demonstrable.
- Project can be defended in interviews for Data Engineer, Big Data Engineer,
  Analytics Engineer, and Data Platform Engineer roles.
- (Extended scope) If the GenAI layer (§13) is built: the Logistics AI Copilot
  answers natural-language questions using only tool-retrieved Gold-warehouse
  data (no hallucinated figures), the text-to-SQL agent is scoped to read-only
  approved tables, RAG answers cite source documents, and the daily brief /
  incident-detection features run on schedule via Airflow.
