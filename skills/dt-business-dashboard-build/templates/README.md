# Dashboard templates

These JSON files are consumed by the `dt-business-dashboard-build` skill.
Do not import them directly into a tenant — they contain literal placeholder
strings (`[CLIENT_NAME]`, `SERVICE-XXXXXXXXXXXX`, `acme.insurance`, …) that
must be substituted at build time. See `../SKILL.md` §Token inventory for the
full substitution table.

| File | Layout | Best for |
| --- | --- | --- |
| `simple-kpi-dashboard.json` | 3 headline single-values + trend charts + one service-perf strip | A single business flow (checkout, signup, claim registration) where the emphasis is on totals, trends, and category slicing |
| `process-journey-dashboard.json` | Horizontal 5-step layout with per-step Business KPI + IT / Security / Carbon health tiles | An end-to-end multi-step process where each step deserves its own health story |

Extending these templates or adding a new one? File a PR against
`dt-playbook-skills`. Do not edit the installed copies (aimgr symlinks them
into `.github/skills/` at install time).
