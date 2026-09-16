# app

Hello world service. Node, no dependencies.

## Deployment facts

| | |
|---|---|
| Subdomain | `app.nerry.tech` |
| Container port | `3000` |
| Health path | `/healthz` |
| Image | `ghcr.io/nerry-labs/app:<sha>` |

Source of truth for subdomain and port is `.github/workflows/deploy.yml`. If
this table and that file disagree, the workflow wins — fix the table.

## Before changing anything about deployment

The deployment contract, server layout, and troubleshooting live in one place
for the whole organization. Read it first:

```bash
gh api repos/nerry-labs/.github/contents/devops.md \
  --jq '.content' | base64 -d
```

Do not copy that file into this repo. It changes, and a stale copy is worse
than no copy.

## Deploying

Actions → Deploy → Run workflow → `deploy`. Requires approval on the
`production` environment.

`destroy` from the same dropdown removes the subdomain and stops the containers.

Rollback runs against the server, not GitHub:

```bash
ssh -T -i ~/.ssh/gha_deploy deploy@$PROD_HOST "rollback app"
```

## Local

```bash
node server.js
curl localhost:3000
curl localhost:3000/healthz
```

## Constraints this repo must keep

Breaking any of these breaks the deploy. Details and reasoning are in
`devops.md`.

- Bind `0.0.0.0`, read the port from `PORT`
- Serve `GET /healthz` with a 2xx
- Keep `LABEL org.opencontainers.image.source` in the Dockerfile
- No published ports in any compose file — Caddy reaches this over the `edge`
  network
- Handle `SIGTERM` so redeploys don't wait out the 10s kill timeout
