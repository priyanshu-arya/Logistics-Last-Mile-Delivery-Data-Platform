# Prompts

Verbatim prompts from the project-scoping conversation that this platform's
requirements, features, and decisions were derived from.

- Source: https://chatgpt.com/share/6aae49c8-abfc-83e8-9b09-4a9ad7a7481b
- Title: "Design logistics platform"
- Captured: 2026-09-19

Full outputs from this conversation were distilled into [REQUIREMENTS.md](REQUIREMENTS.md),
[SKILL.md](SKILL.md), and [DECISIONS.md](DECISIONS.md).

---

## Prompt 1 — Initial project brief

```
Dubai Logistics & Last-Mile Delivery Data Platform

This one is particularly good for a freelance-style project.

Client:

> Logistics company with multiple warehouses and delivery fleets wants centralized operational reporting.

Sources:

Orders
GPS
Driver App
Warehouse
ERP
Fleet
Delivery Partner APIs

Build:

-  delivery SLA dashboard
-  driver performance
-  warehouse throughput
-  failed delivery analysis
-  vehicle utilization
-  route analytics
-  daily operational reports

Add a streaming layer for:

Driver GPS → Kafka → Spark Streaming → Operational Store

Then batch historical data through the lakehouse.
```

## Prompt 2 — Data sourcing question

```
from where we get data or did we need data?
```

## Prompt 3 — Approval to proceed with research

```
yes
```

(This approved the assistant researching and proposing the exact public
datasets/APIs and synthetic-data strategy captured in
[REQUIREMENTS.md §8](REQUIREMENTS.md#8-data-requirements--volumes) and
[DECISIONS.md](DECISIONS.md#d3).)

## Prompt 4 — Add a GenAI layer

A follow-up ChatGPT response proposed extending the platform with a GenAI
"Operations Copilot" layer — chat interface, text-to-SQL agent, RAG over
company docs and data-dictionary definitions, automated daily briefs,
root-cause analysis, streaming-incident detection, and an optional AI
Data Engineer copilot — built on top of the core Data Engineering platform
rather than as a separate project. The full response was distilled directly
into [REQUIREMENTS.md §13](REQUIREMENTS.md#13-genai-operations-copilot-layer-extended-scope),
[SKILL.md §9](SKILL.md#9-genai-operations-copilot-layer-extended-scope), and
[DECISIONS.md D11](DECISIONS.md#d11) rather than duplicated here in full.
