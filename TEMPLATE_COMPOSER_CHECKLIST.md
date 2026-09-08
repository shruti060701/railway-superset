# Railway Template Composer Checklist — Apache Superset

Built by pulling the existing Railway reference template's `serializedConfig` directly (via the
`template(code: "superset")` GraphQL query) rather than guessing from docs, then cross-verified
against Superset's own official `docker-compose-non-dev.yml`, `docker-init.sh`, and
`docker-bootstrap.sh` (fetched live from `github.com/apache/superset`). Built live, deployed, and
one real bug was found and fixed that the reference config doesn't handle.

---

## 1. Healthcheck Settings

### `superset`
Leave blank in the composer — the reference config doesn't set one either, and `/health` works
fine as a manual check but wasn't wired as a Railway healthcheck upstream. (Verified live: `/health`
returns `OK`.)

### `postgres`, `redis`, `superset-worker`, `superset-beat`
Leave blank — no HTTP endpoint on any of them (worker/beat are Celery processes, not web servers).

---

## 2. Custom Start Commands

### `postgres`
None — this uses Railway's own maintained `ghcr.io/railwayapp-templates/postgres-ssl:18` image
(not raw `postgres:*-alpine` like other templates in this series), which handles its own volume
init. `PGDATA` is still set to a subdirectory of the mount as a matter of consistency.

### `redis`
```
/bin/sh -c "rm -rf $RAILWAY_VOLUME_MOUNT_PATH/lost+found/ && exec docker-entrypoint.sh redis-server --requirepass $REDIS_PASSWORD --save 60 1 --dir $RAILWAY_VOLUME_MOUNT_PATH"
```
Same recurring pattern as every Redis-backed template in this series.

### `superset` (the web/gunicorn service)
```
/bin/sh -c "mkdir -p /app/pythonpath && echo \"$SUPERSET_CONFIG_B64\" | base64 -d > /app/pythonpath/superset_config.py && /app/docker/docker-init.sh && /app/docker/docker-bootstrap.sh app-gunicorn"
```
Decodes a base64-encoded `superset_config.py` into place (see §3 for what it contains), runs
`docker-init.sh` (DB migrations, admin user creation, role/permission setup — Superset's own
official init script), then starts the gunicorn web server. **Only this service runs
`docker-init.sh`** — worker and beat below skip it and rely on the web service having already run
migrations.

### `superset-worker` (Celery worker) — fixed from the reference
```
/bin/sh -c "mkdir -p /app/pythonpath && echo \"$SUPERSET_CONFIG_B64\" | base64 -d > /app/pythonpath/superset_config.py && (command -v uv >/dev/null 2>&1 && uv pip install --no-cache-dir psycopg2-binary || pip install --no-cache-dir psycopg2-binary) && /app/docker/docker-bootstrap.sh worker"
```

### `superset-beat` (Celery scheduler) — fixed from the reference
```
/bin/sh -c "mkdir -p /app/pythonpath && echo \"$SUPERSET_CONFIG_B64\" | base64 -d > /app/pythonpath/superset_config.py && (command -v uv >/dev/null 2>&1 && uv pip install --no-cache-dir psycopg2-binary || pip install --no-cache-dir psycopg2-binary) && rm -f /tmp/celerybeat.pid && /app/docker/docker-bootstrap.sh beat"
```
See §3 for why the `psycopg2-binary` install step was added — it's not in the reference template
and both processes crash-looped without it.

### Restart policy on `superset`, `superset-worker`, `superset-beat`
`ON_FAILURE`, max 10 retries (matches the reference). This matters specifically for worker/beat:
they can start before the web service's `docker-init.sh` has finished running migrations, and will
fail fast on a missing-schema error until the web service catches up. The retry policy is what
makes this self-healing rather than a permanent crash loop — deploy web first if you want a clean
log trail while testing, but a real marketplace deploy (all 5 services starting close together)
relies on this retry behavior.

---

## 3. Known Real Bugs Hit During Build

- **`superset-worker` and `superset-beat` crash-loop on every boot with `ModuleNotFoundError: No
  module named 'psycopg2'`** — confirmed live via full traceback. Superset's own
  `docker-bootstrap.sh` explicitly skips installing Postgres driver requirements for `worker` and
  `beat` processes ("Skip postgres requirements installation for workers to avoid conflicts"),
  installing `psycopg2` only for the main `app`/`app-gunicorn` process. This assumption holds in
  Superset's own local Docker Compose setup, where all services are built from the same
  locally-built image and (depending on build layering) may already have it baked in — but the
  published `apache/superset:latest` tag does **not** ship with `psycopg2` pre-installed, so any
  deployment running worker/beat as genuinely separate service instances against that tag (Railway
  included) hits this immediately. **Fixed at the infrastructure level**: both start commands now
  explicitly install `psycopg2-binary` (the prebuilt-wheel package, no compiler needed) before
  invoking `docker-bootstrap.sh`, using `uv` if present (matching the base image's own installer
  preference) and falling back to plain `pip` otherwise. Confirmed live: worker logs
  `celery@... ready.` connected to Redis; beat logs `beat: Starting...`. The existing Railway
  reference template does **not** have this fix — it's unclear whether that's because an older
  `apache/superset:latest` build once included `psycopg2`, or because it was never fully verified
  for worker/beat health. Either way, this build's version is the one to use.
