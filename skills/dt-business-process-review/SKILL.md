---
name: dt-business-process-review
description: >-
  Re-runnable Dynatrace business-process deep-dive playbook. Given one or more
  event.providers and a chosen set of event.types, characterizes how those
  events form a process, surfaces correlation IDs that stitch the steps,
  flags misses/precautions, and recommends business-focused metrics at both
  process and per-event level. Depth-first companion to `dt-bizevents-review`
  (breadth-first). Invoke for business-process review/detail, process funnel
  analysis, or business-process metric design - e.g.
  "business process review", "process detail report", "process funnel",
  "business process metrics", or @dt-business-process-review. Reads
  `dt-playbook-common` FIRST for Step 0, then runs a two-part Step 2 (pick
  providers, pick event.types) before deep-dive queries. Writes to
  `<context-folder>/process-detail-reports/business-process-detail-<YYYY-MM-DD-HHMM>.md`.
  Do NOT use for tenant-wide bizevents inventory (use `dt-bizevents-review`
  first) or log analysis (use `dt-logs-review`).
---

# 🧬 Business Process Review — Playbook

> **Purpose:** Re-runnable recipe for producing a *focused* business-process deep-dive report — given one or more `event.provider`s and a chosen set of `event.type`s, characterize how those events **mesh together to form a process**, surface the correlation IDs that stitch the steps, flag the misses / precautions, and recommend **business-focused** metrics at both the process level and the per-event level.
> **Sibling output:** `<context-folder>/process-detail-reports/business-process-detail-<YYYY-MM-DD-HHMM>.md` (e.g. `busobs-prod-readonly/process-detail-reports/business-process-detail-2026-06-25-2140.md`).
> **Audience:** AI coding agents (Copilot, Claude, etc.) and the engineer driving them.

> 📖 **Read the `dt-playbook-common` skill first.** It owns the prerequisites, the Step 0 kickoff interview, intent confirmation, the per-workspace `.dt-playbook-mappings.yaml`, empty-tenant fast-path conventions, duplicate-snapshot logic, PowerShell quoting note, shared report style rules, and the self-improvement protocol. This skill only contains what is unique to the business-process review.

> 🔁 **Relationship to `dt-bizevents-review`:** the bizevents skill produces a **breadth-first** tenant-wide inventory ("what providers / types exist, are they healthy, where are the anomalies"). This skill is **depth-first** — you already know which providers and types matter, and you want to understand the *process they describe*, the *correlation IDs that join them*, and the *business metrics that should be measured on top of them*. Run `dt-bizevents-review` first if you don't yet know which providers exist; run this one when you want to go deep on a known process.

---

## 🧷 Per-skill parameters (for `dt-playbook-common`)

| Parameter | Value |
| --- | --- |
| `<subfolder>` | `process-detail-reports/` |
| `<filename-stem>` | `business-process-detail` |
| `<records>` (used in fast-path prompts) | `bizevents` |
| Discovery Query 1 (headline) | §1 below |
| Initial-overview queries (run when Query 1 > 0) | §2–4 below |
| Bucket-inventory query (empty-tenant fast path) | §16 below |
| Deep-dive queries (run after Step 2 — focused on selected provider(s)/type(s)) | §5–15 below |

Full output path: `<context-folder>/process-detail-reports/business-process-detail-<YYYY-MM-DD-HHMM>.md` (UTC, never overwritten).

### Extra prerequisites (beyond `dt-playbook-common`)

| Requirement | Notes |
| --- | --- |
| Token scopes: read access to `bizevents` (Grail query scopes) | A read-only context (e.g. `*-readonly`) is enough |
| At least one `event.provider` already present in the tenant | If `bizevents` is empty, fall back to the `dt-bizevents-review` skill's empty-tenant fast path — there is no process to characterize |

---

## 🎯 Step 2 — Focus interview (this skill overrides the common single-question form)

> **Why the override?** The common Step 2 asks **one** focus question. A business-process review is *defined* by its provider+type scope, so this skill splits Step 2 into two sequential questions. Both must be answered before any deep-dive query (§5+) runs. The common Step 0 (mappings + intent confirmation) and Step 1 (headline numbers) are unchanged.

### Step 2a — Pick provider(s)

After §1–§4 print headline numbers + the provider-and-type inventory, ask:

> *"Which `event.provider`(s) do you want to scope this business-process review to? Pick one or several — they should be providers that together describe a single end-to-end process (e.g. `acme.checkout` alone, or `acme.checkout` + `acme.fulfillment` if the process spans two providers)."*

