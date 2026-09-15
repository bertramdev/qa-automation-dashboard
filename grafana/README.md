# Grafana QA Dashboard

`qa-dashboard.json` is a dashboard-as-code definition for the Grafana Loki
sink (see the repo root `README.md`'s "Data schema" and "Going live"
sections). It emulates Cypress Cloud's own run-analytics view: a
**current-state** snapshot of the latest run plus **time-series trends**
that respect Grafana's own time range picker (which already includes Last 7
days / Last 30 days / Last 90 days / Last 1 year among its default quick
ranges — no custom picker needed).

**Status: built from the documented schema, not yet verified against live
data.** As of 2026-09-15, no Grafana instance with a real
`{job="qa-automation-dashboard"}` stream has been located — see "Known gaps"
below. Import this once real data exists and sanity-check it against the
checklist there before treating any panel as trustworthy.

## Why one file, not one per project

Each `*-qa` project's `GRAFANA_LOKI_URL` most likely points at that
project's **own** Grafana instance, not a stack shared across all four (see
"Known gaps"). Rather than hardcode a Loki data source UID, every panel
targets `${DS_LOKI}` — Grafana's standard "export for sharing externally"
placeholder — so this same JSON imports cleanly into any project's own
instance; you just pick that instance's own Loki data source once at import
time. The `$project`/`$environment`/`$browser` dropdowns (top of the
dashboard) still work per-instance even when only one project's data ever
lands there — they just won't have much to filter until more values exist.

## How to import

1. In the target Grafana instance: **Dashboards → New → Import**.
2. Upload `qa-dashboard.json` (or paste its contents).
3. When prompted for the **Loki** data source input, pick that instance's
   Loki data source that receives the `GRAFANA_LOKI_URL` push (**not**
   necessarily the same Loki data source used for that app's own Docker
   container logs — see "Known gaps" below, those are two different things
   in at least two projects checked so far).
4. Save. The `$project` dropdown should populate from
   `label_values({job="qa-automation-dashboard"}, project)` — if it comes
   back empty, no data has reached this Loki instance/data source yet.

To re-import an update later: repeat Import with the same file — Grafana
matches on the dashboard's `uid` (`qa-automation-dashboard-cypress`) and
offers to overwrite in place.

## Panel-by-panel notes

- **Current State row** (Passed / Failed / Flaky / Skipped / Pass Rate /
  Duration / Cost, plus a Recent Runs table) — each stat is
  `last_over_time(...unwrap <field>...[$__range])`, i.e. the most recent
  run's value within whatever time range is selected. If the selected range
  has no runs in it, these show "No data" — expected, not a bug.
- **Pass Rate (gauge)** and **Pass Rate Over Time** — the two panels flagged
  `UNVERIFIED` in their own description. Both compute a ratio client-side
  via chained `calculateField` transformations across two Loki queries
  (Passed, Failed) referenced by their `legendFormat` name. This is a
  well-established Grafana pattern, but it depends on Grafana actually
  naming the resulting fields exactly `Passed`/`Failed` — **check this
  first** once you have real data: open each panel's transform tab and
  confirm the operand names match, or fix them to whatever your Grafana
  version actually names the fields.
- **Recent Runs table** — uses an `extractFields` transform (source: `Line`,
  format: JSON) to split the log line's JSON body into columns. This is the
  version-portable way to get parsed Loki fields into a *dashboard* table
  (Explore's own auto-column-expansion is an Explore-only convenience and
  doesn't carry over). Sanity-check the column set once real log lines
  exist — field names come straight from the schema in the repo root
  README.
- **Test Results Over Time / Run Duration Trend / Spec panels** — single
  Loki metric queries with no cross-series math, the lowest-risk panels
  here. `type="spec"` panels stay empty until a caller also sends
  `spec_timings_json` (optional input, only `leftlane-wheelhouse-qa` wires
  it in today per that repo's own CI).

## Verification checklist (run once real data + Grafana access exist)

1. Import into the instance that actually receives the `GRAFANA_LOKI_URL`
   push for at least one project.
2. Confirm `$project` populates and Recent Runs shows real rows.
3. Open Pass Rate (gauge) and Pass Rate Over Time — if either shows "No
   data" with rows present in Recent Runs, fix the `calculateField` operand
   names (see above).
4. Trigger that project's CI once, then confirm the new run appears within
   a few minutes (respecting the panel's own `refresh: 5m`, or hit refresh
   manually).
5. If `spec_timings_json` is wired for that project, confirm the Spec
   Insights row populates too.

## Known gaps (as of 2026-09-15)

- **leftlane-wheelhouse-qa has no `GRAFANA_LOKI_*` secrets provisioned** —
  its CI wiring is complete (calls this repo's `publish-results` action with
  every field including `spec_timings_json`), but nothing has ever been
  pushed. Someone needs to provision the three secrets per that repo's own
  `docs/GRAFANA_LOKI_SETUP.md` before this dashboard has anything to show
  there.
- **Only equivate-qa has `GRAFANA_LOKI_URL`/`_USER`/`_TOKEN` set** (since
  2026-09-11), but the actual Loki endpoint they point at hasn't been
  located: two candidate self-hosted Grafana instances were checked
  (`grafana.qa.mywheelhouseapp.com` for leftlane-wheelhouse-qa,
  `grafana.qa.equivate.io` for equivate-qa) and **both turned out to be
  each app's own Docker-container-log Grafana** — labels
  `environment=docker`, `service`/`service_name` = that app's own service
  names, no `job` label at all, zero `{job="qa-automation-dashboard"}`
  data over the last 30 days in either. The real Grafana Cloud stack (or
  self-hosted Loki) that `GRAFANA_LOKI_URL` actually names is still
  unidentified — whoever provisioned equivate-qa's secrets on 2026-09-11
  would know which stack they used.
- **ridgeline-insurance-app-qa and mse-zendesk-app-qa** also have complete
  CI wiring and their own `docs/GRAFANA_LOKI_SETUP.md`, but no
  `GRAFANA_LOKI_*` secrets at all yet.
- Once the real instance is found for any one project, re-run the
  verification checklist above before assuming the dashboard is correct for
  the others — the schema is identical across projects, so a pass on one
  is a strong (not certain) signal for the rest.