- **Postgres needs `PGDATA` set to a subdirectory of the mount**, same recurring pattern as every
  other Postgres-backed template in this series, though `ghcr.io/railwayapp-templates/postgres-ssl`
  may already handle this internally — set explicitly anyway for consistency and safety.
- **`superset-worker`/`superset-beat` will show transient crash-loop restarts on a truly
  simultaneous 5-service first deploy** (not a bug — see restart policy note in §2). If testing
  manually service-by-service, deploy `postgres` and `redis` first, then `superset` alone, confirm
  its `docker-init.sh` has completed (logs show `Init Step 3/3 [Complete]`), *then* deploy
  worker/beat, to see clean logs without the expected transient failures.

---

## 4. Variable Descriptions (Add to EVERY variable)

Pulled from the live services via `railway variables --json`, matching the reference's variable
names and reference expressions exactly except where noted.

### `postgres` — 10 total
| Variable | Value | Optional? | Description |
|---|---|---|---|
| `POSTGRES_USER` | `postgres` | No | Default Postgres superuser name. |
| `POSTGRES_PASSWORD` | `${{secret(32)}}` | No | Postgres database user password. Auto-generated. |
| `POSTGRES_DB` | `railway` | No | Initial database created on startup. |
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | No | Data storage directory — a subdirectory of the mount, not the mount root. |
| `PGUSER` | `${{POSTGRES_USER}}` | No | Default Postgres username, exposed under the conventional `PG*` naming. |
| `PGPASSWORD` | `${{POSTGRES_PASSWORD}}` | No | Password for the Postgres user. |
| `PGDATABASE` | `${{POSTGRES_DB}}` | No | Default database name. |
| `PGHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | No | Internal Postgres host address. |
| `PGPORT` | `5432` | No | Postgres server listening port. |
| `DATABASE_URL` | `postgresql://${{PGUSER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{PGDATABASE}}` | No | Internal Postgres connection string — this is what all 3 Superset services reference for their metadata DB. |

**Also present in the reference (not set on this build — optional extras):**
`DATABASE_PUBLIC_URL` (external connection string via TCP proxy, only needed for connecting from
outside Railway), `SSL_CERT_DAYS` (self-signed cert validity, specific to the `postgres-ssl` image),
`RAILWAY_DEPLOYMENT_DRAINING_SECONDS` (graceful shutdown timing).

### `redis` — 6 total
| Variable | Value | Optional? | Description |
|---|---|---|---|
| `REDIS_PASSWORD` | `${{secret(32)}}` | No | Auth password Redis is started with (`--requirepass`). Auto-generated. |
| `REDISHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | No | Internal Redis service hostname. |
| `REDISPASSWORD` | `${{REDIS_PASSWORD}}` | No | Same password under Railway's conventional Redis variable naming. |
| `REDISPORT` | `6379` | No | Redis server listening port. |
| `REDISUSER` | `default` | No | Redis default authentication user (plain `--requirepass`, not ACLs). |
| `REDIS_URL` | `redis://${{REDISUSER}}:${{REDIS_PASSWORD}}@${{REDISHOST}}:${{REDISPORT}}` | No | Internal Redis connection string — used as the Celery broker/result backend AND the Superset cache backend (4 separate logical DB indices, see §3's decoded config). |

