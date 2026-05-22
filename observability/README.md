# Crypto Dashboard Observability

Lightweight single-node LGP stack for the crypto dashboard:

- Grafana: `http://127.0.0.1:3001` behind nginx/TLS.
- Loki: internal only, stores logs in Docker volume `crypto_observability_loki`.
- Promtail: reads Docker JSON logs and pushes to Loki.
- Prometheus: scrapes backend metrics from `backend:8000/metrics`, stores 7 days
  in Docker volume `crypto_observability_prometheus`.

## Deploy

```bash
cd /docker/crypto-observability
cp .env.example .env
chmod 600 .env
# Set GRAFANA_ADMIN_PASSWORD in .env on the server only.
docker volume create crypto_observability_grafana
docker volume create crypto_observability_loki
docker volume create crypto_observability_prometheus
docker volume create crypto_observability_promtail
docker compose config
docker compose up -d
```

Runtime data is stored only in Docker named volumes:
`crypto_observability_grafana`, `crypto_observability_loki`,
`crypto_observability_prometheus`, and `crypto_observability_promtail`.
The stack directory contains configs only and should live under `/docker`, not
under a user's home directory.

Prometheus and Grafana join the Docker network where nginx and the app backend
are resolvable. By default this stack uses `APP_NETWORK=nginx_network`.

Install the nginx vhost:

```bash
sudo cp nginx/promtail.vadim-denisovich.ru.conf /etc/nginx/sites-available/
sudo ln -sf /etc/nginx/sites-available/promtail.vadim-denisovich.ru.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
sudo certbot --nginx -d promtail.vadim-denisovich.ru
```

If nginx itself runs in Docker on `APP_NETWORK`, proxy to
`http://observability-grafana:3000`.

## Grafana Checks

Backend logs:

```logql
{container="backend"} | json | line_format "{{.log}}"
```

The query parses Docker's JSON wrapper and renders only the `log` field, so the
view hides Docker wrapper fields such as `stream` and `time`.

Backend P99:

```promql
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="lightstar_backend"}[5m])) by (le, endpoint))
```

Request rate:

```promql
sum(rate(http_requests_total{job="lightstar_backend"}[1m])) by (endpoint, http_status)
```

Backend CPU approximation:

```promql
rate(process_cpu_seconds_total{job="lightstar_backend"}[1m]) * 100
```

Stress test from the server:

```bash
ab -n 20000 -c 50 -k http://127.0.0.1:8000/healthz
```

The server is overloaded if P99 latency keeps rising, backend CPU is near a full
core, `up{job="lightstar_backend"}` drops to `0`, Prometheus reports scrape
timeouts, or Loki/Grafana queries become noticeably delayed.
