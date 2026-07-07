---
name: dt-logs-review
description: >-
  Re-runnable Dynatrace logs overview playbook. Produces a tenant-wide
  state-of-the-logs report covering volume, top emitters, severity mix, ingest
  sources, data dictionary, top error signatures, PII checks, anomalies, and
  parse-rule / metric opportunities. Invoke when the user asks for a logs
  overview, logs review, or logs report - e.g. "logs overview", "log review",
  "run the logs playbook", "what logs are we collecting",
  "noisy log emitters", "logs PII audit", or @dt-logs-review. Reads
  `dt-playbook-common` FIRST for Step 0 (context/folder confirmation, intent
  check, empty-tenant fast-path). Writes to
  `<context-folder>/logs-overview-reports/logs-overview-<YYYY-MM-DD-HHMM>.md`.
  Uses 1-in-1000 sampling on 24 h discovery queries to cap Grail scan cost.
  Do NOT use for explaining an existing query, bizevents analysis (use
  `dt-bizevents-review`), or trace/span analysis (use `dt-obs-tracing`).
---

# 📜 Logs Environment Review — Playbook

> **Purpose:** Re-runnable recipe for producing a high-level logs state-of-the-environment report (volume, emitters, severity mix, data dictionary, anomalies, correlation IDs, parsing/metric opportunities) for any Dynatrace tenant.
> **Sibling output:** `<context-folder>/logs-overview-reports/logs-overview-<YYYY-MM-DD-HHMM>.md` (e.g. `demo-live-readonly/logs-overview-reports/logs-overview-2026-06-25-2015.md`).
> **Audience:** AI coding agents (Copilot, Claude, etc.) and the engineer driving them.

> 📖 **Read the `dt-playbook-common` skill first.** It owns the prerequisites, the Step 0 kickoff interview, intent confirmation, the per-workspace `.dt-playbook-mappings.yaml`, empty-tenant fast-path conventions, duplicate-snapshot logic, PowerShell quoting note, shared report style rules, and the self-improvement protocol. This skill only contains what is unique to logs.

---

## 🧷 Per-skill parameters (for `dt-playbook-common`)

| Parameter | Value |
| --- | --- |
| `<subfolder>` | `logs-overview-reports/` |
| `<filename-stem>` | `logs-overview` |
| `<records>` (used in fast-path prompts) | `logs` |
| Discovery Query 1 (headline) | §1 below |
| Initial-overview queries (run when Query 1 > 0) | §2–7 below |
| Bucket-inventory query (empty-tenant fast path) | §12 below |
| Deep-dive queries (run after Step 2) | §8–11 below |

Full output path: `<context-folder>/logs-overview-reports/logs-overview-<YYYY-MM-DD-HHMM>.md` (UTC, never overwritten).

### Extra prerequisites (beyond `dt-playbook-common`)

| Requirement | Notes |
| --- | --- |
| **`dt-obs-logs` skill** (from `dynatrace-for-ai`) | Non-optional — supplies log-specific DQL patterns (`matchesPhrase`, JSON parsing, severity filters) the playbook depends on. Installed via this repo's `ai.repo.yaml` manifest (see the repo README). If not in the agent's context at Step 1, follow `dt-playbook-common` §Prerequisites' missing-skill procedure — print the install commands from the README and halt; do not run installers. |
| Token scopes: read access to `logs` (Grail query scopes) | A read-only context (e.g. `*-readonly`) is enough |

> 💡 **Extra cost note:** log buckets are routinely the largest tables in a tenant — **never** drop the `from:` clause and **never** run a `matchesPhrase()` or `parse` over an unbounded window.

### 🛡️ Query guardrails (logs)

Logs are the easiest data source to over-scan by accident. Apply these rules to every query you run in this playbook:

1. **Broad 24 h discovery queries MUST use `samplingRatio: 1000`** (1-in-1000 sample). This applies to §1, §2, §3, §4, and the §11 volume-trend queries. The keyword goes inside the `fetch` clause: `fetch logs, from:-24h, samplingRatio: 1000`.
   - Sampling preserves the **shape** of distributions and top-N rankings, which is what these queries are for.
   - Returned `count()` values are sample-scale — multiply by the sampling ratio (× 1000) to estimate full volume, and label the report column accordingly (e.g. "count (sampled ×1000)" or "est. count").
   - `countDistinct()` under sampling **underestimates** cardinality — treat the returned number as a lower bound ("≥ N distinct values") and note this in the report.
