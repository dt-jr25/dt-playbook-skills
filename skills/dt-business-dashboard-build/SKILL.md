---
name: dt-business-dashboard-build
description: >-
  Re-runnable Dynatrace Business Observability dashboard-builder playbook.
  Given one or more `event.provider`s and a chosen set of `event.type`s,
  produces a ready-to-import Dashboard JSON file (Dynatrace Dashboards app,
  version 20+) with per-KPI single-value tiles, trend charts, and (optional)
  end-to-end process-journey layout with per-step business KPIs and
  IT/Security/Carbon health indicators. Invoke when the user asks to build,
  create, or scaffold a business observability dashboard, business KPI
  dashboard, business process dashboard, or bizevents dashboard - e.g.
  "build a business observability dashboard", "create a KPI dashboard for
  our checkout flow", "scaffold a business process dashboard", "give me a
  bizevents dashboard for provider X", "build a dashboard from that
  process review", "turn last night's business-process report into a
  dashboard", or @dt-business-dashboard-build.
  Reads `dt-playbook-common` FIRST for Step 0 (context/folder confirmation,
  intent check), then interviews the user on template choice + provider/type
  scope, samples the chosen events to discover measure/dimension/correlation
  fields, and finally writes the substituted JSON to
  `<context-folder>/dashboards/<slug>-dashboard-<YYYY-MM-DD-HHMM>.json`.
  Supports a hand-off mode that consumes the scope + correlation ID +
  measure/dimension picks from a preceding `dt-business-process-review` run
  (same-conversation context or written report file) and skips the
  Discovery Queries the review already answered, saving Grail scan cost.
  Do NOT use for tenant-wide bizevents inventory (use `dt-bizevents-review`),
  process characterization / funnel analysis (use `dt-business-process-review`),
  or log dashboards (out of scope).
---

# 📊 Business Observability Dashboard Builder — Playbook

> **Purpose:** Re-runnable recipe for producing a ready-to-import Dynatrace
> Dashboard JSON that visualizes a chosen set of `bizevents` — either as a flat
> Business KPI dashboard (revenue / orders / success rate + trend charts) or as
> a horizontal end-to-end **Business Process Journey** dashboard with per-step
> KPIs and IT / Security / Carbon health indicators.
> **Sibling output:** `<context-folder>/dashboards/<slug>-dashboard-<YYYY-MM-DD-HHMM>.json` (e.g. `acme-prod-readonly/dashboards/checkout-kpi-dashboard-2026-07-02-1420.json`).
> **Audience:** AI coding agents (Copilot, Claude, etc.) and the engineer driving them.

> 📖 **Read the `dt-playbook-common` skill first.** It owns the prerequisites, the Step 0 kickoff interview, intent confirmation, the per-workspace `.dt-playbook-mappings.yaml`, empty-tenant fast-path conventions, duplicate-snapshot logic, PowerShell quoting note, shared style rules, and the self-improvement protocol. This skill only contains what is unique to dashboard building.

> 🔁 **Relationship to sibling skills:**
> - Run `dt-bizevents-review` first if you don't yet know which `event.provider`s and `event.type`s exist in the tenant.
> - Run `dt-business-process-review` first if you want a rigorous funnel/correlation analysis before committing to a dashboard shape — its report tells you which correlation ID stitches steps and which fields are safe to dashboard. **This skill has a dedicated hand-off mode (§Hand-off mode) that consumes a fresh review either from the same conversation's context or from the written report file, skipping every Discovery Query the review already answered.**
> - Run **this** skill when the user is ready to *ship a dashboard*: they know (or you can help them pick) the providers/types, and they want an import-ready JSON file, not a markdown analysis.

---

## 🧷 Per-skill parameters (for `dt-playbook-common`)

| Parameter | Value |
| --- | --- |
| `<subfolder>` | `dashboards/` |
| `<filename-stem>` | `<slug>-dashboard` (slug is derived from the process label — see §Step 2c) |
| `<records>` (used in fast-path prompts) | `bizevents` |
| Discovery Query 1 (headline) | §1 below |
| Initial-overview queries (run when Query 1 > 0) | §2–§3 below |
| Bucket-inventory query (empty-tenant fast path) | §9 below |
| Deep-dive queries (run after Step 2 — field discovery for template substitution) | §4–§8 below |

Full output path: `<context-folder>/dashboards/<slug>-dashboard-<YYYY-MM-DD-HHMM>.json` (UTC, never overwritten).

> ⚠️ **Output extension is `.json`, not `.md`.** Every other `dt-*-review` skill writes markdown; this one writes a Dashboard JSON that the user imports into the Dynatrace Dashboards app **manually**. Do not wrap the JSON in a fenced code block, do not add commentary inside the file, do not prettify beyond a single-pass `JSON.stringify(…, null, 2)` equivalent.

> 🗄️ **The emitted file is a reference copy — it stays on disk and is not auto-imported.** The user controls when, where, and whether the dashboard lands in a tenant. The timestamped, never-overwritten filename means every run produces a fresh snapshot you can diff against prior versions, roll back to, or re-import into a different tenant. The skill never calls a Dashboards write API and never runs an import for the user — see §Step 5 for the manual import checklist.

