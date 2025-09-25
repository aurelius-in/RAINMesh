![RAIN Arcade Logo](RAINarcade.png)

# *Multi-tenant, federated RAG with policy-enforced retrieval and signed evidence logs.*

> Production-ready backend for retrieval over distributed sources without centralizing data. Ships with agents for planning, policy explanation, provenance, freshness, redaction, re-ranking, verification, cost control, drift monitoring, and CDC-driven freshness.

---

## Table of Contents

* [Why RAINMesh](#why-rainmesh)
* [Architecture at a Glance](#architecture-at-a-glance)
* [Core Capabilities](#core-capabilities)
* [Agents](#agents)
* [API](#api)
* [Content Packs](#content-packs)
* [Security & Governance](#security--governance)
* [Multi-Tenancy](#multi-tenancy)
* [Federated Retrieval Modes](#federated-retrieval-modes)
* [Quality, Eval, & Drift](#quality-eval--drift)
* [Observability](#observability)
* [Deploy (Local/Cloud)](#deploy-localcloud)
* [Examples](#examples)
* [Roadmap](#roadmap)
* [Repo Layout](#repo-layout)
* [License](#license)

---

## Why RAINMesh

Enterprises often can’t move data to a single lake or search index. RAINMesh lets you run **Retrieval-Augmented Generation (RAG)** across many tenants and many sources—some pre-indexed, some queried live—while enforcing policy, preserving provenance, and emitting verifiable evidence for every answer.

**Highlights**

* Federated retrieval across SharePoint, Confluence, SQL, S3/Blob, Google Drive, Jira, Git, and more.
* Hard tenant isolation at auth, storage, and index layers (namespaces, keys, ABAC policies).
* Evidence-first design: every answer comes with citations and a signed retrieval/policy log (WORM).
* Built-in agents to plan, verify, redact, re-rank, explain policy, and govern cost.

---

## Architecture at a Glance

```
Multi-Tenant RAG (Federated / Late-Bound Retrieval)

            ┌────────────────────────────────────────────────────────┐
            │                    CONTROL PLANE                       │
            │  Tenant Registry • Content Packs • Eval Harness        │
            │  Policy (OPA) • Telemetry/Billing • Feature Flags      │
            └────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────┐   TLS/JWT   ┌──────────────┐   policy/ABAC   ┌───────────────┐
│  Client  │ ──────── ──▶│ API Gateway │ ───────────────▶│ Orchestrator  │
│ (UI/SDK) │             │  (AuthN/Z)   │                 │  (LangGraph)  │
└──────────┘             └──────────────┘                 └─────┬─────────┘
                                                                │
                                                ┌───────────────┴────────────────┐
                                                │    Retrieval Fan-out Layer     │
                                                │ (per-tenant policy & routing)  │
                                                └───────────────┬────────────────┘
                                                                │
                    ┌──────────────────────────┬────────────────┴──────────────────┐
                    │                          │                                   │
        Pre-indexed, per-tenant            Pre-indexed, per-tenant          Live, late-bound
        vector/search shards               vector/search shards             source adapters
        (A) pgvector/ES/Weaviate           (B) Pinecone/ES/OpenSearch       (C) SQL/SharePoint/
        + KMS + PII tags                   + KMS + PII tags                 Confluence/APIs
                    │                          │                                   │
                    └──────────────┬───────────┴───────────────┬───────────────────┘
                                   ▼                           ▼
                              ┌────────────────────────────────────────────┐
                              │   Aggregator • Dedup • Re-rank (CrossEnc)  │
                              └───────────────────────┬────────────────────┘
                                                      ▼
                                      ┌────────────────────────────────┐
                                      │  Guarded Generation (LLM)      │
                                      │  Grounding + Citations         │
                                      │  PII-aware Redaction           │
                                      └───────────────┬────────────────┘
                                                      ▼
                                      ┌────────────────────────────────┐
                                      │  Answer + Evidence + Signed    │
                                      │  Retrieval/Policy Log (WORM)   │
                                      └────────────────────────────────┘
```

---

## Core Capabilities

* **Federated RAG**: Mix pre-indexed shards and live adapters. No mandatory data centralization.
* **Policy-Enforced Retrieval**: ABAC with OPA at plan time and retrieval time.
* **Verifiable Evidence**: Hashes of retrieved spans, prompts, and model I/O; signed logs.
* **Freshness Awareness**: Per-source recency, CDC integration, and answer-time freshness labels.
* **PII Hygiene**: Redaction before embedding and again at answer time under policy.
* **Per-Tenant Quality Loops**: Golden sets, offline/online eval, auto-tuned reranking.
* **Cost Governance**: Budget-aware path selection, model tiers, and quotas.

---

## Agents

RAINMesh ships with modular agents you can enable per tenant. Names align to the RAIN brand.

### Must‑add

* **RainScout (QueryPlanner)** — Decomposes a query into sub-tasks and routes to the best sources (indexed vs live) to balance recall, precision, and latency.
* **PolicyLens (PolicyExplainer)** — Surfaces OPA decisions in plain English and highlights masked/redacted spans.
* **ProofWeave (EvidenceNotary)** — Hashes chunks + prompts + IO; writes WORM log and optional chain anchor.
* **RainGauge (FreshnessSentinel)** — Tracks recency per source; reweights stale shards and annotates answers with freshness.
* **CleanStream (PIIRedactor)** — Policy-aware chunking/redaction pre-embedding and pre-response.

### Strong Should‑have

* **TuneDeck (RerankStudio)** — Auto-tunes top‑k, chunk size, and cross-encoder choice per tenant using bandits over golden sets.
* **SchemaMapper** — Extracts typed fields (contracts, SKUs, clauses) via tool-calling to enable structured answers.
* **TruthCheck (CiteVerifier)** — Re-reads cited spans, flags hallucinated claims, and returns a confidence delta.
* **BudgetBar (CostGovernor)** — Budget-aware planner that chooses models and retrieval paths per SLA.

### Accelerators

* **Runways (Playbooks)** — Curated task graphs for RFP compare, policy diff, renewal analysis; step-level policies.
* **SkewWatch (DriftWatcher)** — Monitors embedding/prompt drift and triggers targeted evals.
* **PulseSync (CDCResolver)** — Subscribes to change streams and schedules hot re-index jobs.

**LangGraph flow (simplified)**

```python
state = {query, tenant, plan, hits, citations, policy_log, budget}

-> RainScout.plan(state)
-> PolicyLens.preview(state)
-> FederatedRetriever.fanout(state)
-> CleanStream.scrub_chunks(state)
-> TuneDeck.rerank(state)
-> TruthCheck.check(state)
-> BudgetBar.select_models(state)
-> GuardedGenerator.answer(state)
-> ProofWeave.seal(state)
-> RainGauge.annotate(state)
return answer, citations, evidence_log_uri, costs
```

---

## API

Minimal REST surface area. gRPC equivalents optional.

### `POST /v1/query`

Request

```json
{
  "tenant_id": "acme",
  "query": "What’s our FY24 renewal policy for tier-2 customers?",
  "constraints": { "sources": ["sharepoint", "sql:analytics"] },
  "top_k": 8,
  "budget": { "max_usd": 0.12, "latency_sla_ms": 2500 }
}
```

Response

```json
{
  "answer": "Tier-2 renewals follow...",
  "citations": [
    {"source": "sharepoint", "uri": "sp://sites/policies/...#L120-L168", "hash": "b3f..."},
    {"source": "sql:analytics", "table": "renewals", "query_id": "q_7e1..."}
  ],
  "freshness": {"sharepoint": "2025-09-18T14:22Z", "sql:analytics": "2025-09-25T02:09Z"},
  "evidence_log_uri": "s3://rainmesh/logs/acme/2025/09/25/a1b2c3.worm.json",
  "latency_ms": 1840,
  "tokens": {"prompt": 1632, "completion": 214}
}
```

### `POST /v1/ingest`

Attach connectors and ingest rules via content packs.

```json
{
  "tenant_id": "acme",
  "connector": "sharepoint",
  "config": {"site_url": "https://acme.sharepoint.com/sites/policies"},
  "chunking": {"size": 900, "overlap": 120},
  "pii_tags": ["email", "address"]
}
```

### `POST /v1/content-packs`

```json
{
  "tenant_id": "acme",
  "pack_name": "acme_policies_v1",
  "connectors": ["sharepoint", "sql:analytics"],
  "prompts": {"qa": "..."},
  "guardrails": {"blocked_terms": ["ssn", "salary"], "max_tokens": 600},
  "schemas": {"contract": {"fields": ["party", "term", "renewal"]}}
}
```

### `POST /v1/plan`

Returns planned sub-tasks, expected cost, and freshness outlook.

### `POST /v1/verify`

Re-verifies citations and returns a confidence delta and any red flags.

### `POST /v1/playbooks/{name}:run`

Executes a curated, policy-gated workflow. Example: `policy_diff`.

### `GET /v1/tenants/{id}/quality`

Returns eval scores, drift, rerank settings, and freshness metrics.

---

## Content Packs

Declarative bundles that define connectors, chunking, prompts, guardrails, and schemas for a tenant.

Example (YAML)

```yaml
name: acme_policies_v1
connectors:
  - type: sharepoint
    site_url: https://acme.sharepoint.com/sites/policies
  - type: sql
    dsn: analytics-rw
chunking:
  size: 900
  overlap: 120
pii:
  tags: [email, address]
  redact: true
prompts:
  qa: |
    You are an enterprise policy assistant. Cite sources and include section numbers.
guardrails:
  blocked_terms: [ssn, salary]
  max_tokens: 600
schemas:
  contract:
    fields: [party, term, renewal]
```

---

## Security & Governance

* **AuthN/Z**: JWT at the edge, ABAC via OPA. Per-tenant namespaces, keys, and secrets.
* **Data at Rest**: KMS-backed encryption for indexes, object storage, and logs.
* **Data in Motion**: TLS everywhere, VPC peering/private links to enterprise sources.
* **Policy Enforcement**: Applied at plan, retrieval, and generation with signed decisions.
* **Evidence**: Immutable WORM logs (object lock) with optional external anchoring.
* **PII Hygiene**: Redaction prior to embedding; secondary pass on generated output.

---

## Multi-Tenancy

* **Isolation**: Per-tenant DB schemas or clusters; per-tenant index shards/namespaces.
* **Quotas & Billing**: Usage metering per tenant with budgets, alerts, and throttles.
* **Customization**: Content packs and agent toggles per tenant.

---

## Federated Retrieval Modes

* **Federated Indexing**: Create embeddings near sources; store vectors in per-source indexes. Central catalog holds only pointers and minimal metadata.
* **Late-Bound Retrieval**: For sources that can’t be pre-indexed, call native search at query time, embed on the fly or from a short-lived cache, then re-rank.
* **Hybrid**: Mix both per request. The planner chooses based on policy, freshness, and budget.

---

## Quality, Eval, & Drift

* **Golden Sets per Tenant**: Curated QA pairs with authoritative citations.
* **Offline & Online Eval**: Measure hit-rate, MRR@k, answer fidelity, citation accuracy.
* **Auto-Tuning**: TuneDeck adjusts chunking/top‑k/reranker using bandits.
* **Drift Monitoring**: SkewWatch tracks embedding and prompt drift, triggers eval jobs.

---

## Observability

* **Tracing**: OpenTelemetry spans for plan, retrieval, rerank, generation, verify, and seal.
* **Metrics**: P50/P90 latency, hit ratios, verification deltas, redaction counts, unit cost.
* **Logging**: Structured logs with correlation IDs; evidence logs are immutable (WORM).

---

## Deploy (Local/Cloud)

**Local (docker-compose)**

```bash
docker compose up -d
```

Services: API Gateway, Orchestrator, Postgres (pgvector), OPA, MinIO (S3-compatible), Redis, Optional: Elasticsearch, Weaviate.

**Cloud**

* Container orchestration (EKS/AKS/GKE), managed Postgres, managed OpenSearch or Pinecone, object storage with Object Lock, OPA bundle server, private connectivity to sources.

---

## Examples

**Create a tenant**

```bash
rainmesh tenants create --name acme
```

**Attach a content pack**

```bash
rainmesh packs add --tenant acme --file ./packs/acme_policies_v1.yaml
```

**Run a query**

```bash
curl -X POST $API/v1/query \
  -H 'Authorization: Bearer $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "tenant_id":"acme",
    "query":"Summarize renewal clauses for tier-2 customers with citations.",
    "constraints":{"sources":["sharepoint","sql:analytics"]},
    "top_k":8,
    "budget":{"max_usd":0.12,"latency_sla_ms":2500}
  }'
```

**Verify citations**

```bash
curl -X POST $API/v1/verify -d '{"answer_id":"a1b2c3"}'
```

---

## Repo Layout

```
/                         # monorepo root
 ├─ api/                  # REST/gRPC surfaces, auth, request/response schemas
 ├─ orchestrator/         # LangGraph graphs and agent nodes
 ├─ agents/               # RainScout, PolicyLens, ProofWeave, RainGauge, CleanStream, TuneDeck, TruthCheck, BudgetBar, Runways, SkewWatch, PulseSync
 ├─ packs/                # Example content packs (YAML)
 ├─ connectors/           # SharePoint, Confluence, SQL, S3/Blob, GDrive, Jira, Git, etc.
 ├─ retrievers/           # Indexed + live adapters, aggregator, dedup, cross-enc re-rank
 ├─ policy/               # OPA policies, bundle manifests, test vectors
 ├─ eval/                 # Golden sets, eval harness, offline/online jobs
 ├─ observability/        # OTel, metrics, dashboards, log schemas
 ├─ infra/                # Docker compose, IaC, KMS/key policies, object lock config
 └─ docs/                 # Additional diagrams and SRE runbooks
```

---

## Roadmap

* gRPC streaming and server-sent events for partial results.
* Retrieval-time ABAC hints from source systems to shrink candidate sets.
* Query understanding improvements (temporal reasoning, entity linking).
* Hybrid rerank that blends BM25, vector, and learned re-rank scores per tenant.

---

## License

TBD
