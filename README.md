# Monitoring Stack Deployment (Prometheus + Grafana)
 
Containerized hands-on environment built with **Docker Compose**.
 
The goal of this setup is to demonstrate locally the metric collection model based on *scraping* (**Pull**) and its real-time visualization.
 
---
 
## Environment Architecture
 
| Service | Docker Image | Local Port | Function |
| :--- | :--- | :---: | :--- |
| **Node Exporter** | `prom/node-exporter:latest` | `9100` | Exposes Linux hardware and kernel metrics at `/metrics`. |
| **Prometheus** | `prom/prometheus:latest` | `9090` | Collects and stores time series every 5 seconds. |
| **Grafana** | `grafana/grafana:latest` | `3000` | Interactive visualization through dashboards. |
 
---
 
## Prerequisites
 
* Linux operating system (tested on **Ubuntu** / **Debian**).
* **Docker Engine** and the **Docker Compose** plugin (`docker compose version`).
### Quick Docker installation (if not already installed)
 
```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
 
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
 
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```
 
---
 
## Configuration Files
 
### 1. `prometheus.yml`
Defines the collection interval (`scrape_interval`) and the monitored targets through Docker's internal network:
 
```yaml
global:
  scrape_interval: 5s
 
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
 
  - job_name: 'nodo_linux'
    static_configs:
      - targets: ['node-exporter:9100']
```
 
### 2. `docker-compose.yml`
Orchestrates the three containers, mounting the host directories (`/proc`, `/sys`, `/`) in read-only mode so that `node-exporter` can read the real system metrics:
 
```yaml
services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
 
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
 
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
```
 
---
 
## Startup and Verification
 
1. **Start the containers in the background:**
```bash
docker compose up -d
docker ps
```
 
2. **Check metric collection:**
   * **Raw metrics:** `http://localhost:9100/metrics`
   * **Target status in Prometheus:** `http://localhost:9090/targets` (they should show as `UP`).
3. **Configure Grafana:**
   * Go to `http://localhost:3000` (default credentials: `admin` / `admin`).
   * Add a data source under **Connections → Data sources → Prometheus** pointing to `http://prometheus:9090`.
   * Import the official **Node Exporter Full** dashboard under **Dashboards → New → Import** using ID `1860`.
---
 
## Load Test and Shutdown
 
To generate a controlled CPU spike and observe its impact on the Grafana charts:
 
```bash
yes > /dev/null &
sleep 15
killall yes
```
 
To stop and remove the containers when finished:
 
```bash
docker compose down
```
 