### `superset` (web) — 12 total
| Variable | Value | Optional? | Description |
|---|---|---|---|
| `ADMIN_PASSWORD` | Generate a real password (the reference's own default value is a broken placeholder, not usable literally) | No | Bootstrap admin password. Username is always `admin`. Save it — it's how you log in after deploying. |
| `DATABASE_DIALECT` | `postgresql` | No | Triggers `psycopg2` install for the web process specifically (see §3 for why worker/beat needed a separate fix). |
| `DATABASE_URL` | `${{postgres.DATABASE_URL}}` | No | Metadata DB connection string. |
| `DEV_MODE` | `false` | No | Disables the dev-mode editable install. |
| `PYTHONPATH` | `/app/pythonpath` | No | Where the decoded `superset_config.py` gets placed and picked up. |
| `RAILWAY_RUN_UID` | `0` | No | Runs the container as root — required for the runtime `pip`/`uv pip install` steps in the start commands to have write access. |
| `REDIS_URL` | `${{redis.REDIS_URL}}` | No | Redis broker + cache URL, referenced inside the decoded config. |
| `SUPERSET_CONFIG_B64` | See §5 below for the decoded content | No | Base64-encoded `superset_config.py` — wires `DATABASE_URL`/`REDIS_URL` into Superset's actual config format, sets Celery, cache, feature flags, and cookie security settings. |
| `SUPERSET_ENV` | `production` | No | Production mode flag. |
| `SUPERSET_LOAD_EXAMPLES` | `no` | No | Skip loading example dashboards on init. |
| `SUPERSET_PORT` | `8088` | No | Internal gunicorn port — also what the service's public domain should target. |
| `SUPERSET_SECRET_KEY` | `${{secret(64)}}` | No | Flask session signing key. Auto-generate — do not reuse across deployments. |

### `superset-worker`, `superset-beat` — 12 total each
Identical variable set to `superset` (web), except every value is a **reference back to the web
service** instead of an independent value, so all 3 processes share the same config, secret key,
and admin credentials:

| Variable | Value |
|---|---|
| `ADMIN_PASSWORD` | `${{superset.ADMIN_PASSWORD}}` |
| `DATABASE_URL` | `${{postgres.DATABASE_URL}}` |
| `REDIS_URL` | `${{redis.REDIS_URL}}` |
| `SUPERSET_CONFIG_B64` | `${{superset.SUPERSET_CONFIG_B64}}` |
| `SUPERSET_SECRET_KEY` | `${{superset.SUPERSET_SECRET_KEY}}` |
| `DATABASE_DIALECT`, `DEV_MODE`, `PYTHONPATH`, `RAILWAY_RUN_UID`, `SUPERSET_ENV`, `SUPERSET_LOAD_EXAMPLES`, `SUPERSET_PORT` | Same literal values as `superset` (web) — `postgresql`, `false`, `/app/pythonpath`, `0`, `production`, `no`, `8088` |

---

## 5. Secrets That Must Use `${{secret()}}`

- `POSTGRES_PASSWORD` — `${{secret(32)}}`
- `REDIS_PASSWORD` — `${{secret(32)}}`
- `SUPERSET_SECRET_KEY` — `${{secret(64)}}`
- `ADMIN_PASSWORD` — generate a real password; do not use the reference's literal `$` default
  value, which is a broken placeholder, not a working generator expression.

### `SUPERSET_CONFIG_B64` decoded content
```python
import os
SQLALCHEMY_DATABASE_URI = os.environ.get("DATABASE_URL")
r = os.environ.get("REDIS_URL", "redis://localhost:6379").rstrip("/")
class CeleryConfig:
    broker_url = r + "/0"
    result_backend = r + "/1"
CELERY_CONFIG = CeleryConfig
CACHE_CONFIG = {"CACHE_TYPE": "RedisCache", "CACHE_REDIS_URL": r + "/2", "CACHE_DEFAULT_TIMEOUT": 300, "CACHE_KEY_PREFIX": "superset_"}
DATA_CACHE_CONFIG = {"CACHE_TYPE": "RedisCache", "CACHE_REDIS_URL": r + "/3", "CACHE_DEFAULT_TIMEOUT": 300, "CACHE_KEY_PREFIX": "superset_data_"}
FEATURE_FLAGS = {"ALERT_REPORTS": True, "DASHBOARD_RBAC": True, "EMBEDDED_SUPERSET": True}
TALISMAN_ENABLED = False
WTF_CSRF_ENABLED = True
WTF_CSRF_TIME_LIMIT = None
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_SAMESITE = "Lax"
ENABLE_PROXY_FIX = True
```
Redis DB indices 0-3 separate Celery broker, Celery results, general cache, and data/query cache.
If you ever need to change this config, re-encode with `base64 -w0 superset_config.py` (or
`base64 -i superset_config.py` on macOS) and update `SUPERSET_CONFIG_B64` on the `superset` (web)
service — worker and beat both reference it, so one update propagates everywhere.

---

## 6. Volumes

| Service | Mount Path |
|---|---|
| `postgres` | `/var/lib/postgresql/data` (with `PGDATA` pointed at a subdirectory) |
| `redis` | `/data` |

`superset`, `superset-worker`, `superset-beat` need no volumes — all persistent state lives in
Postgres (metadata, dashboards, charts) and Redis (cache, Celery queue).

---

## 7. Post-Deploy Steps

1. Open the deployed domain — redirects to `/login/`.
2. Log in with username `admin` and the password set in `ADMIN_PASSWORD`.
3. Connect a data source (Settings → Database Connections) to start building dashboards.
