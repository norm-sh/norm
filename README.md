<p align="center">
  <h1 align="center">Norm</h1>
  <p align="center">
    <strong>Catch performance regressions before they ship.</strong>
    <br />
    A GitHub Action that runs Lighthouse (and soon Axe &amp; more) on every push — so you know exactly which changes make your site slower.
  </p>
</p>

<p align="center">
  <a href="https://app.norm.sh">Dashboard</a> ·
  <a href="#quick-start">Quick Start</a> ·
  <a href="#configuration">Configuration</a>
</p>

---

## What is Norm?

Norm is a performance regression tool for web apps. It plugs into your CI pipeline via a GitHub Action, runs [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci) against the routes you care about, and reports back with the metrics that matter — **LCP**, **CLS**, **INP**, **TBT**, and more.

The key idea: **Norm doesn't just measure your Core Web Vitals — it tells you which changes make them worse.** When connected to the [Norm dashboard](https://app.norm.sh), Norm compares every run against the latest baseline on your target branch and shows **deltas** for each metric — so you can see at a glance whether a PR introduces a regression or an improvement.

On PRs, Norm posts a **comment** summarizing the performance impact with ↑/↓ indicators. You can also run Norm on pushes to `main` (or any branch) to build a continuous baseline for your project.

<!-- TODO: Replace with screenshot of the PR comment Norm posts -->
![PR Comment](.github/assets/pr-comment.png)

**Unlike other tools, Norm runs against any URL you give it** — including `localhost`. Just build and start your app in CI (`next build && next start`, `npm start`, whatever you use) and point Norm at it. No preview deployments, no external portals, no waiting for staging environments.

> Lighthouse today. Axe, bundle analysis, and more coming soon.

<!-- TODO: Replace with actual dashboard screenshot -->
![Norm Dashboard](.github/assets/dashboard.png)

---

## Quick Start

### 1. Add `normrc.json` to your repo

```json
{
  "settings": {
    "baseUrl": "http://localhost:3000"
  },
  "routes": ["/", "/about", "/pricing"]
}
```

### 2. Create the GitHub Actions workflow

```yaml
# .github/workflows/norm.yml
name: Norm

on:
  pull_request:
  push:
    branches: [main]

jobs:
  perf:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Start your app however you normally would
      - run: npm ci && npm run build && npm start &
      - run: npx -y wait-on http://localhost:3000

      # Run Norm
      - uses: norm-sh/action@v1
        with:
          config: normrc.json
          token: ${{ secrets.NORM_TOKEN }}   # optional — enables dashboard upload
```

### 3. (Optional) Connect to the Dashboard

The dashboard lets you track performance over time, compare runs across PRs, and catch regressions automatically.

#### Sign up

Go to [app.norm.sh](https://app.norm.sh) and sign up. A default workspace is created for you automatically.

#### Install GitHub & generate a token

1. Click on your **workspace** → **Workspace Settings**
2. **Install the GitHub App** to connect your repos
3. Go to the **API Tokens** section and generate a token

<!-- TODO: Replace with screenshot of workspace settings (GitHub install + token generation) -->
![Workspace Settings](.github/assets/workspace-settings.png)

#### Create a project

1. Click **Create Project** — your connected repos will be available to choose from
2. Select the repo you want to track

<!-- TODO: Replace with screenshot of project creation -->
![Create a project](.github/assets/create-project.png)

#### Add the token to GitHub

1. In your GitHub repo, go to **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `NORM_TOKEN`, Value: paste the token from the previous step

That's it. Every PR will now get Lighthouse runs, and results are uploaded to your dashboard for historical tracking and regression detection.

---

## Configuration

Norm is configured via a `normrc.json` file in your repo root.

### Minimal

```json
{
  "settings": {
    "baseUrl": "http://localhost:3000"
  },
  "routes": ["/"]
}
```

### With Performance Budgets

Set thresholds and get warned when a PR exceeds them.

```json
{
  "settings": {
    "baseUrl": "http://localhost:3000",
    "numberOfRuns": 5
  },
  "budgets": {
    "global": {
      "timings": {
        "largest-contentful-paint": 2500,
        "cumulative-layout-shift": 0.1,
        "total-blocking-time": 200
      }
    }
  },
  "routes": ["/", "/about", "/pricing"]
}
```

### With Route-Specific Budgets

Different pages can have different performance targets.

```json
{
  "settings": {
    "baseUrl": "http://localhost:3000",
    "numberOfRuns": 5
  },
  "budgets": {
    "global": {
      "timings": {
        "largest-contentful-paint": 2500,
        "cumulative-layout-shift": 0.1
      }
    },
    "strict": {
      "timings": {
        "largest-contentful-paint": 1000,
        "total-blocking-time": 200
      }
    }
  },
  "routes": [
    "/",
    "/about",
    {
      "path": "/dashboard",
      "budget": "strict"
    }
  ]
}
```

### Reference

#### `settings`

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `baseUrl` | `string` | ✅ | — | The base URL of your running app |
| `numberOfRuns` | `number` | — | `3` | Number of Lighthouse runs per route (median is used) |

#### `budgets`

A map of named budget profiles. The `global` budget applies to all routes. Any other named budget can be assigned to specific routes.

Each budget can include `timings` with thresholds for:

| Metric | Key | Unit |
|--------|-----|------|
| Largest Contentful Paint | `largest-contentful-paint` | ms |
| Cumulative Layout Shift | `cumulative-layout-shift` | score |
| Interaction to Next Paint | `interaction-to-next-paint` | ms |
| Total Blocking Time | `total-blocking-time` | ms |
| First Contentful Paint | `first-contentful-paint` | ms |

#### `routes`

An array of routes to audit. Each entry can be:

- **A string** — e.g. `"/"`, `"/about"` — uses the global budget (if defined)
- **An object** — `{ "path": "/dashboard", "budget": "strict" }` — uses a named budget

---

## Action Inputs & Outputs

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `config` | ✅ | `normrc.json` | Path to your `normrc.json` config file |
| `out` | — | `.norm/norm-run.tgz` | Output path for the results bundle |
| `token` | — | — | Norm API token (enables upload to dashboard) |

### Outputs

| Output | Description |
|--------|-------------|
| `artifact-path` | Path to the generated `.tgz` results bundle |

---

## How It Works

```
normrc.json → Lighthouse CI → Performance Reports → .tgz Bundle → Norm Dashboard
```

1. **Config** — Norm reads your `normrc.json` and transforms it into an LHCI configuration
2. **Collect** — Lighthouse CI runs against each route, multiple times per route for stable medians
3. **Package** — Results (metrics + HTML reports) are bundled into a single `.tgz` artifact
4. **Upload** — If a `token` is provided, the bundle is uploaded to the Norm API for historical tracking and regression detection
