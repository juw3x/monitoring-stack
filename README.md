# monitoring-stack

Готовый monitoring-стек для небольших проектов: **Prometheus + Grafana + Alertmanager + Loki**
с готовыми дашбордами и правилами алертов для типичного сервиса (Nginx, PostgreSQL, Node Exporter).

Один `docker compose up -d` — и у тебя полный observability-стек с демо-нагрузкой.

## Состав

| Компонент | Образ | Назначение |
|---|---|---|
| Prometheus | `prom/prometheus:v3.5.0` | сбор метрик, rules в `prometheus/rules/` |
| Alertmanager | `prom/alertmanager:v0.28.1` | роутинг/подавление алертов |
| Grafana | `grafana/grafana:12.1.0` | дашборды (auto-provisioned, anonymous admin) |
| Loki + Promtail | `grafana/loki:3.6.11`, `grafana/promtail:3.6.11` | логи nginx + docker-контейнеров |
| node-exporter | `prom/node-exporter:v1.9.1` | метрики хоста |
| nginx-exporter | `nginx/nginx-prometheus-exporter:1.4.1` | метрики nginx (stub_status) |
| postgres-exporter | `prometheuscommunity/postgres-exporter:v0.17.1` | метрики PostgreSQL |
| Демо: nginx | `nginx:1.27-alpine` | JSON access log → Loki, stub_status → exporter |
| Демо: postgres | `postgres:16-alpine` | БД `app` для postgres-exporter |

## Быстрый старт

```bash
git clone <repo> && cd monitoring-stack
docker compose up -d
docker compose ps
```

Открыть:

- **Grafana**: http://localhost:3000 — вход не нужен (anonymous admin), дашборды уже provisioned
- **Prometheus**: http://localhost:9090
- **Alertmanager**: http://localhost:9093
- **Nginx demo**: http://localhost:8080

## Что показывает

### Дашборды (Grafana → Dashboards)

- **Node / Host Overview** — CPU, RAM, disk, network + панель логов из Loki
- **Nginx Overview** — RPS по статусам, latency p50/p95/p99, connections + access log
- **PostgreSQL Overview** — размер БД, коннекты, транзакции, deadlocks

### Алерты (`prometheus/rules/`)

- `node.yml` — NodeHighMemoryUsage, NodeHighCPUUsage, NodeDiskAlmostFull, NodeExporterDown, PrometheusTargetDown, PrometheusRuleFailures
- `nginx.yml` — NginxHigh5xxRate (>5%), NginxHighLatency (p95>1s), NginxExporterDown
- `postgres.yml` — PostgresDown, PostgresHighConnections (>80%), PostgresDeadlocks, PostgresReplicationLag

Алерты видны в Alertmanager UI и в Grafana (панель Alert list). Чтобы получать уведомления —
отредактируй `alertmanager/alertmanager.yml` (webhook/email/slack/telegram).

### Логи (Loki)

Promtail собирает:
- JSON access-логи nginx (`./nginx/logs/*.log`)
- stdout/stderr всех docker-контейнеров (через docker.sock)

В Grafana → Explore → datasource **Loki**:

```
{job="nginx"} | json
{job="docker", container="mon-postgres"}
```

## PromQL примеры

```promql
# RPS nginx по статусам
sum by(status) (rate(nginx_http_requests_total[5m]))

# % ошибок 5xx
sum(rate(nginx_http_requests_total{status=~"5.."}[5m]))
/ sum(rate(nginx_http_requests_total[5m]))

# Нагрузка CPU
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Активные коннекты postgres
sum(pg_stat_activity_count) by (datname)
```

## Кастомизация

- Добавить scrape target → `prometheus/prometheus.yml` → `docker compose restart prometheus`
- Добавить алерт → `prometheus/rules/*.yml`
- Свой дашборд → положи JSON в `grafana/dashboards/` (подхватится автоматически)
- Свой источник логов → `promtail/promtail-config.yml`
- PostgreSQL exporter для другой БД → поменяй `DATA_SOURCE_NAME` в compose

## Остановка / очистка

```bash
docker compose down          # остановить
docker compose down -v       # остановить + удалить данные (volumes)
```
