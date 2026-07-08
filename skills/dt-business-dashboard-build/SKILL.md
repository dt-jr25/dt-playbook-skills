---
name: dt-business-dashboard-build
description: Generative Dynatrace Business Observability dashboard builder. Given a business-process scope (from a `dt-business-process-review` sidecar or user-supplied), produces a bespoke, process-focused Dashboard JSON for the Dynatrace Dashboards app (`version` 20+). The agent freely chooses the layout, visualization types, tile mix, and metrics — constrained by four hard guardrails: (1) no PII fields on any tile, (2) never imports to the tenant (user imports manually via UI), (3) built from either a **business** or **technical** perspective the user picks, (4) ≤ 15 data tiles (header/markdown tiles are unlimited). Every tile's DQL is validated with a `dtctl query` dry-run before the file is written. Writes to `<context-folder>/dashboards/<slug>-dashboard-<YYYY-MM-DD-HHMM>.json`. Reads `dt-playbook-common` FIRST for Step 0 (context/folder confirmation, intent check). Companion to `dt-bizevents-review` (inventory) and `dt-business-process-review` (funnel deep-dive).
---

# 📊 Business Observability Dashboard Builder

> **Purpose:** Turn a resolved business-process scope into a bespoke, tenant-ready Dashboard JSON that the user imports manually through the Dynatrace Dashboards app UI. Design is generative — the agent picks the layout, tile types, and metric mix based on the chosen perspective (business vs technical) — but four hard guardrails prevent the output from leaking PII, hitting the tenant, sprawling into noise, or shipping broken DQL.

> ⚠️ **Never overwrite a prior dashboard file.** Each run MUST be written to a new UTC-timestamped file so historical revisions stay intact.

