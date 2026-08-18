# frontend-cd

Desired deployment state for the `frontend` service (the CityPulse Vite/React SPA).
This repo does not contain application code or build logic — it declares which
immutable container image digest should run in each environment, and delegates the
actual deployment mechanics to the reusable workflows in
[`deployment-workflows`](https://github.com/mustapha-smail-org/deployment-workflows).

## Layout

```
service.yaml                  Provider + health-check configuration
environments/
  dev.yaml                    Auto-updated on every merge to frontend main
  staging.yaml                Auto-updated when a semantic release tag is created
  production.yaml             Updated only via reviewed pull request
config/
  app-config.dev.json         Runtime config mounted onto Render (see below)
  app-config.staging.json
  app-config.production.json
.github/
  CODEOWNERS                  Requires data-platform-team review for production.yaml
  workflows/deploy.yml        Updates state, pushes config, calls deploy-render.yml
```

`service.yaml` and each `environments/*.yaml` conform to the schemas in
`deployment-workflows/contracts/`.

## Required repo configuration

- **Variable** `AUTOMATION_APP_ID` and **secret** `AUTOMATION_APP_PRIVATE_KEY` —
  yes, on *this* repo too, not just `frontend`. `deploy.yml` authenticates as the
  `city-pulse-automation` App before checkout/push so the branch ruleset's App
  bypass entry actually applies; without these, `deploy.yml` falls back to the
  default `GITHUB_TOKEN` (identity `github-actions[bot]`), which is a different
  actor than the bypass list allows, and the push is rejected with `GH013`.
- **Secret** `RENDER_API_KEY` — used by `deploy-render.yml` and by `push-config`.
- **Secrets** `RENDER_SERVICE_ID_DEV`, `RENDER_SERVICE_ID_STAGING`,
  `RENDER_SERVICE_ID_PRODUCTION` — the actual Render `srv-...` service IDs (not
  the human-readable service name shown in the Render dashboard URL). Deliberately
  **not** stored in `environments/*.yaml`, and deliberately **not** resolved in a
  `run:` step either — the `deploy` job's `secrets:` block picks the right one via
  a plain `&&`/`||` expression on `inputs.environment` (evaluated before any job
  runs). Routing a secret through a step/job output into another job's
  `with:`/`secrets:` doesn't work: GitHub Actions silently blanks any job output
  derived from a masked value once it crosses that boundary (see
  `deployment-workflows/docs/WORKFLOW_CONTRACTS.md`, "Secret job outputs get
  blanked, not passed through").
- The `main` branch ruleset must list `city-pulse-automation` in its **bypass
  list** with mode **Always** (`Settings → Rules → Rulesets`), or every
  `deploy.yml` run fails to push the desired-state commit.
- No other secrets are required today — see "Runtime config" below for why, and
  for how to add one (e.g. a paid map-tile provider token) when needed.

## Runtime config (`config/app-config.<env>.json`)

Unlike a typical Vite deploy, this service does **not** bake `VITE_*` environment
variables into the JS bundle at build time. Doing so would mean a different image
per environment, breaking this org's "build once, promote the same immutable
digest through every environment" model — the same reason `ci-release.yml` never
rebuilds. Instead:

1. `deploy.yml`'s `push-config` job pushes `config/app-config.<env>.json` to a
   Render secret file mounted at `/etc/secrets/app-config.json`, before the
   `deploy` job triggers the image deploy.
2. The `frontend` image's container entrypoint (`docker/entrypoint.sh` in that
   repo) reads that file at container startup:
   - `API_GATEWAY_URL` is **server-side only** — substituted into nginx's
     `proxy_pass` target so the SPA's same-origin `/api/` requests reach the API
     Gateway. It is never shipped to the browser.
   - Every other key (all `VITE_*`-prefixed, matching `frontend/.env.example`'s
     names) is copied verbatim into `window.__APP_CONFIG__` in
     `public/config.js`, which `frontend/src/shared/config/env.ts` reads and
     merges **over** the build-time `import.meta.env` fallbacks.
3. A missing or malformed config file makes the container fail to start (see
   `frontend`'s entrypoint) rather than silently falling back to build-time
   defaults — in production that would mean serving the public OpenStreetMap tile
   endpoint, which the PRD (`FR-MAP-006`) explicitly forbids for production
   traffic.

**Adding a real secret later** (e.g. a paid map-tile provider API key) follows the
same `%%SECRET:NAME%%`-token pattern as `data-ingestion-cd`/`catalog-service-cd`
(see `deployment-workflows/docs/WORKFLOW_CONTRACTS.md`, "Externalized App Config"):
put the token in the relevant `config/app-config.<env>.json` value, declare one
`SECRET_<NAME>` line in `push-config`'s `env:` block, and add the matching repo
secret. The substitution loop and the unresolved-placeholder guard in
`push-config` already run unconditionally — no other change is needed.

**Before this repo can deploy anything for real**, replace each
`config/app-config.<env>.json`'s `API_GATEWAY_URL` placeholder
(`REPLACE_WITH_API_GATEWAY_<ENV>_RENDER_URL`) with the actual Render URL of that
environment's `api-gateway` service, once it exists.

**Before production go-live**, replace `config/app-config.production.json`'s
`VITE_MAP_TILE_URL`/`VITE_MAP_ATTRIBUTION` with a tile provider CityPulse is
actually entitled to use in production — the public OpenStreetMap endpoint used
here as a placeholder is explicitly not a production CDN (see
`frontend/.env.example` and PRD `FR-MAP-006`). `dev`/`staging` can keep it.

## Render Free Plan note

If this organization has fewer than 3 Render services available (as is currently
the case for `data-ingestion`), point `RENDER_SERVICE_ID_DEV` and
`RENDER_SERVICE_ID_STAGING` at the same service — nothing else changes. Upgrade
later by creating a dedicated staging service and updating
`RENDER_SERVICE_ID_STAGING`.

## How deployments happen

1. **Dev:** `frontend`'s `main.yml` workflow (via the `ci-main-node.yml` template)
   builds an image and dispatches `deploy.yml` here with `environment=dev`.
2. **Staging:** `frontend`'s `release.yml` workflow (via `ci-release.yml`)
   re-tags the existing image for the released commit and dispatches `deploy.yml`
   here with `environment=staging`. No image is rebuilt.
3. **Production:** open a PR here updating `environments/production.yaml` with the
   digest/tag/commit that was validated in staging. Requires CODEOWNERS approval.
   Merging triggers `deploy.yml` with `environment=production` (wire a
   `push`-to-`main`-with-path-filter trigger, or run `deploy.yml` manually via
   `workflow_dispatch`, once the promotion flow is finalized).

`deploy.yml` runs three jobs in sequence: `update-desired-state` (updates and
commits the target `environments/<env>.yaml`), `push-config` (resolves
`config/app-config.<env>.json` and mounts it on Render — see above), then
`deploy` (calls `deploy-render.yml` in `deployment-workflows` to actually trigger
and verify the image deployment, checking `/healthz`).

## Rollback

1. Find the previous known-good digest in this repo's Git history
   (`git log -p environments/<env>.yaml`).
2. Manually run `deploy.yml` (`workflow_dispatch`) with that digest, the
   corresponding source commit, and a tag noting the rollback (e.g.
   `rollback-v1.2.0`) — or open a PR restoring the previous file content for
   `production.yaml`.
3. No rebuild happens; the same image digest is redeployed.
4. Record the reason in the commit/PR description.
