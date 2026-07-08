---
name: dt-bizevents-review
description: >-
  Re-runnable Dynatrace bizevents / business observability overview playbook.
  Produces a tenant-wide state-of-the-environment report covering volume,
  providers, event-types, data dictionary, correlation IDs, anomalies, and
  metric opportunities. Invoke when the user asks for a bizevents overview,
  business observability overview, or business-events review/report - e.g.
  "bizevents overview", "bizevents review", "run the bizevents playbook",
  "what business events are we collecting", or @dt-bizevents-review. Reads
  `dt-playbook-common` FIRST for Step 0 (context/folder confirmation, intent
  check, empty-tenant fast-path). Writes to
  `<context-folder>/event-overview-reports/bizevents-overview-<YYYY-MM-DD-HHMM>.md`.
  Do NOT use for explaining an existing query, business-process / funnel
  deep-dives (use `dt-business-process-review`), or log analysis (use
  `dt-logs-review`).
---

# 📓 Bizevents Environment Review — Playbook

> **Purpose:** Re-runnable recipe for producing a high-level bizevents state-of-the-environment report (data dictionary, anomalies, correlation IDs, metric opportunities) for any Dynatrace tenant.
> **Sibling output:** `<context-folder>/event-overview-reports/bizevents-overview-<YYYY-MM-DD-HHMM>.md` (e.g. `demo-live-readonly/event-overview-reports/bizevents-overview-2026-06-25-1820.md`).
> **Audience:** AI coding agents (Copilot, Claude, etc.) and the engineer driving them.

> 📖 **Read the `dt-playbook-common` skill first.** It owns the prerequisites, the Step 0 kickoff interview, intent confirmation, the per-workspace `.dt-playbook-mappings.yaml`, empty-tenant fast-path conventions, duplicate-snapshot logic, PowerShell quoting note, shared report style rules, and the self-improvement protocol. This skill only contains what is unique to bizevents.

---

## 🧷 Per-skill parameters (for `dt-playbook-common`)

| Parameter | Value |
| --- | --- |
| `<subfolder>` | `event-overview-reports/` |
| `<filename-stem>` | `bizevents-overview` |
| `<records>` (used in fast-path prompts) | `bizevents` |
| Discovery Query 1 (headline) | §1 below |
| Initial-overview queries (run when Query 1 > 0) | §2–6 below |
| Bucket-inventory query (empty-tenant fast path) | §10 below |
| Deep-dive queries (run after Step 2) | §7–9 below |

Full output path: `<context-folder>/event-overview-reports/bizevents-overview-<YYYY-MM-DD-HHMM>.md` (UTC, never overwritten).

### Extra prerequisites (beyond `dt-playbook-common`)

| Requirement | Notes |
| --- | --- |
| Token scopes: read access to `bizevents` (Grail query scopes) | A read-only context (e.g. `*-readonly`) is enough |

> 📚 **No per-skill domain skill beyond the common two.** The two skills the common scaffolding names (`dtctl` operator + `dt-dql-essentials`) are sufficient for every Discovery Query in this playbook. Load them before Step 1.

### Step 2 — Focus question wording (bizevents)

> *"Based on the overview above, are there specific `event.type`s or `event.provider`s you want the rest of the report to focus on? (Leave blank for full coverage of every notable provider.)"*

Present the top ~10 providers and top ~10 event types from the overview as selectable options. If the user picks a focus set, restrict §7 (per-provider schema samples) and §8 (duplication detector) to that set. If the user leaves it blank, cover the top 5 providers + any unusual ones (`*.temp`, `dynatrace.biz.*`, `gen_ai.*`, UUID-looking providers).

---

## 📋 Discovery Query Set

Run these in order. Each is intentionally small.

### 1. Headline numbers — volume, distinct types/providers (last 24 h)

```dql
fetch bizevents, from:-24h
| summarize total = count(),
            distinctTypes = countDistinct(event.type),
            distinctProviders = countDistinct(event.provider)
```

### 2. Top event types (last 24 h, limited)

```dql
fetch bizevents, from:-24h
| summarize count = count(), by:{event.type}
| sort count desc
| limit 50
```

