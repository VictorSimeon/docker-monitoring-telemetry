# Despliegue de Stack de Monitorización (Prometheus + Grafana)

Entorno práctico contenedorizado mediante **Docker Compose** para demostrar la recopilación de telemetría basada en pull (extracción) y su visualización en tiempo real (desarrollado para un seminario técnico en la ETSIIT – UGR).

El objetivo de esta maqueta es demostrar en local el modelo de recolección de métricas por *scraping* (**Pull**) y su visualización en tiempo real.

---

## Arquitectura del Entorno

| Servicio | Imagen Docker | Puerto Local | Función |
| :--- | :--- | :---: | :--- |
| **Node Exporter** | `prom/node-exporter:latest` | `9100` | Expone métricas de hardware y kernel Linux en `/metrics`. |
| **Prometheus** | `prom/prometheus:latest` | `9090` | Recolecta y almacena series temporales cada 5 segundos. |
| **Grafana** | `grafana/grafana:latest` | `3000` | Visualización interactiva mediante paneles (*dashboards*). |

---

## Requisitos Previos

* Sistema operativo Linux (probado en **Ubuntu** / **Debian**).
* **Docker Engine** y plugin **Docker Compose** (`docker compose version`).

### Instalación rápida de Docker (si no está instalado)

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```

---

## Archivos de Configuración

### 1. `prometheus.yml`
Define el intervalo de recolección (`scrape_interval`) y los objetivos monitorizados a través de la red interna de Docker:

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
Orquesta los tres contenedores montando en modo solo lectura (`/proc`, `/sys`, `/`) los directorios del anfitrión para que `node-exporter` pueda leer las métricas reales del sistema:

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

## Puesta en Marcha y Verificación

1. **Levantar los contenedores en segundo plano:**
   ```bash
   docker compose up -d
   docker ps
   ```

2. **Comprobar recolección de métricas:**
   * **Métricas en bruto:** `http://localhost:9100/metrics`
   * **Estado de los targets en Prometheus:** `http://localhost:9090/targets` (deben figurar en estado `UP`).

3. **Configurar Grafana:**
   * Acceder a `http://localhost:3000` (credenciales por defecto: `admin` / `admin`).
   * Añadir origen de datos en **Connections → Data sources → Prometheus** apuntando a `http://prometheus:9090`.
   * Importar el panel oficial **Node Exporter Full** en **Dashboards → New → Import** usando el ID `1860`.

---

## Prueba de Carga y Apagado

Para generar un pico controlado de CPU y observar su impacto en las gráficas de Grafana:

```bash
yes > /dev/null &
sleep 15
killall yes
```

Para detener y eliminar los contenedores al finalizar:

```bash
docker compose down
```
