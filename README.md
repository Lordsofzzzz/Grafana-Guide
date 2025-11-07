# Grafana-Guide

## 1. Purpose
Expanded, practical reference for installing, configuring, operating, and extending Grafana (OSS) in dev, staging, and small-to-medium production setups.

## 2. Core Concepts
- Data source: External system queried (Prometheus, Loki, InfluxDB, PostgreSQL, Tempo, Elasticsearch, Mimir, etc.).
- Panel: Single visualization/query.
- Dashboard: Collection of panels + layout + variables.
- Variable: Dynamic input (templating) that rewrites panel queries.
- Alert rule: Condition over time series; produces alert instances routed to contact points.
- Provisioning: Declaring datasources, dashboards, alerting, plugins as code via YAML/JSON.
- Plugin: Extension (panel, datasource, app).
- Unified Alerting: Modern alert system (rules + contact points + notification policies).
- Access model: Organizations (legacy), folders, RBAC (roles: Viewer, Editor, Admin).

## 3. Installation Options

### 3.1 Docker (quick)
```sh
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -e GF_SECURITY_ADMIN_PASSWORD='changeme!' \
  grafana/grafana:latest
```
Open:
```sh
"$BROWSER" http://localhost:3000
```

### 3.2 Docker Compose (persistent + provision)
```yaml
version: "3.8"
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning
    environment:
      GF_SECURITY_ADMIN_PASSWORD: changeme!
      GF_USERS_ALLOW_SIGN_UP: "false"
volumes:
  grafana-data:
```
Start: `docker compose up -d`

### 3.3 Debian/Ubuntu package
```sh
sudo apt update
sudo apt install -y apt-transport-https software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/grafana.gpg
echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana
sudo systemctl enable --now grafana-server
```

### 3.4 Helm (Kubernetes)
```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana \
  --set adminPassword=changeme \
  --set service.type=ClusterIP
```

## 4. Configuration Basics

### 4.1 grafana.ini overrides (Docker env)
Use GF_<SECTION>_<KEY>=value. Example:
```sh
-e GF_SERVER_ROOT_URL=https://grafana.example.com \
-e GF_AUTH_ANONYMOUS_ENABLED=false \
-e GF_SECURITY_ALLOW_EMBEDDING=false
```

### 4.2 File location (Linux)
- Binary service: /usr/sbin/grafana-server
- Config: /etc/grafana/grafana.ini
- Data (SQLite, dashboards): /var/lib/grafana
- Logs: /var/log/grafana

### 4.3 Database
Default: SQLite (simplicity). For scale, move to PostgreSQL or MySQL:
```ini
[database]
type = postgres
host = db:5432
name = grafana
user = grafana
password = ****
ssl_mode = disable
```

## 5. Securing Grafana
- Change admin password immediately.
- Enforce TLS at reverse proxy (Caddy, Nginx, Traefik).
- Restrict public signups: GF_USERS_ALLOW_SIGN_UP=false.
- Enable role-based access with folders; avoid “Everyone can edit”.
- Configure OAuth/OIDC (GitHub, Keycloak, Azure AD) for centralized auth.
- Limit plugin trust; audit new plugins.
- Backup database (SQLite file or RDS snapshot) regularly.
- Apply least privilege to datasource credentials (read-only for Prometheus, DB).

## 6. Datasource Provisioning
Create file: provisioning/datasources/datasource.yml
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
```

## 7. Dashboard Provisioning
Directory: provisioning/dashboards/
File: dashboards.yml
```yaml
apiVersion: 1
providers:
  - name: Default
    folder: "Infra"
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```
Place JSON dashboards in /var/lib/grafana/dashboards (or mounted path).

Export a dashboard via UI (Dashboard settings -> JSON model) and commit to Git.

## 8. Variables (Templating)
Common Prometheus variable examples:
- Namespaces:
  Query: label_values(kube_pod_info, namespace)
- Pod:
  Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
- Dynamic time aggregation (custom variable):
  Values: 1m,5m,15m,1h (used inside rate() window)

Chaining variables reduces query duplication.

## 9. Query Examples (Prometheus)
- CPU usage (per container):
  sum by (pod, container) (rate(container_cpu_usage_seconds_total{image!=""}[5m]))
- Memory working set (MiB):
  sum by (pod) (container_memory_working_set_bytes{image!=""}) / 1024 / 1024
- HTTP error ratio:
  (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) * 100

Use recording rules in Prometheus for complex expressions to lighten Grafana load.

## 10. Alerting (Unified)
### 10.1 Creating an alert rule (UI)
Alerting -> Alert rules -> New:
- Query: A = rate(http_requests_total[5m])
- Condition: WHEN avg() OF A IS ABOVE 100 FOR 5m
- Labels: severity=warning service=web
- Notification policy routes severity=critical differently than warning.

### 10.2 Contact points
Configure Slack, Email, PagerDuty. Test each.

### 10.3 Provisioning alert rules
File: provisioning/alerting/rules.yml
```yaml
apiVersion: 1
groups:
  - name: web-rates
    interval: 1m
    rules:
      - uid: http_rate_high
        title: High HTTP request rate
        condition: A
        data:
          - refId: A
            datasourceUid: prometheus
            queryType: timeSeriesQuery
            relativeTimeRange:
              from: 300
              to: 0
            model:
              expr: rate(http_requests_total[5m])
        noDataState: NoData
        execErrState: Error
        for: 5m
        annotations:
          summary: High HTTP request rate
        labels:
          severity: warning
