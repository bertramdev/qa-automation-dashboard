# QA Automation Dashboard

A static dashboard aggregating Cypress CI results across `bertramdev`'s `*-qa` test-automation projects: current pass/fail state per project/environment/browser, plus a time-series view over a selectable window (1 day / 1 week / 1 month).

**Status: infrastructure only, not yet live.** `index.html` and the publishing mechanism below are fully built, but deliberately not switched on yet: GitHub Pages is not enabled on this repo, no `DASHBOARD_TOKEN` has been provisioned anywhere, and `data/` is empty. Flipping this on is a deliberate later decision — see "Going live" below — not something to infer from this repo's mere existence.

## How it works

Every `*-qa` project's CI workflow calls this repo's reusable **[`publish-results` composite action](.github/actions/publish-results/action.yml)** (`uses: bertramdev/qa-automation-dashboard/.github/actions/publish-results@main`) from the job that currently emails run results, passing in the metrics that job already computed. The action is **destination-agnostic**: it builds one canonical result JSON object, then hands it to whichever "sinks" are configured via that step's inputs. Today there are two sinks, both independently optional and both silent no-ops (never fail the calling job) when their credentials aren't set:

- **GitHub dashboard sink** — appends the result to this repo's `data/<project>.json` via the GitHub Contents API (get current file SHA, append, PUT — retried a few times on conflict). `index.html` (plain HTML/JS, Chart.js via CDN, no build step) fetches each project's `data/<project>.json` directly — same-origin and rate-limit-free once served from GitHub Pages — and renders the two views client-side.
- **monday.com sink** — a template/reference implementation (see "Adding a new sink" below) that creates one board item per run via monday's `create_item` GraphQL mutation. Not yet exercised against a real board — no board or column schema has been defined for this org.

A calling workflow only ever passes in already-computed metrics; it never needs to know which sinks exist or how they work. Adding a third destination later (or swapping which one is actually live) means changing this one action, not every project's workflow.

## Data schema

The composite action's "Build canonical result JSON" step produces one entry shaped like this — it's both what the GitHub dashboard sink appends to `data/<project>.json` and the source data the monday.com sink (or any future sink) works from:

```json
{
  "project": "leftlane-wheelhouse-qa",
  "repo": "bertramdev/leftlane-wheelhouse-qa",
  "run_id": 33497239286,
  "run_url": "https://github.com/bertramdev/leftlane-wheelhouse-qa/actions/runs/33497239286",
  "timestamp": "2026-09-01T11:05:00Z",
  "environment": "qa",
  "browser": "chrome",
  "status": "FAILED",
  "passed": 812,
  "failed": 4,
  "flaky": 3,
  "skipped": 12,
  "specs_unrun": 0,
  "elapsed_seconds": 5423,
  "cost_usd": 3.12,
  "commit_sha": "a236c53...",
  "cloud_url": null
}
```

## Adding a new project

1. Add the project's slug to the `PROJECTS` array at the top of `index.html`'s `<script>` block (this drives the GitHub-dashboard sink's rendering — skip if that project will only ever use a different sink).
2. In that project's CI workflow, in the job that already computes per-run metrics (usually the same job that emails results), export the metrics to `$GITHUB_ENV` if they aren't already available to a later step, then add a step calling the composite action:
   ```yaml
   - name: Publish results to QA dashboard
     if: always()
     uses: bertramdev/qa-automation-dashboard/.github/actions/publish-results@main
     with:
       project: <project-slug>
       repo: ${{ github.repository }}
       run_id: ${{ github.run_id }}
       run_url: https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}
       environment: ${{ env.DASHBOARD_ENV }}
       browser: ${{ env.DASHBOARD_BROWSER }}
       status: ${{ env.DASHBOARD_STATUS }}
       passed: ${{ env.DASHBOARD_PASSED }}
       failed: ${{ env.DASHBOARD_FAILED }}
       flaky: ${{ env.DASHBOARD_FLAKY }}
       skipped: ${{ env.DASHBOARD_SKIPPED }}
       specs_unrun: ${{ env.DASHBOARD_UNRUN }}
       elapsed_seconds: ${{ env.DASHBOARD_ELAPSED_SECS }}
       cost_usd: ${{ env.DASHBOARD_COST_USD }}
       commit_sha: ${{ env.DASHBOARD_COMMIT_SHA }}
       cloud_url: ${{ env.DASHBOARD_CLOUD_URL }}
       dashboard_token: ${{ secrets.DASHBOARD_TOKEN }}
   ```
   See any of `leftlane-wheelhouse-qa`, `equivate-qa`, `ridgeline-insurance-app-qa`, or `mse-zendesk-app-qa`'s `.github/workflows/cypress.yml` for a working reference — each already has this wired in (as a no-op, pending step 3 below).
3. Add a `DASHBOARD_TOKEN` secret to that project's repo — a fine-grained GitHub PAT scoped to `Contents: Read and write` on this repo only — to actually turn the GitHub-dashboard sink on for that project.

## Adding a new sink

A "sink" is one destination for the canonical result JSON — the GitHub-dashboard append and the monday.com `create_item` call are both examples. To add another (a different SaaS tool, Slack, a data warehouse, etc.), edit only `.github/actions/publish-results/action.yml`:

1. Add an optional input pair for whatever credential/target the new sink needs (e.g. `xyz_token`, `xyz_target_id`), each defaulting to `''` so it's opt-in.
2. Add one step: `if: always() && inputs.xyz_token != ''`, consuming `steps.build.outputs.json` (the canonical result) and sending it however that destination expects. Never fail the job on that sink's failure — `echo "::warning::..."` instead, matching the existing two sinks.
3. Add a matching `published_xyz` output if callers might want to check whether that sink succeeded.

No calling workflow needs to change — every `*-qa` project already calls this action, so a new sink goes live for all of them the moment its credentials are configured, without touching a single project's `cypress.yml` again.

## Going live

Three independent switches, none flipped yet:

1. **Enable GitHub Pages** on this repo (Settings → Pages → source: `main` branch, `/` root) to get a public URL for `index.html`.
2. **Provision a `DASHBOARD_TOKEN`** (fine-grained PAT, `Contents: Read and write`, scoped to this repo only) and add it as a secret on each `*-qa` project that should publish to the GitHub-dashboard sink.
3. **Or/also provision monday.com credentials** (`monday_api_token`, `monday_board_id`, and a `monday_column_values` mapping once a real board/schema exists) if that sink is the one to go live with instead of or alongside the GitHub dashboard.

## Data retention

Currently unbounded — each run appends one entry, forever. If a project's file grows large enough to slow down fetching, revisit with a periodic roll-up (e.g. a scheduled workflow in this repo that condenses entries older than N months into daily summaries) rather than changing the append-only write path.
