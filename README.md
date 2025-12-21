# Torker

Files: `docker-compose.yml` and `prometheus.yml`.

## Setup from scratch

1. Create the external network used by the stack:
   ```zsh
   docker network create home
   ```
2. Set Grafana creds in `.env` (see `.env.example`).
3. Start the stack:
   ```zsh
   docker compose up -d
   ```

## Access

- Grafana: http://localhost:9111 (login with `.env` creds)
- Prometheus: http://localhost:9114
- cAdvisor UI: http://localhost:9112
- Node exporter: http://localhost:9113/metrics

## Grafana datasource

- Add Prometheus datasource pointing to `http://prometheus:9090` (inside the Docker network).

## Grafana dashboard

- Community dashboard **ID 893** (“Docker and system monitoring” using cAdvisor/Prometheus).
- Import via Grafana: _Dashboards → Import → enter `893` → select Prometheus datasource_.

## Files

- `docker-compose.yml`: Prometheus, Grafana, cAdvisor, node-exporter; uses named volumes (`grafana-data`, `prometheus-data`) for data; ports mapped to 9111/9112/9113/9114; attaches to external network `home`.
- `prometheus.yml`: scrapes Prometheus, cAdvisor, and node-exporter.
- `.env.example` / `.env`: Grafana admin credentials (loaded via `env_file`).
