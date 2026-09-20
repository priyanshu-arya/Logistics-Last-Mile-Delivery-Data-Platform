# Decisions

Architecture and scope decisions agreed during project scoping
(see [PROMPT.md](PROMPT.md) for the source conversation). Each entry records the
decision, the alternative considered, and why it was chosen.

---

## D1 — Project positioning: portfolio project, not a claimed client engagement
**Decision:** Present this as an independent/portfolio Data Engineering project
built around a fictional Dubai logistics client, e.g. "Dubai Logistics &
Last-Mile Delivery Data Platform — Freelance-style / Portfolio Project."
**Rejected alternative:** Presenting it as a completed real freelance engagement
for an actual Dubai company.
**Why:** No real client data or engagement exists. Misrepresenting it as a real
client project would be dishonest on a resume/portfolio and hard to defend in an
interview. The business requirements, architecture, and synthetic-data
methodology can still be explained honestly and are what interviewers actually
probe.

## D2 — Hybrid batch + real-time architecture
**Decision:** Build both a streaming path (driver GPS/delivery events →
Kafka → Spark Structured Streaming → Redis/PostgreSQL → live dashboard) and a
batch path (ERP/WMS/Fleet/Partner APIs → S3 → PySpark → Delta Lake → Warehouse →
BI), rather than either alone.
**Rejected alternative:** Batch-only ETL pipeline.
**Why:** A batch-only pipeline reads as a generic ETL demo. The hybrid design
lets the project support live operational questions (driver location, SLA
breach risk in real time) as well as historical/analytical reporting, and is
significantly stronger for Data/Big Data/Platform Engineer interviews.

## D3 — Data strategy: hybrid public + synthetic + simulated, not one downloaded dataset
**Decision:** Combine (a) real public data — OpenStreetMap road/geo data and
Dubai Open Data/Dubai Pulse reference data; (b) a public logistics dataset
(e.g. Kaggle 25K delivery-logistics dataset) used only to inform realistic
distributions/schema, never relabeled as Dubai client data; (c) a self-built
synthetic generator for customers, orders, drivers, vehicles, warehouses, and
deliveries; and (d) a self-built real-time GPS/event simulator feeding Kafka.
**Rejected alternatives:**
- Sourcing real operational data from an actual logistics company (not
  accessible/appropriate for a portfolio project).
- Downloading a single Kaggle CSV and loading it as-is (reads as a toy project;
  no data-integration problem to demonstrate).
- Renaming an India-based dataset's records to Dubai (fabricates provenance).
**Why:** Companies cannot publish real customer/order/driver data, so synthetic
data is the normal and expected approach — but grounding it in real Dubai
geography and realistic public-data distributions makes it far more credible
than a single generic dataset, and building the generator is itself a
demonstrable engineering artifact.

## D4 — Medallion lakehouse for batch data
**Decision:** Structure all batch-ingested data through Bronze (raw) → Silver
(cleaned/standardized) → Gold (business-ready) layers in Delta Lake on S3.
**Why:** Standard, defensible lakehouse pattern; gives a natural place to
demonstrate data-quality handling (dedup, malformed GPS, missing timestamps,
timezone normalization, schema evolution, late-arriving events) between Silver
and Gold.

## D5 — Star-schema dimensional model for the warehouse layer
**Decision:** Model Gold-layer/warehouse data as fact tables (`fact_orders`,
`fact_deliveries`, `fact_delivery_events`, `fact_gps_events`,
`fact_route_execution`, `fact_vehicle_usage`, `fact_warehouse_operations`)
against shared dimensions (`dim_customer`, `dim_driver`, `dim_vehicle`,
`dim_warehouse`, `dim_route`, `dim_delivery_partner`, `dim_date`, `dim_location`).
**Why:** Gives the project a proper analytical-warehouse component distinct
from the raw lakehouse, and supports the required dashboards directly.

