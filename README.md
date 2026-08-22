# frontend-cd

Desired deployment state for the `frontend` service (the CityPulse Next.js app).
This repo holds no application code or build logic — it declares which immutable
container image digest runs in each environment and delegates the deployment
mechanics to the reusable workflows in
[`deployment-workflows`](https://github.com/mustapha-smail-org/deployment-workflows).

## Layout

```
service.yaml                  Image repository + health-check configuration
environments/
  staging.yaml                Auto-updated when a semantic release tag is created
  production.yaml             Updated only via reviewed pull request
.github/
  CODEOWNERS                  Requires review for production.yaml
  workflows/deploy.yml        Updates desired state, calls deploy-vps.yml
```

`service.yaml` and each `environments/*.yaml` conform to the schemas in
`deployment-workflows/contracts/`.

## Runtime config

The Next.js app is configured entirely through environment variables, set per
environment in `platform-infrastructure`'s `docker-compose.yml` (not delivered
from this repo):

- `CITYPULSE_API_BASE_URL` — internal gateway URL (`http://api-gateway:8080`).
  Read server-side by the app's BFF route handlers (`src/lib/proxy.ts`), which
  proxy the browser's same-origin `/api/*` calls to the gateway.
- `NEXT_PUBLIC_SITE_URL` — public site origin, used server-side for metadata.

Both are consumed server-side, so the single image built once in `frontend` is
promoted unchanged through staging and production — no per-environment rebuild,
and no runtime secret file. (The old nginx/`app-config.json` mechanism is gone.)

## Required repo configuration

- **Variable** `AUTOMATION_APP_ID` and **secret** `AUTOMATION_APP_PRIVATE_KEY` —
  on *this* repo too. `deploy.yml` authenticates as the `city-pulse-automation`
  App before checkout/push so the branch ruleset's App bypass applies; without
  them it falls back to `GITHUB_TOKEN` (`github-actions[bot]`), a different actor
  than the bypass list allows, and the push is rejected with `GH013`.
- The `main` branch ruleset must list `city-pulse-automation` in its **bypass
  list** with mode **Always**, or every `deploy.yml` run fails to push the
  desired-state commit.
- VPS access (`VPS_HOST`, `VPS_USER`, `VPS_SSH_PRIVATE_KEY`, `VPS_KNOWN_HOSTS`)
  is consumed by the `deploy` job via `deploy-vps.yml`.

## How deployments happen

1. **Staging:** `frontend`'s `release.yml` (via `ci-release.yml`) dispatches
   `deploy.yml` here with `environment=staging` and the released image digest.
   No image is rebuilt.
2. **Production:** open a PR here updating `environments/production.yaml` with the
   digest/tag/commit validated in staging (CODEOWNERS approval required), or run
   `deploy.yml` manually via `workflow_dispatch`.

`deploy.yml` runs `update-desired-state` (commits the target
`environments/<env>.yaml`) then `deploy` (calls `deploy-vps.yml`, which re-pins
the digest on the box, recreates the service, and health-gates it on `/healthz`).

## Rollback

1. Find the previous known-good digest (`git log -p environments/<env>.yaml`).
2. Run `deploy.yml` (`workflow_dispatch`) with that digest, its source commit,
   and a tag noting the rollback (e.g. `rollback-v1.2.0`) — or open a PR
   restoring the previous `production.yaml` content.
3. No rebuild happens; the same image digest is redeployed.
4. Record the reason in the commit/PR description.