> 🔒 **Read-only skill.** The agent MUST NOT edit this file at runtime. Improvements flow back via PR against `dt-playbook-skills` (see the common skill's self-improvement protocol).

> 🚫 **The agent NEVER imports the dashboard to the tenant, even if a future `dtctl` verb exposes a Dashboards write API.** The final delivery is always a JSON file on disk plus manual-import instructions in chat.

---

## 🎯 Hard guardrails (non-negotiable)

These four rules are checked mechanically before the file is written. Any violation halts the write.

| # | Guardrail | How it is enforced |
| --- | --- | --- |
| 1 | **No PII fields on any tile.** Fields the process review flagged in `pii_exclude` (and any obvious equivalents like `password`, `ssn`, `taxId`) MUST NOT appear anywhere in a tile's DQL, title, subtitle, `by:` clause, `filter`, or `fields` list. | Step 5 runs a case-insensitive **word-boundary** scan (regex `\b<term>\b`, case-insensitive) of every tile's DQL against the PII list, with a whitelist for Dynatrace-standard `.name`-suffix fields (see Step 5.2). Any hit → fail the tile, regenerate without the offending field. |
| 2 | **Never import to the tenant.** No `dtctl documents create`, no HTTP calls to `/platform/document-service/`, no MCP dashboard-create verbs. Output is a file on disk; the user imports via **⋯ → Upload JSON**. | Step 7 emits the manual import checklist only; no write API is ever called. |
| 3 | **Build from exactly one perspective.** Step 2 forces the user to pick **Business** *or* **Technical**. The picked perspective drives every downstream metric choice. Mixed dashboards are not offered. | Step 2's `vscode_askQuestions` requires a selection; the perspective label appears in the dashboard title and in the chat summary. |
| 4 | **≤ 15 data tiles.** "Data tiles" = anything that runs a DQL query (single-value, timeseries, table, honeycomb, top list, funnel, etc.). Header / markdown / divider tiles are **unlimited** and do not count. | Step 5 counts data tiles from the assembled JSON. > 15 → agent must consolidate or drop tiles before writing. |

---

## 🧭 What this skill does (in one paragraph)

Given a business-process scope (providers, ordered event types, correlation IDs, measures, dimensions, PII exclusions — usually inherited from a `dt-business-process-review` sidecar), the agent **designs** a bespoke dashboard: it chooses the layout (rows, grid, swim-lane, or something novel), the tile visualization types (single-value, timeseries line/bar, table, honeycomb, funnel, top list, etc.), the DQL for each tile, and the arrangement — guided by the perspective (business vs technical) the user picks and by the tile budget. The two files under `reference-examples/` are consulted for JSON shape only (which tile fields exist, how `layouts` are keyed, which `visualizationSettings` are valid) — they are **not** copied wholesale. Every tile's DQL is validated with a `dtctl query` dry-run before the JSON is written to disk. The user then imports the file manually.

---

## 🧰 Prerequisites

| Requirement | Why |
| --- | --- |
| `dtctl` CLI installed + authenticated (`dtctl auth whoami` succeeds) | Every tile's DQL is dry-run-validated via `dtctl query`. |
| **`dt-playbook-common` skill** | Owns Step 0 (context/folder confirmation, mappings file, final intent confirmation). Read it FIRST. |
| **`dt-app-dashboards` skill** (from `dynatrace-for-ai`) | Owns the Dashboards app JSON schema — `version`, `tiles`, `layouts`, `variables`, `visualizationSettings`, `unitsOverrides`. Consult before assembling the JSON. |
| **`dt-dql-essentials` skill** (from `dynatrace-for-ai`) | Every tile embeds DQL. Consult for syntax, `timeseries` vs `makeTimeseries`, `dedup`, safe aggregation patterns, and cost-safe windows. |
| **`dtctl` operator skill** (from the `dtctl` repo) | Exact `dtctl query` invocations, `-f -` stdin usage, PowerShell quoting caveats. Do not shell out from memory. |

If any skill above is not in your context by the point in the flow where it's needed, follow the missing-skill procedure in `dt-playbook-common` — print the install commands from the repo README and halt.

> 💡 **Cost guardrails:** Dry-run validation uses `| limit 1` with the tightest window that still returns a row (usually `from:-1h`). Tile-published queries use `from:now()-24h` or whatever the perspective calls for, but always with an explicit `from:`.

---

## 📥 Inputs

The skill accepts scope from three sources, in order of preference:

1. **Sidecar hand-off** (recommended, produced by `dt-business-process-review`) — a YAML file at
   `<context-folder>/process-detail-reports/business-process-detail-<YYYY-MM-DD-HHMM>.handoff.yaml`
   alongside the paired markdown report. This is the richest source and skips most of Step 1.
2. **Report path** — the user names a `business-process-detail-*.md` explicitly (e.g. `--from-report <path>`). The agent looks for a sidecar next to it; if absent, extracts scope from the report body.
3. **Fresh interview** — no prior review. The agent runs a minimal Step 1 to establish providers, event types, and correlation.

### Sidecar schema (as emitted by `dt-business-process-review`)

The dashboard skill reads these keys:

| Sidecar key | Used for |
| --- | --- |
| `scope.providers`, `scope.event_types_ordered`, `scope.process_label`, `scope.process_slug` | Filters, tile titles, output filename |
| `correlation.field`, `correlation.strength`, `correlation.alt_fields` | Cross-step joins, funnel/duration tiles |
| `measure.field`, `measure.candidates` | Revenue / throughput single-values and trends |
| `dimensions.primary`, `dimensions.candidates`, `dimensions.cardinality` | Top-N breakdowns, honeycombs, mix charts |
| `status_field.field`, `status_field.success_literal` | Success/attrition tiles (respect the review's warning if only one literal exists) |
| `pii_exclude` | Guardrail #1 — the mechanical PII scan list |
| `recommended_template` | **Treated as a hint only.** The agent is free to override based on the picked perspective and its own design judgment; the sidecar field pre-dates the generative flow. |

Missing sidecar fields → the agent asks the user for that specific fact, or runs a targeted DQL to discover it. Never fabricate.

---

## 🏃 How to run

### Step 0 — Playbook-common kickoff (mandatory, every run)

Read `dt-playbook-common` and execute its Step 0 verbatim:

- Mappings file lookup (`.dt-playbook-mappings.yaml`).
- `dtctl auth whoami`.
- Context/folder interview (single-click accept if the mapping already exists).
- Folder creation for `<context-folder>/dashboards/` if it doesn't exist.
- Mappings file update.
- **Final intent confirmation** — fires on every run, even when hand-off mode is active.

**No `dtctl query` runs until Step 0 completes with an explicit "Proceed".**

Skill-specific parameters passed to `dt-playbook-common`:

| Parameter | Value |
| --- | --- |
| `<subfolder>` | `dashboards/` |
| `<filename-stem>` | `<process_slug>-dashboard` (e.g. `deposit-lifecycle-dashboard`) |
| `<records>` | `bizevents` |

### Step 1 — Resolve scope

1. **Detect hand-off mode.**
   - If the current chat contains a completed `@dt-business-process-review` run whose sidecar is available in the workspace, use it.
   - If the user named a report path or passed `--from-report <path>`, read the sidecar next to it.
   - Otherwise → fresh mode.
2. **Hand-off mode:** load the sidecar, echo the resolved scope back to the user in a short bullet list (providers, ordered event types, correlation, measure, dimensions, PII fields), and ask *"Use this scope, or override any of it?"* — one `vscode_askQuestions` call, "Use as-is" as the recommended default.
3. **Fresh mode:** run just enough discovery to fill the minimum-required inputs:
   - **Providers** — 24 h `summarize count() by:{event.provider}`, top 10, ask the user to pick 1-3.
   - **Event types (ordered)** — 24 h `summarize count(), min(timestamp) by:{event.type}` filtered to the picked providers, ask the user to confirm the process order.
   - **Measure candidate** — 1 h sample `fields event.type, keys(...)` (or a small `sample`), look for numeric fields; ask the user to name the revenue/throughput measure.
   - **PII list** — ask the user to list any known-sensitive fields on these events, and default-add `password`, `ssn`, `taxId`, `cardNumber`, `cvv`, `email`, `name`, `address`, `phone`, `client.ip` as a starter safety list.
   - Skip anything the user already stated verbally in the current chat.

Do **not** run the process review's full Discovery Query Set here — that's the review skill's job. If the user wants a deep-dive, offer to hand back to `@dt-business-process-review`.

### Step 2 — Perspective interview (mandatory, single required question)

Use `vscode_askQuestions` with one question, two options, no free-form:

> **"Which perspective should this dashboard be built for?"**
>
> - 🧑‍💼 **Business** *(recommended for exec / product / ops audiences)* — revenue, order volume, success/attrition, geo/channel mix, trend vs prior period.
> - 🛠️ **Technical** *(SRE / dev / product engineering)* — per-step latency p50/p95/p99, failure counts, correlation coverage, stalled processes, top failing dimensions.

The user's answer becomes `<perspective>` and drives every metric pick below. It also gets stamped into the dashboard title (e.g. *"Deposit Lifecycle — Business View"*).

### Step 3 — Design the dashboard

This is where the agent's judgment does the heavy lifting. The agent MUST:

1. **Consult (do not copy) the reference examples.** Read `reference-examples/simple-kpi-dashboard.json` and `reference-examples/process-journey-dashboard.json` to learn the JSON shape (`version`, `tiles` keyed by tile-id, `layouts` keyed by the same tile-id, `visualizationSettings`, `unitsOverrides`, variable syntax). Do **not** clone their layout, tile mix, or titles — those templates felt rigid; the point of this skill is to generate something better fitted to *this* process. Treat the reference examples the way you'd treat a documentation snippet, not a starting point.
2. **Design a layout that fits the process shape** — number of steps, whether it's linear vs branching, and the chosen perspective. Rows-of-KPIs, swim-lanes, hero + grid, funnel-on-top-with-drilldowns, dense honeycomb-per-step, etc. are all fair game. Pick what makes the story *for this specific process* obvious in ≤ 5 seconds of scanning.
3. **Include both process-level AND per-step tiles.** A good dashboard answers "how is the whole process doing?" *and* "which step is the problem?" in the same view. Neither alone is enough.
4. **Respect the tile budget: ≤ 15 data tiles.** Header / markdown / divider tiles do **not** count and should be used liberally to give the dashboard visual hierarchy and to explain non-obvious metrics.
5. **Pick visualization types to match the metric.**

| Metric shape | Preferred viz | DQL shape | Notes |
| --- | --- | --- | --- |
| Single number for the whole process (revenue, volume, success rate) | `singleValue` | `summarize` to one row | Keep sparklines on if the underlying query can produce a timeseries cheaply. |
| Trend over time (single or multi-metric) | `lineChart`, `areaChart`, **timeseries** `barChart` | `makeTimeseries` / `timeseries` | Do **not** put `from:` in the query — the dashboard's global time selector drives the window. See Step 4 “Timeframe rule”. |
| Comparison across a low-cardinality dimension (≤ ~15 values) | **`categoricalBarChart`** — not `barChart` | `summarize <agg>, by:{field}` | Common footgun: `barChart` is a **time-series** viz and will fail validation if the query has no time axis. For “revenue by card type” / “sessions by country” use `categoricalBarChart`. |
| Proportion of a whole (mix, share) | **`donutChart`** (preferred) or `pieChart` | `summarize <agg>, by:{field}` | Some tenants render `pieChart` inconsistently; default to `donutChart` with `circleChartSettings` unless the user asks for a pie. |
| Ranked list (top N failing steps, top N countries by volume) | Top list or `table` | `summarize … by:{}` \| `sort` \| `limit` | Cap with `limit`; include the raw count column. |
| Funnel across ordered event types | `table` with per-step count + conversion column, or Funnel viz | `filter in(event.type, …)` \| `summarize by:{event.type}` | Use `dedup` on the correlation field if the review flagged duplicates. |
| Per-step health matrix | `honeycomb` keyed by step | `summarize by:{event.type}` | One cell per step, colour by SLO / threshold. |

> **Viz ↔ DQL matrix (memorise these two).** `barChart` + `summarize by:{}` → tile is invalid. `pieChart` with a `pieChartSettings` block is not the sanctioned shape — `donutChart` with `circleChartSettings` is. Whenever you pick a chart type, cross-check it against the DQL shape column above.

6. **Perspective-driven metric picks:**

**Business view — recommended metric palette:**
- Total process volume in the last 24 h (single value + sparkline).
- Total revenue / throughput in the last 24 h using `measure.field` with `dedup event.id`.
- Success rate — **but only if `status_field.notes` confirms both a success and failure literal exist**. Otherwise use "completions" as a plain count and add a markdown note explaining why success-rate is not shown (link to the misses in the review).
- Trend of volume and revenue vs prior 24 h (timeseries or two single-values with delta).
- Top N breakdown by `dimensions.primary` (bar or honeycomb).
- Geographic mix if `geo.country.name` is in `dimensions.candidates`.
- Per-step completion counts as a lightweight funnel table.

**Technical view — recommended metric palette:**
- Per-step count over 24 h (timeseries multi-line).
- Per-step p50/p95/p99 wall-clock latency using the correlation field (join adjacent steps).
- Failure count per step (if a failure literal exists) or "silent drop-off" count (start event without matching finish event within N minutes) — use the review's `correlation` guidance.
- Correlation coverage — % of records with the correlation field populated per step.
- Stalled processes — count of correlation IDs where the last observed step is *not* the final step and the last event is > N minutes old.
- Top N failing dimensions (which country / card type / channel is driving errors).
- End-to-end duration histogram (percentile bands).

7. **Every tile gets a clear title + subtitle** that names the metric and the time window in plain English (e.g. *"Revenue — last 24 h vs prior 24 h"*).

### Step 4 — Assemble the JSON

Follow `dt-app-dashboards` conventions verbatim. Non-negotiable structural rules:

- `version` ≥ 20.
- Top-level keys: `version`, `variables` (may be empty `{}`), `tiles`, `layouts`, `settings` (optional).
- `tiles` and `layouts` are objects keyed by tile-id (stable, human-readable, e.g. `revenue-headline`, `step-2-latency-p95`). Every tile-id in `tiles` must appear in `layouts` and vice versa.
- Tile types the agent may use: `data` (with `query` + `visualization` + `visualizationSettings`), `markdown` (for headers/explanations), `document-link` (rare), `dql-query`.
- Layout entries carry `x`, `y`, `w`, `h` (grid units). Keep to a 12- or 24-column grid; avoid tile overlaps.
- Never wrap the JSON in a fenced code block, never add JSON5 comments, never trailing-comma. Emit valid RFC-8259 JSON.

#### Timeframe rule (mandatory)

Data-tile DQL queries MUST NOT contain `from:` / `to:` bounds. Every tile inherits the dashboard's global time selector, so an inline timeframe silently overrides the user's chosen window and breaks the whole “change time → all tiles update” contract of a dashboard. Two narrow exceptions are allowed, and each MUST carry a one-line `//` DQL comment above the `fetch` explaining why:

1. **Fixed-window comparisons** where the tile *must* pin its own window (e.g. week-over-week deltas, prior-period baselines). Use `from:` / `to:` explicitly on both sides of the comparison.
2. **Reference / lookup subqueries** that need bounded enumeration independent of the main window (e.g. “baseline over the last 30 d, compared to the current selection”).

Variable-input queries (`variables[].input`) are a separate story — they do not inherit the time selector because they evaluate on dashboard load. Give them a reasonable bounded window (`from:-24h` or `from:-7d`) sized to the cardinality you expect.

#### Variables (filters) — required when the data has an obvious dimension

`variables` is **not optional** when the scope has one or more natural business dimensions (country, region, product line, tier, card type, tenant). At minimum, wire one multi-select query variable for each dimension the business team is likely to slice by. Skip variables only when there is truly no useful dimension (e.g. a single-service, single-region KPI board).

Canonical shape (multi-select, all values selected by default):

```json
{
  "version": 2,
  "key": "Country",
  "type": "query",
  "visible": true,
  "editable": true,
  "input": "fetch bizevents, from:-24h\n| filter event.provider == \"<provider>\" and event.type == \"<eventType>\"\n| filter isNotNull(geo.country.name) and geo.country.name != \"\"\n| dedup geo.country.name\n| fields geo.country.name\n| sort geo.country.name asc",
  "multiple": true,
  "defaultValue": "3420b2ac-f1cf-4b24-b62d-61ba1ba8ed05*"
}
```

Reference the variable in every tile whose data model **actually carries the field**:

```dql
| filter in(geo.country.name, array($Country))
```

Rules:

- Variable-input queries MUST return a single field, MUST filter out `null` / `""`, and MUST use `dedup` (not `summarize by:`) for distinct values.
- Multi-select variables MUST include `"defaultValue": "3420b2ac-f1cf-4b24-b62d-61ba1ba8ed05*"` (the “select all” magic token) so tiles are not empty on first load.
- Only wire a variable into a tile whose underlying event model has that field. Wiring `$Country` (a RUM-side field) into a backend-only revenue tile silently empties the tile. Note the domain split in the reading-notes markdown panel.
- Every variable in `variables` MUST be referenced by at least one tile. No dead variables.

Substitute values from the sidecar (or the fresh-mode answers) into every tile. Fields substituted from a `field name` — never from a sampled `value` (which could contain PII).

### Step 5 — Validate (mandatory, mechanical)

Run all seven checks **in order**. Any failure requires fixing before the file is written.

1. **Placeholder scan.** Search the assembled JSON for any leftover `<placeholder>`, `[PLACEHOLDER]`, `TODO`, `XXXXXXX`, `FIXME`, or reference-example strings (`acme.insurance`, `SERVICE-XXXXXXXXXXXX`, `[CLIENT_NAME]`, etc.). Any hit → agent regenerates the offending tile(s).
2. **PII scan (guardrail #1).** For each tile's DQL, title, and subtitle, case-insensitive **word-boundary** regex search (`\b<term>\b`) against the union of `pii_exclude` from the sidecar plus the safety starter list (`password`, `ssn`, `taxId`, `cardNumber`, `cvv`, `email`, `name`, `address`, `phone`, `client.ip`). Word-boundary matching prevents false positives against Dynatrace-standard fields whose identifiers happen to end in `.name` — the following are **whitelisted** and MUST NOT be flagged by a bare `name` term: `geo.country.name`, `geo.region.name`, `geo.city.name`, `browser.name`, `os.name`, `device.name`, `service.name`, `host.name`, `container.name`, `k8s.pod.name`, `k8s.node.name`, `k8s.namespace.name`, `k8s.cluster.name`, `k8s.container.name`, `k8s.deployment.name`, `k8s.statefulset.name`, `k8s.daemonset.name`, `dt.entity.*.name` (any entity type). Terms that end in `.name` in the PII list itself (e.g. a custom `customer.name`) are matched literally as full field paths and still fail the scan. Any true hit → drop or replace the tile.
3. **Tile budget scan (guardrail #4).** Count only tiles whose `type` runs a DQL query (`data`, `dql-query`). Header, markdown, and divider tiles are unlimited. If data-tile count > 15 → the agent must consolidate (merge two single-values into one two-line tile, drop the lowest-value tile) and re-count. **Target 10–15 data tiles for a proper business or technical review dashboard.** Below ~8 data tiles the dashboard usually looks empty; only ship a bare-bones dashboard when the scope is genuinely narrow (single-metric board, embedded widget, etc.) and note the reason in the chat summary.
4. **Timeframe scan.** For every `data` / `dql-query` tile, grep the `query` string for `from:` or `to:`. Any hit → the tile MUST carry a DQL `//` comment on the line above the offending `fetch` / `timeseries` justifying it (per the Step 4 “Timeframe rule” exceptions list). No comment → strip the inline timeframe and let the tile inherit the global selector. Variable-input queries are exempt.
5. **Viz↔DQL shape scan.** For every `data` tile, cross-check the picked `visualization` against the DQL shape:
   - `barChart` / `lineChart` / `areaChart` / `bandChart` → query MUST contain `makeTimeseries` or `timeseries`. If it does not, replace with `categoricalBarChart` (or the right categorical/scalar viz).
   - `categoricalBarChart` / `pieChart` / `donutChart` → query MUST contain `summarize … by:{}`.
   - `singleValue` / `meterBar` / `gauge` → query MUST return a single row.
   - `pieChart` → rewrite to `donutChart` with `circleChartSettings` unless the user explicitly asked for a pie.
6. **Variables scan.** Every variable declared in `variables` MUST be referenced by at least one tile via `$Key` or `array($Key)`; every multi-select variable MUST carry the `"3420b2ac-f1cf-4b24-b62d-61ba1ba8ed05*"` default. If the sidecar/interview surfaced an obvious business dimension (country, region, product, card type, tier) and no matching variable exists → add one before writing.
7. **DQL dry-run.** For each tile, run its DQL through `dtctl query` with `| limit 1` appended (or use the smallest window that returns a row). Use PowerShell here-string quoting per `dt-playbook-common`:

   ```powershell
   @'
   fetch bizevents, from:-1h
   | filter event.provider == "<provider>" and event.type == "<eventType>"
   | summarize revenue = sum(amount), by:{cardType}
   | limit 1
   '@ | dtctl query -f - -o json
   ```

   - **Exit code 0 + valid JSON row** → tile passes.
   - **Non-zero exit or parser error** → the DQL is broken. The agent MUST fix it (consult `dt-dql-essentials`) or replace the tile with an equivalent working one. Repeat the dry-run until it passes. Never write a dashboard containing a tile whose DQL failed dry-run.
   - **Zero rows returned** is acceptable (some tiles will legitimately be empty at 1 h), but only if the query is syntactically valid. Use a wider window (`-24h`) to disambiguate before deciding.

Do not batch these into a single `dtctl` call; run one dry-run per tile so failure attribution is unambiguous.

### Step 6 — Write the file

Path (per `dt-playbook-common`):

```
<context-folder>/dashboards/<process_slug>-dashboard-<YYYY-MM-DD-HHMM>.json
```

- Filename uses UTC minute-precision. Collision → bump to next minute or append seconds. **Never overwrite.**
- Emit one JSON blob, pretty-printed (2-space indent). No BOM, no trailing newline mess.
- Do not write any sidecar or auxiliary files unless the user asked for one.

### Step 7 — Chat summary + manual import instructions

Emit a concise chat summary containing:

1. The full workspace-relative path of the dashboard file (as a markdown link).
2. The chosen perspective and a one-sentence rationale.
3. Tile inventory as a compact table: tile title | viz type | swim-lane / row it belongs to. Group by row.
4. Any dry-run failures that were auto-corrected (be transparent — "3 tiles regenerated because the original DQL failed; final versions all passed").
5. Any tiles that were dropped by the tile-budget scan and why.
6. **Manual import instructions (never an API call):**
   > 1. Open the Dynatrace **Dashboards** app on the `<context>` tenant.
   > 2. Click **⋯** (top-right) → **Upload JSON**.
   > 3. Select `<dashboard-file-path>`.
   > 4. Save the dashboard with your preferred title / owner / sharing settings.

Do not offer a `dtctl` command that would import the dashboard, even if one becomes available.

### Step 8 — Self-improvement prompt (via `dt-playbook-common`)

Follow the common skill's self-improvement protocol. If — and only if — you noticed a concrete, generalisable gap during the run (a metric palette that was clearly missing for a perspective; a new visualization pattern that worked well; a DQL pitfall that tripped multiple tiles), surface **one** batched suggestion offering a PR draft against `dt-playbook-skills`. Otherwise stay silent.

---

## 🚫 What this skill NEVER does

- Import, push, publish, or otherwise transmit the dashboard to the tenant. Not via `dtctl`, not via HTTP, not via MCP.
- Include any field named in `pii_exclude`, or any obvious PII/PCI/secret equivalent, anywhere in the JSON.
- Copy a reference-example layout wholesale. The examples exist to show JSON shape, not to be duplicated.
- Build a dashboard mixing business and technical perspectives in one file. Two perspectives → two runs → two files.
- Emit a file whose data-tile count > 15.
- Write a file whose tile DQL failed dry-run.
- Ask the user to run `dtctl documents create` or any other write command as a "convenience next step".
- Overwrite a prior dashboard file, or write the file if the target folder does not exist and cannot be created.
- Edit any installed `SKILL.md` or `.agent.md` at runtime. Improvements flow via the common skill's self-improvement protocol.

---

## 📚 Reference examples (`reference-examples/`)

The two JSON files under `reference-examples/` are **shape references**, not templates. Consult them to answer:

- "What's the exact key path for a single-value visualization's units override?"
- "How does the Dashboards app render a multi-metric timeseries tile?"
- "What layout grid keys are valid?"

Do **not** clone their layouts or tile mixes. See [`reference-examples/README.md`](reference-examples/README.md) for the intended use.

---

## 🔗 Related skills

- **`dt-playbook-common`** — Step 0 kickoff, mappings file, final intent confirmation, self-improvement protocol.
- **`dt-business-process-review`** — depth-first companion that produces the sidecar this skill consumes.
- **`dt-bizevents-review`** — breadth-first companion for tenant-wide bizevents inventory (usually run *before* the process review).
- **`dt-app-dashboards`** (from `dynatrace-for-ai`) — the Dashboards app JSON schema reference.
- **`dt-dql-essentials`** (from `dynatrace-for-ai`) — DQL syntax, pitfalls, cost patterns.
- **`dtctl` operator skill** — exact `dtctl query` invocations, PowerShell quoting.