## D6 — Fictional third-party delivery-partner API via self-built FastAPI service
**Decision:** Simulate delivery-partner integrations with a small FastAPI
service (`GET /deliveries/{id}`, `POST /delivery-events`, `GET /tracking/{id}`)
pulled periodically by Airflow, rather than trying to integrate a real
delivery-partner API.
**Why:** No real third-party delivery API is accessible for a portfolio project;
a simulator still demonstrates the API → ingestion → lake → transformation
pattern end to end.

## D7 — Two data-volume tiers: development vs. production simulation
**Decision:** Generate at development scale (10K customers / 100K orders / 1M
GPS events) for local runs, and separately generate — but not necessarily
run end-to-end locally — a production-simulation scale (500K customers / 10M+
orders / 100M+ GPS events) to demonstrate the architecture scales.
**Why:** Full production-scale volumes are unnecessary and costly to run
locally; the goal is to prove the pipeline design and configuration scale,
not to operate at that scale on a laptop.

## D8 — Technology stack
**Decision:**
- Processing: Python, PySpark, Apache Kafka, Spark Structured Streaming, Apache Airflow
- Storage: AWS S3, Delta Lake, PostgreSQL
- Warehouse: one of Snowflake / BigQuery / Redshift
- Serving: Redis, PostgreSQL
- BI: Power BI or Apache Superset
- Infrastructure: Docker, Terraform, AWS, GitHub Actions
- Data quality: Great Expectations or Soda
**Why:** This is the standard, in-demand toolset for Data Engineer / Big Data
Engineer / Analytics Engineer / Data Platform Engineer roles, and each choice
maps directly to a requirement (streaming → Kafka/Spark Streaming; lakehouse →
Delta Lake/S3; orchestration → Airflow; analytics warehouse → Snowflake/BigQuery/
Redshift).

## D9 — Route realism via real road network
**Decision:** Use real Dubai locations (Downtown Dubai, Jumeirah, Business Bay,
Dubai Marina, Deira, Bur Dubai, Al Quoz, Jebel Ali, etc.) and a routing engine
(OSRM) over OpenStreetMap data to generate routes and GPS trajectories, rather
than random lat/long generation.
**Why:** Random coordinates would make route-deviation and GPS-simulation
features meaningless; real road geometry is required for those metrics to
tell a believable story.

## D10 — Realistic (non-random) failure modeling
**Decision:** Model delivery failures with causal relationships (e.g. higher
traffic + distance + peak hour → higher delay probability; failed first
attempt → scheduled second attempt) instead of assigning failure status via
uniform `random()`.
**Why:** Needed so failed-delivery analytics (§4.5 of REQUIREMENTS.md) actually
surfaces meaningful patterns rather than noise, since that is one of the
platform's core analytical use cases.

## D11 — Add a GenAI Operations Copilot layer on top of the core DE platform
**Decision:** Extend the platform with a GenAI layer (chat copilot,
text-to-SQL, RAG over operational docs and data-dictionary, automated daily
briefs, root-cause analysis, streaming-incident detection, and an optional
AI Data Engineer copilot) built on the existing Gold warehouse and streaming
outputs, rather than treating GenAI/Agentic AI as a separate project.
**Rejected alternative:** Building the Data Engineering platform and a
GenAI/agentic project as two disconnected portfolio pieces.
**Why:** A bolted-on chatbot UI with no real data behind it is a weak GenAI
demo, and a DE-only platform doesn't showcase GenAI/agentic skills at all.
Wiring the GenAI layer directly into the Gold warehouse and streaming
pipeline lets one project credibly demonstrate both skill sets together,
which is a stronger and more differentiated portfolio story. See
[REQUIREMENTS.md §13](REQUIREMENTS.md#13-genai-operations-copilot-layer-extended-scope)
and [SKILL.md §9](SKILL.md#9-genai-operations-copilot-layer-extended-scope).

**Constraint carried over from D10 and D5:** the LLM must never invent
business numbers — every metric it reports must come from a tool call into
the SQL warehouse or Spark output; the LLM's job is retrieval-grounded
explanation, not computation.
