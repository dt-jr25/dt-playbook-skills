---
name: dt-playbook-common
description: Shared scaffolding for every Dynatrace environment-review playbook (dt-bizevents-review, dt-logs-review, dt-business-process-review). Owns the Step 0 kickoff interview, dtctl context/folder confirmation, intent confirmation, the per-workspace mapping file (.dt-playbook-mappings.yaml), empty-tenant fast-path, duplicate-snapshot logic, PowerShell quoting rules, shared report style, and the self-improvement protocol. Read this skill FIRST whenever any dt-*-review skill or agent is invoked — it sets up the run so the per-data-source skill can focus on its Discovery Query Set and report shape.
---

# 🧰 Playbook Common Scaffolding

> **Purpose:** Shared workflow that every data-source review skill in this repo inherits. The per-data-source skills (`dt-bizevents-review`, `dt-logs-review`, `dt-business-process-review`) reference this skill for the Step 0 interview, intent confirmation, mapping persistence, empty-tenant fast-path conventions, duplicate-snapshot logic, PowerShell quoting note, and self-improvement protocol. They only own the data-source-specific Discovery Query Set, What-to-Look-For checklist, report skeleton, and empty-tenant report skeleton.

> ⚠️ **Never overwrite a prior report.** Each run MUST be written to a new file with a UTC date+time suffix so historical snapshots stay intact and can be diffed.

> 🔒 **Read-only skill.** This skill is symlink-installed by `aimgr`. The agent MUST NOT edit this file at runtime. Persistent state (context↔folder mappings) lives in the per-workspace `.dt-playbook-mappings.yaml` (see below).

---

## 🧭 How to use this skill

The per-data-source skill supplies:

| Per-skill parameter | Example (`dt-bizevents-review`) | Example (`dt-logs-review`) |
| --- | --- | --- |
| `<subfolder>` | `event-overview-reports/` | `logs-overview-reports/` |
| `<filename-stem>` | `bizevents-overview` | `logs-overview` |
| Discovery Query 1 (headline) | bizevents Query 1 | logs Query 1 |
| Bucket-inventory query | bizevents Query 10 | logs Query 12 |
| Initial-overview query range | Queries 2–6 | Queries 2–7 |
| Step 2 focus question wording | provider / event-type picker | emitter / namespace / `log.source` picker |
| Report skeleton | bizevents §Output Document Structure | logs §Output Document Structure |

The full output path is always:

```
<context-folder>/<subfolder>/<filename-stem>-<YYYY-MM-DD-HHMM>.md
```

Everything else lives below.

---

## 🧰 Prerequisites (shared)

| Requirement | Notes |
| --- | --- |
| `dtctl` CLI installed and on PATH | `dtctl version` to verify. Install via Homebrew, `install.ps1`, or `install.sh` — aimgr does **not** install the binary. |
| Authenticated context for the target tenant | `dtctl auth whoami` to confirm; switch with `dtctl ctx use <name>` |
| **`dtctl` operator skill** (from the `dtctl` repo) | Non-optional. Teaches the agent the exact `dtctl` verbs, flags, `--agent` envelope, output shape, and PowerShell quoting caveats — so it doesn't shell out to `dtctl` from memory. Installed via this repo's `ai.repo.yaml` manifest (see the repo README). |
| **`dt-dql-essentials` skill** (from `dynatrace-for-ai`) | Non-optional. Every Discovery Query in every playbook is DQL. Installed via this repo's `ai.repo.yaml` manifest (see the repo README). |
| A target context folder in the workspace (e.g. `dt-playground-readonly/`, `<client>-sprint-readonly/`) | Folder name must match the `dtctl` context name. Reports live in `<context-folder>/<subfolder>/` per the skill. |

> 📚 **Read the `dtctl` operator skill AND `dt-dql-essentials` before Step 1 fires.** The host agent's routing model may not auto-load them from the catalog once it's already executing inside a `@dt-*-review` agent — name them explicitly in your read list. Skipping either is the single biggest cause of the agent shelling out with wrong `dtctl` flags or writing invalid DQL.

