# dt-playbook-skills

Re-runnable Dynatrace environment-review playbooks, packaged as
[`aimgr`](https://github.com/dynatrace-oss/ai-config-manager)-installable skills
and custom agents for GitHub Copilot.

Three playbooks are bundled:

| Playbook | Skill | Agent | Output subfolder |
| --- | --- | --- | --- |
| Bizevents / Business Observability overview | `dt-bizevents-review` | `@dt-bizevents-review` | `event-overview-reports/` |
| Logs overview | `dt-logs-review` | `@dt-logs-review` | `logs-overview-reports/` |
| Business Process detail | `dt-business-process-review` | `@dt-business-process-review` | `process-detail-reports/` |

All three depend on a shared scaffolding skill **`dt-playbook-common`** that
owns the Step 0 kickoff interview, intent confirmation, empty-tenant fast-path,
duplicate-snapshot logic, PowerShell quoting, shared report style, and the
self-improvement protocol.

## Prerequisites

| Requirement | Notes |
| --- | --- |
| [`dtctl`](https://github.com/dynatrace-oss/dtctl) installed and on `PATH` | `dtctl version` to verify. Install via Homebrew or `install.ps1` / `install.sh` — aimgr does **not** install the binary, only its operator skill. |
| [`aimgr`](https://github.com/dynatrace-oss/ai-config-manager) installed | `aimgr --version` to verify |
| Authenticated `dtctl` context | `dtctl auth login --context <name> --environment "https://<env>.apps.dynatrace.com"` |

> The `dtctl` operator skill and the Dynatrace domain skills (`dt-dql-essentials`,
> `dt-obs-logs`, etc.) are pulled in automatically by the manifest install
> below — no separate `dtctl skills install` / `npx skills add` step needed.

## Install (one-time per workspace) — manifest one-liner

This repo ships an [`ai.repo.yaml`](ai.repo.yaml) manifest that also registers
the [`dtctl`](https://github.com/dynatrace-oss/dtctl) operator skill and the
[`dynatrace-for-ai`](https://github.com/Dynatrace/dynatrace-for-ai) domain
skills (DQL, observability, platform, migration) as aimgr sources.

```powershell
# 1. Apply the manifest — registers all three repos as aimgr sources
aimgr repo apply-manifest https://raw.githubusercontent.com/<owner>/dt-playbook-skills/main/ai.repo.yaml

# 2. Target GitHub Copilot
aimgr config set install.targets copilot

# 3. Install everything in one shot (playbook skills + agents + dtctl + dynatrace-for-ai)
cd <your-workspace>
aimgr install "skill/*" "agent/dt-*"
```

After install, `.github/skills/` and `.github/agents/` are populated with
symlinks back to the aimgr repo cache.

### Alternative: this repo only

If you don't want the `dtctl` and `dynatrace-for-ai` skills pulled in (e.g. you
already manage them separately), skip the manifest and add only this repo:

```powershell
aimgr repo add gh:<owner>/dt-playbook-skills --ref v1.0.0
aimgr config set install.targets copilot
aimgr install "skill/dt-*" "agent/dt-*"
```

## Run a playbook

In any Copilot chat inside the workspace:

```
@dt-bizevents-review
@dt-logs-review
@dt-business-process-review
```

The agent reads `dt-playbook-common` first (Step 0 interview → context/folder
confirmation → final intent check), then runs the paired playbook skill end-to-end
and writes the report to
`<context-folder>/<subfolder>/<filename-stem>-<YYYY-MM-DD-HHMM>.md`.

Natural-language invocation also works because each skill's `description` field
carries trigger keywords:

> "Run a bizevents overview against my prod context."
> "Logs overview report for the demo tenant."
> "Business process detail on the checkout funnel."

## Workspace state file

Persistent context↔folder mappings live in
`<workspace-root>/.dt-playbook-mappings.yaml`, created on first run. Do not
edit by hand while a playbook is running. See `dt-playbook-common`'s
*Workspace mapping file* section for the schema.

## Versioning

Releases follow semver:

- **patch** — typo / clarification fixes.
- **minor** — additive (new query, new report section, new anomaly check).
- **major** — breaking report-shape change, removed query, renamed skill/agent.

Pin a `--ref` tag in `aimgr repo add` if you want predictable upgrades:

```powershell
aimgr repo add gh:<owner>/dt-playbook-skills --ref v1.0.0
```

## Contributing improvements

When a playbook needs a new query, anomaly check, or report section, file a PR
against this repo. The skills themselves are symlink-installed and read-only
from any consuming workspace — do not edit `.github/skills/*/SKILL.md` in
place.