### Extra prerequisites (beyond `dt-playbook-common`)

| Requirement | Notes |
| --- | --- |
| Token scopes: read access to `bizevents` (Grail query scopes) | A read-only context (e.g. `*-readonly`) is enough for the discovery queries — the dashboard is imported through the UI, no write scopes needed |
| Dynatrace **Dashboards** app installed on the target tenant | Import target. Version handshake: templates in this skill are `"version": 20` (simple-kpi) and `"version": 21` (process-journey). Newer Dashboards apps import them fine; older ones may need a manual version bump |
| At least one `event.provider` already present in the tenant | If `bizevents` is empty, fall back to the `dt-bizevents-review` skill's empty-tenant fast path — there is nothing to dashboard |
| **Domain skills loaded for the agent:** `dt-app-dashboards`, `dt-dql-essentials`, and the `dtctl` operator skill | Non-optional. `dt-app-dashboards` supplies the tile / layout / `visualizationSettings` schema this skill's Step 4 depends on; `dt-dql-essentials` supplies the DQL syntax rules for every embedded tile query; the `dtctl` operator skill supplies the exact CLI invocations for §Discovery Queries. All three are installed via this repo's `ai.repo.yaml` manifest (see the repo README). If any is not in the agent's context at Step 4, follow `dt-playbook-common` §Prerequisites' missing-skill procedure — print the install commands from the README and halt; do not run installers. |

> 📚 **Read these three domain skills BEFORE Step 2 fires** — Step 4's substitution work assumes their knowledge is already in the agent's context. Skipping them is the single biggest cause of slow, error-prone dashboard builds (wrong tile shape, invalid DQL inside a tile, wrong `dtctl` flag). Read `dt-playbook-common` first for Step 0 kickoff, then this skill, then load the three domain skills, then run Discovery Queries §1–§9.

---

## 🗂️ Templates shipped with this skill

Both live under `skills/dt-business-dashboard-build/templates/` and are the **only** dashboard skeletons this skill produces. Do not invent a third layout at runtime — extend a template or file a PR against the repo.

| Template file | Best for | Key tiles |
| --- | --- | --- |
| `simple-kpi-dashboard.json` | A single business flow (checkout, signup, order placement, claim registration, …) where the user wants revenue / volume / success-rate at a glance plus trend + category breakdown | 3 headline single-values (Revenue, Orders, Success Rate), 2 timeseries (Orders over time, Revenue over time), 1 donut (Orders by Category), 1 service-perf timeseries, 1 service error-rate timeseries, 1 recent-events table |
| `process-journey-dashboard.json` | A multi-step end-to-end business process (5 steps by default) where each step needs its own Business KPI + IT-Issues + Security + Carbon tile row, plus per-step trend/breakdown charts | Horizontal 5-column journey layout: markdown step header → 4 health tiles (IT / Security / Business KPI / Carbon) → drilldown links → step-specific trend chart → step-specific breakdown chart |

### Token / placeholder inventory

The agent's job at §Step 4 is to replace every placeholder in the chosen template with a real value from the tenant. Do **not** leave placeholders in the emitted JSON — an unresolved placeholder either blocks the import or renders a broken tile.

> **Sentinel convention:** the `simple-kpi` template uses unique `__NAME__` sentinels (double underscore, upper snake) for every field/currency substitution, so string-level replacement can't collide with real tenant field names. Anything wrapped in `__…__` **must** be replaced before the file is written; the §Step 4.3 leak scan enforces this.

#### `simple-kpi-dashboard.json` placeholders

| Placeholder (literal string in the JSON) | Replace with | Where it appears | Expected count |
| --- | --- | --- | --- |
| `[CLIENT_NAME]` | Human-readable customer / process name (e.g. `Acme Insurance – Claim Registration`) | Tile 0 (markdown header) | 1 |
| `your.order.event.type` | The chosen `event.type` (single) | Every `filter event.type == "…"` clause | 7 |
| `__MEASURE_FIELD__` | The chosen **business measure** field name discovered in §5 (e.g. `order.total`, `claim.amount`, `payment.value`) | Revenue single-value, Revenue-over-time timeseries, Recent Events table | 3 |
| `__STATUS_FIELD__` | The chosen **status field** discovered in §6 (e.g. `status`, `result`, `outcome`) | Success Rate single-value, Recent Events table | 2 |
| `__SUCCESS_LITERAL__` | The success-value literal discovered in §6 (e.g. `success`, `OK`, `COMPLETED`) | Success Rate single-value | 1 |
| `__DIMENSION_FIELD__` | The chosen **category dimension** discovered in §7 (e.g. `order.channel`, `product.category`) | Orders-by-Category donut | 1 |
| `__CURRENCY_CODE__` | The tenant's business currency ISO code — lowercase (`usd`, `eur`, `gbp`, `jpy`, …). If unknown, substitute `usd` and flag it in the chat summary — do **not** leave `__CURRENCY_CODE__` in the file. | Tile 1 `unitsOverrides.baseUnit` | 1 |
| `SERVICE-XXXXXXXXXXXX` | The `dt.entity.service` ID of the primary owning service, discovered in §8 | Service Performance timeseries + Error Rate timeseries | 2 |