### 3. Top providers + cardinality survey (1 h, fast)

```dql
fetch bizevents, from:-1h
| fieldsSummary event.type, event.provider, event.category, event.id, dt.system.bucket
```

### 4. Ingest source split

```dql
fetch bizevents, from:-1h
| summarize cnt = count(), by:{dt.openpipeline.source}
| sort cnt desc
```

### 5. Correlation-ID coverage

```dql
fetch bizevents, from:-1h
| fieldsSummary trace_id, span_id, traceId, spanId, dt.trace_id, user.id, session.id
```

### 6. Data-quality counters

```dql
fetch bizevents, from:-1h
| summarize cnt = count(),
            distinctEventIds = countDistinct(event.id),
            missingId       = countIf(isNull(event.id)),
            missingProvider = countIf(isNull(event.provider)),
            missingType     = countIf(isNull(event.type)),
            withTrace       = countIf(isNotNull(trace_id))
```

### 7. Sample one record per major provider (schema discovery)

For each provider that surfaces in §3, pull a single record to learn its schema:

```dql
fetch bizevents, from:-15m
| filter event.provider == "<provider>"
| limit 1
```

Do this for at least the top 5 providers plus any unusual ones (e.g. `*.temp`, `dynatrace.biz.*`, `gen_ai.*`).

### 8. Duplication detector (provider/type pairs)

```dql
fetch bizevents, from:-1h
| filter matchesValue(event.provider, "<prefix>.*")
| summarize cnt = count(), by:{event.provider, event.type}
```

Look for **near-identical counts across paired types** (e.g. `foo.bar` vs `foo.nginx.bar`) or **paired providers** with the same event types (e.g. `acme.x` and `acme.x.temp`).

### 9. Optional follow-ups (run if first pass shows interesting signals)

- High-cardinality check on a suspect field:
  ```dql
  fetch bizevents, from:-15m | summarize c = countDistinct(<field>)
  ```
- Per-event-type ingest-source breakdown (proves which pipeline produced a given event):
  ```dql
  fetch bizevents, from:-1h
  | summarize cnt = count(), by:{event.type, dt.openpipeline.source}
  | sort cnt desc
  ```
- OpenPipeline rule attribution:
  ```dql
  fetch bizevents, from:-15m | summarize cnt = count(), by:{dt.openpipeline.pipelines}
  ```

### 10. Bucket inventory (used by the empty-tenant fast path)

Confirms the `bizevents` table is provisioned, regardless of whether anything has been written to it. Distinguishes "no producers" from "no bucket".

```dql
fetch dt.system.buckets
| filter dt.system.table == "bizevents"
| fields name, display_name, retention_days, records, estimated_uncompressed_bytes
```

If this returns at least one row, the table exists and the absence of data is an ingest-side issue (no producers, paused producer, or pipeline drop rule). If it returns zero rows, the tenant doesn't have business-events provisioned at all.

---

## 🔍 What to Look for in the Results

Map findings to the report sections using this checklist.

### Misconfigurations / anomalies
- 🚨 **Duplicate events** — paired types with identical counts (service + nginx, app + sidecar, `.temp` twins).
- 🚨 **Missing `event.id`** — usually all `data_extraction`-sourced events; breaks idempotency.
- 🚨 **`event.category` mostly empty/null** — inconsistent population across sources.
- 🔒 **PII in payload fields** — look for raw credit-card numbers, CVV, email, addresses in fields like `fullRequestBody`, `payload`, `response`, `gen_ai.prompt`.
- 🪪 **Always-null ID fields** — `user.id`, `session.id`, `dt.trace_id` etc. that are referenced but never populated.
- 🧹 **Dormant providers** — < ~5 events/hour. Candidate to decommission.
- 🆔 **Unexpected high cardinality** — anything other than `event.id` / trace IDs / business UUIDs being near-unique is suspicious.

### Schema drift
- Same concept under multiple keys: `trace_id` vs `trace.id` vs `traceparent`; `orderId` vs `order.id`; `span_id` vs `span.id`.
- Capture all variants in the dictionary so consumers know to `coalesce()`.