```

## 11. Annotations
Use event markers (deploys) from external sources:
```sh
curl -X POST \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3000/api/annotations \
  -d '{"dashboardId":0,"time":'$(date +%s%3N)',"text":"Deployment v1.2.3"}'
```

## 12. API Usage
Create API key (Server Admin -> Service Accounts or Legacy API keys). Example fetch dashboards:
```sh
curl -s -H "Authorization: Bearer $GRAFANA_TOKEN" \
  http://localhost:3000/api/search?query=&type=dash-db | jq
```

## 13. Plugins
Install at build time (Dockerfile):
```Dockerfile
FROM grafana/grafana:latest
RUN grafana-cli plugins install grafana-piechart-panel \
 && grafana-cli plugins install grafana-github-datasource
```
List installed:
```sh
docker exec grafana grafana-cli plugins ls
```

## 14. Performance & Scaling
- Switch to PostgreSQL for > ~100 users or heavy alerting.
- Enable caching with reverse proxy if embedding public dashboards.
- Reduce panel interval (Min interval field) to avoid high cardinality query spam.
- Prefer wide dashboards with fewer queries; group metrics logically.
- Use Explore for ad-hoc queries, keep dashboards stable.

## 15. Logs / Traces / Metrics Stack
- Metrics: Prometheus or Mimir.
- Logs: Loki (query via {app="api"} | line_format "{{.msg}}").
- Traces: Tempo (exemplars and trace ID linking).
Linking:
- Enable exemplars in Prometheus client libraries.
- Add traceID label / metadata and configure Tempo data source.

## 16. Backup & Restore
- SQLite: Stop Grafana, copy /var/lib/grafana/grafana.db
- PostgreSQL:
  pg_dump -Fc grafana > grafana.dump
- Dashboards (provisioned): Already in Git; treat Git as source of truth.
- Restore: Replace DB file or pg_restore, then restart service.

## 17. Upgrades
- Read release notes (breaking changes).
- Test upgrade in staging (copy DB + plugins).
- Keep pinned version in production (grafana/grafana:10.4.2) to avoid accidental major jumps.
- After upgrade: Validate logs, alert rule states, plugin compatibility.

## 18. Troubleshooting
Commands:
```sh
docker logs grafana | tail -n 50
lsof -iTCP:3000 -sTCP:LISTEN
curl -I http://localhost:3000/login
```
Common issues:
- 404 for /api endpoints: token missing or insufficient roles.
- Slow dashboards: excessive series returned (filter labels, use recording rules).
- Alert no data: datasource timing out; inspect Explore for raw query.

## 19. Environment Variables Quick Reference
- GF_SECURITY_ADMIN_PASSWORD
- GF_SERVER_ROOT_URL
- GF_USERS_ALLOW_SIGN_UP
- GF_AUTH_GITHUB_ENABLED (when using GitHub)
- GF_INSTALL_PLUGINS (comma-separated plugin list)

Example:
```sh
docker run -d -p 3000:3000 \
  -e GF_SECURITY_ADMIN_PASSWORD=changeme \
  -e GF_INSTALL_PLUGINS=grafana-piechart-panel,grafana-github-datasource \
  grafana/grafana:latest
```

## 20. Embedding & Sharing
- Use snapshot (static) for offline share.
- Signed URLs (public dashboards) require proper auth config.
- Disable anonymous mode unless intentional.

## 21. JSON Dashboard Structure (Minimal)
```json
{
  "title": "Simple",
  "panels": [
    {
      "type": "timeseries",
      "title": "HTTP Rate",
      "targets": [
        { "expr": "rate(http_requests_total[5m])" }
      ]
    }
  ]
}
```

## 22. Example Infra Dashboard Panels
- Node CPU: avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)
- FS usage: (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes
- Network RX/TX: rate(node_network_receive_bytes_total[5m]), rate(node_network_transmit_bytes_total[5m])

## 23. Dev Automation Snippet
Export all dashboards to local dir:
```sh
mkdir -p exported
curl -s -H "Authorization: Bearer $GRAFANA_TOKEN" \
  http://localhost:3000/api/search?query=&type=dash-db | jq -r '.[] | .uid' |
while read uid; do
  curl -s -H "Authorization: Bearer $GRAFANA_TOKEN" \
    http://localhost:3000/api/dashboards/uid/$uid | \
    jq '.dashboard' > exported/$uid.json
done
```

## 24. Housekeeping Checklist
- Review orphaned datasources monthly.
- Audit API keys (rotation).
- Validate alert noise (reduce duplicate notifications).
- Update pinned Grafana minor version quarterly.

## 25. Resources
- Official docs: https://grafana.com/docs/
- Community: https://community.grafana.com
- Play site (examples): https://play.grafana.org
- Prometheus docs: https://prometheus.io/docs/
- Loki docs: https://grafana.com/docs/loki/

## 26. Next Steps
Add Helm values customizations, SSO configuration examples, and automated dashboard tests (JSON lint).