2. **Narrow / specific / summary queries MAY omit sampling**, provided the existing guardrails are kept:
   - A bounded `from:` window (default to ≤ 1 h, prefer 15 m for anything touching `content` like `matchesRegex`, `parse`, `splitString`, `fieldsAdd`).
   - A selective `filter` (e.g. specific `dt.process_group.id`, `status`, `k8s.namespace.name`).
   - A `summarize` + `limit` (top-N) rather than returning raw records, unless you genuinely need one sample row (in which case use `limit 1`).
3. **If a 1 h or 15 m query still feels heavy** (very large tenant, very wide pipeline, or the `fieldsSummary` in §6/§7 is slow) add `samplingRatio: 100` or `scanLimitGBytes: 50` to the `fetch` as a safety net, and call it out in the report.
4. **Never** remove the `from:` clause, and never run `matchesPhrase()` / `matchesRegex()` / `parse` over an unbounded window — sampling does not protect against that pattern.

### Step 2 — Focus question wording (logs)

> *"Based on the overview above, are there specific process groups, hosts, Kubernetes namespaces, or `log.source` paths you want the rest of the report to focus on? (Leave blank for full coverage of the noisiest + most error-prone emitters.)"*

Present the top ~10 process groups (or pods/namespaces if this is a K8s-heavy tenant) and top ~10 `log.source` paths from the overview as selectable options. If the user picks a focus set, restrict §8 (per-emitter schema samples), §9 (top error signatures), and §10 (PII probes) to that set. If the user leaves it blank, cover the top 5 emitters by volume **plus** the top 5 emitters by ERROR/SEVERE count (these are often different sets) plus any unusual ones (sources logging stack traces, sources with > 70 % unparsed `content`, sources with PII hits).

---

## 📋 Discovery Query Set

Run these in order. Each is intentionally small.

### 1. Headline numbers — volume, distinct severities/emitters (last 24 h, sampled ×1000)

```dql
fetch logs, from:-24h, samplingRatio: 1000
| summarize sampledTotal       = count(),
            distinctStatuses     = countDistinct(status),
            distinctProcessGroups = countDistinct(dt.process_group.id),
            distinctHosts        = countDistinct(dt.host.id),
            distinctLogSources   = countDistinct(log.source),
            earliest             = min(timestamp),
            latest               = max(timestamp)
| fieldsAdd estimatedTotal = sampledTotal * 1000
```

> ℹ️ `sampledTotal` is the raw count from the 1-in-1000 sample; `estimatedTotal` is the scaled full-volume estimate. `distinct*` values are **lower bounds** because sampling underestimates cardinality.

### 2. Severity distribution (last 24 h, sampled ×1000)

```dql
fetch logs, from:-24h, samplingRatio: 1000
| summarize sampledCount = count(), by:{status}
| fieldsAdd estimatedCount = sampledCount * 1000
| sort sampledCount desc
```

> If `status` is mostly `NONE` / null, also try the parsed log level:
> ```dql
> fetch logs, from:-24h, samplingRatio: 1000
> | summarize sampledCount = count(), by:{loglevel}
> | fieldsAdd estimatedCount = sampledCount * 1000
> | sort sampledCount desc
> ```

### 2b. `loglevel` vs `status` cross-tabulation (last 1 h — detect normalization gaps)

```dql
fetch logs, from:-1h
| summarize count = count(), by:{status, loglevel}
| sort count desc
| limit 20
```

> If `loglevel: DEBUG` appears but maps to `status: INFO`, severity normalization is collapsing DEBUG into INFO. If `loglevel: CRITICAL`/`FATAL` maps to `status: ERROR` that is correct but worth confirming. Any `loglevel` value without a `status` counterpart indicates a parsing gap.

### 3. Top emitters — process groups + hosts (last 24 h, sampled ×1000)

```dql
fetch logs, from:-24h, samplingRatio: 1000
| summarize sampledCount = count(),
            sampledErrorCount = countIf(status == "ERROR" or status == "SEVERE"),
            by:{dt.process_group.detected_name, dt.process_group.id}
| fieldsAdd estimatedCount = sampledCount * 1000,
            estimatedErrorCount = sampledErrorCount * 1000
| sort sampledCount desc
| limit 25
```