Present the top providers from §2 as multi-select options (recommended ≤ 3 providers per run — more providers blur the process focus). Always include a free-text option in case the user wants to scope to a less-common provider.

Required answer: **at least one** `event.provider`. Do not proceed to Step 2b without one.

### Step 2b — Pick event.type(s) from the chosen provider(s)

Run §3 (filtered to the providers selected in 2a) to enumerate every `event.type` the user might want. Then ask:

> *"Within the provider(s) `<list>`, which `event.type`s do you want to treat as steps of the process? Pick the events that you believe are part of the same logical flow — including any failure / exception branches you want characterized (e.g. `claim.fraud.detected` alongside the happy-path `claim.*` events)."*

Present the event types from §3 as multi-select options, ordered by count (highest first) and grouped by provider. Always include:
- A **"Select all"** shortcut for users that want to start broad.
- A free-text option for types not in the top-N.

Required answer: **at least two** `event.type`s (a "process" with only one step is just a single event — fall back to the `dt-bizevents-review` skill in that case and tell the user so).

### Persisting the scope in the report

Record the final scope at the top of the deliverable as:

```
> **Scope:** providers = [`<p1>`, `<p2>`, …]; event types = [`<t1>`, `<t2>`, …]
> **Process label (user-supplied or inferred):** `<short name, e.g. "checkout funnel">`
```

If the user did not name the process, infer one from the dominant provider (e.g. `aviva.claims` → "claims lifecycle") and explicitly mark it as `(inferred)`.

---

## 📋 Discovery Query Set

Run §1–§4 during Step 1 (overview). Run §5–§15 only **after** Step 2 has produced a concrete `providers` + `eventTypes` scope. Throughout the queries below, treat `<providers>` and `<eventTypes>` as placeholder lists that the agent substitutes inline (DQL `in(field, {"a", "b"})`).

### 1. Headline numbers — bizevents volume + provider/type cardinality (last 24 h)

```dql
fetch bizevents, from:-24h
| summarize total = count(),
            distinctTypes = countDistinct(event.type),
            distinctProviders = countDistinct(event.provider),
            earliest = min(timestamp),
            latest   = max(timestamp)
```

### 2. Provider inventory (last 24 h)

```dql
fetch bizevents, from:-24h
| summarize count = count(), distinctTypes = countDistinct(event.type), by:{event.provider}
| sort count desc
| limit 50
```

Use this to power the Step 2a multi-select.

### 3. Event-type inventory **scoped to the chosen providers** (last 24 h)

```dql
fetch bizevents, from:-24h
| filter in(event.provider, {<providers>})
| summarize count = count(),
            distinctIds = countDistinct(event.id),
            firstSeen = min(timestamp),
            lastSeen  = max(timestamp),
            by:{event.provider, event.type}
| sort count desc
| limit 200
```

Use this to power the Step 2b multi-select. Note any `event.type`s with `count < ~5/h` — flag them as candidate exception branches when presenting options.

### 4. Schema-level cardinality survey (last 1 h, scoped to chosen providers)

```dql
fetch bizevents, from:-1h
| filter in(event.provider, {<providers>})
| fieldsSummary event.id, event.category, event.provider, event.type,
                dt.openpipeline.source, dt.openpipeline.pipelines, dt.system.bucket
```

---

> 🔻 **The queries below run only after Step 2 has produced both `<providers>` and `<eventTypes>`.** All windows default to 24 h for funnel/correlation analysis (need enough volume to see end-to-end completions) and drop to 1 h or 15 m for schema discovery / payload checks.

### 5. Per-step volume + relative share (24 h)

```dql
fetch bizevents, from:-24h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| summarize count = count(),
            distinctEventIds = countDistinct(event.id),
            firstSeen = min(timestamp),
            lastSeen  = max(timestamp),
            by:{event.provider, event.type}
| sort count desc
```

> If `distinctEventIds < count` for any step, flag duplicates and capture the ratio in the report.

### 6. Sample one record per selected event.type (schema discovery — 15 m, ingest-lag-aware)

For each event.type in `<eventTypes>`:

```dql
fetch bizevents, from:-15m
| filter event.provider == "<provider>" and event.type == "<eventType>"
| limit 1
```