> 🚦 **How to handle a missing skill (applies to every skill named in any playbook's prereqs, not just the two above).** The expected user workflow is that skills are pre-installed via `aimgr repo apply-manifest` + `aimgr install "skill/*"` (see the repo README). The agent's job is to **detect** a missing skill and **report** it — never to install it. Concretely:
> 
> 1. Treat "installed" as "the skill's guidance is in your current context" — do not probe the filesystem, do not run `aimgr`, `dtctl skills`, `npx`, or any other package manager to check.
> 2. If a required skill is not in your context by the point in the flow where it's needed, print one chat message quoting the exact install commands from the repo README (`aimgr repo apply-manifest …` then `aimgr install …`) and halt.
> 3. Do **not** offer to run those commands yourself, even if the user asks. Installation is a user-side decision because it touches their agent-client config and their `aimgr` state.
> 4. Resume only after the user confirms they've installed the missing skill and re-invoked the agent.

> 💡 **Cost guardrails:** Every Discovery Query in any review skill uses tight `from:` windows (≤ 24 h for headline numbers, ≤ 1 h or 15 min for cardinality / samples / pattern scans) and `limit`. Do not omit them — Grail tables can be very large.

The per-data-source skill may add data-source-specific requirements (e.g. `dt-obs-logs` for the logs skill; `dt-app-dashboards` for the dashboard-build skill; extra token scopes for bizevents). Those are also non-optional — read them at the same point in the flow. The missing-skill procedure above applies uniformly.

---

## 🗂️ Workspace mapping file

Persistent `<dtctl context> → <context-folder>` pairings live in
`<workspace-root>/.dt-playbook-mappings.yaml`. This file is the agent's memory
of which output folder to use for a given tenant. It is workspace-local —
each workspace tracks its own mappings — and is shared across all `dt-*-review`
skills (the same context maps to the same folder regardless of which data
source is being reviewed; only the `<subfolder>` differs).

### File location

```
<workspace-root>/.dt-playbook-mappings.yaml
```

### Schema

```yaml
# Auto-maintained by dt-playbook-* skills. Do not edit while a playbook is
# running. Safe to commit to the workspace repo so the team shares mappings.
version: 1
mappings:

```

Field rules:

- **`context`** — exact `dtctl` context name. Primary key. One row per context.
- **`environment_url`** — apps URL of the target tenant. Sourced from `dtctl auth whoami` on creation; do not invent.
- **`folder`** — workspace-relative folder path (trailing slash). Defaults to `<context>/` for brand-new contexts.
- **`last_used_utc`** — ISO-8601 UTC timestamp of the most recent run that used this mapping.
- **`last_data_source`** — `bizevents` | `logs` | `business-process`. Helps with cross-skill audits.

### Agent behaviour

1. **At the start of Step 0**, the agent reads `.dt-playbook-mappings.yaml` from the workspace root. If the file does not exist, treat the mappings list as empty (create the file lazily at step 5).
2. **At step 5 (after the user confirms `<context> → <context-folder>`)**, the agent updates `.dt-playbook-mappings.yaml`:
   - If a row for the chosen `context` already exists, **overwrite** its `folder`, `last_used_utc`, and `last_data_source` fields in place.
   - If no row exists, **append** a new entry. Keep `mappings` alphabetized by `context`.
   - Create the file with `version: 1` and an empty `mappings: []` on the very first run, then add the row.
3. **Never** write secrets, tokens, or PII into this file. Only the four fields above.

---

## 🏃 How to Run a Review Report

The user kicks off a review by either `@`-mentioning the paired agent
(`@dt-bizevents-review`, `@dt-logs-review`, `@dt-business-process-review`) or
typing a natural-language trigger (e.g. "run a bizevents overview", "logs
overview report"). Copilot routes to the right skill based on either the agent
mention or the skill description match. The agent then fills in everything else
by interviewing the user. **No `dtctl query` calls run until Step 0 is complete.**

### Step 0 — Mandatory kickoff interview (run BEFORE any query)

1. **Read `.dt-playbook-mappings.yaml`** at the workspace root. That file is the agent's memory of which output folder pairs to which `dtctl` context. It is shared across all `dt-*-review` skills.
2. **Run `dtctl auth whoami`** to see the current context. Compare it to the mappings file.
3. **Ask the user two questions** using the `vscode_askQuestions` tool (or a plain prompt if that tool is unavailable). Include the current context and any matching mapping as recommended defaults so the user can accept with one click:
   - **"Which `dtctl` context should I use for this run?"** — present the following options. Always include the first three; the fourth is only offered if the current `dtctl auth whoami` context differs from a context known in the mappings file.
     - **✅ Use current context `<name>`** (recommended) — proceed with the context returned by `dtctl auth whoami`.
     - **🔄 Switch to a different existing context** — show the contexts known from the mappings file as sub-options. If picked, run `dtctl ctx use <name>` followed by `dtctl auth whoami` to confirm.
     - **➕ Create a new context (authenticate a new tenant)** → go to **Step 3a** below before continuing.
     - **🛑 Cancel** → stop the playbook, no queries executed.
   - **"Which workspace context folder should the report be written to?"** — recommend the folder paired to the chosen context in the mappings file. **If the chosen context has no row in the mappings file (i.e. it is a brand-new context, including one just created via Step 3a), the default folder name MUST equal the context name verbatim** (e.g. context `acme-prod-readonly` → folder `acme-prod-readonly/`). The user can override, but do not invent a different name from the tenant short-id. The agent always appends `<subfolder>/<filename-stem>-<YYYY-MM-DD-HHMM>.md` automatically using the per-data-source skill's parameters — **do not prompt for the subfolder or filename**, and do not overwrite an existing file.

#### Step 3a — Optional: create a new `dtctl` context

Only run this step if the user picked **"Create a new context"** in step 3. The goal is to authenticate a brand-new tenant in one shot so the playbook can target it immediately, without forcing the user to leave the chat to run `dtctl` manually.

1. **Collect the three required inputs** via `vscode_askQuestions` (one question with three fields, or three sequential questions if the tool only supports single-field input). Do not guess any of these — they must come from the user.
   - **Context name** — short, lowercase-with-dashes, ideally ending in a safety suffix (e.g. `acme-prod-readonly`, `acme-sprint-readonly`). Reject names that already appear in `dtctl ctx list` (offer to switch to the existing context instead, looping back to step 3).
   - **Environment URL** — the apps URL of the target tenant, e.g. `https://abc12345.apps.dynatrace.com` or `https://xyz98765.sprint.apps.dynatracelabs.com`. Must start with `https://`.
   - **Safety level** — present the four supported values as buttons with a strong recommendation for `readonly`:
     - **🔒 `readonly`** (recommended for review playbooks) — queries only; cannot mutate tenant state.
     - **✏️ `readwrite-mine`** — read + write to objects the authenticated user owns.
     - **✏️ `readwrite-all`** — read + write across the tenant.
     - **⚠️ `dangerously-unrestricted`** — full unrestricted access; only pick if the user explicitly requests it and acknowledges the risk.

2. **Confirm before running.** Echo the resolved command back to the user as a final yes/no check, since `dtctl auth login` triggers a browser-based OAuth flow that the user must complete interactively:

   > *"About to create + authenticate a new context with `dtctl auth login --context <name> --environment "<url>" --safety-level <level>`. This will open a browser tab for SSO. Proceed?"*

   Options:
   - **✅ Proceed — open the auth flow**
   - **🔄 Edit the inputs** → loop back to step 3a.1
   - **🛑 Cancel** → return to step 3 without creating the context.

3. **Run the command.** Use the workspace terminal tool. On Windows / PowerShell, the environment URL is quoted with double quotes (the example below works on PowerShell, bash, and zsh):

   ```powershell
   dtctl auth login --context <context-name> --environment "<env-url>" --safety-level <readonly|readwrite-mine|readwrite-all|dangerously-unrestricted>
   ```

   The command is interactive — the user completes the SSO flow in their browser. **Do not** answer any prompts on the user's behalf and **never** ask for or echo their password / SSO PIN / token via `vscode_askQuestions`. If the terminal pauses for input that is clearly a secret, tell the user to type it directly into the terminal.

4. **Verify the new context.** After the command exits successfully:

   ```powershell
   dtctl auth whoami
   ```

   Confirm the output shows the new context name and the expected environment URL. If `whoami` reports a different context or the URL doesn't match, surface the discrepancy and stop — do **not** silently proceed.

5. **Treat the new context as the chosen one.** Continue with step 3's second question (folder name), defaulting to the new context name. Then step 4 (folder creation), step 5 (mappings file update — this will be a brand-new entry), and step 6 (final intent confirmation) all run normally.

6. **If the user cancels or `dtctl auth login` fails.** Do not write a partial mapping entry. Return to step 3 and let the user pick a different option.

4. **Create both folders if they don't exist yet.** After the user confirms the context-folder name, check whether `<workspace-root>/<context-folder>/` and `<workspace-root>/<context-folder>/<subfolder>/` exist. If either is missing, create it before writing the report — use the workspace file tool (e.g. `create_directory`) or a write tool that auto-creates parent directories. Do not fail the run because the folder is new; new contexts are expected to produce new folders.
5. **Persist the answer.** After the user confirms both values, the agent MUST update `<workspace-root>/.dt-playbook-mappings.yaml` so future runs auto-recommend the same pairing. Follow the schema and rules in the *Workspace mapping file* section above. If the file does not exist, create it with `version: 1` and a single-entry `mappings:` list.
6. **🛑 Final intent confirmation (mandatory — fires every run, even when the mapping is already known).** Before any `dtctl query` call, summarize the resolved target back to the user and require explicit go-ahead. Use `vscode_askQuestions` with a single yes/no question titled something like *"Confirm target environment before querying"* and a message body that contains, on separate lines:

   - **Context:** `<dtctl context name>`
   - **Environment URL:** `<value from dtctl auth whoami>`
   - **Report destination:** `<context-folder>/<subfolder>/<filename-stem>-<YYYY-MM-DD-HHMM>.md`
   - **Windows to be queried:** last 24 h (initial), expanding to 7 d / 30 d / 90 d only via the empty-tenant fast path

   Options:
   - **✅ Proceed — run the playbook against this environment** (recommended)
   - **🔄 Pick a different `dtctl` context** → loop back to step 3
   - **🛑 Cancel** → stop the playbook, no queries executed

   This step is **never skipped**, even if the same context was confirmed earlier in the same conversation. Its only job is to prevent the agent from querying the wrong environment when the user assumed a different context was active. If the user picks "Cancel", the agent halts without running Step 1 and reports back with the context it would have queried.

> **Assumption rule (scope):** Once the user confirms `<context> → <context-folder>` in steps 3–5, treat the **mapping itself** as canonical for the rest of the conversation and for all future runs (do not re-ask which folder to use). The **step-6 intent confirmation is separate** and fires on every playbook invocation — it confirms *that* the user wants to query this environment *now*, not *which* folder it maps to.

#### Prior completed-report check (before running Query 1)

Before firing Query 1, list existing `<filename-stem>-*.md` files in the target report subfolder (`<context-folder>/<subfolder>/`). If the most recent file is **not** an empty-tenant report **and** was written **today (same UTC date)**, surface it and ask the user:

> *"A completed `<filename-stem>` report already exists from `<HH:MM UTC>` today (`<filename>`). Do you want to re-run the full playbook, or work from the existing report?"*

Present the options via `vscode_askQuestions`:
- **📄 Work from the existing report** (recommended) → skip Step 1 and all deep-dive queries; jump directly to whatever the per-skill's "use existing report" flow specifies. For `dt-business-process-review`, this means reading the scope, correlation ID, measures, dimensions, and PII flags from the existing report and proceeding straight to the *Follow-up hand-off — offer to build a dashboard* section. For `dt-bizevents-review` and `dt-logs-review`, present the existing report path in chat and ask what the user wants to do next.
- **🔄 Re-run the full playbook** → continue normally with Query 1.
- **🛑 Cancel** → stop.

Skip this check entirely if:
- No prior report file exists in the subfolder, or
- The most recent file was written on a **different UTC date** (re-running a fresh day's playbook is expected and should not be gated).

---

### Step 1 — Initial overview

Run the skill's **Discovery Query 1** (24 h headline numbers) first. Branch based on the result:

- **If `total > 0`:** run the rest of the skill's **initial-overview queries** at the same 24 h window (or 1 h where the skill says so for cardinality queries). Summarize the results inline to the user (a short bullet list is fine — no report yet). Proceed to Step 2.
- **If `total == 0`:** do **not** run the rest of the initial-overview queries yet. Trigger the **empty-tenant fast path** below.

#### Empty-tenant fast path

The goal is to confirm "no `<records>` are being captured" in 2–3 queries instead of 8+.

1. **Re-run Query 1 with `from:-7d`.** Catches tenants where ingest is bursty/paused but recent.
2. **Run the skill's bucket-inventory query** to confirm the relevant Grail table is provisioned. This distinguishes "no producers" from "no bucket".
3. **Branch on the 7 d result:**
   - **If `total > 0` over 7 d:** the tenant has historical data but is currently silent. Proceed with Step 1 (initial-overview queries) using `from:-7d` (or whatever window contains the latest data), and flag the ingest gap as the top anomaly.
   - **If `total == 0` over 7 d:** ask the user **one** question before going further:

     > *"No `<records>` found in the last 7 days. Should I assume this tenant has no `<records>` ingest configured and write a short empty-tenant report, or do you want me to keep digging (30 d / 90 d windows, etc.)?"*

     Present the choices as buttons via `vscode_askQuestions`:
     - **Assume none configured (recommended)** → write the short empty-tenant report per the skill's *Empty-tenant report structure*, then stop. Skip Steps 2 and 3.
     - **Keep digging** → run Query 1 at `from:-30d` and then `from:-90d`. If still zero, proceed to the empty-tenant report anyway but note the windows you checked. Do not run downstream queries against an empty stream.

> 🔁 **Rule of thumb:** never run downstream queries against a window that returned zero in Query 1. Every query past Query 1 assumes there is at least one record to characterize.

#### Duplicate-snapshot check (before writing an empty-tenant report)

Before writing the empty-tenant report, list existing `<filename-stem>-*.md` files in the target report subfolder (`<context-folder>/<subfolder>/`). If the most recent one is **also** an empty-tenant report (zero-row verdict) **and** less than 24 h old, ask the user:

> *"An empty-tenant snapshot already exists from `<HH:MM UTC>` today with the same verdict. Should I skip writing a duplicate, append a one-line note to that file, or write a fresh timestamped report anyway?"*

Present the options via `vscode_askQuestions`:
- **Skip (recommended)** → do not write a new file. Confirm in chat with one sentence (e.g. "Verdict unchanged from `<HH:MM UTC>` snapshot — no new report written.").
- **Append a note** → add a one-line entry to the existing report's `## 📊 Headline Numbers` section recording the re-check time and verdict.
- **Write a fresh report** → proceed with the normal empty-tenant report.

Only auto-write a fresh report (no prompt) when the ingest verdict has **changed** since the previous snapshot (zero → non-zero, or non-zero → zero).

### Step 2 — Focus interview (run AFTER the overview, BEFORE deep dives)

Once the user has seen the headline numbers, ask the **skill-specific focus question** (see the per-data-source skill for exact wording and which dimensions to offer as picker options). The user's answer narrows downstream sampling.

Regardless of the data source, the agent must:
- Still produce the full document skeleton, but explicitly note in the report's header that the deep-dive scope was narrowed ("Focus: …").
- Keep the headline / volume / global tables global so the report still gives a complete picture of ingest health.
- Fall back to the skill's default coverage if the user leaves the question blank.

### Step 3 — Deep dive + write report

Proceed with the skill's deep-dive queries and produce the deliverable per its Output Document Structure.

### PowerShell quoting note (Windows)

Single-quoted DQL strings containing `"..."` break PowerShell parsing. Use **stdin via here-string**:

```powershell
@'
fetch logs, from:-15m | filter status == "ERROR" | limit 1
'@ | dtctl query -f - -o json
```

On bash/zsh, plain single-quoted argument works:

```bash
dtctl query 'fetch logs, from:-15m | filter status == "ERROR" | limit 1' -o json
```

---

## 🧱 Shared style rules for every report

The per-data-source skill owns the section skeleton, but every report produced by any `dt-*-review` skill must follow:

- **Tables over prose** for any list of facts.
- One leading emoji per heading as a scan marker.
- For each anomaly: one line of **Impact**, one line of **Fix**.
- Include 2–3 ready-to-paste DQL snippets at the end so the reader can verify and re-investigate.
- Avoid timing claims ("takes 5 min"). State volumes + windows precisely.
- **Never paste raw PII / secrets / customer identifiers** into the report — paraphrase or redact (`a***@example.com`, `41XX-XXXX-XXXX-1234`).
- Filename uses **UTC** minute-precision (`-<YYYY-MM-DD-HHMM>`). If a file with that exact minute already exists, bump to the next minute or append seconds — never overwrite.
- Reports live under `<context-folder>/<subfolder>/`, never directly in the context-folder root.

---

## 🔁 Re-running Against a Different Environment

The user should not need to remember context names or output folders. They just `@`-mention the agent (or use a trigger phrase) and Step 0 walks them through it. The `.dt-playbook-mappings.yaml` file preserves prior pairings so subsequent runs auto-recommend the right folder.

Manual override (if the user wants to skip the interview):

```powershell
dtctl ctx use <other-context>
dtctl auth whoami   # confirm
# then invoke the agent; the agent appends /<subfolder>/<filename-stem>-<YYYY-MM-DD-HHMM>.md automatically
```

The agent should always **re-run all queries** for the new environment rather than copying numbers — every tenant has different emitters, providers, and quirks.

---

## 🛠️ Playbook self-improvement protocol

> **When this applies:** during any review run, if the agent notices something that *would have* improved this or future runs — a new query, a missing report section, a recurring anomaly pattern, a useful follow-up, a phrasing fix, etc. — it should offer to fold the learning back into the upstream source repo via PR.

**Rules:**

1. **Never edit installed skill files at runtime.** Installed SKILL.md files are symlinked from the `aimgr` cache and shared across workspaces. Any improvement to the playbook itself must flow back to the upstream `dt-playbook-skills` repository via PR — the agent MUST NOT write to `.github/skills/dt-*/SKILL.md` in the workspace. The only runtime write is the per-workspace `.dt-playbook-mappings.yaml`.
2. **Surface the suggestion at the end of the run**, after the report is written. Don't interrupt the discovery flow mid-investigation.
3. **Phrase the proposal as a yes/no decision the user can accept in one click.** Use `vscode_askQuestions` when available. Template:

   > *"While generating this report I noticed `<short description of the gap>`. I think the `<skill-name>` playbook would benefit from `<concrete addition: e.g. a new Query N for X, a new anomaly check Y in §What to Look for, an extra row in the report skeleton, …>`. Would you like me to draft a PR description against the dt-playbook-skills repo?"*

   Include 1–3 bullet points showing the proposed wording / DQL / table row so the user is approving the actual change, not a vague promise.
4. **Only one batch of proposals per run.** Group all observations from a single report into one question (with multi-select) rather than asking repeatedly. If there are no improvement ideas, say nothing — do not invent suggestions.
5. **On approval — produce a PR-ready snippet** with:
   - Target file (`skills/dt-bizevents-review/SKILL.md`, `skills/dt-playbook-common/SKILL.md`, etc.).
   - Suggested commit message.
   - The exact markdown diff to paste.
   - Keep additions short. Tables/bullets over prose. Match existing tone and emoji conventions.
   Tell the user one sentence about what to do with the snippet (commit to a fork, open a PR, etc.).
6. **On rejection or skip:** do nothing. Do not record the suggestion elsewhere (no separate TODO file). The user can re-raise it later.
7. **Things worth proposing** (non-exhaustive):
   - A query that was needed during this run but isn't in the Discovery Query Set.
   - A misconfiguration pattern observed across ≥ 2 tenants that isn't yet in §What to Look for.
   - A report section that the user asked for verbally and would clearly help future runs.
   - A DQL pitfall / workaround discovered while debugging a query (especially PowerShell-quoting or operator-availability issues).
8. **Things NOT worth proposing:**
   - Tenant-specific findings (those belong in the report, not the skill).
   - Generic DQL tips already covered by the `dt-dql-essentials` or related skills.
   - Reformatting / cosmetic preferences unless the user explicitly asked.
   - New `dtctl` context entries — those go to `.dt-playbook-mappings.yaml` automatically and do not require a PR.

---

## 🔗 Related References

- `dt-dql-essentials` skill (from `dynatrace-for-ai`, installed via `aimgr install skill/dt-dql-essentials` or `npx skills add dynatrace/dynatrace-for-ai`) — DQL syntax, pitfalls, operator reference.
- Data-source-specific skills as listed in each playbook (`dt-obs-logs`, etc.).
- Sibling skills in this repo: `dt-bizevents-review`, `dt-logs-review`, `dt-business-process-review`.
- Dynatrace docs: per data source — linked from each playbook's Related References section.