```dql
fetch logs, from:-24h, samplingRatio: 1000
| summarize sampledCount = count(),
            sampledErrorCount = countIf(status == "ERROR" or status == "SEVERE"),
            by:{host.name, dt.host.id}
| fieldsAdd estimatedCount = sampledCount * 1000,
            estimatedErrorCount = sampledErrorCount * 1000
| sort sampledCount desc
| limit 25
```

If Kubernetes is in use, also run:
```dql
fetch logs, from:-24h, samplingRatio: 1000
| filter isNotNull(k8s.namespace.name)
| summarize sampledCount = count(),
            sampledErrorCount = countIf(status == "ERROR" or status == "SEVERE"),
            by:{k8s.cluster.name, k8s.namespace.name, k8s.container.name}
| fieldsAdd estimatedCount = sampledCount * 1000,
            estimatedErrorCount = sampledErrorCount * 1000
| sort sampledCount desc
| limit 25
```

> ℹ️ Sampling preserves the *ranking* of top emitters but rare error spikes (< 1 in 1000) may not appear. Re-run the survivor list against a 1 h window without sampling if you need exact error counts for the report.

### 4. Top `log.source` paths (last 24 h, sampled ×1000)

```dql
fetch logs, from:-24h, samplingRatio: 1000
| summarize sampledCount = count(), by:{log.source}
| fieldsAdd estimatedCount = sampledCount * 1000
| sort sampledCount desc
| limit 25
```

### 5. Ingest source + pipeline split (last 1 h)

```dql
fetch logs, from:-1h
| summarize cnt = count(), by:{dt.openpipeline.source}
| sort cnt desc
```

```dql
fetch logs, from:-1h
| summarize cnt = count(), by:{dt.openpipeline.pipelines}
| sort cnt desc
| limit 25
```

### 6. Field-coverage & cardinality survey (last 1 h)

```dql
fetch logs, from:-1h
| fieldsSummary status, loglevel, content,
                dt.process_group.id, dt.host.id, log.source,
                dt.entity.process_group, dt.entity.host, dt.entity.service,
                k8s.cluster.name, k8s.namespace.name, k8s.pod.name, k8s.container.name,
                service.name, cloud.provider, cloud.region,
                dt.system.bucket, dt.openpipeline.source, dt.openpipeline.pipelines
```

### 7. Correlation-ID coverage (last 1 h)

```dql
fetch logs, from:-1h
| fieldsSummary trace_id, span_id, trace.id, span.id, dt.trace_id,
                user.id, session.id,
                dt.entity.host, dt.entity.process_group, dt.entity.service
```

### 8. Sample one record per major emitter (schema discovery)

For each process group (or namespace/container) that surfaces in §3, pull a single record to learn its schema and how `content` looks (raw text, JSON, key=value, multi-line stack trace, etc.):

```dql
fetch logs, from:-15m
| filter dt.process_group.id == "<pg-id>"
| fields timestamp, status, loglevel, content, log.source, dt.openpipeline.pipelines
| limit 1
```

> ⏱️ **Ingest-lag-aware window:** §1 returns a `latest` timestamp — if it is more than ~10 min in the past (host idle / overnight gap / OneAgent paused), `from:-15m` will return zero rows in §8–§10. Widen those narrow-window queries to span at least `(now - latest)` plus a small buffer (e.g. `-1h` or `-2h`). Compute the lag once with:
> ```dql
> fetch logs, from:-2h
> | summarize latest = max(timestamp),
>             lagMinutes = (toLong(now()) - toLong(max(timestamp))) / 60000000000
> ```
> and pick the §8–§10 window accordingly.

Do this for at least the top 5 emitters by volume **and** the top 5 by `errorCount`. Note for each:
- Is `content` plain text, JSON, or key=value?
- Is `status` populated, or does the real severity live inside `content`?
- Are there parse-worthy fields embedded in `content` (trace IDs, user IDs, request IDs, durations)?

### 9. Top error signatures (last 1 h)

```dql
fetch logs, from:-1h
| filter in(status, {"ERROR", "SEVERE"})
| summarize count = count(), by:{dt.process_group.detected_name, content}
| sort count desc
| limit 25
```

