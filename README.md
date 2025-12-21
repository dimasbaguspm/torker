# Torker

Lightweight monitoring stack for local/dev environments. This repository provides a small Docker Compose stack that runs Prometheus, Grafana, cAdvisor, node-exporter, Loki and Promtail for metrics and logs.

Files of interest:
- `docker-compose.yml` — service definitions and volumes
- `prometheus.yml` — Prometheus scrape targets
- `promtail-config.yaml` — Promtail log discovery and relabeling
- `loki-config.yaml` — (optional) custom Loki config; by default the Loki image's internal config is used

---

## Prerequisites

- Docker & Docker Compose (or Docker Compose v2 `docker compose`).
- An external Docker network named `home` is used by the compose file. Create it once if missing:

```zsh
docker network ls | grep -q home || docker network create home
```

- Provide Grafana admin credentials in an `.env` file (example below).

---

## Quick start

1. Create or update `.env` with Grafana credentials (example):

```env
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=changeme
```

2. Start the stack:

```zsh
docker compose up -d
```

3. Open services:

- Grafana: http://localhost:9111
- Prometheus: http://localhost:9114
- cAdvisor: http://localhost:9112
- Node Exporter metrics: http://localhost:9113/metrics
- Loki (inside network): http://loki:3100 (host mapped to port 9115)

---

## Services & Ports (summary)

- `prometheus` — image `prom/prometheus`, container port `9090` mapped to host `9114`.
- `grafana` — image `grafana/grafana`, container port `3000` mapped to host `9111`.
- `cadvisor` — container UI on host `9112`.
- `node-exporter` — metrics on host `9113`.
- `loki` — Loki log storage, container port `3100` mapped to host `9115`.
- `promtail` — collects container logs and pushes to Loki.

Volumes created by compose:
- `grafana-data`, `prometheus-data`, `loki-data`, `promtail-positions`.

---

## Grafana setup

1. Login to Grafana at `http://localhost:9111` with credentials in `.env`.
2. Add Prometheus datasource (UI):
   - URL: `http://prometheus:9090` (inside Docker network)
3. Add Loki datasource (UI):
   - URL: `http://loki:3100` (inside Docker network). Host port is `9115` if you need it from the host.

You can also provision datasources using Grafana provisioning files, but this repository defaults to manual setup so you can pick datasources at runtime.

---

## Logs (Loki + Promtail)

Promtail is configured to discover Docker containers (reads `/var/run/docker.sock`) and relabel discovered metadata into labels such as:

- `container_name` — container name
- `compose_service` — docker-compose service name (if present)
- `compose_project` — compose project name (if present)

Promtail persists positions to `promtail-positions` so it resumes after restarts.

Use Grafana Explore (Loki datasource) or add Logs panels to dashboards to query logs per container.

### Quick checks

```bash
docker compose logs -f promtail    # watch promtail discover containers
docker compose logs -f loki       # watch loki
```

---

## Creating a Logs panel in Grafana

1. Create a dashboard variable `container`:
   - Type: Query, Data source: your Loki datasource
   - Query: `label_values({job="docker-sd"}, container_name)`

2. Add a Logs panel and use queries like:

```logql
{container_name="$container"}
{container_name="$container"} |= "error"
```

3. Example LogQL snippets:

- Tail logs for a container:
  ```logql
  {container_name="torker_promtail"}
  ```
- Count errors over time (graph panel):
  ```logql
  sum by (container_name) (count_over_time({job="docker-sd"} |= "error"[5m]))
  ```
- Top noisy containers (errors in last hour):
  ```logql
  topk(5, sum by (container_name) (count_over_time({job="docker-sd"} |= "error"[1h])))
  ```

Tip: In Explore, run `{job="docker-sd"}` and inspect a single log line to see which labels Promtail attached.

---

## Troubleshooting

- If Promtail shows no labels or no logs:
  - Verify the compose mounts: `/var/run/docker.sock` and `/var/lib/docker/containers` must be mounted into `promtail`.
  - Check Promtail logs: `docker compose logs -f promtail`.
- If Loki fails to start with a YAML parsing error, remove any custom config mount (the compose file defaults to the image config) or provide a Loki v3-compatible config matching the image version.
- If Grafana shows combined logs, use `container_name` or other labels in queries to filter to a single container.

---

## Useful commands

```zsh
# create external network if missing
docker network ls | grep -q home || docker network create home

# start all services
docker compose up -d

# restart a single service
docker compose up -d promtail

# tail logs for a service
docker compose logs -f promtail
docker compose logs -f loki
docker compose logs -f grafana

# list running torker containers
docker ps --filter name=torker_
```

---

## Want me to add more?

- I can create an importable Grafana dashboard JSON with a `$container` variable, a Logs panel and an error-count graph.
- I can add a small `.env.example` file to the repo.

If you'd like either, tell me which and I'll add it.