### Correlation IDs to surface
- Always: `trace_id`, `span_id`, `dt.entity.*`, `dt.smartscape.*`, `k8s.*`, `service.name`.
- Business: `orderId`, `sessionId`, `userId`, `claimId`, etc. — discover from samples.
- GenAI: `gen_ai.model`, `gen_ai.system`, `gen_ai.type`.

### Metric / SLO opportunities
- Funnels (e.g. checkout success → ship → deliver), success/failure ratios, throughput-per-business-unit, FinOps (LLM tokens, carbon), guardian pass-rate. Suggest the DQL sketch for each.

---

## 🧱 Output Document Structure

Save to `<context-folder>/event-overview-reports/bizevents-overview-<YYYY-MM-DD-HHMM>.md` (UTC). See the `dt-playbook-common` skill's *Shared style rules* section for filename/overwrite rules.

Section skeleton:

```
# 🧭 Bizevents Overview — <tenant short name>
> Snapshot date, context name, scope of windows used.

## 📊 Headline Numbers (24 h)
## 🏷️ Top Providers
## 🗺️ Event Families (Logical Domains)
## 📘 Data Dictionary — Common / Shared Fields
   ### 🔗 Correlation IDs
   ### 🔄 Naming Inconsistencies
## 🚨 Misconfigurations & Anomalies   (numbered, each with Impact + Fix)
## 💡 Opportunities — Metrics & Correlations Worth Building
   ### 📈 Candidate metrics
   ### 🔗 Correlation chains
   ### 🧱 Suggested follow-ups
## 🧪 Quick Verification Queries
```

### 🪹 Empty-tenant report structure (bizevents)

When the empty-tenant fast path concludes that no bizevents are being ingested, write a much shorter document with this skeleton:

```
# 🧭 Bizevents Overview — <tenant short name>
> Snapshot date, context name, windows checked (e.g. 24 h / 7 d / 30 d / 90 d), verdict.

## 📊 Headline Numbers
   - One row per window checked, all showing `total = 0`.
   - Earliest/latest = null.

## 🗄️ Bucket Configuration
   - Output of §10 (bucket name, retention, record count, bytes).
   - State whether the bucket exists and is healthy.

## 🚨 Findings
   ### 1. 🛑 Tenant has no bizevent ingest path active
       - **Observation:** zero rows across the windows checked.
       - **Impact:** nothing to dashboard/alert/correlate on.
       - **Fix:** list the four standard ingest paths (OneAgent HTTP capture rule, RUM `sendBizEvent`, public `/api/v2/bizevents/ingest`, OpenPipeline log-to-bizevent promotion).

## 🧪 Quick Re-check Queries
   - Three short DQL snippets the user can paste once ingest is enabled to confirm flow.
```

Skip the data dictionary, correlation IDs, anomalies (beyond #1), and opportunities sections entirely — they have no observations to populate.

---

## ✅ Final Self-Review Checklist

- [ ] Headline numbers, top providers, top types, ingest-source split all present.
- [ ] Every provider mentioned in tables actually appeared in a query result (no hallucination).
- [ ] Each anomaly has a concrete fix.
- [ ] Correlation IDs section names the actual field variants observed in this tenant (don't copy verbatim from another tenant's report).
- [ ] Data dictionary lists coverage (% populated) for fields where it is interesting.
- [ ] At least one PII / sensitive-payload check was performed and explicitly stated (positive or negative).
- [ ] Document renders cleanly in a markdown preview (tables aligned, no broken links).
- [ ] Sensitive values (real customer IDs, real CC numbers, real emails) are **not** pasted into the document — paraphrase or redact.

---

## 🔗 Related References

- `dt-playbook-common` skill — shared scaffolding (Step 0 interview, mappings file, style rules).
- `dt-dql-essentials` skill — DQL syntax, pitfalls, operator reference.
- Sibling skill: `dt-business-process-review` — depth-first follow-up once you know which providers/types matter.
- Dynatrace docs: *Business Events*, *OpenPipeline*, *Schema `builtin:bizevents.http.incoming`*.
