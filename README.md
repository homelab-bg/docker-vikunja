# docker-vikunja

A [Vikunja](https://vikunja.io) task tracker, single all-in-one container
(API + frontend), SQLite.

This repo is public - `.env.example` uses placeholder values (a generic
domain) deliberately. Real values only ever go in your own gitignored `.env`.

## Why these choices

- **SQLite, not Postgres/MySQL** - Vikunja officially supports it, and this
  is a homelab-scale task tracker for a handful of users; running (and
  backing up) a separate database container isn't worth it at this scale.
  Switching to Postgres later is a `VIKUNJA_DATABASE_*` env var change plus
  adding a `db` service, not a rebuild of this repo.
- **The official `vikunja/vikunja` image**, not a custom build - unlike
  `docker-postfix`, nothing here needs wiring the upstream image doesn't
  already provide.
- **`VIKUNJA_SERVICE_SECRET` as a required env var, not a Docker secret** -
  it's an internal JWT-signing secret for Vikunja's own sessions, not a
  credential granting access to an external account/service (contrast
  `docker-postfix`'s AWS keys), and it's unconfirmed whether Vikunja's config
  loader supports reading secrets from a file at all. A required var sourced
  from the gitignored `.env` is simple and matches the actual risk here.
- **Named volumes for `/db` and `/app/vikunja/files`**, not bind mounts -
  both are opaque app state (SQLite file, uploaded attachments), not
  hand-edited config - same reasoning as `docker-postfix`'s `acme_data`.
- **Self-registration disabled by default** (`VIKUNJA_SERVICE_ENABLEREGISTRATION=false`)
  - sane default for an internal tool with a known, small user set; not
  that this host is internet-reachable either way (see the project's
  Traefik-is-LAN-only note).

## Setup

```sh
cp .env.example .env
# edit .env: VIKUNJA_PUBLICURL, VIKUNJA_SERVICE_SECRET at minimum
docker compose up -d
```

Generate the secret with:

```sh
openssl rand -hex 32
```

First user to register (before disabling registration, or via the CLI -
see Vikunja's own docs) becomes the only account unless more are added.

### Traefik (optional)

To put Vikunja behind the shared Traefik instance with a real HTTPS
hostname instead of the bare `:3456` port:

```sh
docker compose -f docker-compose.yml -f docker-compose.traefik.yml up -d
```

Requires `VIKUNJA_HOST` (matching the host in `VIKUNJA_PUBLICURL`) and the
external `traefik` network (`NETWORK` in `.env`) to already exist - see
`docker-traefik-portainer`.

## Observability

Deploy the [`docker-grafana-stack`](https://github.com/homelab-bg/docker-grafana-stack)
Alloy agent alongside this container (agent-only profile) if it isn't already
running on this host - `STACK_NAME` above is already wired for it.

## Known limitations / not yet done

- Not yet deployed or tested live - compose files and config are built and
  reasoned through against Vikunja's own docs (confirmed current, real
  `vikunja/vikunja:2.8.0` tag), but `docker compose up -d` hasn't actually
  been run anywhere yet.
- No GitHub remote yet - working from this local repo until one's created.