> ⏱️ **Ingest-lag-aware window:** if §1 returned `latest` older than ~10 min, widen to `-1h` or `-2h` so §6 actually returns rows. Compute the lag once with:
> ```dql
> fetch bizevents, from:-2h
> | filter in(event.provider, {<providers>})
> | summarize latest = max(timestamp),
>             lagMinutes = (toLong(now()) - toLong(max(timestamp))) / 60000000000
> ```

For each sampled record, capture in the report:

- Top-level keys present + their types.
- Any field that looks like an identifier (`*Id`, `*Uuid`, `*Ref`, `*Number`, `*Code`) — these become correlation-ID candidates in §8.
- Any field that looks like a **business measure** (`amount`, `total`, `price`, `qty`, `count`, `duration*`, `score`) — these become candidate per-event business metrics in §14.
- Any field that looks like a **business dimension** (`region`, `branch`, `channel`, `productCategory`, `currency`, `tenantId`, `customerSegment`) — these are dashboard slicers and feed metric dimensions in §14.

### 7. Field-coverage survey across the selected event.types (1 h)

For each event.type, run:

```dql
fetch bizevents, from:-1h
| filter event.provider == "<provider>" and event.type == "<eventType>"
| fieldsSummary <candidateIds>, <candidateMeasures>, <candidateDimensions>,
                trace_id, span_id, dt.trace_id,
                user.id, session.id
```

