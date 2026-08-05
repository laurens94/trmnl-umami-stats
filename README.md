# Umami Stats — TRMNL plugin

A [TRMNL](https://usetrmnl.com) e-ink plugin that shows your [Umami](https://umami.is)
web-analytics at a glance: **visitors**, **pageviews**, **visits**, and **bounce
rate** for the period, each with its **change vs. the previous period**, plus a
**daily chart** of visitors (solid) and pageviews (dotted) with dated axis ticks.

Built on TRMNL **Framework v3.2** and tuned for **TRMNL X**: responsive sizing,
a light-gray-with-white-cards default that follows your device's **theme and dark
mode**, and an **adaptive chart** that repaints when the screen's theme, mode or
scale changes. Renders across all four layouts: `full`, `half_horizontal`,
`half_vertical`, and `quadrant`.

## How it works

Each refresh, the plugin makes two Umami API calls (exposed to the templates as
`IDX_0` / `IDX_1`):

- `GET {base}/websites/{id}/stats` → period totals (`visitors`, `pageviews`,
  `visits`, `bounces`) plus a `comparison` block with the previous period's
  totals, which drives the ▲/▼ deltas on each tile.
- `GET {base}/websites/{id}/pageviews?unit=day` → the two daily series behind the
  chart: `pageviews`, and `sessions` (distinct visitors per day). Umami exposes no
  daily **visits** series, so the chart pairs visitors with pageviews.

`{base}` is `https://api.umami.is/v1` (Umami Cloud) or `{umami_url}/api`
(self-hosted), chosen by the **Umami Hosting** field — which also switches the
auth header (`x-umami-api-key` vs. `Authorization: Bearer`).

Umami wants an absolute `[startAt, endAt]` window in epoch **milliseconds**, so
the polling URL derives it in Liquid: `endAt = now`, `startAt = now − date_from
days`. The four tiles show each metric with its change vs. the previous period
(percentage-point change for bounce rate); Umami returns no `comparison` data
when the preceding window is empty, so those tiles then read `—`.

## Configuration

Set these custom fields (in the TRMNL UI, or in `.trmnlp.yml` for local dev):

| Field | Description |
| --- | --- |
| `umami_hosting` | **Umami Cloud** or **Self-hosted** — sets the base URL and auth scheme |
| `umami_url` | Self-hosted only: instance base URL, e.g. `https://analytics.example.com` (no trailing slash). Ignored for Cloud |
| `umami_token` | Umami Cloud API key, or a self-hosted bearer token (see below) |
| `umami_website_id` | The website's UUID (Umami → Settings → Websites → Edit → Website ID) |
| `project_name` | Shown in the footer |
| `date_from` | Days of history to fetch (e.g. `30`) |
| `chart_lines` | Which lines the chart draws: `both`, `visitors`, or `pageviews` |
| `chart_axes` | With both lines: `single` shared y axis, or `dual` (left = visitors, right = pageviews) |

### Getting an API key / token

**Umami Cloud:** create an API key in your account settings
([docs](https://umami.is/docs/cloud/api-key)) and use it as `umami_token`.

**Self-hosted:** the API uses a bearer token. A helper script wraps the
[login endpoint](https://umami.is/docs/api/authentication) — it reads `UMAMI_URL`
from `.env` (or prompts), asks for your username/password, and prints the token:

```bash
./bin/umami-token                 # prompts for everything, prints the token
./bin/umami-token admin --write   # log in as "admin" and store it in .env
```

No repo checked out? Grab a token with a single command — fill in your host,
username, and password, then paste the printed token into the plugin's **API
Key / Token** field in the TRMNL UI:

```bash
curl -s -X POST "https://analytics.example.com/api/auth/login" -H 'Content-Type: application/json' -d '{"username":"admin","password":"YOUR_PASSWORD"}' | jq -r '.token'
```

No `jq`? Replace the final `| jq -r '.token'` with `| grep -o '"token":"[^"]*"' | cut -d'"' -f4`.

## Local development

Uses [`trmnlp`](https://github.com/usetrmnl/trmnlp), run via Docker (no Ruby needed):

```bash
cp .env.example .env     # then fill in your Umami credentials
./bin/trmnlp serve       # open http://localhost:4567
```

`bin/trmnlp` auto-loads `.env`, and `.trmnlp.yml` reads the secrets via
`{{ env.* }}`. Note: only fields used in the polling request are
env-interpolated — `project_name` must be a plain literal.

Every push to `master` runs `trmnlp lint` and deploys to TRMNL. Add a
`TRMNL_API_KEY` repository secret (see `.github/workflows/trmnl.yml`).
