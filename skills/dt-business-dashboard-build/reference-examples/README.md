# Dashboard reference examples

These JSON files are **shape references** for the `dt-business-dashboard-build`
skill. They are NOT templates — the skill does not copy them, substitute
placeholders in them, or use them as a starting scaffold. Cloning either file
was the reason earlier runs produced rigid, generic dashboards that didn't
adapt to the specific business process under review.

## What they are for

Consult these files when you need to answer a **JSON-shape** question while
generating a bespoke dashboard, for example:

- What are the valid top-level keys? (`version`, `variables`, `tiles`,
  `layouts`, `settings`)
- What does a `data` tile look like end-to-end (`query` + `visualization` +
  `visualizationSettings` + `unitsOverrides`)?
- How is a `markdown` tile keyed and where does its content live?
- How is a multi-metric timeseries tile configured?
- What grid keys does a `layouts` entry expect (`x`, `y`, `w`, `h`)?
- How is a variable referenced from a tile query?

For anything the JSON shape doesn't answer, defer to the `dt-app-dashboards`
skill from `dynatrace-for-ai`.

## What they are NOT for

- ❌ Do not copy the layout, tile mix, tile titles, or column structure.
- ❌ Do not treat any string in these files as a placeholder to substitute
  (e.g. `[CLIENT_NAME]`, `SERVICE-XXXXXXXXXXXX`, `acme.insurance`) — those
  strings are illustrative and will trip the skill's Step 5 placeholder scan
  if they survive into an emitted dashboard.
- ❌ Do not import these files directly into a tenant. They are illustrative.

## Files

| File | What it illustrates |
| --- | --- |
| `simple-kpi-dashboard.json` | A flat business-KPI layout — headline single-values, trend charts, one service-perf strip. Useful for seeing how single-value + timeseries + table tiles co-exist. |
| `process-journey-dashboard.json` | A horizontal per-step swim-lane layout — per-step business KPI + IT / Security / Carbon health tiles. Useful for seeing how large tile grids are keyed and arranged. |

## Contributing

To add a new shape reference or update an existing one, open a PR against the
`dt-playbook-skills` repo. Keep additions genuinely illustrative — if a
reference example starts to feel like a template, split its useful patterns
into the SKILL.md text instead and delete the redundant JSON.