For stack-trace-heavy emitters, truncate to the first line to cluster similar errors:
```dql
fetch logs, from:-1h
| filter in(status, {"ERROR", "SEVERE"})
| fieldsAdd firstLine = splitString(content, "\n")[0]
| summarize count = count(), by:{dt.process_group.detected_name, firstLine}
| sort count desc
| limit 25
```

### 10. PII / sensitive-content probe (last 15 min, capped)

Run **each** check below independently. They are deliberately narrow regexes — count-only first, then inspect a single record if the count is non-zero.

```dql
// Credit-card-ish digit runs (Luhn not checked — flag for review)
fetch logs, from:-15m
| filter matchesRegex(content, "\\b(?:\\d[ -]?){13,16}\\b")
| summarize hits = count(), by:{dt.process_group.detected_name}
| sort hits desc
| limit 10
```

```dql
// Email addresses
fetch logs, from:-15m
| filter matchesRegex(content, "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}")
| summarize hits = count(), by:{dt.process_group.detected_name}
| sort hits desc
| limit 10
```

```dql
// "password=" / "token=" / "secret=" / "authorization: " leaks
fetch logs, from:-15m
| filter matchesRegex(content, "(?i)(password|passwd|pwd|token|secret|api[_-]?key|authorization)\\s*[=:]\\s*\\S+")
| summarize hits = count(), by:{dt.process_group.detected_name}
| sort hits desc
| limit 10
```

If any of the above is non-zero, fetch **one** sample to confirm (paraphrase / redact in the report — never paste the raw value):
```dql
fetch logs, from:-15m
| filter dt.process_group.detected_name == "<pg>"
  and matchesRegex(content, "<the regex that matched>")
| fields timestamp, content
| limit 1
```

### 11. Optional follow-ups (run if first pass shows interesting signals)

- Volume trend over 24 h (detect spikes / drops, sampled ×1000):
  ```dql
  fetch logs, from:-24h, samplingRatio: 1000
  | summarize sampledCnt = count(), by:{bin = bin(timestamp, 1h)}
  | fieldsAdd estimatedCnt = sampledCnt * 1000
  | sort bin asc
  ```
- Volume trend for the noisiest emitter (sampling optional — narrow filter already bounds the scan; drop `samplingRatio` if the emitter is small or you need exact bin counts):
  ```dql
  fetch logs, from:-24h, samplingRatio: 1000
  | filter dt.process_group.detected_name == "<pg>"
  | summarize sampledCnt = count(), by:{bin = bin(timestamp, 1h)}
  | fieldsAdd estimatedCnt = sampledCnt * 1000
  | sort bin asc
  ```
- Bucket-level cost survey (which bucket is hottest):
  ```dql
  fetch logs, from:-1h
  | summarize cnt = count(),
              bytes = sum(if(isNotNull(content), len(content), else: 0)),
              by:{dt.system.bucket}
  | sort bytes desc
  ```
- Unparsed-JSON detector (rows where `content` looks like JSON but no `parse` rule fired):
  ```dql
  fetch logs, from:-15m
  | filter startsWith(content, "{") and endsWith(content, "}")
  | summarize cnt = count(), by:{dt.process_group.detected_name}
  | sort cnt desc
  | limit 10
  ```
- Multi-line / stack-trace pressure (rows with > 5 newlines):
  ```dql
  fetch logs, from:-15m
  | fieldsAdd lines = arraySize(splitString(content, "\n"))
  | filter lines > 5
  | summarize cnt = count(), avgLines = avg(lines), by:{dt.process_group.detected_name}
  | sort cnt desc
  | limit 10
  ```

### 12. Bucket inventory (used by the empty-tenant fast path)

Confirms that at least one `logs`-typed bucket is provisioned, regardless of whether anything has been written to it. Distinguishes "no producers" from "no bucket".

```dql
fetch dt.system.buckets
| filter dt.system.table == "logs"
| fields name, display_name, retention_days, records, estimated_uncompressed_bytes
| sort estimated_uncompressed_bytes desc
```

If this returns at least one row, log tables exist and the absence of data is an ingest-side issue (no OneAgent log capture rules, OTLP not wired, ingest API not in use, or a pipeline drop rule). If it returns zero rows, the tenant doesn't have log storage provisioned at all.

---

## 🔍 What to Look for in the Results

Map findings to the report sections using this checklist.

