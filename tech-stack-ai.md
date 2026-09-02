# Tech Stack (AI)

## 1. Run Questions

### 1a. Config Files

| Config File | Location | Config Value | What it's for | How it's used |
|---|---|---|---|---|
| .env.template | learn-ops-api/.env.template | LEARN_OPS_DJANGO_SECRET_KEY | Django's cryptographic signing key | Read via `os.getenv` into `SECRET_KEY` in `LearningPlatform/settings.py`, used by Django for session/cookie signing |

| .env.template | learn-ops-api/.env.template | LEARN_OPS_ALLOWED_HOSTS | Comma-separated list of hostnames the Django app will serve | Read via `os.getenv` into `ALLOWED_HOSTS` in `LearningPlatform/settings.py` |

| .env.template | learn-ops-api/.env.template | SLACK_BOT_TOKEN | Auth token for the Slack Web API | Read via `os.getenv("SLACK_BOT_TOKEN")` in `LearningAPI/utils.py` and several views (e.g. `capstone_view.py`, `student_view.py`) to send Slack notifications |

| learn-ops-api.yaml | learn-ops-api/config/learn-ops-api.yaml | databases[].engine | Database engine for the DigitalOcean-managed database | Read by DigitalOcean App Platform when provisioning the `learnops` database (not read by app code) |

| learn-ops-api.yaml | learn-ops-api/config/learn-ops-api.yaml | services[].http_port | Port the deployed API container listens on | Read by DigitalOcean App Platform to route traffic to the `learn-ops-api` service |

| learn-ops-api.yaml | learn-ops-api/config/learn-ops-api.yaml | services[].instance_size_slug | Compute size tier for the deployed API instance | Read by DigitalOcean App Platform when sizing the `learn-ops-api` droplet/container |

| .env.template | service-monarch/.env.template | GH_PAT | GitHub personal access token | Read via `os.getenv("GH_PAT")` into `Settings.GITHUB_TOKEN` in `service/config/settings.py`, used for GitHub API auth |

| .env.template | service-monarch/.env.template | VALKEY_HOST | Hostname of the Valkey (Redis-compatible) instance | Read via `os.getenv("VALKEY_HOST", "localhost")` into `Settings.VALKEY_HOST` in `service/config/settings.py` |

| .env.template | service-monarch/.env.template | VALKEY_PORT | Port of the Valkey instance | Referenced alongside `VALKEY_HOST` for connecting to Valkey from the Monarch service |

| settings.py | service-monarch/service/config/settings.py | GITHUB_API_URL | Base URL for the GitHub REST API | Hardcoded default on the `Settings` pydantic model, used when the service makes GitHub API calls |

| settings.py | service-monarch/service/config/settings.py | PROMETHEUS_PORT | Port the service exposes Prometheus metrics on | Set on the `Settings` model, used to bind the metrics endpoint for scraping |

| settings.py | service-monarch/service/config/settings.py | SLACK_BOT_TOKEN | Auth token for Slack Web API calls | Read via `os.getenv("SLACK_BOT_TOKEN")` into `Settings.SLACK_BOT_TOKEN`, consumed in `service/notifications/slack.py` as a Bearer token |

| .env | learn-ops-client/.env | REACT_APP_API_URI | Base URL of the API the client talks to | Read via `process.env.REACT_APP_API_URI` in `src/components/Settings.js` as `apiHost` |

| .env | learn-ops-client/.env | CHOKIDAR_USEPOLLING | Forces the dev server to use polling for file-watching | Consumed by `react-scripts`' dev server (not app code), useful in Docker where native FS events don't propagate |

| .env | learn-ops-client/.env | GENERATE_SOURCEMAP | Toggles whether the production build emits source maps | Consumed by `react-scripts` at build time (not app code) |

| .env | learn-ops-infrastructure/.env | POSTGRES_DB | Name of the Postgres database to create | Passed via `env_file` into the `database` service in `docker-compose.yml`; also used in its healthcheck (`pg_isready -d`) |

| .env | learn-ops-infrastructure/.env | POSTGRES_USER | Postgres superuser created on container init | Passed via `env_file` into the `database` service; also used in its healthcheck (`pg_isready -U`) |

| .env | learn-ops-infrastructure/.env | DATA_SOURCE_NAME | Postgres connection string for the metrics exporter | Passed via `env_file` into the `postgres_exporter` service, which uses it to connect to `database` |

