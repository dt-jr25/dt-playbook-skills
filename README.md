# dt-playbook-skills

Re-runnable Dynatrace environment-review playbooks, packaged as
[`aimgr`](https://github.com/dynatrace-oss/ai-config-manager)-installable skills
and custom agents for GitHub Copilot.

## Contents

| Playbook | Agent | Output location |
| --- | --- | --- |
| Bizevents / Business Observability overview | `@dt-bizevents-review` | `OUTPUT/<context>/reports/` |
| Logs overview | `@dt-logs-review` | `OUTPUT/<context>/reports/` |
| Business Process detail | `@dt-business-process-review` | `OUTPUT/<context>/reports/` |
| Business Observability dashboard builder | `@dt-business-dashboard-build` | `OUTPUT/<context>/dashboards/` |
| **Umbrella / flexible entry point** (routes to one of the above) | `@dt-business-observability` | *(delegates — inherits the target skill's location)* |

The four named agents are 1:1 with a playbook skill and own explicit `@name`
invocation. The **umbrella agent** (`@dt-business-observability`) is the
flexible entry point: it reads the **`dt-playbook-router`** skill to resolve an
exploratory or cross-skill request to exactly one of the four playbooks and
runs it verbatim. New team skills are reachable through the umbrella agent
without needing their own agent — see
[Contributing](#contributing-a-new-team-skill).

All agents depend on the shared **`dt-playbook-common`** skill (Step 0 kickoff
interview, context/folder confirmation, empty-tenant fast-path,
duplicate-snapshot logic, PowerShell quoting rules, shared report style,
self-improvement protocol, and the **Shared agent boilerplate** every agent
references for its Step-0 opener and hard-rules core).

## Prerequisites

| Requirement | Verify with |
| --- | --- |
| [`dtctl`](https://github.com/dynatrace-oss/dtctl) on `PATH`, authenticated to at least one context | `dtctl version` · `dtctl ctx current` |
| [`aimgr`](https://github.com/dynatrace-oss/ai-config-manager) installed | `aimgr --version` |

`aimgr` does **not** install the `dtctl` binary — install it separately per its
README. `aimgr` **does** clone the `dtctl` operator skill and all
`dynatrace-for-ai` domain skills (DQL, observability, platform, migration) into
its own cache when you run the manifest install below — so if you're using this
`aimgr` workflow you can skip the alternative install paths documented in the
`dynatrace-for-ai` and `dtctl` READMEs (`npx skills add …`, `claude plugin
marketplace add …`, `dtctl skills install`). Those are equivalent, mutually
exclusive alternatives — pick one, not several.

## Install (one-time per workspace)

This repo ships an [`ai.repo.yaml`](ai.repo.yaml) that also registers the
[`dtctl`](https://github.com/dynatrace-oss/dtctl) operator skill and the
[`dynatrace-for-ai`](https://github.com/Dynatrace/dynatrace-for-ai) domain
skills as `aimgr` sources.

```powershell
# 1. Apply the manifest — registers all three source repos
aimgr repo apply-manifest https://raw.githubusercontent.com/dt-jr25/dt-playbook-skills/main/ai.repo.yaml

# 2. Target GitHub Copilot
aimgr config set install.targets copilot

# 3. Install into your workspace
cd <your-workspace>
aimgr install "skill/*" "agent/dt-*"
```

After install, `.github/skills/` and `.github/agents/` are populated with
symlinks back to the aimgr repo cache.

For predictable upgrades, pin an immutable ref in your local
`ai.repo.yaml` (edit and re-run `aimgr repo sync`) or, if you only want this
repo and none of the transitive sources, add it directly with a pinned ref:

```powershell
aimgr repo add gh:dt-jr25/dt-playbook-skills --ref v1.0.0
```

## Run a playbook

In any Copilot chat inside the workspace, invoke the agent by name:

```
@dt-bizevents-review
@dt-logs-review
@dt-business-process-review
@dt-business-dashboard-build
```

**Not sure which one?** Use the umbrella agent — it routes your request to the
right playbook (and asks you to pick if two plausibly fit):

```
@dt-business-observability
```

Rule of thumb: **know the exact playbook → name it directly** (`@dt-<name>`);
**exploratory / cross-skill / "help me review this tenant" → use
`@dt-business-observability`**. The umbrella never blends two playbooks — it
resolves to exactly one per run via the `dt-playbook-router` skill.

Natural-language invocation also works — each skill's `description` carries
trigger keywords:

> "Run a bizevents overview against my prod context."
> "Build a Business Observability dashboard for the checkout funnel."

Every agent reads `dt-playbook-common` first (Step 0 → context/folder
confirmation → intent check), then runs the paired skill end-to-end and writes
the deliverable to
`OUTPUT/<context-folder>/<kind>/<stem>-<YYYY-MM-DD-HHMM>.<ext>`
(`<kind>` = `reports/` for reviews `.md`, `dashboards/` for dashboards `.json`).

### Where output goes

All generated artifacts live under a single top-level **`OUTPUT/`** directory,
one subfolder per `dtctl` context, and within each context exactly two artifact
folders:

```
OUTPUT/
└── <context-folder>/          # = dtctl context name, e.g. acme-prod-readonly/
    ├── reports/               # every .md review (bizevents, logs, process) + .handoff.yaml sidecars
    └── dashboards/            # every dashboard .json
```

The review skills all share one `reports/` folder — filenames stay unique per
playbook (`bizevents-overview-…`, `logs-overview-…`, `business-process-detail-…`),
so finding a report is "`OUTPUT/` → tenant → `reports/`". `OUTPUT/` is
gitignored. The `dt-playbook-common` skill prepends `OUTPUT/` automatically; the
`.dt-playbook-mappings.yaml` `folder` field stores only the context-relative
folder name.

### Chaining process review → dashboard build

At the end of a `@dt-business-process-review` run the agent offers a one-click
hand-off to `@dt-business-dashboard-build`. A `.handoff.yaml` sidecar carries
the resolved scope, correlation ID, business measure + dimension, PII
exclusions, and step order forward; the dashboard agent's hand-off mode skips
every Discovery Query the review already answered — roughly halving Grail scan
cost on paired runs.

Manual hand-off later (after reading the report):

```
@dt-business-dashboard-build --from-report OUTPUT/<context-folder>/reports/business-process-detail-<YYYY-MM-DD-HHMM>.md
```

## Common commands (tips & tricks)

### `aimgr` — sources, install, and overrides

| Action | Command |
| --- | --- |
| Show current repo state (sources, resource counts, active overrides) | `aimgr repo info` |
| Print the shareable local manifest | `aimgr repo show-manifest` |
| List resources known to the aimgr repo | `aimgr repo list` |
| Re-import from all configured sources | `aimgr repo sync` |
| Preview a sync without touching anything | `aimgr repo sync --dry-run` |
| Reconcile stale source-owned resources after include/subpath changes | `aimgr repo sync --prune` |
| Full soft-reset: clear imported state then re-import from `ai.repo.yaml` | `aimgr repo rebuild` |
| Temporarily redirect a normally-remote source to a local checkout | `aimgr repo override-source dt-playbook-skills local:C:\path\to\dt-playbook-skills` |
| Clear an active override (restores the original remote definition + auto-syncs) | `aimgr repo override-source dt-playbook-skills --clear` |
| Re-apply the shared manifest (picks up ref/URL changes) | `aimgr repo apply-manifest <manifest-url>` |
| Remove a source and its imported resources | `aimgr repo remove <name>` |
| Remove a source but keep the files | `aimgr repo remove <name> --keep-resources` |
| Install everything declared in the current workspace's `ai.package.yaml` | `aimgr install` |
| Install a specific resource or pattern | `aimgr install skill/dt-bizevents-review` · `aimgr install "skill/dt-*"` |
| List what's installed in the current workspace | `aimgr list` |
| Reconcile the workspace with `ai.package.yaml` (fix missing/orphaned installs) | `aimgr repair` |
| Uninstall from the current workspace | `aimgr uninstall skill/<name>` |

> **Override gotcha #1 — clearing:** `--clear` is mutually exclusive with the
> `local:/path` argument. Run `aimgr repo override-source <name> --clear` on
> its own, no path. The error message says so verbatim.
>
> **Override gotcha #2 — `apply-manifest` doesn't clear overrides:** an active
> override always wins over the manifest baseline, so `apply-manifest` can
> silently mask a stale override and later `repo sync` may reject with a
> resource-name collision. Always `--clear` first, then `apply-manifest`.
>
> **Override gotcha #3 — local-only sources can't be overridden:**
> `override-source` is only for switching a *remote-backed* source to a local
> checkout. If a source is already declared as `local:` in `ai.repo.yaml`,
> edit the manifest directly (or `repo remove` + `repo add`) instead.

### `dtctl` — contexts and query smoke-tests

| Action | Command |
| --- | --- |
| List every context (starred = current) | `dtctl ctx` |
| Show which context is active | `dtctl ctx current` |
| Switch to another context | `dtctl ctx <name>` |
| Create a new context and log in via browser OAuth | `dtctl auth login --context <name> --environment "https://<env>.apps.dynatrace.com"` |
| Re-authenticate the current context after token expiry | `dtctl auth login` |
| Show OAuth session status | `dtctl auth status` |
| Health-check config, connectivity, and auth | `dtctl doctor` |
| Ad-hoc DQL from a here-string (avoids shell quoting) | ``@'...DQL...'@ \| dtctl query -f - -o json`` |
| Run a DQL query from a file | `dtctl query -f query.dql -o json` |
| Dry-run to cap Grail cost | append `\| limit 1` and use a small window (e.g. `from:-1h`) |

> **PowerShell quoting:** always use `@'...'@` (single-quoted here-string) when
> piping DQL to `dtctl query -f -`. Double-quoted here-strings (`@"..."@`)
> attempt to expand `$` variables in your DQL — that breaks dashboard-variable
> references like `$Country` or `$CardType` and any string starting with `$`.

## Workspace state file

Persistent context↔folder mappings live in
`<workspace-root>/.dt-playbook-mappings.yaml`, created on first run. Do not
edit by hand while a playbook is running — see `dt-playbook-common`'s
*Workspace mapping file* section for the schema.

## Versioning

Releases follow semver:

- **patch** — typo / clarification fixes.
- **minor** — additive (new query, new report section, new anomaly check, a
  new team skill + its `dt-playbook-router` dispatch-table row).
- **major** — breaking report-shape change, removed query, renamed skill/agent.

Adding a new team skill bumps `dt-playbook-router` (and, if you also add a
dedicated agent, that agent) by **minor** — each playbook still carries its own
independent semver.

## Contributing a new team skill

Adding a playbook or capability to this repo is intentionally cheap. The
committed source of truth is only four things — `skills/dt-*/SKILL.md`,
`agents/dt-*.md`, `ai.repo.yaml`, and this `README.md`. Everything under
`.github/`, `.claude/`, and `ai.package.yaml` is **gitignored aimgr runtime
state** — never hand-edit it, and never add your new skill to a resource list;
`ai.repo.yaml` auto-discovers the `skills/` and `agents/` directories.

### Required: the skill

Create `skills/dt-<name>/SKILL.md` with YAML frontmatter and a body:

```markdown
---
name: dt-<name>
description: >-
  One paragraph that doubles as the natural-language trigger surface. State
  what the skill produces, the exact phrases that should invoke it (e.g.
  "run an X review", "@dt-<name>"), that it reads `dt-playbook-common` FIRST
  for Step 0, where it writes output, and a "Do NOT use for …" line pointing
  at sibling skills so routing stays clean.
---

# 📓 <Title> — Playbook

## Parameters (consumed by dt-playbook-common)
| Parameter | Value |
| --- | --- |
| `<kind>`          | `reports/` (or `dashboards/`) |
| `<filename-stem>` | `<name>-overview` |
| `<records>`       | bizevents \| logs \| … |

## Discovery Query Set        # §1…§N — your DQL, tight time windows + limits
## What to Look For           # the checklist that drives findings
## Output Document Structure  # the report skeleton
```

Follow the conventions the existing skills already establish:

- **Read `dt-playbook-common` first.** Your skill inherits Step 0, the mapping
  file, the empty-tenant fast-path, duplicate-snapshot logic, PowerShell
  quoting rules, the self-improvement protocol, and the **Shared agent
  boilerplate**. Only own what is data-source-specific (queries, checklist,
  report shape).
- **Cost guardrails.** Every Discovery Query uses tight `from:` windows and a
  `limit` (see `dt-playbook-common` §Prerequisites) — Grail tables are large.
- **Declare prerequisites.** List the domain skills your queries need (`dtctl`
  operator skill and `dt-dql-essentials` at minimum; add data-source skills
  like `dt-obs-logs` or `dt-app-dashboards` as needed). The agent detects and
  reports a missing skill — it never installs one.

### Required: one row in the router

Add a row to `skills/dt-playbook-router/SKILL.md`'s **Dispatch table** (target
skill · intent · output folder — `reports/` or `dashboards/`) and bump that
skill's version (minor). This
is what makes your new skill reachable through `@dt-business-observability`.

### Optional: a dedicated agent

Only add `agents/dt-<name>.md` if the playbook deserves a first-class `@name`
entry point (a common, frequently-invoked workflow). Keep it thin — reference
the **Shared agent boilerplate** from `dt-playbook-common` and add only your
skill-specific steps, parameter table, required-domain-skill list, and any
hard rules unique to the playbook. A minimal shim:

```markdown
---
description: Runs the <…> playbook end-to-end. Invoke with `@dt-<name>` when …
---
# dt-<name> agent

## How to run
1. **Apply the Shared agent boilerplate from `dt-playbook-common`** (§Shared agent boilerplate) — run its Step 0 opener first and observe its shared hard rules.
2. **Read the `dt-<name>` skill next.** Follow its Discovery Query Set, checklist, and Output Document Structure verbatim. Honour its parameter table.
3. … skill-specific steps …

## Hard rules
Apply the **Shared agent boilerplate** hard rules from `dt-playbook-common` in full, plus:
- … only rules unique to this playbook …
```

If you skip the dedicated agent, the skill is still fully usable via
`@dt-business-observability` and (where the host supports it) its own
description-based auto-trigger — this is how the agent layer stays flat as the
skill count grows.

### Checklist

- [ ] `skills/dt-<name>/SKILL.md` created (frontmatter + parameters + queries + report shape).
- [ ] Row added to the `dt-playbook-router` dispatch table (+ minor version bump).
- [ ] Contents table row added in this README.
- [ ] Optional `agents/dt-<name>.md` shim added **only** if a first-class `@name` is warranted.
- [ ] No edit to `ai.package.yaml` / `.github/` / `.claude/` (gitignored, auto-discovered).
- [ ] Version bumped per [Versioning](#versioning) (minor for an additive skill).

## Contributing improvements

File a PR against this repo when a playbook needs a new query, anomaly check,
or report section. Installed skills are symlinked and read-only from consuming
workspaces — do **not** edit `.github/skills/*/SKILL.md` in place.
