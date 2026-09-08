# Apache Superset on Railway

Apache Superset — open-source, self-hosted business intelligence and data visualization platform. SQL-based exploration, dashboards, and scheduled reports, proven at real production scale. Deploy on Railway with one click.

## Architecture

This template deploys the same 5-service architecture as Railway's existing reference template
(pulled directly via its `serializedConfig`, then cross-checked against Superset's own official
`docker-compose-non-dev.yml`, `docker-init.sh`, and `docker-bootstrap.sh`):

- **superset** — the web app (`apache/superset:latest`): runs migrations + admin setup on boot, then serves gunicorn
- **superset-worker** — Celery worker (`apache/superset:latest`): background jobs, chart thumbnails
- **superset-beat** — Celery scheduler (`apache/superset:latest`): scheduled alerts and reports
- **postgres** — metadata store (`ghcr.io/railwayapp-templates/postgres-ssl:18`)
- **redis** — Celery broker + cache (`redis:8.2.1`)

## How to use

Before deploying, set `ADMIN_PASSWORD` to a real generated value — the reference config's own
default value is a broken placeholder, not something to use literally. Once deployed, open your
domain (redirects to `/login/`) and log in as `admin` with that password.

## Notes

- **Real bug fixed here that the reference template doesn't handle**: `superset-worker` and
  `superset-beat` both crash-loop on `apache/superset:latest` with `ModuleNotFoundError: No module
  named 'psycopg2'`. Superset's own `docker-bootstrap.sh` explicitly skips installing the Postgres
  driver for worker/beat processes, assuming it's already present — it isn't, on this published
  image tag. Fixed by installing `psycopg2-binary` explicitly in both start commands before
  invoking Superset's bootstrap script. Confirmed live: worker logs `celery@... ready.`, beat logs
  `beat: Starting...`.
- Only the `superset` (web) service runs `docker-init.sh` (migrations + admin setup). Worker and
  beat rely on that having completed — their `ON_FAILURE` restart policy (max 10 retries) makes
  this self-healing on a simultaneous first deploy rather than a permanent crash loop.
- Postgres uses `PGDATA=/var/lib/postgresql/data/pgdata` (a subdirectory under the mount), the
  same recurring fix as every other Postgres-backed template in this series.
- Data persists across redeploys via 2 volumes: Postgres (all dashboards, charts, queries) and
  Redis (cache, Celery queue).

See `TEMPLATE_COMPOSER_CHECKLIST.md`, `TEMPLATE_DESCRIPTION.md`, and `TEMPLATE_BLOG.md` in this
repo for the full build documentation.