| docker-compose.yml | learn-ops-infrastructure/docker-compose.yml | services.api.ports | Host↔container port mappings for the API container | Docker Compose maps these ports (e.g. app port and debugger port) from the host into the `api` container |

| docker-compose.yml | learn-ops-infrastructure/docker-compose.yml | services.database.healthcheck | Command Compose runs to determine when Postgres is ready | Used by `depends_on: condition: service_healthy` on the `api` service so it waits for the DB |

| docker-compose.yml | learn-ops-infrastructure/docker-compose.yml | networks.learningplatform | Shared Docker network name joining all services | Every service (`database`, `api`, `client`, `prometheus`, `grafana`, `postgres_exporter`) attaches to it so they can reach each other by service name |

| prometheus.yml | learn-ops-infrastructure/prometheus.yml | scrape_interval | How often Prometheus polls targets for metrics | Set under `global`, applied to all scrape jobs unless overridden |

| prometheus.yml | learn-ops-infrastructure/prometheus.yml | scrape_configs[].metrics_path | HTTP path Prometheus requests metrics from | Set on the `django` job (`/metrics/metrics`) so Prometheus scrapes the API's custom metrics endpoint instead of the default `/metrics` |

| prometheus.yml | learn-ops-infrastructure/prometheus.yml | scrape_configs[].targets | Host:port of each service to scrape | Points the `django` job at `api:8000` and the `postgresql` job at `postgres_exporter:9187`, resolved via the shared Docker network |

### 1b. How to Start It

Findings from `learn-ops-infrastructure/Makefile`, which wraps `docker compose` (and `scripts/setup.sh`/`scripts/teardown.sh`) for this repo:

- **`make setup`** — first-time bootstrap, not a start command by itself. Runs `scripts/setup.sh`, which checks prerequisites (Docker, GitHub SSH, etc.), clones sibling repos, writes each service's `.env` from its `.env.template`, and only then *offers* to run `docker compose up -d` for you. This is the one you run once before anything else will work.

- **`make doctor`** — runs `scripts/setup.sh --doctor`, a read-only subset of setup (platform/prereq/Docker checks only). It diagnoses whether your machine is ready for `make setup`; it never touches `.env` files, images, or containers.

- **`make up`** — the normal "start the whole stack" command. Runs `pull` first (`docker compose pull --ignore-buildable`), then `docker compose up --build -d` (rebuilds images if changed, starts every service in `docker-compose.yml` detached), then tails all logs with `docker compose logs -f`.

- **`make up-api`** — starts only the `api` service (plus its `depends_on`, i.e. `database`), rebuilding it if needed. Use this when you only need the backend/database, not the React client or monitoring stack.

