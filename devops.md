# devops.md

Deployment contract for `nerry-labs`. Single source of truth — do not copy this
file into application repositories. Each repo carries a short `CLAUDE.md` that
states its own subdomain and port and points here.

Written for both humans taking over and Claude Code making infrastructure
changes. If you are an agent: read [Rules for agents](#rules-for-agents) before
changing anything.

---

## The system in one paragraph

One DigitalOcean droplet (`nerrylabs-core`, Ubuntu 24.04, 2 vCPU / 4 GB) runs
Docker. A single Caddy container owns ports 80 and 443 and terminates TLS for
every `*.nerry.tech` subdomain. Application containers publish no ports at all;
they join a shared Docker network called `edge` and Caddy reaches them by
service name. Deploys are manual, triggered from a repository's Actions tab.
GitHub builds the image, pushes it to GHCR, then SSHes to the server as a
restricted `deploy` user that can only run one script.

```
GitHub Actions
  ├─ build image ──────────────► ghcr.io/nerry-labs/<repo>:<sha>
  └─ ssh deploy@server ────────► /usr/local/bin/app-deploy
                                    ├─ docker pull
                                    ├─ write /srv/apps/<app>/docker-compose.yml
                                    ├─ write /srv/caddy/sites/<app>.caddy
                                    ├─ docker compose up --wait
                                    └─ caddy reload
```

---

## The deploy contract

A repository is deployable when it satisfies all six of these. Nothing else is
required, and nothing on the server needs touching to add a new app.

| # | Requirement | Why |
|---|---|---|
| 1 | Repository name is the app name: lowercase, `[a-z0-9-]`, max 32 chars | Used as the directory name, compose service name, and GHCR image name |
| 2 | `Dockerfile` at the repo root | The pipeline builds with `context: .` |
| 3 | `LABEL org.opencontainers.image.source="https://github.com/nerry-labs/<repo>"` | Links the GHCR package to the repo. Without it, the server's pull returns 403 |
| 4 | App reads `PORT` from the environment and binds `0.0.0.0` | The pipeline injects `PORT`. Binding `127.0.0.1` makes it unreachable from Caddy |
| 5 | App serves `GET /healthz` returning 2xx | Drives the container healthcheck and the deploy gate |
| 6 | `.github/workflows/deploy.yml` calling the reusable workflow | See below |

### The workflow

```yaml
name: Deploy

on:
  workflow_dispatch:
    inputs:
      mode:
        description: What to do
        type: choice
        options: [deploy, destroy]
        default: deploy

concurrency:
  group: deploy-${{ github.repository }}
  cancel-in-progress: false

jobs:
  run:
    uses: nerry-labs/.github/.github/workflows/deploy.yml@main
    with:
      subdomain: <app>.nerry.tech
      port: <port>
      mode: ${{ inputs.mode }}
    secrets: inherit
```

`subdomain` and `port` are the entire deploy interface. They are the source of
truth — do not duplicate them in prose elsewhere in the repo.

### Repository settings

Both are required, and both produce confusing errors when missing.

1. Settings → Environments → New environment → `production`, with yourself as a
   required reviewer. Missing: the job queues forever with no error.
2. The four organization secrets must be scoped to this repo:
   `PROD_SSH_KEY`, `PROD_KNOWN_HOSTS`, `PROD_HOST`, `PROD_USER`.

---

## Adding a new application

No server access needed.

1. Create `nerry-labs/<name>` with the six contract items above.
2. Pick a subdomain under `nerry.tech` that nothing else uses. Check with
   `ssh -T deploy@<host> "status <name>"` or by listing
   `/srv/caddy/sites/` if you have shell access.
3. Add a DNS record for the subdomain if the wildcard doesn't cover it. `*.nerry.tech`
   is proxied through Cloudflare; the apex `nerry.tech` is **not** covered by the
   wildcard and needs its own record.
4. Create the `production` environment and confirm the secrets are scoped.
5. Actions → Deploy → Run workflow → `deploy`.

The server creates `/srv/apps/<name>/` and `/srv/caddy/sites/<name>.caddy` on
first deploy.

## Changing a subdomain or port

Edit the `with:` block in the repo's workflow, then run a deploy. The pipeline
rewrites the compose file and the Caddy site config and reloads Caddy. The old
site config is overwritten, not orphaned.

Changing a subdomain leaves the old certificate in Caddy's store. Harmless; it
expires and is cleaned up.

## Removing an application

Actions → Deploy → Run workflow → `destroy`. This stops the containers, removes
the Caddy site config, reloads Caddy, and deletes `/srv/apps/<name>`. The
subdomain returns 404 afterwards. Images are cleaned up by the weekly prune.

## Rolling back

Runs against the server directly so it works when GitHub is down:

```bash
ssh -T -i ~/.ssh/gha_deploy deploy@<PROD_HOST> "rollback <app>"
```

Rollback restores the previous image tag, subdomain, and port together — the
server records all three on each deploy. There is one level of history, not a
stack.

---

## Server reference

| Path | Owner | Purpose |
|---|---|---|
| `/srv/caddy/docker-compose.yml` | `deploy` | Caddy stack. Only container with published ports |
| `/srv/caddy/Caddyfile` | `deploy` | Global config: ACME email, `:80` catch-all 404, `import sites/*.caddy` |
| `/srv/caddy/sites/<app>.caddy` | pipeline | Generated per app. Do not hand-edit — the next deploy overwrites it |
| `/srv/apps/<app>/docker-compose.yml` | pipeline | Generated per app. Same warning |
| `/srv/apps/<app>/.env` | pipeline | `IMAGE_TAG` only, mode 600 |
| `/srv/apps/<app>/.previous_tag` | pipeline | Rollback target |
| `/usr/local/bin/app-deploy` | `root` | The deploy script. Hand-maintained |
| `/var/log/app-deploy.log` | `deploy` | Every deploy, weekly rotation |

Docker network `edge` is external and shared. Both stacks declare
`networks: {edge: {external: true}}`.

### Deploy script command surface

The `deploy` user's SSH key is pinned to `/usr/local/bin/app-deploy` via a
forced command in `authorized_keys`, with `no-pty` and no forwarding. It accepts
only these:

```
deploy <app> <sha> <subdomain> <port>
rollback <app>
destroy <app>
status <app>
```

Validation, all of which are security controls rather than convenience checks:

| Argument | Pattern |
|---|---|
| `app` | `^[a-z0-9][a-z0-9-]{0,31}$` |
| `sha` | `^[0-9a-f]{40}$` |
| `subdomain` | `^[a-z0-9-]{1,63}\.nerry\.tech$` |
| `port` | `^[0-9]{2,5}$` |

Anything else exits non-zero before touching Docker. The `sha` and `subdomain`
patterns in particular prevent shell injection through `SSH_ORIGINAL_COMMAND`
and stop a repository claiming a hostname outside `nerry.tech`.

### Registry authentication

The server holds no long-lived registry credential. Each deploy pipes the
workflow's own `GITHUB_TOKEN` over stdin; the script uses it for
`docker login ghcr.io` and logs out on exit. The token expires with the job.

### Security boundaries worth knowing

- The `deploy` user is in the `docker` group, which is root-equivalent. The
  forced command is what limits the blast radius of a stolen deploy key — treat
  any change to `authorized_keys` as a security change.
- Published container ports bypass UFW entirely. A `DOCKER-USER` chain,
  reapplied by `docker-user-rules.service`, drops everything except 80 and 443.
  Adding a publicly published port means editing
  `/usr/local/sbin/docker-user-rules.sh` **and** the compose file.
- There is no cloud firewall yet. UFW and `DOCKER-USER` are the only
  enforcement. See [When to update this file](#when-to-update-this-file).

---

## Troubleshooting

Ordered by how often each has actually happened.

**`workflow was not found` when calling the reusable workflow.**
Usually the access setting, not the path. `nerry-labs/.github` → Settings →
Actions → General → Access → accessible from repositories in the organization.
Also confirm the file is at `.github/workflows/deploy.yml` *inside* the repo
named `.github` — the doubled path is correct.

**Job stuck queued.**
The `production` environment is waiting for approval; look for the Review
deployments banner. If there's no banner, the environment doesn't exist, or
Actions minutes are exhausted.

**`containers unhealthy, rolling back`.**
The healthcheck failed. The container is already gone, so reproduce by hand:

```bash
cd /srv/apps/<app>
docker compose up -d
docker inspect --format '{{range .State.Health.Log}}exit={{.ExitCode}} out={{.Output}}{{end}}' <app>-<app>-1
```

`Connection refused` from inside a running container is almost always the
IPv4/IPv6 split: busybox `wget` resolves `localhost` to `::1` first, and most
apps bind IPv4 only. Healthchecks use `127.0.0.1` for this reason. If you see
`localhost` in a generated compose file, the script template has regressed.

**403 pulling from GHCR.**
The `org.opencontainers.image.source` label is missing from the Dockerfile, so
the package isn't linked to the repo and the job's token has no access.

**Subdomain returns 404 with `server: cloudflare`.**
You're seeing Cloudflare, not the origin. Test the origin directly:

```bash
curl -sk --resolve <subdomain>:443:127.0.0.1 https://<subdomain>/
```

If the origin is fine, check Cloudflare SSL/TLS is **Full (strict)**. On
Flexible, Cloudflare connects to port 80, lands in the catch-all block, and
returns 404 regardless of configuration. If the origin also 404s, check
`/srv/caddy/sites/<app>.caddy` exists — a failed deploy removes it during
rollback.

**`PTY allocation request failed`.**
Expected. `no-pty` is set on the deploy key. Use `ssh -T`. The line after the
warning is the real output.

**Locked out of SSH.**
DO panel → Droplet → Access → Launch Droplet Console, log in as `root` with the
console password. `PermitRootLogin no` does not affect the console. Then:

```bash
rm -f /etc/ssh/sshd_config.d/00-hardening.conf
sshd -t && systemctl restart ssh
```

Diagnose which layer broke with `nc -vz <host> 22` from outside: `Connection
refused` means sshd is down; a timeout means a firewall.

---

## Rules for agents

If you are Claude Code or another agent working in an application repository:

1. **Read the repo's `CLAUDE.md` first.** It states that repo's subdomain, port,
   and health path. This document covers everything shared.
2. **Do not edit generated files.** Anything under `/srv/caddy/sites/` or
   `/srv/apps/*/` is rewritten on the next deploy. Changes belong in
   `/usr/local/bin/app-deploy` or in the repo's workflow inputs.
3. **Do not weaken the argument validation** in the deploy script. Those regexes
   are security controls.
4. **Do not add published ports** to an application compose file. Apps reach the
   internet through Caddy on the `edge` network only.
5. **Do not commit secrets.** The four `PROD_*` values are organization secrets.
6. **Changing `subdomain` or `port`** is a workflow-input edit plus a redeploy.
   Never hand-edit a Caddy site file to achieve it.
7. **If a change doesn't fit the contract above**, stop and say so rather than
   working around it. Working around the contract is how the next person loses a
   day.

---

## When to update this file

Update `devops.md` when any of these change. This list is the boundary between
what belongs here and what belongs in a repo.

**Always update:**

- The deploy script's command surface or argument validation
- Server directory layout, the `edge` network, or the Caddy stack
- The reusable workflow's inputs, or the org secrets it consumes
- Anything in the six-item deploy contract
- Registry, authentication, or the `deploy` user's restrictions
- Firewall posture — notably adding the DigitalOcean cloud firewall, which is
  currently deferred
- A second server, or anything that makes "the droplet" ambiguous
- A troubleshooting entry, once an issue has cost someone more than an hour

**Never put here** (these belong in the app repo's `CLAUDE.md` or workflow):

- A specific app's subdomain, port, or health path
- Application build steps, dependencies, or framework detail
- Anything true of one repository only

**Process.** Change the server and this file in the same session; a server
change that isn't written down here is the failure mode this document exists to
prevent. Open a PR against `nerry-labs/.github` rather than pushing to `main`,
since every repo's agent reads this. Note the date and what changed at the
bottom.

---

## Access inventory

Review quarterly. Rotate on any departure.

| Credential | Location | Used by | Rotate |
|---|---|---|---|
| `do_app_ops` SSH key | Individual laptops | Humans, as `ops` | On departure |
| `gha_deploy` SSH key | Org secret `PROD_SSH_KEY`, copy in password manager | CI, and manual rollback | Quarterly |
| Droplet console password | Password manager | Lockout recovery | On departure |
| DigitalOcean account | Password manager | Infrastructure changes | — |

The password-manager copy of `gha_deploy` is deliberate: rollback must work when
GitHub does not.

---

## Known gaps

Honest list. None are blocking, all are worth revisiting.

- **Single droplet, no HA.** Any deploy or reboot is downtime. Mitigation is
  weekly DO backups plus a rebuild, not redundancy.
- **No cloud firewall.** UFW and `DOCKER-USER` are the only enforcement, so a
  `ufw disable` left in place has no backstop.
- **No restore drill has been run.** The recovery time is currently a guess.
- **One deploy key for all apps.** Adding repos widens the blast radius of that
  key. Per-app keys and users are the upgrade when it matters.
- **`@main` on the reusable workflow.** Anyone who can push to `.github` changes
  what every repo deploys with. Tag and pin once it's stable.

---

*Last updated: 2026-09-16 — initial version.*
