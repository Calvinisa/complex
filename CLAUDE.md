# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture

A multi-container Fibonacci calculator. Four app containers plus Postgres and Redis:

- **client/** — Create React App (react-scripts 1.1.4, React 16, react-router 4). `Fib.js` is the only meaningful component; it POSTs an index to `/api/values` and polls `/api/values/current` and `/api/values/all`.
- **server/** — Express API on port 5000. Writes the submitted index to Postgres (`values` table, created on boot via `CREATE TABLE IF NOT EXISTS`), seeds Redis with `'Nothing yet!'` for that index, and publishes the index on the Redis `insert` channel. Hard-rejects `index > 40` with HTTP 422 to prevent runaway recursion in the worker.
- **worker/** — Subscribes to the Redis `insert` channel, computes `fib(index)` with the naive recursive implementation, and writes the result back to the `values` Redis hash. There is no queue and no concurrency control — duplicate submissions recompute.
- **nginx/** — Reverse proxy on port 80 (mapped to host 3050 in dev). `/` → `client:3000`, `/api/*` → `api:5000` (the `/api` prefix is stripped via `rewrite /api(.*) /$1 break`), and `/sockjs-node` is proxied with WebSocket upgrade headers so CRA hot-reload works through the proxy.

Two Redis clients are used per service: one for normal commands and a `.duplicate()` for pub/sub (node-redis requires this — a subscribed connection can't issue other commands).

`server/keys.js` and `worker/keys.js` are identical and read all config from env vars. There are no defaults — the containers will fail at startup if the docker-compose env block is missing or the prod environment doesn't set these.

## Dev vs. prod split

Each app dir has both `Dockerfile.dev` and `Dockerfile`:

- **Dockerfile.dev** — Single-stage `node:alpine`, runs `npm run dev` (nodemon) or `npm run start` (CRA). Used by `docker-compose.yml` with bind mounts (`./server:/app` etc.) and an anonymous `/app/node_modules` volume to preserve container-installed deps. The client nginx is not used in dev — CRA's dev server serves the client directly on port 3000 and the top-level nginx proxies to it.
- **Dockerfile** (prod) — For the client, a multi-stage build that compiles with `npm run build` and serves the static bundle via its own nginx (`client/nginx/default.conf`, listening on 3000, with SPA `try_files` fallback). For server/worker, a plain `npm run start`. These prod images are pushed to Docker Hub as `calvinisa/multi-{client,server,worker,nginx}` and consumed by `Dockerrun.aws.json` on Elastic Beanstalk. Postgres and Redis are intentionally absent from `Dockerrun.aws.json` — prod expects external managed services (RDS/ElastiCache) reached via the env vars.

When editing the client's prod nginx config, edit `client/nginx/default.conf` (baked into the client image), not the top-level `nginx/default.conf` (the routing proxy). They serve different roles.

## Common commands

Run everything locally (from repo root):

```bash
docker-compose up --build      # first run / after Dockerfile or package.json changes
docker-compose up              # subsequent runs
```

App is at `http://localhost:3050`. Source edits in `client/`, `server/`, `worker/` hot-reload via the bind mounts; changes to `package.json` or any `Dockerfile.dev` require `--build`.

Run the client test suite the same way CI does:

```bash
docker build -t calvinisa/react-test -f ./client/Dockerfile.dev ./client
docker run -e CI=true calvinisa/react-test npm test
```

Or interactively against a running stack: `docker exec -it <client-container> npm test`. There are no tests in `server/` or `worker/`; `client/src/App.test.js` is an empty smoke test.

Build the prod images locally (matches the `after_success` step in `.travis.yml`):

```bash
docker build -t calvinisa/multi-client ./client
docker build -t calvinisa/multi-nginx  ./nginx
docker build -t calvinisa/multi-server ./server
docker build -t calvinisa/multi-worker ./worker
```

## CI/CD

`.travis.yml` builds the client dev image, runs its tests with `CI=true`, then on success builds and pushes all four prod images to Docker Hub under `calvinisa/`. The Elastic Beanstalk deploy block is currently commented out, so deploys are manual: upload `Dockerrun.aws.json` to an EB multi-container Docker environment that already has RDS Postgres and ElastiCache Redis attached and the `PG*`/`REDIS_*` env vars configured.