`<candidateIds>` / `<candidateMeasures>` / `<candidateDimensions>` come from §6. The aim is a per-event-type coverage matrix in the report: which fields are 100 % populated (safe to dashboard), which are sparse (need a fallback or note), which are absent (don't reference in metrics).

### 8. Correlation-ID candidate hunt — shared identifiers across the selected event.types (1 h)

Run **once per candidate ID field** discovered in §6. The goal: prove the field is populated *across* the chosen event.types with values that overlap (otherwise it can't stitch steps together).

```dql
fetch bizevents, from:-1h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| filter isNotNull(<candidateIdField>)
| summarize count = count(),
            distinctValues = countDistinct(<candidateIdField>),
            by:{event.type}
| sort count desc
```

Then quantify cross-step overlap (does the **same** identifier value appear in multiple steps?):

```dql
fetch bizevents, from:-1h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| filter isNotNull(<candidateIdField>)
| summarize stepsTouched = countDistinct(event.type),
            steps = collectDistinct(event.type),
            by:{corrId = <candidateIdField>}
| summarize correlations = count(),
            avgStepsTouched = avg(stepsTouched),
            maxStepsTouched = max(stepsTouched),
            by:{stepsTouched}
| sort stepsTouched desc
```

> 💡 **What "good" looks like:** for a viable correlation ID, the majority of values touch **≥ 2** of the selected event.types. If `stepsTouched == 1` dominates, the field is a *per-event* ID (like `event.id`) — not a process correlator.

If multiple candidate IDs survive, rank them by:
1. Coverage (% of events where the field is non-null).
2. Mean `stepsTouched` (higher = stitches more steps).
3. Stability (does the same value really mean the same business entity, or is it reused?).

Standard always-check correlation candidates (run §8 against each even if §6 didn't surface them — they're cross-cutting):

- `trace_id`, `span_id`, `dt.trace_id` (technical, but tells you whether the events are already trace-linked).
- `user.id`, `session.id`, `customer.id`.
- Any field matching `^(order|transaction|claim|account|case|ticket|reservation|booking|policy|invoice|payment)Id$` (or its `.id` / `.number` / `.ref` variants).

### 9. Process step ordering — for each correlation-ID value, what order do the steps fire? (24 h)

```dql
fetch bizevents, from:-24h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| filter isNotNull(<bestCorrelationIdField>)
| sort timestamp asc
| summarize stepsInOrder = collectArray(event.type),
            firstStep    = takeFirst(event.type),
            lastStep     = takeFirst(event.type, sort:{timestamp desc}),
            firstTs      = min(timestamp),
            lastTs       = max(timestamp),
            stepCount    = count(),
            by:{corrId = <bestCorrelationIdField>}
```

Then aggregate to discover the most common sequences:

```dql
... | fieldsAdd seq = concatArray(stepsInOrder, " → ")
    | summarize correlations = count(), by:{seq}
    | sort correlations desc
    | limit 25
```

> Use the top-N sequences to draw the *canonical happy path* and any *common branches* (e.g. happy-path vs. fraud-branch) in the report's Process Map section.

### 10. Step-to-step transition matrix (24 h)

For each correlation-ID value, expand its event timeline into adjacent (step_n, step_n+1) pairs and count transitions:

```dql
fetch bizevents, from:-24h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| filter isNotNull(<bestCorrelationIdField>)
| sort timestamp asc
| summarize stepsInOrder = collectArray(event.type),
            tsInOrder    = collectArray(timestamp),
            by:{corrId = <bestCorrelationIdField>}
| expand idx = sequence(0, arraySize(stepsInOrder) - 2)
| fieldsAdd fromStep = stepsInOrder[idx],
            toStep   = stepsInOrder[idx + 1],
            transitionMs = (toLong(tsInOrder[idx + 1]) - toLong(tsInOrder[idx])) / 1000000
| summarize transitions = count(),
            avgTransitionMs = avg(transitionMs),
            p50TransitionMs = percentile(transitionMs, 50),
            p95TransitionMs = percentile(transitionMs, 95),
            by:{fromStep, toStep}
| sort transitions desc
```

> ⚠️ If your tenant's DQL build does not expose `expand` + `sequence`, fall back to repeated `lookup` joins or compute transitions in two queries. Note the fallback in the report so the next agent knows.

### 11. Funnel completion + drop-off (24 h)

Given a hypothesised process order `[t1, t2, … tN]` (use the dominant sequence from §9):

```dql
fetch bizevents, from:-24h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| filter isNotNull(<bestCorrelationIdField>)
| summarize hasT1 = countIf(event.type == "<t1>") > 0,
            hasT2 = countIf(event.type == "<t2>") > 0,
            hasTN = countIf(event.type == "<tN>") > 0,
            by:{corrId = <bestCorrelationIdField>}
| summarize startedT1   = countIf(hasT1),
            reachedT2   = countIf(hasT1 and hasT2),
            reachedTN   = countIf(hasT1 and hasT2 and ... and hasTN),
            droppedAtT2 = countIf(hasT1 and not hasT2)
```

The agent should template this per-tenant — number of steps varies per process. Use the row counts to compute step-by-step conversion percentages for the *Funnel attrition* table in the report.

### 12. End-to-end process duration (24 h, scoped to **completed** correlations)

```dql
fetch bizevents, from:-24h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| filter isNotNull(<bestCorrelationIdField>)
| summarize firstTs = min(timestamp),
            lastTs  = max(timestamp),
            stepsTouched = countDistinct(event.type),
            by:{corrId = <bestCorrelationIdField>}
| filter stepsTouched == <expectedTerminalStepCount>     // only completed processes
| fieldsAdd durationMs = (toLong(lastTs) - toLong(firstTs)) / 1000000
| summarize completed = count(),
            avgMs = avg(durationMs),
            p50Ms = percentile(durationMs, 50),
            p95Ms = percentile(durationMs, 95),
            p99Ms = percentile(durationMs, 99),
            maxMs = max(durationMs)
```

### 13. Open / stalled processes (correlations that started but didn't end within the window)

```dql
fetch bizevents, from:-24h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| filter isNotNull(<bestCorrelationIdField>)
| summarize firstTs = min(timestamp),
            lastTs  = max(timestamp),
            stepsTouched = countDistinct(event.type),
            terminalReached = countIf(event.type == "<terminalSuccessType>") > 0
                           or countIf(event.type == "<terminalFailureType>") > 0,
            by:{corrId = <bestCorrelationIdField>}
| filter not terminalReached
| fieldsAdd ageMs = (toLong(now()) - toLong(lastTs)) / 1000000
| summarize stalled = count(),
            avgAgeMin = avg(ageMs) / 60000,
            p95AgeMin = percentile(ageMs, 95) / 60000
```

> Use the result to size the "stalled processes" risk in the report. Combine with §11 drop-off for a full picture.

### 14. Business-measure & dimension sweep on the selected event.types (1 h)

For each candidate business measure / dimension from §6, confirm it's actually usable:

```dql
fetch bizevents, from:-1h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| summarize count = count(),
            nonNull = countIf(isNotNull(<measureOrDimensionField>)),
            distinctVals = countDistinct(<measureOrDimensionField>),
            sampleValues = collectDistinct(<measureOrDimensionField>, limit: 5),
            by:{event.type}
| sort count desc
```

> For *measures*: confirm it parses as numeric (`toDouble(<field>)` round-trip), check min/avg/max ranges, and flag obvious unit ambiguity (cents vs dollars, ms vs seconds).
> For *dimensions*: confirm cardinality is < ~50 unique values for dashboard use (anything higher needs a top-N or bucketing).

### 15. PII / sensitive-payload probe scoped to the selected event.types (15 m, capped)

Even though the `dt-bizevents-review` skill covers tenant-wide PII checks, this skill re-runs **scoped to the selected event.types** because the audience for a process-detail report often dashboards these events directly and needs to know which fields to suppress.

```dql
fetch bizevents, from:-15m
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| fieldsSummary <candidatePayloadFields>
```

Then sample one redactable record per type:

```dql
fetch bizevents, from:-15m
| filter event.provider == "<provider>" and event.type == "<eventType>"
| fields timestamp, event.id, <candidatePayloadFields>
| limit 1
```

Paraphrase / redact anything sensitive before pasting into the report.

### 16. Bucket inventory (used by the empty-tenant fast path)

```dql
fetch dt.system.buckets
| filter dt.system.table == "bizevents"
| fields name, display_name, retention_days, records, estimated_uncompressed_bytes
```

Identical purpose to the `dt-bizevents-review` skill §10 — distinguishes "no producers" from "no bucket". If this returns zero rows, halt and tell the user to run the `dt-bizevents-review` empty-tenant report instead; a process review cannot run.

---

## 🔍 What to Look for in the Results

Map findings into the report sections using this checklist.

### Process structure
- 🧩 **Canonical happy path** — the dominant sequence from §9 + §10. Express it as `t1 → t2 → … → tN` with counts and percentages.
- 🌿 **Common branches** — non-happy-path sequences that still account for ≥ 1 % of correlations (e.g. fraud branch, refund branch, cancel branch). Each branch deserves its own row in the Process Map.
- 🔁 **Loop / retry steps** — same event.type repeating for the same correlation ID. Note typical loop count and whether it indicates a retry pattern or a genuine business loop (e.g. multiple `payment.attempted` events before a `payment.completed`).
- 📉 **Drop-off cliffs** — steps with > 10 percentage-point conversion drop from the previous step. Highlight as candidates for SLO targets.
- ⏱️ **Slow transitions** — transition pairs in §10 with `p95TransitionMs` an order of magnitude larger than the others.

### Correlation ID quality
- ✅ **Strong correlator** — field is non-null on ≥ 95 % of the selected events, and most values touch ≥ 2 of the chosen event.types in §8. Recommend it as the primary correlation ID for dashboards & metrics.
- ⚠️ **Weak correlator** — populated only on some types, or values rarely overlap across types. Document it and explain *why* it's weak (so the next person doesn't try to use it).
- ❌ **Decoy correlator** — looks promising (matches the naming convention) but in practice each step has its own value (so `stepsTouched == 1` dominates). Call it out explicitly so consumers don't try to join on it.
- 🪪 **Naming inconsistencies** — same business concept under multiple keys (`orderId` vs `order.id` vs `orderNumber`). Capture all variants and recommend a `coalesce()` pattern.

### Misses, precautions, things to note
- 🚨 **Missing terminal step** — selected types include a "started" event but no clear "completed" event. The process can't be measured for completion rate without one.
- 🚨 **Missing failure / exception step** — only happy-path events selected, with no equivalent failure branch. Operators cannot tell failed processes from stalled ones.
- 🚨 **Out-of-order timestamps** — events received out-of-order at the ingest pipeline (later timestamps arriving before earlier ones). Note that this breaks any "first event vs. last event" calculation done over a *narrow* window.
- 🚨 **Duplicate events** — `distinctEventIds < count` for any step in §5. Risks double-counting in metrics; recommend dedup-on-`event.id`.
- 🚨 **Cross-provider mismatch** — when the process spans ≥ 2 providers, check that the chosen correlation ID is consistent across providers (some teams reset IDs at the provider boundary). If they reset, the process is *not* end-to-end joinable without a mapping table.
- 🪲 **Ingest gaps** — `latest` timestamp older than expected (compare §1 lag vs. the user's expected freshness). Flag because every funnel/duration metric will under-count until the gap closes.
- 🔒 **PII in payload** — same checks as the `dt-bizevents-review` skill but scoped to the selected types (per §15).

### Business metric opportunities (the headline output for this playbook)

For each candidate metric, write down: **name**, **what it answers for the business**, **DQL sketch**, **suggested dimensions**, **suggested alert/threshold**, and **owner persona (Ops / Product / Finance / Risk)**. Examples by domain — *use these as inspiration, the actual list must come from §5–§14 results*:

- **Process Completion Rate** — % of started correlations that reach the terminal success step. Slice by region / channel / segment. Owner: Product + Ops.
- **Process Abandonment Rate** — % of correlations that started but did not reach a terminal step within the SLA window. Owner: Product.
- **Average / p95 Process Duration** — wall-clock from first step to terminal success. Owner: Ops.
- **Step Conversion Rate** — per-step retention (step n → step n+1 %). Lets the business pinpoint *which* step is causing drop-off. Owner: Product.
- **Step Latency (p50/p95)** — per-transition timing from §10. Owner: Ops + Engineering.
- **Business Value Throughput** — `sum(<amountField>)` per unit time on the terminal success step. Owner: Finance + Ops.
- **Failure-Branch Volume / Share** — count + % of correlations hitting the failure branch (fraud, refund, cancel). Owner: Risk + Product.
- **Top Failing Step** — for non-terminal correlations, which step is last? (Pareto chart.) Owner: Engineering.
- **Stalled Process Count** — correlations open for > N minutes without a terminal event. Owner: Ops.
- **Revenue-At-Risk** — `sum(<amountField>)` on *non-terminal* correlations that have been open > N minutes. Owner: Finance.
- **Per-Customer Process Rate** — count of distinct correlations per `<customer.id>` per day. Surfaces VIPs and abusers. Owner: Product + Risk.
- **Process Mix** — share of correlations by business dimension (region, channel, product category) — proves whether the funnel behaves the same across segments. Owner: Product.

For each metric, include the DQL sketch in the report so the consumer can paste it into a metric definition. Two examples:

```dql
// Process Completion Rate, 1 h granularity
fetch bizevents, from:-7d
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| filter isNotNull(<bestCorrelationIdField>)
| summarize hasStart = countIf(event.type == "<startStep>") > 0,
            hasEnd   = countIf(event.type == "<terminalSuccessStep>") > 0,
            by:{corrId = <bestCorrelationIdField>, bin = bin(timestamp, 1h)}
| filter hasStart
| summarize started = count(),
            completed = countIf(hasEnd),
            completionRate = countIf(hasEnd) * 100.0 / count(),
            by:{bin}
| sort bin asc
```

```dql
// Revenue throughput, 5 min granularity, terminal step only
fetch bizevents, from:-24h
| filter event.provider == "<provider>" and event.type == "<terminalSuccessStep>"
| fieldsAdd amt = toDouble(<amountField>)
| summarize value = sum(amt), txns = count(), by:{bin = bin(timestamp, 5m), region}
| sort bin asc
```

---

## 🧱 Output Document Structure

Save to `<context-folder>/process-detail-reports/business-process-detail-<YYYY-MM-DD-HHMM>.md` (UTC). See the `dt-playbook-common` skill's *Shared style rules* section for filename/overwrite rules.

Section skeleton:

```
# 🧬 Business Process Detail — <tenant short name>
> Snapshot date, context name, scope of windows used.
> **Scope:** providers = [...]; event types = [...]
> **Process label:** <name> (inferred or user-supplied)

## 📊 Headline Numbers (24 h, scoped)
   - Total events in scope, distinct correlation values, started vs completed counts.
   - Mention the bucket the events live in and the ingest source (`dt.openpipeline.source`).

## 🧭 Process Map
   ### 🛣️ Canonical happy path
       - Diagram (mermaid or ASCII) of `t1 → t2 → … → tN` with per-step counts + conversion %.
   ### 🌿 Common branches
       - One sub-section per branch (fraud, refund, cancel, retry…) with its own diagram fragment and share.
   ### 🔁 Loop / retry behaviour (if any)

## 📘 Per-Event Data Dictionary
   - One sub-table per `event.type` from the scope.
   - Columns: field, type, coverage %, sample value (redacted), role (id / measure / dimension / metadata / sensitive).

## 🔗 Correlation IDs
   ### ✅ Recommended primary correlation
       - Field name, coverage, mean `stepsTouched`, why it was chosen.
   ### ⚠️ Secondary / weak candidates
   ### ❌ Decoy fields (look like correlators but aren't)
   ### 🪪 Naming inconsistencies / multi-variant fields

## ⏱️ Step Transitions & Timing
   - Transition matrix table (from §10): `fromStep`, `toStep`, transitions, p50, p95 ms.
   - Highlight the slow transitions (> p95 outliers).

## 🎯 Funnel & Duration
   - Funnel attrition table (started → step2 → … → terminal) with absolute counts and %.
   - End-to-end duration table (avg / p50 / p95 / p99) on completed correlations only.
   - Stalled process count + age stats from §13.

## 🚨 Misses, Precautions & Anomalies (numbered, each with Impact + Fix)
   - Missing terminal steps, missing failure branches, duplicate events, out-of-order events,
     cross-provider correlation mismatches, ingest gaps, etc.

## 🔒 PII / Sensitive Content Findings (scoped)
   - Always include this section — state "No hits in the sampled window" if all checks were clean.

## 💡 Recommended Business Metrics
   ### 🏢 Process-level metrics (operate on the whole funnel)
       - Table: name | answers (business question) | DQL sketch | dimensions | suggested alert | owner.
   ### 🧱 Event-level metrics (per `event.type`)
       - Table with the same columns, one row per `event.type` in scope.
   ### 📊 Dashboard composition
       - 3–5 tile-level recommendations: which metric goes where, which dimension to slice by,
         and what a "good" / "bad" reading looks like.

## 🧪 Quick Verification Queries
   - 3 ready-to-paste DQL snippets that let the next reader re-confirm completion rate,
     duration, and the top failing step on demand.
```

### 🪹 Empty-scope report structure (no rows for the chosen scope)

If §5 returns zero rows for the chosen providers + types (e.g. the user picked a provider that has events today but no events in the chosen types within 24 h), do not proceed to §6+. Instead write a short report with this skeleton:

```
# 🧬 Business Process Detail — <tenant short name>
> Snapshot date, context name, windows checked, verdict.
> **Scope:** providers = [...]; event types = [...]

## 📊 Headline Numbers
   - Per-window counts (24 h, 7 d, 30 d) — all zero for the chosen scope.
   - Per-window counts at provider granularity (to show whether the provider itself has activity).

## 🚨 Findings
   ### 1. 🛑 Selected `event.type`(s) had no traffic in the windows checked
       - **Observation:** zero rows for the chosen types across the windows checked.
       - **Impact:** the process review cannot be produced — there is nothing to stitch.
       - **Fix:** verify the type names spelled correctly; consider widening the type selection;
         consider running the `dt-bizevents-review` skill to confirm the provider's actual current types.

## 🧪 Quick Re-check Queries
   - Two short snippets to re-confirm the scope once the user re-selects types.
```

Skip the data dictionary, correlation IDs, transitions, funnel, PII, and recommendations sections entirely — there is nothing to populate.

---

## ✅ Final Self-Review Checklist

- [ ] Headline numbers, provider+type scope, and process label are stated at the top of the report.
- [ ] Process Map shows the canonical happy path **and** at least one branch if §9 surfaced one.
- [ ] Every event.type mentioned in tables actually appeared in a query result (no hallucination).
- [ ] At least one correlation ID is recommended *or* the report explicitly states "no usable correlation ID found" with the evidence (decoy fields excluded).
- [ ] Each anomaly has a concrete fix.
- [ ] Funnel attrition table covers every step in scope (no silently-dropped step).
- [ ] Duration stats are calculated on **completed** correlations only — stalled / partial flows are reported separately.
- [ ] PII section is **always** present (even if all checks returned zero — state that explicitly).
- [ ] Recommended Business Metrics section has both **process-level** and **event-level** rows.
- [ ] Each recommended metric includes a DQL sketch, suggested dimensions, suggested alert, and an owner persona.
- [ ] Metrics are described in **business** terms ("completion rate", "revenue-at-risk", "abandonment rate"), not purely technical ones ("event count", "latency ms").
- [ ] Document renders cleanly in a markdown preview (tables aligned, no broken links, mermaid diagrams render).
- [ ] Sensitive values (real customer IDs, real CC numbers, real emails) are **not** pasted into the document — paraphrase or redact.

---

## 🔗 Related References

- `dt-playbook-common` skill — shared scaffolding (Step 0 interview, mappings file, style rules).
- `dt-bizevents-review` skill — breadth-first sibling. Run first if you don't yet know which providers/types exist.
- `dt-dql-essentials` skill — DQL syntax, pitfalls, operator reference.
- Dynatrace docs: *Business Events*, *Business processes (process mining)*, *Funnels & conversion DQL patterns*.