### Misconfigurations / anomalies
- 🚨 **Single emitter > 50 % of volume** — usually a misbehaving app or a log loop. Candidate for a sampling/drop rule.
- 🚨 **Status mostly `NONE` / null** — severity isn't being parsed; downstream alerting is blind. Suggest a processor rule.
- 🚨 **JSON-shaped `content` with no `parse` applied** — fields are buried and can't be filtered/aggregated. Recommend a `parse` rule or a pipeline processor.
- 🚨 **Stack-trace fragmentation** — same exception split across many records (no multiline grouping). Recommend multiline capture config at the agent or pipeline.
- 🚨 **Repeated identical ERROR every N seconds** — health-check noise or a watchdog log loop. Candidate to drop or sample.
- 🚨 **Volume spike vs baseline** — investigate; could be incident, debug-mode left on, or a new noisy deployment.
- 🚨 **Volume drop to zero on a previously busy emitter** — ingest gap (agent down, pipeline drop, file rotation broken).
- 🔒 **PII in `content`** — credit-card-like digit runs, raw emails, `password=`/`token=`/`authorization:` leaks. Recommend pipeline masking processors.
- 🔒 **Secrets in JSON payloads** — fields named `apiKey`, `bearer`, `cookie`, `set-cookie` populated with real values.
- 🪪 **Always-null correlation fields** — `trace_id`, `span_id`, `user.id`, `session.id` referenced but never populated. Limits join surface.
- 🧹 **Dormant log sources** — `log.source` paths or process groups with < 5 events/h despite the workload still running. Stale capture rules.
- 🆔 **Unexpected high cardinality** — `content` distinct ≈ count is expected; cardinality near total on `log.source` or `service.name` is suspicious (probably a per-pod path leaking pod UID into the filename).

### Schema drift
- Same concept under multiple keys: `trace_id` vs `trace.id` vs `traceparent` vs `dt.trace_id`; `user.id` vs `userId` vs `usr`; `request.id` vs `requestId` vs `correlationId`.
- Mixed severity vocabularies: `status` uses Dynatrace levels (`ERROR`/`SEVERE`) while `loglevel` or parsed JSON uses app-native levels (`error`/`fatal`/`critical`/`50`).
- Capture all variants in the dictionary so consumers know to `coalesce()`.

### Correlation IDs to surface
- Always: `trace_id`, `span_id`, `dt.entity.host`, `dt.entity.process_group`, `dt.entity.service`, `k8s.*`, `service.name`.
- Business: `orderId`, `userId`, `sessionId`, `requestId`, `correlationId` — discover from samples.
- Cloud: `cloud.account.id`, `cloud.region`, `aws.region`, `azure.subscription.id`, `gcp.project.id`.

### Metric / SLO opportunities
- Error rate per service (`countIf(status == "ERROR") / count()`), exception-class top-N, log-volume-per-deployment (cost), parse-failure rate, latency extracted from request-completion logs, security-event count (auth failures, 401/403 surges).

### Parsing-rule opportunities
- For every emitter where §8 shows JSON-shaped `content`: propose a `parse content, "JSON:log"` snippet and the 3–5 fields worth extracting.
- For key=value formatted logs (e.g. Apache, syslog): propose a `parse content, "<KVP>"` template.
- For stack traces: recommend `dt.process_group.detected_name`-scoped grouping by first line.

---

## 🧱 Output Document Structure

Save to `<context-folder>/logs-overview-reports/logs-overview-<YYYY-MM-DD-HHMM>.md` (UTC). See the `dt-playbook-common` skill's *Shared style rules* section for filename/overwrite rules.

Section skeleton:

