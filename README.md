# carle7-edge

The single reverse proxy for everything on carle7.com. It's the only container on the host that
binds ports 80/443 or terminates TLS; every app running on this host (currently
[energytracker](https://github.com/mattcarle/energytracker)) runs its own backend + static-file
Caddy behind this one, reachable only on the shared `carle7-edge` Docker network, and gets a path
prefix here (e.g. `/energytracker/`). The bare domain (`/`) serves a small landing page
(`site/index.html`) linking to whichever apps are deployed.

## Why a separate repo

Each app stays self-contained and independently deployable - it never needs to know it's sharing
a domain. This repo is the only thing that needs to know the full list of apps and their path
prefixes.

## Deploying

1. Copy `.env.example` to `.env` and set `SITE_ADDRESS` to your real domain (DNS must already
   point at this host, with ports 80/443 reachable from the internet, for Caddy to obtain a
   Let's Encrypt certificate). Leave it unset/`localhost` for local testing only.
2. Bring this stack up **first** - it creates the `carle7-edge` Docker network that every app
   attaches to. Any app's `docker compose up` fails if this network doesn't exist yet.

   ```bash
   docker compose up -d
   ```
3. Bring up each app's own stack (see that app's own README/deployment scripts).

## Adding a new app

Once the app has its own repo, backend container, and a Caddy/static-file container that joins
the `carle7-edge` network under a network alias (see energytracker's `docker-compose.yml` for the
pattern), add it here:

1. In `Caddyfile`, add a `handle /<name>` (redirect to trailing slash) + `handle /<name>/*`
   (`reverse_proxy <name>:80`) pair, using that app's network alias.
2. Add a link to `site/index.html`.
3. `docker compose up -d` (no rebuild needed for the Caddyfile/site change - Caddy reloads from
   the bind-mounted files; `docker compose restart caddy` if it doesn't pick it up automatically).
