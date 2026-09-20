# Features to Build

Feature backlog derived from [PROMPT.md](PROMPT.md), organized by platform layer.
Each item maps to requirements in [REQUIREMENTS.md](REQUIREMENTS.md).

## 1. Synthetic Data Generator (`data-generator/`)
- [ ] `customers.py` — generate customers with zone + lat/long
- [ ] `drivers.py` — generate drivers with home zone, hire date, employment status
- [ ] `vehicles.py` — generate vehicles with type, capacity, fuel type
- [ ] `warehouses.py` — generate Dubai warehouses (e.g. `WH-DXB-001`) on real coordinates
- [ ] `orders.py` — generate orders linked to customers/warehouses with SLA fields
- [ ] `deliveries.py` — generate deliveries with realistic failure relationships
  (traffic + distance + peak hour → delay probability; not pure `random()`)
- [ ] `routes.py` — generate planned routes over real Dubai road network (OSRM)
- [ ] `delivery_events.py` — generate delivery status/lifecycle events
- [ ] `gps_simulator.py` — continuously emit GPS events along a route in real time

## 2. Streaming Pipeline
- [ ] Kafka topics: `driver-gps-events`, `delivery-events`, `order-events`,
      `vehicle-events`, `warehouse-events`, `partner-events`
- [ ] Kafka producer wired to `gps_simulator.py`
- [ ] Spark Structured Streaming job: validation → deduplication → windowed aggregation
- [ ] Write streaming results to Redis/PostgreSQL operational store
- [ ] Real-time metrics: current driver location, speed-violation detection,
      delivery-delay calculation, route-deviation detection, driver idle-time detection,
      warehouse backlog
- [ ] Checkpointing and late/duplicate-event handling

## 3. Batch / Lakehouse Pipeline
- [ ] Bronze ingestion of ERP, WMS, Driver App, Fleet, Partner API sources into S3
- [ ] Silver-layer cleaning: dedup, malformed-GPS handling, missing-timestamp handling,
      timezone normalization, schema-change handling, late-arriving events,
      invalid driver/vehicle ID handling
- [ ] Gold-layer aggregates: `delivery_sla_daily`, `driver_performance`,
      `warehouse_throughput`, `vehicle_utilization`, `route_performance`,
      `failed_delivery_analysis`, `daily_operations_summary`
- [ ] Airflow DAGs orchestrating the batch pipeline end to end

## 4. Dimensional Warehouse Model
- [ ] Fact tables: `fact_orders`, `fact_deliveries`, `fact_delivery_events`,
      `fact_gps_events`, `fact_route_execution`, `fact_vehicle_usage`,
      `fact_warehouse_operations`
- [ ] Dimension tables: `dim_customer`, `dim_driver`, `dim_vehicle`, `dim_warehouse`,
      `dim_route`, `dim_delivery_partner`, `dim_date`, `dim_location`
- [ ] Load into chosen warehouse (Snowflake / BigQuery / Redshift)

## 5. Dashboards & Reports
- [ ] Delivery SLA dashboard (orders, delivered, SLA %, failed — by warehouse/hour)
- [ ] Driver performance dashboard
- [ ] Warehouse throughput dashboard (inbound → picking → packing → dispatch)
- [ ] Failed delivery analysis (by warehouse, driver, route, time, location, partner, reason)
- [ ] Vehicle utilization dashboard (utilization %, km driven, idle time, fuel, maintenance)
- [ ] Route analytics (planned vs. actual distance/duration/deviation)
- [ ] Daily operations report, auto-generated and scheduled via Airflow

## 6. Delivery Partner API Simulator
- [ ] FastAPI service exposing `GET /deliveries/{id}`, `POST /delivery-events`,
      `GET /tracking/{id}`
- [ ] Airflow task to periodically pull this API into the batch pipeline

## 7. Data Quality & Observability
- [ ] Great Expectations or Soda checks (NOT NULL, valid FK, GPS bounds, valid timestamp)
- [ ] Schema-evolution handling (e.g. GPS API v1 → v2 without breaking pipelines)
- [ ] Lineage tracking from source → Bronze → Silver → Gold → Warehouse → BI
- [ ] Monitoring: Kafka consumer lag, Airflow failures, Spark job health, data
      freshness, pipeline latency, record counts, data-quality failure counts

## 8. Infrastructure
- [ ] Dockerized local dev environment (Kafka, Spark, Postgres, Redis, Airflow)
- [ ] Terraform for AWS deployment (S3, MSK, EMR, warehouse)
- [ ] CI/CD via GitHub Actions

## 9. GenAI Operations Copilot Layer (extended scope)
See [REQUIREMENTS.md §13](REQUIREMENTS.md#13-genai-operations-copilot-layer-extended-scope)
and [DECISIONS.md D11](DECISIONS.md#d11).

- [ ] FastAPI backend + Next.js/React/Tailwind chat UI ("Logistics AI Copilot")
- [ ] Text-to-SQL agent scoped to read-only queries against approved Gold tables
- [ ] Tool-calling flow: LLM → intent detection → SQL agent → warehouse → results → LLM explanation
- [ ] Operational RAG: chunk/embed company docs (SLA policy, driver handbook,
      warehouse SOP, failed-delivery policy, vehicle maintenance policy,
      escalation policy, partner API docs) into a vector DB (Qdrant/Pinecone)
- [ ] Data-dictionary RAG: vector index over table/metric definitions so
      agents don't guess business definitions (e.g. "what does SLA % mean")
- [ ] Automated daily operations brief: Airflow → Gold metrics → LLM →
      narrative summary → email/Slack
- [ ] AI root-cause analysis: LLM narrates correlations across
      orders/GPS/routes/traffic/warehouse/weather computed in SQL/Spark
- [ ] AI incident detection: anomaly detection on Spark Streaming output
      (e.g. warehouse backlog spike) → LLM-generated incident summary
- [ ] (Optional/advanced) AI Data Engineer Copilot: tool-calling agent
      reporting Airflow/Kafka/Spark/data-quality/warehouse health in natural language
- [ ] Guardrails: LLM never computes metrics itself; all figures come from
      tool calls; RAG answers include source citations
