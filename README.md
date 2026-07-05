# Umami Stats — TRMNL plugin

A [TRMNL](https://usetrmnl.com) e-ink plugin that shows your [Umami](https://umami.is)
web-analytics at a glance: **visitors**, **pageviews**, **visits**, and **bounce
rate** for the period, each with its **change vs. the previous period**, plus a
pageviews **spline sparkline** in the headline.

Built on TRMNL **Framework v3** and tuned for **TRMNL X** (responsive sizing,
light-gray theme with white cards). Renders across all four layouts: `full`,
`half_horizontal`, `half_vertical`, and `quadrant`.

## How it works

Each refresh, the plugin makes two Umami API calls (exposed to the templates as
`IDX_0` / `IDX_1`):

- `GET {base}/websites/{id}/stats` → period totals (`visitors`, `pageviews`,
  `visits`, `bounces`) plus a `comparison` block with the previous period's
  totals, which drives the ▲/▼ deltas on each tile.
- `GET {base}/websites/{id}/pageviews?unit=day` → the daily pageviews series
  that draws the hero sparkline.

`{base}` is `https://api.umami.is/v1` (Umami Cloud) or `{umami_url}/api`
(self-hosted), chosen by the **Umami Hosting** field — which also switches the
auth header (`x-umami-api-key` vs. `Authorization: Bearer`).

Umami wants an absolute `[startAt, endAt]` window in epoch **milliseconds**, so
the polling URL derives it in Liquid: `endAt = now`, `startAt = now − date_from
days`. The headline shows period **Visitors** with the **Bounce Rate**
alongside; the four tiles show each metric with its change vs. the previous
period (percentage-point change for bounce rate).

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

Or by hand — `POST {UMAMI_URL}/api/auth/login` with `{"username","password"}`
returns `{ "token": "eyJ…" }`. Use it as `umami_token` (custom field) /
`UMAMI_TOKEN` (`.env`).

## Local development

Uses [`trmnlp`](https://github.com/usetrmnl/trmnlp), run via Docker (no Ruby needed):

```bash
cp .env.example .env     # then fill in your Umami credentials
./bin/trmnlp serve       # open http://localhost:4567
```

`bin/trmnlp` auto-loads `.env`, and `.trmnlp.yml` reads the secrets via
`{{ env.* }}`. Note: only fields used in the polling request are
env-interpolated — `project_name` must be a plain literal.

Every push to `main` runs `trmnlp lint` and deploys to TRMNL. Add a
`TRMNL_API_KEY` repository secret (see `.github/workflows/trmnl.yml`).
