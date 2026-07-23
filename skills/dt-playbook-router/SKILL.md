---
name: dt-playbook-router
description: >-
  Dispatch layer for the Dynatrace Business Observability team playbooks. Given
  a flexible, exploratory, or "not sure which report I need" request, resolves
  it to exactly ONE team-specific playbook skill (dt-bizevents-review,
  dt-logs-review, dt-business-process-review, dt-business-dashboard-build, or
  any future dt-* team skill) and then hands off to that skill to run verbatim.
  Read by the `@dt-business-observability` umbrella agent AFTER
  `dt-playbook-common` Step 0. This skill owns ONLY routing + disambiguation —
  it holds no Discovery Queries, report skeletons, or hard rules of its own;
  those stay owned by the target skill. Do NOT use this skill to explain an
  existing query, and do NOT invoke it from a specific `@dt-<name>` agent (those
  already know their target skill).
---

# 🧭 Playbook Router — Dispatch Layer

> **Purpose:** Turn a broad / ambiguous / cross-skill request into a run of
> exactly one team playbook skill. This skill is the brain of the
> `@dt-business-observability` umbrella agent. It does **not** contain any
> playbook logic — once it has picked a target skill, that skill owns the run
> end-to-end exactly as if the user had invoked the skill's dedicated
> `@dt-<name>` agent.

> 🔒 **Read-only skill.** Symlink-installed by `aimgr`. Never edit at runtime.

---

## 🧩 When this skill runs

The `@dt-business-observability` agent reads this skill **after**
`dt-playbook-common` Step 0 has completed with an explicit "Proceed" from the
user. By the time you are here:

- The active `dtctl` context and output folder are already confirmed.
- The user's intent is known (their original prompt).
- No Discovery Query has run yet.

Your single job: **select the one target skill** that matches the user's
intent, then read it and run it verbatim.

---

## 🗺️ Dispatch table

Match the user's intent against the **Intent** column. Keep this table in sync
with each skill's own frontmatter `description` — the description is the source
of truth for that skill's trigger surface; this table is a routing summary.

| Target skill | Route here when the user wants… | Output folder |
| --- | --- | --- |
| `dt-bizevents-review` | A tenant-wide **bizevents / business-observability overview** — what business events exist, volume, providers, event types, data dictionary, correlation IDs, anomalies, metric opportunities. Breadth-first inventory. | `OUTPUT/<context>/reports/` |
| `dt-logs-review` | A tenant-wide **logs overview** — volume, top emitters, severity mix, ingest sources, data dictionary, top error signatures, PII checks, parse-rule / metric opportunities. | `OUTPUT/<context>/reports/` |
| `dt-business-process-review` | A **deep-dive on one business process** — given providers + event types, characterize the happy path, branches, correlation IDs, funnel attrition, end-to-end duration, and recommend process/event metrics. Depth-first. | `OUTPUT/<context>/reports/` |
| `dt-business-dashboard-build` | To **build / scaffold / generate a dashboard** — a Business Observability, business-KPI, business-process, or bizevents dashboard JSON for the Dashboards app. | `OUTPUT/<context>/dashboards/` |

> ➕ **Adding a new team skill?** Add one row here (target skill · intent ·
> output folder — `reports/` for `.md` playbooks, `dashboards/` for JSON
> builders) and bump this skill's version (minor). No agent file is required
> for the new skill to be reachable through the umbrella agent — see the repo
> README's *Contributing* section.

---

## 🔀 Disambiguation contract

Resolve the user's intent to a target using exactly these three cases. This is
mandatory behavior, not a suggestion.

### Case 0 — no match
The request doesn't correspond to any row in the dispatch table (e.g. it's a
pure DQL question, an infra/host question, or unrelated to the team
playbooks).
→ **Do not run any playbook.** Tell the user plainly that no team playbook
covers the request, list the available playbooks (the Intent column above), and
ask whether they want one of them or a different kind of help. Stop.

### Case 1 — exactly one match
The request clearly maps to one row.
→ Announce the pick in one short line (e.g. "This maps to the **bizevents
overview** playbook — running it now."), then **read that skill and execute it
verbatim**, following its own Discovery Query Set, focus interview, and Output
Document Structure. From this point the target skill owns the run completely —
run it exactly as its dedicated `@dt-<name>` agent would. Do not inject any
guidance of your own.

### Case 2 — two or more plausible matches
The request could reasonably map to more than one row (e.g. "review our
business events and show me the checkout funnel" → bizevents-review **and**
business-process-review; or "give me a dashboard of our logs" → ambiguous
between a logs review and a dashboard build).
→ **Do not guess and do not blend.** Ask the user which single playbook they
want, via one multiple-choice question listing only the plausible targets (plus
their one-line intents). Run exactly one per invocation. If the user genuinely
wants two (e.g. a review *then* a dashboard), run the first to completion; the
review skills that support hand-off (e.g. `dt-business-process-review` →
`dt-business-dashboard-build`) will offer their own chained follow-up.

---

## 🚫 Router hard rules

- **One skill per run.** Never load or interleave the Discovery Queries, report
  skeletons, or hard rules of two target skills in the same run.
- **Never guess on ambiguity.** If Case 2 applies, ask — do not silently pick.
- **Own nothing.** This skill adds no queries, no report content, and no
  guardrails of its own. Every query, guardrail, and report shape comes from
  the target skill (and `dt-playbook-common`, which the umbrella agent already
  read at Step 0).
- **Respect existing hand-offs.** Do not re-implement the
  `dt-business-process-review` → `dt-business-dashboard-build` `.handoff.yaml`
  hand-off — let the target skill run it if the user wants a chained dashboard.
- **Prereqs belong to the target skill.** The domain-skill prerequisites
  (`dtctl` operator skill, `dt-dql-essentials`, and any data-source skill like
  `dt-obs-logs` or `dt-app-dashboards`) are verified by the target skill at its
  own Step 1 per `dt-playbook-common` §Prerequisites — the router does not
  check or install them.