#### `process-journey-dashboard.json` placeholders

This template is fully populated with a working `acme.insurance` claim-lifecycle example. It doubles as a **showcase** — the user is expected to keep the layout and swap the domain wording. Unlike `simple-kpi`, this template does **not** use `__…__` sentinels — substitutions are literal-value swaps of the example domain. Be extra careful with string-level replacement and always JSON-round-trip parse after every batch of swaps.

| Placeholder / literal | Replace with | Where it appears |
| --- | --- | --- |
| `acme.insurance` | The chosen `event.provider` | Every `filter event.provider == "…"` clause |
| Step markdown titles (`## 📋 Claim registered`, `## 📱 Document submission`, …) | The 5 user-chosen step labels (with matching emoji) | Tiles 0–4 |
| Step event types (`claim.registered`, `documents.submitted`, `internal.review`, `third.party.review`, `claim.settled`, `customer.call`, `further.evidence`, `documents.submitted.assets.corrupt`, `warehouse.received`, `item.picked`, `order.packed`) | The user's actual step event types (from §2b) — map in order Step 1 → Step 5 | Every scoped tile query |
| Correlation ID field `claim.id` | The correlation ID discovered in §4 (typically `order.id`, `session.id`, `case.id`, etc.) | All duration / funnel / lookup tiles |
| Business measure fields inside donuts / categorical bars (`claim.type`, `location`) | The dimensions discovered in §7 for the chosen provider | Tiles 11, 24, 78 |
| Drilldown-URL host `wkf10640` (inside `https://wkf10640.apps.dynatrace.com/...`) | The environment host from `dtctl auth whoami` (the leading subdomain of the apps URL — e.g. `abc12345`) | Drilldown markdown tiles (13, 53, 54, 55, 56) |
| **Base64 tenant IDs inside drilldown URLs** (e.g. `vu9U3hXa3q0AAAABAChhcHA6ZHluYXRyYWNlLmJpei5mbG93OmJpei1mbG93LXNldHRpbmdz…`) | Regenerate per-tenant by opening the target app (Business Flow, Vulnerabilities, Carbon) in the destination tenant and copying its URL. **Do not ship the source tenant's IDs to another tenant** — the link will 404. If regeneration is out of scope for this run, strip the encoded segment so the URL points at the app root. | Drilldown markdown tiles (13, 53, 54, 55, 56) |
| **Anomaly-detector `davis.componentState.[…].query` fields** that duplicate their tile's `query` | Substitute **both** the tile-level `query` AND the analyzer's inner `query` at the same time. If they drift, the tile renders one dataset while Davis analyzes a different one. Search the file for the string `"AutoAdaptiveAnomalyDetectionAnalyzer"` to find every affected tile. | Tiles 14, 22, 33, 37, … (any tile with `visualization":"davis"`) |
| Placeholder health tiles that hard-code `problemExample = 0`, `securityExample = "4/10"`, `carbonExample = 452` | Leave the hard-coded fallback **but** keep the existing in-file comment block pointing the user at the two integration paths (Davis problems query vs. topology lookup) | All IT / Security / Carbon tiles (6, 18, 25, 66–77) |
| Currency `usd` in `unitsOverrides` | Tenant's business currency if known — only relevant if the KPI is monetary | Tile 26 (`avgCallRegister`) |

> 💡 **Fewer than 5 steps?** The template's 5-column layout does not gracefully collapse, **and** several derived tiles compute values *between* steps (e.g. tile 26 spans `customer.call` → `claim.registered`, tile 28 spans `documents.submitted` → `documents.submitted.assets.corrupt`, tile 32 spans `internal.review` → `third.party.review`, tile 37 same pair). Removing a step-column leaves these derived tiles referencing deleted event.types → broken tiles. Recommend keeping exactly 5 steps; if the user insists on 3 or 4, walk them through which derived tiles must also be deleted (grep the file for the removed event.type name to enumerate the dependencies before writing). If they pick more than 5, tell them the process-journey template is capped at 5 and offer to split into two dashboards.

---

## 🤝 Hand-off mode (from `dt-business-process-review` or a prior review report)

