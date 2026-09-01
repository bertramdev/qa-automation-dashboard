# QA Automation Dashboard

A static dashboard aggregating Cypress CI results across `bertramdev`'s `*-qa` test-automation projects: current pass/fail state per project/environment/browser, plus a time-series view over a selectable window (1 day / 1 week / 1 month).

**Live site:** enable GitHub Pages on this repo (Settings → Pages → source: `main` branch, `/` root) to get a public URL.

## How it works

Each `*-qa` project's CI workflow (in the job that currently emails run results) appends one JSON entry describing that run to `data/<project>.json` in this repo, via the GitHub Contents API (get current file SHA, append, PUT — retried a few times on conflict). `index.html` is a plain HTML/JS page (Chart.js via CDN, no build step) that fetches each project's `data/<project>.json` directly (same-origin on GitHub Pages, no API rate limit) and renders the two views client-side.

## Data schema

Each `data/<project>.json` file is a JSON array of entries shaped like:

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

1. Add the project's slug to the `PROJECTS` array at the top of `index.html`'s `<script>` block.
2. In that project's CI workflow, add a "Publish results to QA dashboard" step (see `leftlane-wheelhouse-qa`'s `.github/workflows/cypress.yml` for the reference implementation) that PUTs to `data/<project>.json` using the schema above.
3. Add a `DASHBOARD_TOKEN` secret to that project's repo — a fine-grained GitHub PAT scoped to `Contents: Read and write` on this repo only.

## Data retention

Currently unbounded — each run appends one entry, forever. If a project's file grows large enough to slow down fetching, revisit with a periodic roll-up (e.g. a scheduled workflow in this repo that condenses entries older than N months into daily summaries) rather than changing the append-only write path.