- **`make up-client-api`** — starts `api` and `client` together (and `database` via `api`'s dependency), skipping Prometheus/Grafana/postgres_exporter. A middle ground between `up-api` and the full `up`.

- **`make restart`** — `pull` + `docker compose down` + `docker compose up --build -d` + log tail. A full stop/rebuild/start cycle, useful after pulling new code or changing `.env` files, since `up` alone won't stop already-running containers.

- **`make down`** — stops and removes containers via `docker compose down`, but leaves named volumes (e.g. Postgres data) intact.

- **`make reset`** — `docker compose down -v --remove-orphans`, i.e. the same as `down` but also deletes volumes (wipes the database) and removes orphaned containers. Destructive; used to get back to a clean slate.

- **`make logs`** / **`make ps`** — not startup commands, but companions: `logs` tails all service logs, `ps` shows current container status.

Key differences at a glance:
- `setup`/`doctor` prepare the environment (prereqs, `.env` files); `up`/`restart`/`up-api`/`up-client-api` actually launch containers.

- `up`, `up-api`, and `up-client-api` differ only in **which services** they bring up (all vs. `api` only vs. `api`+`client`) — same underlying `docker compose up --build -d` mechanism.

- `up` vs `restart` differ in whether existing containers are torn down first — `restart` guarantees a clean stop before starting, `up` does not.

- `down` vs `reset` differ in **data persistence** — `down` keeps volumes, `reset` deletes them.

### 1c. Where to Access It

| Compose File | Service | Port | URL | What It Does |
|---|---|---|---|---|
| `learn-ops-infrastructure/docker-compose.yml` | client (React app) | 3000 | http://localhost:3000 | Student/instructor-facing web UI (Create React App dev server) |

| `learn-ops-infrastructure/docker-compose.yml` | api (Django REST API) | 8000 (app), 5678 (debugpy remote debugger, attach only) | http://localhost:8000 | Backend REST API — auth, cohorts, students, assessments, capstones, etc.; also serves Django admin and the LogViewer |

| `learn-ops-infrastructure/docker-compose.yml` | database (Postgres 16) | 5433 (host) → 5432 (container) | `localhost:5433` (Postgres connection, not HTTP) | Primary relational datastore for the API |

| `learn-ops-infrastructure/valkey/docker-compose.yml` | valkey | 6379 | `localhost:6379` (Valkey/Redis protocol, not HTTP) | Redis-compatible cache/key-value store used by the API and Monarch |

| `learn-ops-infrastructure/valkey/docker-compose.yml` | valkey-monitor | none exposed | N/A — CLI (`valkey-cli monitor`) attached to the `valkey` service, not network-accessible | Streams every command Valkey receives, for local debugging of cache activity |

| `learn-ops-infrastructure/docker-compose.yml` | prometheus | 9090 | http://localhost:9090 | Scrapes and stores metrics from the API and postgres_exporter |

| `learn-ops-infrastructure/docker-compose.yml` | grafana | 3001 (host) → 3000 (container) | http://localhost:3001 | Dashboards for visualizing the metrics Prometheus collects |

| `learn-ops-infrastructure/docker-compose.yml` | postgres_exporter | 9187 | http://localhost:9187/metrics | Exposes Postgres server stats in Prometheus format so Prometheus can scrape the database's health |

| `service-monarch/docker-compose.yml` | monarch | 8080 (Prometheus metrics), 8081 (log web interface) | http://localhost:8080/metrics, http://localhost:8081 | Background service that manages student GitHub repos/forks and sends Slack notifications; exposes its own metrics and a log viewer |


### 1d. Service Dependencies

| Service | Depends On | Why |
|---|---|---|
| api | database | Django's ORM stores every piece of app data (cohorts, students, assessments, capstones, etc.) in Postgres; without it the API has nothing to read or write. Compose declares `depends_on: database: condition: service_healthy`, so the API container won't start until Postgres is actually accepting connections, not just running. |

| api | valkey | `LearningAPI/views/popular_query.py` and `team_maker_view.py` connect to Valkey directly to cache search results and to `publish()` on the `channel_migrate_issue_tickets` channel; those endpoints break without a reachable Valkey. This isn't a compose `depends_on`, so there's no startup-order guarantee — only a functional one. |

| api | GitHub OAuth (external) | `LearningAPI/views/github_login.py` uses `django-allauth`'s GitHub adapter plus `LEARNING_GITHUB_CALLBACK` to let students/instructors log in with GitHub; login is unavailable if GitHub's OAuth endpoints aren't reachable. |

| api | Slack API (external) | Multiple views (`utils.py`, `capstone_view.py`, `proposal_timeline.py`, `student_view.py`) call the Slack API with `SLACK_BOT_TOKEN` to post notifications (e.g. capstone/proposal events); those notifications silently depend on Slack being reachable. |

| client | api | The React app is built with `REACT_APP_API_URI` baked in and calls that URL for auth, cohort data, assessments, etc. (`src/components/Settings.js`); the UI loads but is non-functional without the API. Not a compose `depends_on` — it's a browser-side runtime dependency, not a container-start-order one. |

| grafana | prometheus | Declared via `depends_on: prometheus`. Grafana's dashboards are meant to query Prometheus as the metrics source, so it needs Prometheus running to have any data to visualize. (No Grafana provisioning file was found in the repo, so the Prometheus datasource must currently be added manually in the UI.) |

| monarch | valkey | `service/core/monarch.py` opens a `ResilientPubSub` subscription on the `channel_migrate_issue_tickets` channel — the same channel the API's `team_maker_view.py` publishes to. This is how team/ticket-migration events get from the API to Monarch; Monarch does nothing on that flow without Valkey. Not a compose dependency (Monarch lives in a separate compose file from Valkey). |

| monarch | GitHub API (external) | Monarch's core purpose is managing student repos/forks; it authenticates with `GH_PAT` and calls `GITHUB_API_URL` for essentially all of its work (cloning, forking, issue/ticket migration). Without GitHub reachable, Monarch can't do its job. |

| monarch | Slack API (external) | `service/notifications/slack.py` uses `SLACK_BOT_TOKEN` as a Bearer token to post notifications; Monarch's Slack-notification feature depends on this external API. |

| postgres_exporter | database | Declared via `depends_on: database`. It uses `DATA_SOURCE_NAME` to open a connection to Postgres and translate its internal stats into Prometheus metrics — it has nothing to export without the database. |

| prometheus | api | Declared via `depends_on: api`. `prometheus.yml`'s `django` job scrapes `api:8000/metrics/metrics`; Prometheus needs the API up to have a metrics endpoint to poll. |

| prometheus | postgres_exporter | `prometheus.yml`'s `postgresql` job scrapes `postgres_exporter:9187`, so Prometheus needs it running to get DB metrics — though this isn't reflected in the compose `depends_on` list (only `api` is), so it's a functional dependency without a guaranteed startup order. |

| valkey-monitor | valkey | Declared via `depends_on: valkey` in `valkey/docker-compose.yml`. It runs `valkey-cli -h valkey monitor`, which attaches to the `valkey` container and streams its command traffic — it's just an observer process and can't function without Valkey running. |

### 1e. Main Entry Points

| Service | Startup Files | Routes | URL Config File |
|---|---|---|---|
| api | `manage.py` (Django management entry, run via `entrypoint.sh` → `python3 manage.py runserver 0.0.0.0:8000`, wrapped in `debugpy` when `DEBUG=True`); underlying WSGI app object is `LearningPlatform/wsgi.py` | `/` (DRF router — `assessments`, `students`, `cohorts`, `courses`, etc.), `/accounts`, `/auth/...` (incl. GitHub OAuth), `/logs/` (LogViewer), `/admin`, `/metrics/` (django-prometheus) | `LearningPlatform/urls.py` (root URLconf; includes `LogViewer/urls.py`, `dj_rest_auth.urls`, `django_prometheus.urls`) |

| client | `src/index.js` (`ReactDOM.render`, wraps app in `react-router-dom`'s `BrowserRouter`); launched via `react-scripts start` (`package.json` `start` script) | Role-based route trees defined via `<Route>` components — `Dashboard`, `Cohorts`, `Courses`, `Assessments`, `Teams`, etc. | `src/components/ApplicationViews.js` (shared routes), composed inside `src/components/LearnOps.js` with `src/components/StaffViews.js` and `src/components/StudentViews.js` for role-specific routes |

| database | N/A — off-the-shelf `postgres:16` image, no application code; started per its `command`/entrypoint baked into the image | N/A — not an HTTP service | N/A |

| valkey | N/A — off-the-shelf `valkey/valkey:latest` image, started with `valkey-server --save 900 1 --loglevel notice` | N/A — not an HTTP service | N/A |

| valkey-monitor | N/A — off-the-shelf `valkey/valkey:latest` image, started with `valkey-cli -h valkey monitor` (entrypoint override) | N/A — not an HTTP service | N/A |

| prometheus | N/A — off-the-shelf `prom/prometheus:latest` image, started with `--config.file=/etc/prometheus/prometheus.yml` | `/` (Prometheus UI) | `learn-ops-infrastructure/prometheus.yml` (defines what it scrapes, not routes it serves) |

| grafana | N/A — off-the-shelf `grafana/grafana:latest` image, default entrypoint | `/` (Grafana UI) | N/A — no provisioning/config file found in this repo; dashboards/datasources are configured through the UI or Grafana's own volume state |

| postgres_exporter | N/A — off-the-shelf `quay.io/prometheuscommunity/postgres-exporter` image, default entrypoint | `/metrics` | N/A — configured entirely via the `DATA_SOURCE_NAME` env var, no routes file |

| monarch | `service/main.py` (`asyncio.run(TicketMigrator().run())`); the actual HTTP servers are started inside `service/core/monarch.py` — `start_http_server()` for Prometheus metrics (port 8080) and a threaded `web_interface.start_web_interface()` for the log viewer (port 8081) | Log viewer (Flask): `/`, `/health`, `/api/logs`, `/api/log-levels`, `/api/services`; Metrics: default `prometheus_client` `/metrics` | `service/custom_logging/web_interface.py` (Flask `@app.route` definitions for the log viewer); metrics endpoint has no separate routes file — it's registered by `prometheus_client` itself |

## 2. Services

| Service Name | Tech Stack (including version) | Purpose |
|---|---|---|
| | | |

## 3. System Overview