```
# 📜 Logs Overview — <tenant short name>
> Snapshot date, context name, scope of windows used, focus scope (if narrowed).

## 📊 Headline Numbers (24 h)
   - Total volume (label clearly as sampled estimate, e.g. "~N events (24 h, est. from 1-in-1000 sample)"), distinct severities, distinct process groups / hosts / log.sources (label as lower bounds because of sampling)
   - Earliest / latest timestamp
   - Ingest-source split (oneagent / otel / api / data_extraction / ...)
   - Bucket footprint (rows + bytes per bucket)

## 🏷️ Top Emitters
   ### By volume (process groups / hosts / K8s containers)
   ### By ERROR/SEVERE count
   ### By log.source path

## 📶 Severity Distribution
   - Table of status counts and share
   - Note on `loglevel` mismatch with `status` if present

## 🗺️ Log Families (Logical Domains)
   - Group emitters into families (e.g. "App tier — Java services", "Web tier — nginx access logs", "K8s system — kube-system", "OTel demo")
   - One row per family with representative process groups, formats (text / JSON / kvp), ingest path

## 📘 Data Dictionary — Common / Shared Fields
   - Coverage table for the fields surfaced by §6 + §7
   ### 🔗 Correlation IDs
   ### 🔄 Naming Inconsistencies (multi-variant fields)

## 🚨 Misconfigurations & Anomalies   (numbered, each with Impact + Fix)

## 🔥 Top Error Signatures
   - Table of (process group, first-line signature, count, suggested grouping/alerting)

## 🔒 PII / Sensitive Content Findings
   - One row per check (credit-card-ish, email, password/token/auth header)
   - Always include this section — state "No hits in the sampled window" if all checks were clean

## 💡 Opportunities — Metrics, Parsing & Correlations Worth Building
   ### 📈 Candidate metrics / SLOs
   ### 🧠 Suggested parse rules
   ### 🔗 Correlation chains
   ### 🧱 Suggested follow-ups

## 🧪 Quick Verification Queries
```

### 🪹 Empty-tenant report structure (logs)

When the empty-tenant fast path concludes that no logs are being ingested, write a much shorter document with this skeleton:

```
# 📜 Logs Overview — <tenant short name>
> Snapshot date, context name, windows checked (e.g. 24 h / 7 d / 30 d / 90 d), verdict.

## 📊 Headline Numbers
   - One row per window checked, all showing `total = 0`.
   - Earliest/latest = null.

## 🗄️ Bucket Configuration
   - Output of §12 (bucket name, retention, record count, bytes).
   - State whether at least one `logs`-typed bucket exists and is healthy.

## 🚨 Findings
   ### 1. 🛑 Tenant has no log ingest path active
       - **Observation:** zero rows across the windows checked.
       - **Impact:** nothing to search/dashboard/alert/correlate on; tracing and problems lose log context.
       - **Fix:** list the standard ingest paths (OneAgent log capture rules, OTLP `/api/v2/otlp/v1/logs`, public `/api/v2/logs/ingest`, syslog ingest, OpenPipeline log routing rules, K8s log forwarder via OneAgent / OTel Collector / Fluent Bit).

## 🧪 Quick Re-check Queries
   - Three short DQL snippets the user can paste once ingest is enabled to confirm flow.
```

Skip the data dictionary, correlation IDs, anomalies (beyond #1), top errors, PII, and opportunities sections entirely — they have no observations to populate.

---

## ✅ Final Self-Review Checklist

- [ ] Headline numbers, top emitters, severity distribution, ingest-source split all present.
- [ ] Counts derived from 24 h sampled queries (§1–§4, §11 trends) are labelled as estimates (e.g. "~N", "est.", or explicit "sampled ×1000"); `countDistinct` values from sampled queries are presented as lower bounds.
- [ ] Every emitter / process group mentioned in tables actually appeared in a query result (no hallucination).
- [ ] Each anomaly has a concrete fix.
- [ ] Correlation IDs section names the actual field variants observed in this tenant (don't copy verbatim from another tenant's report).
- [ ] Data dictionary lists coverage (% populated) for fields where it is interesting.
- [ ] PII section is **always** present (even if all checks returned zero — state that explicitly).
- [ ] Top error signatures section lists ≥ 5 rows if any ERROR/SEVERE traffic exists, or explicitly states "no error traffic in window".
- [ ] At least one parse-rule suggestion is included when any emitter shows JSON-shaped or key=value `content`.
- [ ] Document renders cleanly in a markdown preview (tables aligned, no broken links).
- [ ] Sensitive values (real customer IDs, real CC numbers, real emails, real bearer tokens) are **not** pasted into the document — paraphrase or redact.

---

## 🔗 Related References

- `dt-playbook-common` skill — shared scaffolding (Step 0 interview, mappings file, style rules).
- `dt-dql-essentials` skill — DQL syntax, pitfalls, operator reference.
- `dt-obs-logs` skill — log-specific DQL patterns (`matchesPhrase`, JSON parsing, severity filters).
- Sibling skill: `dt-bizevents-review` — same workflow applied to business events.
- Dynatrace docs: *Log Management and Analytics*, *OpenPipeline for Logs*, *Log ingest API*, *OTLP logs ingest*.