> **Why this exists:** the `dt-business-process-review` skill already runs almost every Discovery Query this skill would run — §1 headline, §2 provider inventory, §3 event-type inventory, §8 correlation-ID discovery, §14 business-measure + dimension sweep, §15 PII probe. Re-running them wastes Grail scan budget and clock time. Hand-off mode inherits those results so this skill only runs the **incremental** queries it truly needs (template-specific field discoveries the review didn't perform).

### Preferred hand-off transport — the structured packet sidecar

Every non-empty-scope `dt-business-process-review` run writes a YAML sidecar at:

```
<context-folder>/process-detail-reports/business-process-detail-<YYYY-MM-DD-HHMM>.handoff.yaml
```

See `dt-business-process-review` §Structured hand-off packet for the full schema (fields: `scope.providers`, `scope.event_types_ordered`, `scope.process_label`, `scope.process_slug`, `correlation.field`, `correlation.strength`, `measure.field`, `measure.numeric_confirmed`, `dimensions.primary`, `status_field.field`, `status_field.success_literal`, `pii_exclude`, `recommended_template`).

**This is the fast path.** Prefer it over any of the fallbacks below. When the packet exists, the mapping to this skill's Discovery Queries is unambiguous:

| Packet field → | This skill's substitution / decision |
| --- | --- |
| `scope.providers` | `<providers>` — skip §2 |
| `scope.event_types_ordered` | `<eventTypes>` — skip §3; for `process-journey`, this is the step order (Step 1 = index 0) |
| `scope.process_label` | `[CLIENT_NAME]` substitution + basis for `<slug>` (use `scope.process_slug` verbatim if present) |
| `correlation.field` + `correlation.strength = strong` | Substitute `claim.id` in process-journey; skip §4. If `strength = weak`, re-run §4 to confirm. |
| `measure.field` + `measure.numeric_confirmed = true` | Substitute `__MEASURE_FIELD__`; skip §5. If `numeric_confirmed = false`, re-run §5 to sanity-check. |
| `dimensions.primary` | Substitute `__DIMENSION_FIELD__`; skip §7 |
| `status_field.field` + `status_field.success_literal` (nullable) | Substitute `__STATUS_FIELD__` + `__SUCCESS_LITERAL__`; skip §6. If either is `null`, re-run §6 (simple-kpi only). |
| `pii_exclude` | Fields to remove from the Recent Events table (`simple-kpi` tile 9) before writing |
| `recommended_template` | Pre-select in Step 2a (still require explicit user confirmation) |

`__CURRENCY_CODE__`, `SERVICE-XXXXXXXXXXXX`, step labels, and slug (if not in the packet) are **not** in the review — always collect them in Step 2c / §8 as normal.

### Detection

Enter hand-off mode when **any** of the following is true — in this priority order:

1. **Packet in current conversation.** A prior turn in this chat wrote (or referenced) a `.handoff.yaml` path. Read it from disk and use it as the source of truth.
2. **Packet named explicitly.** The user says something like *"build a dashboard from `<path>.handoff.yaml`"* or invokes `@dt-business-dashboard-build --from-report <path>` (accept either the `.md` or the `.handoff.yaml` — they share a stem). Read the packet; if only the markdown exists, fall through to fallback (4).
3. **Packet auto-discovered.** The user says "build a dashboard from the last review" (or similar) and exactly one recent `.handoff.yaml` exists under `<context-folder>/process-detail-reports/`. Read that packet.
4. **Markdown-only fallback (legacy).** No packet exists but a `business-process-detail-*.md` report does. Grep the report for the fields listed in the fallback table below. This is slower and more brittle — flag it in the chat summary and recommend the user re-run the review to generate a packet.

If none of the above matches, run the full workflow (§1–§9). Do **not** silently guess.

### Markdown fallback — field-extraction table (used only when the packet is missing)

| Dashboard-builder input | Report section to grep |
| --- | --- |
| `<providers>` | Report header `> **Scope:** providers = [...]` |
| `<eventTypes>` | Report header `> **Scope:** event types = [...]` |
| `<processLabel>` | Report header `> **Process label:** <name>` |
| `<bestCorrelationIdField>` | `## 🔗 Correlation IDs → ### ✅ Recommended primary correlation` |
| `<measureField>` (candidates) | `## 💡 Recommended Business Metrics → 🧱 Event-level metrics` — look for `sum(<field>)` in DQL sketches |
| `<dimensionField>` (candidates) | `## 💡 Recommended Business Metrics → 📊 Dashboard composition` |
| `<statusField>` + success literal (simple-kpi only) | `## 📘 Per-Event Data Dictionary` — field with role = "status" and a low-cardinality sample |
| PII exclusions | `## 🔒 PII / Sensitive Content Findings` |
| Terminal / start step | `## 🧭 Process Map → ### 🛣️ Canonical happy path` |

If a field the builder needs is not in the report either (e.g. status field for simple-kpi, weak correlator, missing measure), fall back to the corresponding §Discovery Query in this skill — do not invent a value.

### Which discovery queries to skip vs. still run

| This skill's query | With packet | Markdown-only fallback |
| --- | --- | --- |
| §1 headline | ✅ Skip | ✅ Skip |
| §2 provider inventory | ✅ Skip | ✅ Skip |
| §3 event-type inventory | ✅ Skip | ✅ Skip |
| §4 correlation ID (process-journey only) | ✅ Skip if `correlation.strength = strong`; re-run if `weak` | ⚠️ Re-run (strength unknown from markdown) |
| §5 business measure | ✅ Skip if `measure.numeric_confirmed = true`; re-run otherwise | ⚠️ Re-run to sanity-check |
| §6 status / success literal (simple-kpi only) | ✅ Skip if `status_field.field` is non-null; re-run otherwise | ⚠️ Usually re-run — rarely surfaced by the review |
| §7 category dimension | ✅ Skip if `dimensions.primary` is non-null; re-run otherwise | ✅ Skip if a dimension is named in the report |
| §8 owning service (simple-kpi only) | ❌ Always run | ❌ Always run |
| §9 bucket inventory | ✅ Skip | ✅ Skip |

**Rule of thumb:** never skip §8 (owning service) for `simple-kpi` — the review does not produce it.

### What still requires a user decision even in hand-off mode

Hand-off inherits *data*, not *user preference*. These Step 2 questions still fire:

- **Step 0 intent confirmation** — always fires per `dt-playbook-common` (non-negotiable, even inside the same conversation).
- **Step 2a — template choice** (simple-kpi vs process-journey). Pre-select `recommended_template` from the packet, but still require explicit confirmation.
- **Step 2c narrowing** — the packet's `event_types_ordered` may be broader than what belongs on a single dashboard. For `simple-kpi`, still ask the single-select headline event.type; for `process-journey`, confirm the ordered subset + step labels.
- **Currency**, **process/dashboard slug** (if not in packet), **step labels** — user-facing text the review does not produce.

### Chat summary must credit the hand-off source

The scope summary printed after the file is written (per §Persisting the scope in the emitted file) must add one line:

```
> Hand-off from: <packet path> (packet v<version>) | <report path> (markdown fallback) | prior conversation turn
> Queries skipped via hand-off: [§1, §2, §3, …]  (Grail scans saved: <N>)
> Queries still executed: [§6, §8, …]
```

This gives the user a receipt for the cost-saving choice and lets them audit whether the builder took a shortcut it shouldn't have.

---

## 📋 Discovery Query Set

Run §1–§3 during Step 1 (overview). Run §4–§8 only **after** Step 2 has produced concrete `<providers>`, `<eventTypes>`, and template choice. Throughout, treat `<providers>` and `<eventTypes>` as placeholder lists that the agent substitutes inline (DQL `in(field, {"a", "b"})`).

### 1. Headline numbers — bizevents volume (last 24 h)

```dql
fetch bizevents, from:-24h
| summarize total = count(),
            distinctTypes = countDistinct(event.type),
            distinctProviders = countDistinct(event.provider),
            earliest = min(timestamp),
            latest   = max(timestamp)
```

If `total == 0`, halt and tell the user to run `@dt-bizevents-review` (or its empty-tenant fast path) first — a dashboard cannot be built on an empty stream. Use §9 to distinguish "no producers" from "no bucket".

### 2. Provider inventory (last 24 h)

```dql
fetch bizevents, from:-24h
| summarize count = count(), distinctTypes = countDistinct(event.type), by:{event.provider}
| sort count desc
| limit 50
```

Use this to power the Step 2b multi-select.

### 3. Event-type inventory **scoped to the chosen providers** (last 24 h)

```dql
fetch bizevents, from:-24h
| filter in(event.provider, {<providers>})
| summarize count = count(),
            firstSeen = min(timestamp),
            lastSeen  = max(timestamp),
            by:{event.provider, event.type}
| sort count desc
| limit 200
```

Use this to power the Step 2c multi-select (event types to include). Note any `event.type`s with `count < ~5/h` — flag them as noisy candidates the user probably doesn't want on a KPI dashboard.

---

> 🔻 **The queries below run only after Step 2 has produced `<providers>`, `<eventTypes>`, and `<templateChoice>`.**

### 4. Correlation-ID field discovery (1 h, scoped) — process-journey template only

```dql
fetch bizevents, from:-1h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| fieldsSummary trace_id, span_id, dt.trace_id,
                user.id, session.id, customer.id,
                order.id, orderId, claim.id, case.id, transaction.id, booking.id,
                policy.id, invoice.id, payment.id
```

Pick the field with the highest coverage across the selected event.types. If multiple candidates tie, prefer the domain-specific one (`claim.id` beats `session.id` for an insurance flow) over the generic one. Record the choice in the report header. Rerun the `dt-business-process-review` §8 cross-step-overlap check if the user wants strict validation before committing.

### 5. Business-measure field discovery (1 h, scoped)

Sample one record per chosen event.type and inspect the payload for numeric fields:

```dql
fetch bizevents, from:-15m
| filter event.provider == "<provider>" and event.type == "<eventType>"
| limit 1
```

For each numeric-looking field (`amount`, `total`, `price`, `value`, `qty`, `count`, `score`, `duration*`), confirm it's actually populated + numeric at scale:

```dql
fetch bizevents, from:-1h
| filter event.provider == "<provider>" and event.type == "<eventType>"
| summarize nonNull = countIf(isNotNull(<field>)),
            distinctVals = countDistinct(<field>),
            minVal = min(toDouble(<field>)),
            maxVal = max(toDouble(<field>)),
            avgVal = avg(toDouble(<field>)),
            total = count()
```

Pick the field with the highest coverage that also passes a sanity check (min/max within a plausible business range). This is the substitute for `__MEASURE_FIELD__` in the simple-kpi template.

### 6. Status / success-flag field discovery (1 h, scoped) — simple-kpi template only

```dql
fetch bizevents, from:-1h
| filter event.provider == "<provider>" and event.type == "<eventType>"
| fieldsSummary status, result, outcome, state, success
```

Then for each candidate, enumerate its value set:

```dql
fetch bizevents, from:-1h
| filter event.provider == "<provider>" and event.type == "<eventType>"
| summarize cnt = count(), by:{<candidateStatusField>}
| sort cnt desc
| limit 20
```

Identify the success literal (`"success"`, `"OK"`, `"COMPLETED"`, `"true"`, …). If no such field exists, tell the user the Success Rate tile will be left as a placeholder in the emitted JSON with a comment above it explaining the missing input.

### 7. Category / dimension discovery (1 h, scoped)

```dql
fetch bizevents, from:-1h
| filter event.provider == "<provider>" and event.type == "<eventType>"
| fieldsSummary <candidateDimensions>
```

`<candidateDimensions>` = every non-numeric, non-ID field discovered in §5 (`channel`, `region`, `productCategory`, `claim.type`, `currency`, `tenantId`, `customerSegment`). Filter to fields with `cardinality < 50` for donut / categorical-bar use.

### 8. Owning-service discovery (1 h, scoped) — simple-kpi template only, tile 7 & 8

The simple-kpi template has a `SERVICE-XXXXXXXXXXXX` placeholder in tiles 7 and 8. Two ways to fill it:

- **Preferred — from bizevent → span linkage:** if any of the events carry `trace_id`, look up the associated service:
  ```dql
  fetch bizevents, from:-1h
  | filter event.provider == "<provider>" and event.type == "<eventType>"
  | filter isNotNull(trace_id)
  | limit 1
  | lookup [ fetch spans, from:-1h | fields trace.id, dt.entity.service, service.name ],
           sourceField:trace_id, lookupField:trace.id
  | fields trace_id, lookup.dt.entity.service, lookup.service.name
  ```
- **Fallback — ask the user.** Present the top ~5 services from the tenant and let them pick:
  ```dql
  fetch dt.entity.service
  | fields entity.name, id
  | limit 25
  ```

If neither path resolves a single service, replace tiles 7 and 8's `filter:{dt.entity.service == "SERVICE-XXXXXXXXXXXX"}` clause with a broader filter (`filter:{service.name matches "*<hint>*"}`) and flag it in the emitted-JSON summary. Do **not** ship a dashboard containing the literal `SERVICE-XXXXXXXXXXXX` — it will produce broken tiles.

### 9. Bucket inventory (used by the empty-tenant fast path)

Same as the bizevents-review skill §10:

```dql
fetch dt.system.buckets
| filter dt.system.table == "bizevents"
| fields name, display_name, retention_days, records, estimated_uncompressed_bytes
```

---

## 🎯 Step 2 — Focus interview (this skill overrides the common single-question form)

Step 2 splits into **three** sequential questions. All must be answered before any deep-dive query (§4+) runs. Step 0 (mappings + intent confirmation) and Step 1 (headline + provider/type inventory) are unchanged from `dt-playbook-common`.

### Step 2a — Pick a template

> *"Which dashboard shape do you want?"*

Present the two options via `vscode_askQuestions`:

- **📊 Simple Business KPI dashboard** (recommended for a single flow) — revenue / orders / success-rate headline, trend + category charts, one service-perf strip. Ships as `simple-kpi-dashboard.json`. Best when the user wants **one dashboard for one process** and the emphasis is on totals, trends, and category slicing.
- **🧭 Process Journey dashboard** — horizontal 5-step layout, per-step Business KPI + IT / Security / Carbon health tiles, drilldown links, per-step trend and breakdown charts. Ships as `process-journey-dashboard.json`. Best when the user wants **end-to-end visibility over a multi-step flow** and each step deserves its own health story.
- **🛑 Cancel**

Do not offer a "custom" or "hybrid" option — extend a template later via PR.

### Step 2b — Pick provider(s)

> *"Which `event.provider`(s) should this dashboard cover? Pick one or several — they should describe the same business flow."*

Present the top providers from §2 as multi-select. Recommend ≤ 2 providers for `simple-kpi` (it's tuned for a single filter), ≤ 3 for `process-journey`. Require at least one.

### Step 2c — Pick event.type(s) and (for `process-journey`) step ordering

Run §3 filtered to the chosen providers.

- **Simple KPI:** ask *"Which `event.type` should headline this dashboard?"* — single-select. The simple-kpi template assumes **one** event.type drives every tile. If the user needs more, offer to run the skill twice (one dashboard per event.type) or switch to `process-journey`.
- **Process Journey:** ask *"Which `event.type`s are the steps of your process, in order?"* — ordered multi-select of exactly 5 (or 3–5, see the "Fewer than 5 steps" note in §Token inventory). Also ask *"What short label should each step use? (e.g. `Claim registered`, `Document submission`)"* — one text input per step, with the event.type name as the default.

Also collect:

- **Process / dashboard label** (free text) — becomes both the `[CLIENT_NAME]` substitution (simple-kpi) and the `<slug>` for the output filename (kebab-cased, lowercased, ASCII-only).
- **(Optional) Business currency** — three-letter code for the currency `unitsOverrides` swap.

### Persisting the scope in the emitted file

Do not add comments inside the JSON — the Dashboards app importer rejects them. Instead, print a short chat summary after the file is written that records:

```
> Template: simple-kpi | process-journey
> Providers: [<p1>, <p2>, …]
> Event types (ordered): [<t1>, <t2>, …]
> Correlation ID (process-journey only): <field>
> Business measure: <field>
> Status field + success literal (simple-kpi only): <field>=<literal>
> Category dimension: <field>
> Owning service (simple-kpi only): <name> (<id>)
> Currency: <code>
> Output: <context-folder>/dashboards/<slug>-dashboard-<YYYY-MM-DD-HHMM>.json
```

Keep the same summary in the chat log after import so the user can re-run the skill and reproduce the substitution.

---

## 🧱 Step 4 — Build the JSON

1. **Load the chosen template file verbatim** from `skills/dt-business-dashboard-build/templates/<template>.json`. When the skill is aimgr-installed, the runtime path is typically `.github/skills/dt-business-dashboard-build/templates/<template>.json` (symlink into the aimgr cache). Try both paths in that order and use the first that resolves. Do not re-encode line endings, do not re-order keys.
2. **Apply every substitution from the §Token inventory** using string-level replacement (the templates deliberately embed placeholders inside DQL strings, tile titles, and markdown content). For process-journey, remember the anomaly-detector query duplication — substitute both the tile-level `query` and the analyzer's inner `query` in the same pass.
3. **Guard against unresolved placeholders (leak scan).** After all substitutions, fail loudly (do **not** write the file) if the resulting JSON still contains any of the following literals. Report the offending token to the user and stop.

   Common (both templates):
   - `[CLIENT_NAME]`
   - `your.order.event.type`
   - `SERVICE-XXXXXXXXXXXX`

   `simple-kpi` sentinels (any occurrence is a bug — these are unique double-underscore markers with no valid tenant use):
   - `__MEASURE_FIELD__`
   - `__STATUS_FIELD__`
   - `__SUCCESS_LITERAL__`
   - `__DIMENSION_FIELD__`
   - `__CURRENCY_CODE__`

   `process-journey` example-data leaks (fail only if the user's substitution differs from the value — e.g. skip `acme.insurance` if the tenant's provider genuinely is `acme.insurance`):
   - `acme.insurance`
   - `claim.id`, `claim.type`, `claim.registered`, `documents.submitted`, `internal.review`, `third.party.review`, `customer.call`, `warehouse.received`, `item.picked`, `order.packed`
   - `wkf10640` (source tenant host in drilldown URLs)

   For process-journey, also print a **soft warning** (do not block the write) if any drilldown URL still contains a base64 blob starting with `vu9U3hXa` — those are source-tenant IDs that will 404 in the destination tenant.
4. **Set the top-level `settings.defaultTimeframe`** to a sensible default based on the tenant's ingest lag:
   - Simple KPI: `now()-7d` → `now()` (matches template default).
   - Process Journey: `now()-24h` → `now()` (matches template default).
   Keep the template default unless the user explicitly asked for a different window.
5. **Validate** by parsing the substituted string back into an object. Use the workspace's persistent PowerShell terminal: `Get-Content -Raw <path> | ConvertFrom-Json` — non-zero exit or exception means the substitution corrupted the JSON. Do **not** write the file in that case; report the offending region and stop.
6. **Write the file** to `<context-folder>/dashboards/<slug>-dashboard-<YYYY-MM-DD-HHMM>.json`. UTC, minute-precision timestamp, never overwrite (bump the minute or append seconds if a collision occurs).

### PowerShell / file-write note

Dashboard JSON is UTF-8 without BOM. Use the workspace file-write tool (which writes UTF-8 without BOM by default). If you must use PowerShell, use `Set-Content -Encoding utf8NoBOM` — the Dynatrace importer rejects files with a BOM.

---

## 📥 Step 5 — Manual import instructions (surfaced in chat after write)

The skill's job ends when the JSON file is on disk. **Do not attempt to import the dashboard for the user** — importing is a mutating operation the user explicitly controls, and keeping it manual has three concrete benefits:

- The emitted file stays on disk as a **reference copy** the user can diff, edit, or re-import against a different tenant later.
- The user chooses **when** the dashboard lands (e.g. after a code review, after a demo, into a shared folder vs. a personal drafts folder).
- Nothing in the tenant changes without an explicit UI action, which matches the read-only posture of the recommended `*-readonly` `dtctl` context.

Even if a future Dashboards write API becomes available via `dtctl`, this skill must continue to write-and-hand-off — do **not** add an auto-import branch.

Print this checklist to chat once the file is written:

1. The reference copy has been written to `<context-folder>/dashboards/<slug>-dashboard-<YYYY-MM-DD-HHMM>.json`. Keep it in version control if you want an audit trail of dashboard revisions.
2. Open the Dynatrace **Dashboards** app on the target tenant when you're ready to import.
3. Click **⋯ → Upload JSON** (or **New dashboard → Import**, depending on Dashboards app version).
4. Pick the file from step 1.
5. Rename the dashboard in the UI if the auto-generated name is too long — the file name is only a filesystem convention, not a display name.
6. Share with the appropriate group. Do **not** share dashboards containing hard-coded service IDs across tenants — regenerate the JSON per-tenant instead.

If the tenant's Dashboards app is on a version < 20 (rare), the import may reject the file with a schema mismatch. In that case bump `"version"` down by one, re-import, and file an issue against `dt-playbook-skills`.

---

## 🔍 What to look for while building

Map decisions to the emitted file using this checklist:

### Quality gates before writing
- ✅ Every placeholder from the §Token inventory has been replaced with a real value discovered in §4–§8.
- ✅ The chosen business-measure field is genuinely numeric (§5 sanity check passed).
- ✅ For simple-kpi: `dt.entity.service` filter is a real service ID (or has been rewritten to a `service.name matches` filter with the user's blessing).
- ✅ For process-journey: correlation-ID field is populated across ≥ 2 of the chosen event.types (`dt-business-process-review` §8-style check).
- ✅ JSON round-trip parse succeeds.

### Common failure modes
- 🚨 **Placeholder leaked into the emitted JSON** — always run the substitution scan in §Step 4.3 before writing.
- 🚨 **Currency unit not swapped** — Revenue tile still renders `£` in a USD tenant. Ask the user for the currency; don't guess from geography.
- 🚨 **`makeTimeseries` bin count too low for the chosen window** — the templates use `bins:50` which is fine for 24 h / 7 d; if the user picks a much longer window, drop to `interval:1d` to avoid empty leading bins.
- 🚨 **Donut / categorical-bar dimension has > 50 distinct values** — the tile will render an illegible legend. Fall back to a top-N `summarize … | sort … | limit 10 | fieldsAdd Other = …` pattern or pick a lower-cardinality dimension.
- 🔒 **PII in the Recent Events table** — the simple-kpi template's tile 9 lists raw event fields including `__MEASURE_FIELD__` and `__STATUS_FIELD__`. If the sampled event carries `email`, `phone`, `payment.card.*`, or `customer.address.*`, remove that field from the `fields` list before writing.

### Opportunities to flag to the user
- 💡 If the correlation-ID field also appears in other bizevent streams (e.g. `session.id` shows up in RUM), suggest a follow-up dashboard cross-referencing frontend health with the business flow.
- 💡 If the Davis / Security / Carbon health tiles are still using their hard-coded fallbacks, offer to open a `@dt-business-process-review` follow-up to wire them into real problem / vulnerability / carbon queries.

---

## 🧪 Post-build verification (in chat)

Do not run additional Grail queries after the file is written. Print a 3-line verification hint the user can paste into a Notebook to sanity-check each headline tile:

```dql
// Verify the Revenue tile
fetch bizevents, from:now()-7d
| filter event.provider == "<provider>" and event.type == "<eventType>"
| summarize totalRevenue = sum(<measureField>)
```

```dql
// Verify the Success Rate tile (simple-kpi only)
fetch bizevents, from:now()-7d
| filter event.provider == "<provider>" and event.type == "<eventType>"
| summarize success = countIf(<statusField> == "<successLiteral>"), total = count()
| fieldsAdd successRate = (toDouble(success) / toDouble(total)) * 100
```

```dql
// Verify the correlation ID (process-journey only)
fetch bizevents, from:now()-24h
| filter in(event.provider, {<providers>}) and in(event.type, {<eventTypes>})
| filter isNotNull(<corrIdField>)
| summarize stepsTouched = countDistinct(event.type), by:{corrId = <corrIdField>}
| summarize goodCorrelations = countIf(stepsTouched >= 2), totalCorrelations = count()
```

---

## ✅ Final Self-Review Checklist

- [ ] Template file loaded verbatim from `skills/dt-business-dashboard-build/templates/`.
- [ ] Every entry in the §Token inventory has been replaced with a tenant-specific value.
- [ ] Placeholder scan (§Step 4.3) passed — no `__…__` sentinels, no `[CLIENT_NAME]`, `SERVICE-XXXXXXXXXXXX`, `your.order.event.type`, and (unless intentional) no `acme.insurance` / `claim.id` / `wkf10640` / source-tenant base64 blob left in the file.
- [ ] JSON round-trip parse succeeded.
- [ ] File written to `<context-folder>/dashboards/<slug>-dashboard-<YYYY-MM-DD-HHMM>.json` (UTC, no overwrite).
- [ ] Chat summary of the substitution decisions printed.
- [ ] Import checklist printed.
- [ ] Post-build verification DQL snippets printed.
- [ ] No PII (real customer IDs, real CC numbers, real emails) shipped inside the JSON — the substitutions come from field **names**, not sampled values.

---

## 🔗 Related References

- `dt-playbook-common` skill — shared scaffolding (Step 0 interview, mappings file, style rules).
- `dt-bizevents-review` skill — run first if you don't yet know the tenant's providers / event.types.
- `dt-business-process-review` skill — run first (or after) for a rigorous funnel / correlation analysis that this dashboard can then visualize.
- `dt-app-dashboards` skill (from `dynatrace-for-ai`) — DQL + tile-schema reference for the Dashboards app.
- `dt-dql-essentials` skill — DQL syntax, pitfalls, operator reference.
- Dynatrace docs: *Dashboards app*, *Business Events*, *OpenPipeline*.
