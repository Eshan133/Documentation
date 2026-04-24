# Grafana Monitoring Stack — Setup Guide
> Ubuntu VPS with Snap Docker + Docker Compose

## Stack Overview

| Component | Version | Purpose |
|---|---|---|
| Prometheus | latest | Metrics scraping & storage |
| cAdvisor | v0.47.2 | Per-container metrics (must use this version on snap Docker) |
| Node Exporter | latest | Host system metrics |
| Grafana | latest | Dashboards & visualization |

---

## Prerequisites

- Ubuntu VPS with Docker installed via **Snap**
- Docker Compose v2
- Ports available: `9090` (Prometheus), `8080` (cAdvisor), `9100` (Node Exporter), and one free port for Grafana (e.g. `3002`)

---

## Step 1 — Create monitoring directory

```bash
mkdir -p ~/monitoring && cd ~/monitoring
```

---

## Step 2 — Create `prometheus.yml`

```bash
nano prometheus.yml
```

Paste the following:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    metric_relabel_configs:
      - source_labels: [container_label_com_docker_compose_service]
        regex: (.+)
        target_label: name
      - source_labels: [container_label_com_docker_compose_project]
        regex: (.+)
        target_label: compose_project
      - regex: 'container_label_.*'
        action: labeldrop
```

> The `metric_relabel_configs` section extracts clean container names from Docker Compose labels and drops the noisy raw labels.

---

## Step 3 — Create `docker-compose.yml`

```bash
nano docker-compose.yml
```

Paste the following:

```yaml
networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:

services:

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    command:
      - '--docker_only=true'
      - '--store_container_labels=true'
    volumes:
      - /:/rootfs:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=changeme123   # Change this!
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - "3002:3000"   # Change 3002 to any free port on your VPS
    networks:
      - monitoring
```

> **Important notes:**
> - Use `gcr.io/cadvisor/cadvisor:v0.47.2` — newer versions (v0.55+) cannot connect to snap Docker
> - Mount `/var/run/docker.sock` directly (not `/var/run:/var/run`) — snap Docker's read-only filesystem blocks the latter
> - Do NOT mount `/var/lib/docker` — it's read-only on snap Docker VPS
> - Change the Grafana port (`3002`) if already in use. Check with: `sudo ss -tlnp | grep PORT`

---

## Step 4 — Start the stack

```bash
docker compose up -d
```

Verify all 4 containers are running:

```bash
docker compose ps
```

Expected output — all should show `Up`:
```
cadvisor        Up    0.0.0.0:8080->8080/tcp
grafana         Up    0.0.0.0:3002->3000/tcp
node-exporter   Up    9100/tcp
prometheus      Up    0.0.0.0:9090->9090/tcp
```

---

## Step 5 — Verify everything is working

**Check cAdvisor sees your containers:**
```bash
curl -s "http://localhost:8080/metrics" | grep container_memory_usage_bytes | grep 'name=' | head -5
```

**Check Prometheus is scraping all targets:**
```bash
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -E "health|job"
```
All targets should show `"health": "up"`.

**Check container metrics are in Prometheus:**
```bash
curl -s 'http://localhost:9090/api/v1/query?query=container_memory_usage_bytes{name!=""}' | python3 -m json.tool | grep '"name"'
```
You should see your container names like `pgadmin`, `backend`, `postgres`, etc.

---

## Step 6 — Configure Grafana

1. Open browser: `http://YOUR_VPS_IP:3002`
2. Login: `admin` / `changeme123`

### Add Prometheus data source

1. Go to **Connections → Data Sources → Add data source**
2. Select **Prometheus**
3. Set URL to `http://prometheus:9090`
4. Click **Save & Test** — should show green "Successfully queried"

### Import dashboards

Go to **Dashboards → New → Import**, paste each ID, click **Load**, select **Prometheus** as data source, click **Import**:

| Dashboard | ID | Shows |
|---|---|---|
| Node Exporter Full | `1860` | Host CPU, RAM, disk, network |
| cAdvisor Docker Insights | `19908` | Per-container CPU, RAM, network |

---

## Troubleshooting

### cAdvisor shows only root `/` container, no app containers
- Make sure you're using `v0.47.2`, not `latest`
- Ensure `/var/run/docker.sock:/var/run/docker.sock:ro` is mounted
- Check: `docker logs cadvisor | grep -i "docker\|registration"`

### Port already in use error
```bash
# Find free ports
for port in 3000 3001 3002 3003 4000; do
  sudo ss -tlnp | grep ":$port " && echo "PORT $port IN USE" || echo "PORT $port FREE"
done
```

### Grafana dashboard shows N/A
- Verify Prometheus target is UP: `curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep health`
- Wait 1-2 minutes after startup for first scrape
- Check dashboard variable dropdowns at the top — select the correct instance

### Read-only filesystem errors
- Do NOT mount `/var/lib/docker` — snap Docker stores it at `/var/snap/docker/common/var-lib-docker`
- Do NOT mount `/var/run:/var/run` as a directory — mount the socket file directly

---

## File Structure

```
~/monitoring/
├── docker-compose.yml
└── prometheus.yml
```

---

## Useful Commands

```bash
# Start stack
docker compose up -d

# Stop stack
docker compose down

# Restart a single service
docker compose restart prometheus

# View logs
docker logs cadvisor --tail 50
docker logs prometheus --tail 50
docker logs grafana --tail 50

# Check all container status
docker compose ps

# Check metrics manually
curl http://localhost:8080/metrics | head -20   # cAdvisor
curl http://localhost:9090/api/v1/targets       # Prometheus targets
```
