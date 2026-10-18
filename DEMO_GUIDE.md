# Deployment and Demonstration Guide

**Repository:** [selimanaw/DefectDojo](https://github.com/selimanaw/DefectDojo) · branch `dev`

This guide covers three routes to a running DefectDojo instance that demonstrates the Phase 1 and
Phase 2 improvements. Read the comparison table first, then follow the section that matches your situation.

---

## Comparison: Three Deployment Options

| | Option A — Local Docker Compose | Option B — ngrok Tunnel | Option C — Railway |
|---|---|---|---|
| **Setup time** | ~10 min | ~15 min | ~25 min |
| **URL type** | `localhost:8080` | `https://<id>.ngrok-free.app` | `https://<name>.up.railway.app` |
| **URL lifetime** | As long as Docker runs | As long as the ngrok session runs | Persistent |
| **Internet required** | For first build only | Yes | Yes |
| **Cost** | Free | Free (ngrok basic) | Free (Starter plan) |
| **Best for** | Local testing and running tests | Sharing a live URL during a demo | Persistent judge access |
| **Resource requirements** | 8 GB RAM, 20 GB disk | Same as Option A + ngrok account | Railway cloud (no local resource) |
| **Data persistence** | Local Docker volume | Local Docker volume | PostgreSQL addon |
| **Risk** | None | Low | Low |

---

## Option A — Local Docker Compose (recommended for testing)

### Prerequisites

- Docker Desktop ≥ 4.x with WSL2 backend (Windows) or Docker Engine (Linux/macOS)
- ~8 GB free RAM
- ~20 GB free disk space (images + database)
- Git

### Steps

```bash
# 1. Clone the fork
git clone https://github.com/selimanaw/DefectDojo.git
cd DefectDojo

# 2. Verify you are on dev and that both PRs are present
git log --oneline -5
# Expected output (most recent first):
#   44257e481f Merge pull request #2 from selimanaw/perf/bulk-import-persist
#   aa96dfb74b fix(tests): update ZAP import query-count baselines...
#   3e172517e5 WIP: preserve Phase 2 bulk import optimization
#   083b85e7f2 Merge pull request #1 from selimanaw/fix/hash-correctness
#   5ed6aafd13 fix(dedup): fix bytes.strip() ordering...

# 3. Start the full stack
#    First run builds images — allow 5–10 minutes.
#    Subsequent starts take ~30 seconds.
docker compose up -d

# 4. Watch the initializer finish
docker compose logs -f initializer
# Wait for: "Admin password: <password>"  (Ctrl-C to stop following)

# 5. Confirm all services are healthy
docker compose ps
# All six services should show status "running" or "exited 0" (initializer exits after setup)

# 6. Open the application
#    URL:      http://localhost:8080
#    Username: admin
#    Password: <from step 4>
```

### Services started by `docker compose up`

| Service | Role | Port |
|---|---|---|
| `nginx` | Reverse proxy, static files | 8080 (HTTP), 8443 (HTTPS) |
| `uwsgi` | Django application server | internal |
| `celeryworker` | Background task executor | internal |
| `celerybeat` | Periodic task scheduler | internal |
| `postgres` | PostgreSQL 14 database | internal |
| `valkey` | Redis-compatible broker/cache | internal |

### Stopping and cleaning up

```bash
# Stop containers, preserve data
docker compose down

# Stop containers AND delete all data (clean slate)
docker compose down -v
```

### Running the regression and performance tests

The unit-test compose override mounts the source tree and runs tests against a test database.

```bash
# Phase 1: hash normalisation fix
docker compose -f docker-compose.override.unit_tests.yml run --rm uwsgi \
  python manage.py test \
    unittests.test_deduplication_logic.TestHashFieldsNormalisation \
    unittests.test_deduplication_logic.TestLegacyFallbackWarnings \
  -v 2

# Phase 2: bulk-persist regression (19 tests + 1 skipped)
docker compose -f docker-compose.override.unit_tests.yml run --rm uwsgi \
  python manage.py test unittests.test_bulk_persist_new_findings -v 2

# Phase 2: ZAP query-count baselines (confirms 269/295 numbers)
docker compose -f docker-compose.override.unit_tests.yml run --rm uwsgi \
  python manage.py test \
    unittests.test_tag_inheritance_perf \
    unittests.test_importers_performance \
  -v 2 --tag performance
```

Alternatively, use the project's shell convenience script (runs inside a fresh container automatically):

```bash
./run-unittest.sh --test-case unittests.test_deduplication_logic.TestHashFieldsNormalisation
./run-unittest.sh --test-case unittests.test_bulk_persist_new_findings
./run-unittest.sh --test-case unittests.test_tag_inheritance_perf
```

---

## Option B — ngrok Tunnel (fastest public URL)

ngrok creates an HTTPS tunnel from a public subdomain to your local Docker Compose instance.
Use this when you need a shareable URL quickly for a judge or reviewer.

### Prerequisites

- Option A running (`docker compose up -d`)
- A free [ngrok account](https://ngrok.com) (required for stable URLs and HTTPS)
- ngrok CLI installed: `winget install ngrok` (Windows) or `brew install ngrok` (macOS)

### Steps

```bash
# 1. Authenticate the ngrok CLI (one-time setup, token from ngrok dashboard)
ngrok config add-authtoken <YOUR_AUTHTOKEN>

# 2. Start the tunnel pointing at the local nginx port
ngrok http 8080

# Output will include a line like:
#   Forwarding  https://abc123de.ngrok-free.app -> http://localhost:8080
```

The URL `https://abc123de.ngrok-free.app` is now publicly accessible for as long as the ngrok process
runs. Share it with judges. The application uses the same admin credentials as Option A.

### Important limitation

The free ngrok URL changes every time you restart the ngrok process. If you need a **persistent** URL,
use Option C, or upgrade to an ngrok paid plan with a reserved domain.

### Tip: set AllowedHosts for the tunnel URL

DefectDojo defaults `DD_ALLOWED_HOSTS=*` in the docker-compose environment, so no extra configuration
is needed for the ngrok URL to work.

---

## Option C — Railway (persistent hosted URL)

Railway can deploy DefectDojo directly from your GitHub fork using the included `docker-compose.yml`.
This gives judges a stable URL that survives between sessions.

### Prerequisites

- A [Railway account](https://railway.app) (free Starter plan)
- GitHub account connected to Railway

### Caveats verified against this repository

The `docker-compose.yml` in this fork defines six services. Railway's Docker Compose support handles
multi-service files, but the free Starter plan has a 512 MB RAM limit per service and a $5 monthly
credit. A full DefectDojo stack (postgres + valkey + celery + uwsgi + nginx) will exceed the free
tier resources. **Use Railway only if you have a Trial or Pro plan, or if you want to demonstrate
the concept for judges at a higher level.**

A more practical Railway approach is to deploy only the Django application + managed PostgreSQL:

### Recommended Railway approach

```
1. Go to railway.app → New Project → Deploy from GitHub repo
2. Select: selimanaw/DefectDojo, branch: dev
3. Railway detects Dockerfile.django-debian automatically
4. Add a PostgreSQL plugin (Railway managed)
5. Set environment variables:
     DD_DATABASE_URL       = ${{Postgres.DATABASE_URL}}
     DD_SECRET_KEY         = <generate a random 50-char string>
     DD_ALLOWED_HOSTS      = *
     DJANGO_SUPERUSER_PASS = <choose a password>
     DD_INITIALIZE         = True
6. Deploy. First deploy takes ~8 minutes.
7. Railway assigns a URL: https://defectdojo-fork.up.railway.app
```

Because the Celery workers are not deployed separately in this minimal setup, asynchronous tasks
(deduplication, notifications) will run synchronously via `CELERY_TASK_ALWAYS_EAGER=True`. This is
sufficient for a demonstration but is not production-equivalent.

---

## Demonstrating the Phase 1 and Phase 2 improvements

### What to show for Phase 1

1. Open `dojo/finding/models.py` and show line `compute_hash_code()` — the `.casefold().strip().encode("utf-8")` order.
2. Run the regression test live:
   ```bash
   ./run-unittest.sh --test-case unittests.test_deduplication_logic.TestHashFieldsNormalisation
   ```
   Nine tests pass, including the pinned regression value and three Unicode whitespace variants.
3. Open `dojo/finding/deduplication.py` and show the `WARNING`-level log message with scan-type interpolation.

### What to show for Phase 2

1. Open `dojo/importers/base_importer.py` — show the `bulk_create()` block inside `persist_new_findings()`.
2. Run the query-count test live:
   ```bash
   ./run-unittest.sh --test-case unittests.test_tag_inheritance_perf
   ```
   The test asserts `EXPECTED_ZAP_IMPORT_V2 = 269` and `EXPECTED_ZAP_IMPORT_V3 = 295`. It passes,
   confirming the 20-query reduction.
3. To show the before/after contrast, display the original commit diff:
   ```bash
   git show 3e172517e5 -- dojo/importers/base_importer.py
   ```

### Importing a real scan for a live demo

Sample scan files are included in `unittests/scans/`. To demonstrate a ZAP import:

1. Log into the running DefectDojo (`http://localhost:8080`, credentials from `docker compose logs initializer`).
2. Create a Product → Create an Engagement.
3. Go to **Import Scan** → select **ZAP Scan** as scan type.
4. Upload `unittests/scans/zap/0_zap_sample.xml` (or any file from `unittests/scans/zap/`).
5. The findings appear immediately. Show the deduplication settings in System Settings → Deduplication.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `docker compose up` fails immediately | Docker not running | Start Docker Desktop |
| Port 8080 already in use | Another service on that port | `docker compose down` or change `DD_PORT` in `.env` |
| Initializer exits with error | Postgres not ready | Wait 30 sec and re-run `docker compose up -d` |
| Admin password not in logs | Initializer already ran (data volume exists) | `docker compose down -v` then `up -d` again |
| ngrok shows `ERR_NGROK_3200` | Not authenticated | Run `ngrok config add-authtoken <token>` |
| Railway deploy fails OOM | Free plan RAM limit | Upgrade to Trial plan or use Option A + ngrok |
