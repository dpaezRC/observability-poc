# 🚀 Infrastructure Monitoring Stack - Plan de Implementación

> Stack completo de observabilidad production-grade con cAdvisor + Prometheus + Grafana + Alertmanager

---

## 📁 Estructura del Proyecto

```
infrastructure-monitoring-stack/
├── docker-compose.yml
├── prometheus.yml
├── prometheus-rules.yml
├── alertmanager.yml
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── datasource.yml
│   │   └── dashboards/
│   │       └── dashboard.yml
│   └── dashboards/
│       ├── container-overview.json
│       └── api-performance.json
├── demo-api/
│   ├── Dockerfile
│   ├── package.json
│   ├── server.js
│   └── .dockerignore
├── scripts/
│   ├── stress-test.sh
│   └── cleanup.sh
├── .env
├── .gitignore
└── README.md
```

---

## 🎯 Roadmap de Implementación

### **Día 1: Core Stack (3-4 horas)**

#### Tarea 1.1: Docker Compose Stack
**Archivo:** `docker-compose.yml`

**Servicios a configurar:**
- ✅ **demo-api**: API Node.js con endpoints de prueba (puerto 3001)
- ✅ **cadvisor**: Monitoreo de contenedores (puerto 8080)
- ✅ **prometheus**: Time-series database (puerto 9090)
- ✅ **alertmanager**: Sistema de alertas (puerto 9093)
- ✅ **grafana**: Visualización (puerto 3000)

**Configuraciones críticas:**
```yaml
# Network compartida para todos los servicios
networks:
  monitoring:
    driver: bridge

# Volumes persistentes
volumes:
  prometheus-data:
  alertmanager-data:
  grafana-data:

# Health check ejemplo (demo-api)
healthcheck:
  test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/health"]
  interval: 10s
  timeout: 5s
  retries: 3

# cAdvisor necesita acceso privilegiado
privileged: true
volumes:
  - /:/rootfs:ro
  - /var/run:/var/run:ro
  - /sys:/sys:ro
  - /var/lib/docker/:/var/lib/docker:ro
  - /dev/disk/:/dev/disk:ro
devices:
  - /dev/kmsg
```

---

#### Tarea 1.2: Demo API
**Archivos:** `demo-api/package.json`, `demo-api/server.js`, `demo-api/Dockerfile`

**Endpoints a implementar:**

```javascript
GET  /health
// Response: { status: "ok", uptime: process.uptime(), timestamp: Date.now() }

GET  /metrics
// Formato Prometheus (usar prom-client)

POST /cpu-intensive
// Body: { iterations: 35 }
// Calcula Fibonacci recursivo para generar carga CPU

POST /memory-leak
// Body: { sizeMB: 100, durationSeconds: 60 }
// Crea array que crece progresivamente

POST /slow-query
// Body: { delayMs: 5000 }
// Simula query lenta con setTimeout

GET  /stats
// Response: { cpu: cpuUsage(), memory: memoryUsage(), connections: activeConns }
```

**Dependencies necesarias:**
```json
{
  "express": "^4.18.2",
  "prom-client": "^15.1.0"
}
```

**Dockerfile (multi-stage build):**
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY server.js .
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

**Métricas custom a exponer:**
- `http_requests_total` (Counter): Total requests por endpoint
- `http_request_duration_seconds` (Histogram): Latencia por endpoint
- `active_connections` (Gauge): Conexiones activas
- `memory_leak_size_bytes` (Gauge): Tamaño del leak simulado

---

#### Tarea 1.3: Configuración Prometheus
**Archivo:** `prometheus.yml`

**Scrape configs:**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "rules.yml"

scrape_configs:
  # Self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Container metrics
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  # Application metrics
  - job_name: 'demo-api'
    static_configs:
      - targets: ['demo-api:3000']
    metrics_path: '/metrics'
```

**Verificación:**
```bash
# Levantar stack
docker compose up -d

# Verificar targets en Prometheus
curl http://localhost:9090/api/v1/targets

# Deben aparecer 3 targets: prometheus, cadvisor, demo-api (todos UP)
```

---

### **Día 2: Dashboards y Visualización (3 horas)**

#### Tarea 2.1: Grafana Datasource
**Archivo:** `grafana/provisioning/datasources/datasource.yml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: "15s"
```

---

#### Tarea 2.2: Dashboard Provisioning
**Archivo:** `grafana/provisioning/dashboards/dashboard.yml`

```yaml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

---

#### Tarea 2.3: Dashboard 1 - Container Overview
**Archivo:** `grafana/dashboards/container-overview.json`

**Paneles a incluir:**

1. **Total Containers Running** (Stat)
   - Query: `count(container_last_seen)`
   - Threshold: Green < 10, Yellow < 20, Red >= 20

2. **CPU Usage per Container** (Time Series)
   - Query: `sum(rate(container_cpu_usage_seconds_total{name!=""}[5m])) by (name)`
   - Unit: Percent (0-100)
   - Legend: `{{name}}`

3. **Memory Usage per Container** (Gauge)
   - Query: `container_memory_usage_bytes{name!=""} / 1024 / 1024`
   - Unit: MB
   - Thresholds: 0-500 (green), 500-1000 (yellow), 1000+ (red)

4. **Network I/O** (Graph)
   - Query RX: `rate(container_network_receive_bytes_total[5m])`
   - Query TX: `rate(container_network_transmit_bytes_total[5m])`
   - Unit: Bytes/sec

5. **Filesystem Usage** (Bar Gauge)
   - Query: `container_fs_usage_bytes / container_fs_limit_bytes * 100`
   - Unit: Percent

**Variables:**
- `container`: `label_values(container_last_seen, name)`

---

#### Tarea 2.4: Dashboard 2 - API Performance
**Archivo:** `grafana/dashboards/api-performance.json`

**Paneles a incluir:**

1. **Request Rate** (Stat + Sparkline)
   - Query: `sum(rate(http_requests_total[5m]))`
   - Unit: req/s

2. **Response Time p95/p99** (Time Series)
   - Query p95: `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))`
   - Query p99: `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))`
   - Unit: seconds

3. **Error Rate** (Graph)
   - Query: `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100`
   - Unit: Percent
   - Alert threshold: > 1%

4. **Requests by Endpoint** (Pie Chart)
   - Query: `sum(rate(http_requests_total[5m])) by (endpoint)`

5. **Active Connections** (Gauge)
   - Query: `active_connections`
   - Threshold: Warn > 50, Critical > 100

6. **Memory Leak Detection** (Time Series)
   - Query: `memory_leak_size_bytes / 1024 / 1024`
   - Unit: MB
   - Color: Red gradient

**Annotations:**
- Agregar annotation layer para mostrar cuando se disparan alertas

---

### **Día 3: Alerting (2-3 horas)**

#### Tarea 3.1: Alerting Rules
**Archivo:** `prometheus-rules.yml`

```yaml
groups:
  - name: container_alerts
    interval: 30s
    rules:
      # Alerta 1: CPU alta
      - alert: HighCPUUsage
        expr: rate(container_cpu_usage_seconds_total{name="demo-api"}[1m]) > 0.8
        for: 2m
        labels:
          severity: warning
          component: demo-api
        annotations:
          summary: "Alto uso de CPU detectado"
          description: "El contenedor {{ $labels.name }} está usando {{ $value | humanizePercentage }} de CPU durante los últimos 2 minutos"

      # Alerta 2: Memoria alta
      - alert: HighMemoryUsage
        expr: (container_memory_usage_bytes{name="demo-api"} / container_spec_memory_limit_bytes{name="demo-api"}) > 0.8
        for: 2m
        labels:
          severity: warning
          component: demo-api
        annotations:
          summary: "Alto uso de memoria"
          description: "Memoria en {{ $value | humanizePercentage }} - posible memory leak"

      # Alerta 3: Contenedor caído
      - alert: ContainerDown
        expr: up{job="demo-api"} == 0
        for: 1m
        labels:
          severity: critical
          component: demo-api
        annotations:
          summary: "Servicio no disponible"
          description: "El contenedor demo-api no responde a health checks"

  - name: api_performance_alerts
    interval: 30s
    rules:
      # Alerta 4: Latencia alta
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Latencia p95 excede 1 segundo"

      # Alerta 5: Error rate alto
      - alert: HighErrorRate
        expr: (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) > 0.05
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "Tasa de errores > 5%"
```

---

#### Tarea 3.2: Alertmanager Configuration
**Archivo:** `alertmanager.yml`

```yaml
global:
  resolve_timeout: 5m

# Configuración de rutas
route:
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  
  # Rutas específicas por severidad
  routes:
    - match:
        severity: critical
      receiver: 'critical-alerts'
      continue: true
    
    - match:
        severity: warning
      receiver: 'warning-alerts'

# Receivers (por ahora webhooks de ejemplo)
receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://localhost:5001/alerts'
        send_resolved: true

  - name: 'critical-alerts'
    webhook_configs:
      - url: 'http://localhost:5001/critical'
        send_resolved: true
    # Aquí podrías agregar:
    # slack_configs:
    #   - api_url: 'YOUR_SLACK_WEBHOOK'
    #     channel: '#alerts-critical'

  - name: 'warning-alerts'
    webhook_configs:
      - url: 'http://localhost:5001/warnings'

# Inhibition rules (suprimir alertas redundantes)
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'component']
```

**Testing de alertas:**
```bash
# Disparar alerta de CPU
curl -X POST http://localhost:3001/cpu-intensive -d '{"iterations": 40}'

# Ver alertas activas
curl http://localhost:9093/api/v1/alerts

# Ver en Grafana
# Dashboard → Alerting → Alert Rules
```

---

### **Día 4: Scripts y Automatización (2 horas)**

#### Tarea 4.1: Stress Test Script
**Archivo:** `scripts/stress-test.sh`

```bash
#!/bin/bash

set -e

API_URL="http://localhost:3001"
GRAFANA_URL="http://localhost:3000"
PROMETHEUS_URL="http://localhost:9090"

echo "🚀 Infrastructure Monitoring - Stress Test Demo"
echo "================================================"

# Verificar que el stack esté corriendo
echo "📡 Verificando servicios..."
for service in demo-api cadvisor prometheus grafana alertmanager; do
  if ! docker ps | grep -q $service; then
    echo "❌ Error: $service no está corriendo"
    echo "Ejecuta: docker compose up -d"
    exit 1
  fi
done
echo "✅ Todos los servicios activos"

# Health check
echo ""
echo "🏥 Health check..."
curl -s $API_URL/health | jq .
sleep 2

# Test 1: CPU Spike
echo ""
echo "⚡ Test 1: CPU Intensive (Fibonacci 38)"
echo "Esto debería disparar HighCPUUsage alert en ~2 minutos"
for i in {1..5}; do
  curl -s -X POST $API_URL/cpu-intensive \
    -H "Content-Type: application/json" \
    -d '{"iterations": 38}' &
done
echo "✅ 5 requests paralelas enviadas"
echo "📊 Monitorea CPU en: $GRAFANA_URL/d/container-overview"

sleep 30

# Test 2: Memory Leak
echo ""
echo "💾 Test 2: Memory Leak (100MB en 60s)"
echo "Esto debería disparar HighMemoryUsage alert"
curl -s -X POST $API_URL/memory-leak \
  -H "Content-Type: application/json" \
  -d '{"sizeMB": 100, "durationSeconds": 60}'
echo "✅ Memory leak iniciado"

sleep 30

# Test 3: Latency Spike
echo ""
echo "⏱️  Test 3: Slow Queries"
for i in {1..10}; do
  curl -s -X POST $API_URL/slow-query \
    -H "Content-Type: application/json" \
    -d '{"delayMs": 3000}' &
done
echo "✅ 10 slow queries enviadas"

echo ""
echo "================================================"
echo "🎯 Demo completada!"
echo ""
echo "📊 Dashboards:"
echo "   Grafana:      $GRAFANA_URL (admin/admin)"
echo "   Prometheus:   $PROMETHEUS_URL"
echo "   Alertmanager: http://localhost:9093"
echo ""
echo "🔔 Alertas activas (espera 2-3 min):"
echo "   curl http://localhost:9093/api/v1/alerts | jq"
echo ""
echo "🧹 Para limpiar: ./scripts/cleanup.sh"
```

**Hacer ejecutable:**
```bash
chmod +x scripts/stress-test.sh
```

---

#### Tarea 4.2: Cleanup Script
**Archivo:** `scripts/cleanup.sh`

```bash
#!/bin/bash

echo "🧹 Limpiando stack de monitoreo..."

# Detener y eliminar contenedores
docker compose down

# Eliminar volúmenes (opcional - descomentar si querés reset completo)
# docker volume rm $(docker volume ls -q | grep monitoring)

# Limpiar imágenes huérfanas
docker image prune -f

echo "✅ Cleanup completado"
echo "Para reiniciar: docker compose up -d"
```

---

#### Tarea 4.3: Variables de Entorno
**Archivo:** `.env`

```bash
# Grafana
GRAFANA_ADMIN_PASSWORD=admin
GF_SECURITY_ADMIN_PASSWORD=admin
GF_USERS_ALLOW_SIGN_UP=false

# Prometheus
PROMETHEUS_RETENTION_TIME=15d
PROMETHEUS_RETENTION_SIZE=10GB

# API
API_PORT=3001
NODE_ENV=production

# Puertos
GRAFANA_PORT=3000
PROMETHEUS_PORT=9090
CADVISOR_PORT=8080
ALERTMANAGER_PORT=9093
```

---

#### Tarea 4.4: .gitignore
**Archivo:** `.gitignore`

```
# Data volumes
*-data/
grafana-data/
prometheus-data/
alertmanager-data/

# Node
node_modules/
npm-debug.log
yarn-error.log

# Environment
.env.local
.env.*.local

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# IDE
.vscode/
.idea/
*.swp
*.swo
```

---

#### Tarea 4.5: .dockerignore (demo-api)
**Archivo:** `demo-api/.dockerignore`

```
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.env.local
```

---

### **Día 5: Documentación Final (2 horas)**

#### Tarea 5.1: README Principal
**Archivo:** `README.md`

```markdown
# 🚀 Infrastructure Monitoring Stack

> Production-grade observability stack basado en tecnologías usadas por Netflix, Spotify y Uber

[![Docker](https://img.shields.io/badge/Docker-20.10+-blue.svg)](https://www.docker.com/)
[![Prometheus](https://img.shields.io/badge/Prometheus-2.40+-orange.svg)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-9.0+-orange.svg)](https://grafana.com/)

---

## 🎯 Overview

Stack completo de monitoreo de infraestructura que incluye:
- **Métricas de contenedores** en tiempo real (cAdvisor)
- **Time-series database** optimizada (Prometheus)
- **Alertas inteligentes** con agrupación y deduplicación (Alertmanager)
- **Dashboards interactivos** pre-configurados (Grafana)
- **API de demostración** con simulación de problemas comunes

### ¿Por qué este stack?

- ✅ **Arquitectura industry-standard**: Mismo stack usado por companies Fortune 500
- ✅ **Zero-config deployment**: Un solo comando para levantar todo
- ✅ **Declarative configuration**: GitOps-ready, reproducible
- ✅ **Escalable**: De 1 a 1000+ contenedores sin cambios arquitectónicos

---

## 🚀 Quick Start (< 3 minutos)

### Prerequisitos
- Docker 20.10+
- Docker Compose 2.0+
- 4GB RAM libre
- Puertos disponibles: 3000, 3001, 8080, 9090, 9093

### Deployment

```bash
# 1. Clonar repositorio
git clone <tu-repo>
cd infrastructure-monitoring-stack

# 2. Levantar stack completo
docker compose up -d

# 3. Verificar que todo esté corriendo
docker compose ps

# 4. Esperar ~30 segundos para inicialización completa
```

### Acceder a las interfaces

| Servicio      | URL                        | Credenciales  |
|---------------|----------------------------|---------------|
| Grafana       | http://localhost:3000      | admin/admin   |
| Prometheus    | http://localhost:9090      | -             |
| Alertmanager  | http://localhost:9093      | -             |
| cAdvisor      | http://localhost:8080      | -             |
| Demo API      | http://localhost:3001      | -             |

---

## 🧪 Demo Interactiva

### Opción 1: Script Automatizado (Recomendado)

```bash
./scripts/stress-test.sh
```

Esto ejecutará:
1. ✅ Health check de todos los servicios
2. ⚡ CPU spike (Fibonacci intensivo)
3. 💾 Memory leak simulado
4. ⏱️ Latency spike (queries lentas)

**Resultado esperado:**
- Alertas `HighCPUUsage` y `HighMemoryUsage` disparan en ~2 minutos
- Dashboards muestran spikes en tiempo real
- Alertmanager agrupa y notifica

### Opción 2: Tests Manuales

```bash
# CPU Intensive
curl -X POST http://localhost:3001/cpu-intensive \
  -H "Content-Type: application/json" \
  -d '{"iterations": 40}'

# Memory Leak
curl -X POST http://localhost:3001/memory-leak \
  -H "Content-Type: application/json" \
  -d '{"sizeMB": 150, "durationSeconds": 90}'

# Slow Query
curl -X POST http://localhost:3001/slow-query \
  -H "Content-Type: application/json" \
  -d '{"delayMs": 5000}'

# Ver alertas activas
curl http://localhost:9093/api/v1/alerts | jq
```

---

## 📊 Dashboards Incluidos

### 1. Container Overview
**Path:** Grafana → Dashboards → Container Overview

**Visualizaciones:**
- 📈 CPU usage per container (time series)
- 💾 Memory usage with limits (gauges)
- 🌐 Network I/O (RX/TX)
- 💿 Filesystem usage (bar chart)
- 🔢 Total containers running (stat)

**Queries destacadas:**
```promql
# CPU por contenedor
sum(rate(container_cpu_usage_seconds_total[5m])) by (name)

# Memoria con porcentaje
container_memory_usage_bytes / container_spec_memory_limit_bytes * 100
```

### 2. API Performance
**Path:** Grafana → Dashboards → API Performance

**Visualizaciones:**
- ⚡ Request rate (req/s)
- ⏱️ Response time p95/p99 (heatmap)
- ❌ Error rate 5xx (graph)
- 🔗 Active connections (gauge)
- 🚨 Memory leak detection (time series)

---

## 🔔 Alertas Configuradas

| Alerta            | Condición                       | Severidad | Duración |
|-------------------|---------------------------------|-----------|----------|
| HighCPUUsage      | CPU > 80%                       | Warning   | 2 min    |
| HighMemoryUsage   | Memory > 80% limit              | Warning   | 2 min    |
| ContainerDown     | Health check fail               | Critical  | 1 min    |
| HighLatency       | p95 latency > 1s                | Warning   | 5 min    |
| HighErrorRate     | Error rate > 5%                 | Critical  | 3 min    |

### Ver alertas activas

```bash
# Via API
curl http://localhost:9093/api/v1/alerts | jq '.data[] | {name: .labels.alertname, status: .status.state}'

# Via UI
open http://localhost:9093
```

---

## 🏗️ Arquitectura

```
┌─────────────┐
│  Demo API   │ ◄─── Simula carga/problemas
└──────┬──────┘
       │ expone /metrics
       ▼
┌─────────────┐      ┌──────────────┐
│  cAdvisor   │──────┤  Prometheus  │ ◄─── Scrape cada 15s
└─────────────┘      └──────┬───────┘
                            │ evalúa rules cada 30s
                            ▼
                     ┌──────────────┐
                     │ Alertmanager │ ◄─── Agrupa/deduplica
                     └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   Grafana    │ ◄─── Visualización
                     └──────────────┘
```

### Flujo de datos

1. **Collection**: cAdvisor scrape métricas de Docker daemon cada 15s
2. **Storage**: Prometheus almacena en TSDB optimizada (retención 15 días)
3. **Evaluation**: Prometheus evalúa alerting rules cada 30s
4. **Alerting**: Alertmanager recibe alerts, agrupa, y notifica
5. **Visualization**: Grafana consulta Prometheus vía PromQL

---

## 💻 Stack Tecnológico

| Componente    | Versión | Propósito                          | Por qué esta tool                    |
|---------------|---------|-------------------------------------|--------------------------------------|
| cAdvisor      | latest  | Métricas de contenedores            | Google-made, native Docker support   |
| Prometheus    | 2.47+   | Time-series database                | Industry standard, PromQL poderoso   |
| Alertmanager  | 0.26+   | Alert routing & grouping            | Integración nativa con Prometheus    |
| Grafana       | 10.0+   | Dashboarding & visualization        | UI superior, plugin ecosystem        |
| Node.js       | 18      | Demo API                            | Lightweight, fácil de instrumentar   |
| prom-client   | 15.1    | Prometheus client library           | Official Node.js client              |

---

## 📈 Métricas Recolectadas

### Métricas de Contenedores (cAdvisor)
- `container_cpu_usage_seconds_total`: CPU time consumido
- `container_memory_usage_bytes`: Memoria en uso
- `container_network_receive_bytes_total`: Bytes recibidos
- `container_network_transmit_bytes_total`: Bytes transmitidos
- `container_fs_usage_bytes`: Uso de filesystem

### Métricas de Aplicación (Demo API)
- `http_requests_total`: Total requests por endpoint/status
- `http_request_duration_seconds`: Latencia por endpoint
- `active_connections`: Conexiones TCP activas
- `memory_leak_size_bytes`: Tamaño del leak simulado

### Queries PromQL útiles

```promql
# Top 5 contenedores por CPU
topk(5, sum(rate(container_cpu_usage_seconds_total[5m])) by (name))

# Memoria total del sistema
sum(container_memory_usage_bytes{id="/"})

# Request rate por endpoint
sum(rate(http_requests_total[5m])) by (endpoint)

# Latencia p99
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

---

## 🛠️ Configuración Avanzada

### Ajustar retención de Prometheus

Editar `docker-compose.yml`:
```yaml
prometheus:
  command:
    - '--storage.tsdb.retention.time=30d'
    - '--storage.tsdb.retention.size=20GB'
```

### Agregar Slack notifications

1. Crear Incoming Webhook en Slack
2. Editar `alertmanager.yml`:

```yaml
receivers:
  - name: 'slack-critical'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts-critical'
        title: '🚨 {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

### Custom dashboards

```bash
# Exportar dashboard desde UI
Grafana UI → Dashboard → Share → Export → Save to file

# Importar dashboard
cp mi-dashboard.json grafana/dashboards/
docker compose restart grafana
```

---

## 🧹 Mantenimiento

### Limpiar y reiniciar
```bash
./scripts/cleanup.sh
docker compose up -d
```

### Ver logs
```bash
# Todos los servicios
docker compose logs -f

# Servicio específico
docker compose logs -f prometheus

# Últimas 100 líneas
docker compose logs --tail=100 demo-api
```

### Backup de datos
```bash
# Prometheus data
docker run --rm -v infrastructure-monitoring-stack_prometheus-data:/data -v $(pwd):/backup alpine tar czf /backup/prometheus-backup.tar.gz /data

# Grafana dashboards
docker cp grafana:/var/lib/grafana/dashboards ./grafana-backup/
```

---

## 🎓 Conceptos Clave

### ¿Qué es Observability?
La habilidad de entender el estado interno de un sistema basándose en sus outputs externos. Los 3 pilares:
1. **Metrics**: Valores numéricos en el tiempo (ej: CPU usage)
2. **Logs**: Eventos discretos (ej: "Request failed")
3. **Traces**: Path de una request a través del sistema

Este proyecto cubre **Metrics** comprehensivamente.

### PromQL Basics
```promql
# Sintaxis básica
metric_name{label="value"}[time_range]

# Rate (velocidad de cambio)
rate(http_requests_total[5m])

# Aggregation
sum(metric) by (label)

# Comparación temporal
metric offset 1h
```

### Alerting Best Practices
- ✅ Alert on symptoms, not causes (ej: "latency alta" no "CPU alta")
- ✅ Incluir contexto en annotations
- ✅ Agrupar alertas relacionadas
- ✅ Definir runbooks para cada alerta
- ❌ No alertar en métricas ruidosas

---

## 🐛 Troubleshooting

### Prometheus no scraped targets

**Síntoma:** Targets en state "DOWN" en http://localhost:9090/targets

**Solución:**
```bash
# Verificar conectividad de red
docker compose exec prometheus wget -O- http://cadvisor:8080/metrics

# Verificar DNS resolution
docker compose exec prometheus nslookup demo-api

# Reiniciar Prometheus
docker compose restart prometheus
```

### Grafana no muestra datos

**Síntoma:** Dashboards vacíos o "No data"

**Checklist:**
1. ¿Datasource configurado? Grafana → Configuration → Data Sources
2. ¿Time range correcto? (últimos 15 min, no últimos 7 días)
3. ¿Prometheus tiene datos? `curl http://localhost:9090/api/v1/query?query=up`
4. ¿Query correcta? Testar en Prometheus UI primero

### Alertas no disparan

**Debugging:**
```bash
# Ver reglas cargadas
curl http://localhost:9090/api/v1/rules | jq '.data.groups[].rules[] | {alert: .name, state: .state}'

# Ver evaluación de regla específica
curl 'http://localhost:9090/api/v1/query?query=ALERT{alertname="HighCPUUsage"}' | jq

# Logs de Alertmanager
docker compose logs alertmanager | grep -i "firing\|resolved"
```

### cAdvisor no muestra contenedores

**Problema común en macOS:** cAdvisor necesita acceso a `/sys` que no existe en Docker Desktop

**Workaround:**
```yaml
# En docker-compose.yml, reducir volumes:
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - /var/lib/docker/:/var/lib/docker:ro
```

---

## 🚀 Próximos Pasos

### Features a agregar

- [ ] **Tracing** con Jaeger/Tempo
- [ ] **Logs** con Loki
- [ ] **Service Discovery** automático
- [ ] **Multi-cluster monitoring**
- [ ] **Custom exporters** (Postgres, Redis, etc)
- [ ] **SLI/SLO tracking**

### Integraciones útiles

```yaml
# Node Exporter (métricas del host)
node-exporter:
  image: prom/node-exporter:latest
  volumes:
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro

# Blackbox Exporter (synthetic monitoring)
blackbox-exporter:
  image: prom/blackbox-exporter:latest
  
# Postgres Exporter
postgres-exporter:
  image: prometheuscommunity/postgres-exporter
  environment:
    DATA_SOURCE_NAME: "postgresql://user:pass@postgres:5432/db"
```

---

## 📚 Referencias

### Documentación Oficial
- [Prometheus Docs](https://prometheus.io/docs/)
- [Grafana Tutorials](https://grafana.com/tutorials/)
- [cAdvisor GitHub](https://github.com/google/cadvisor)
- [Alertmanager Guide](https://prometheus.io/docs/alerting/latest/alertmanager/)

### Recursos de Aprendizaje
- [PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)
- [Grafana Dashboards Library](https://grafana.com/grafana/dashboards/)
- [SRE Book (Google)](https://sre.google/sre-book/table-of-contents/)

### Repositorios Inspiración
- [Awesome Prometheus](https://github.com/roaldnefs/awesome-prometheus)
- [kube-prometheus-stack](https://github.com/prometheus-operator/kube-prometheus)

---

## 🤝 Contribuciones

Este es un proyecto de portfolio/demo, pero sugerencias son bienvenidas:

1. Fork el repo
2. Crear feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit cambios (`git commit -m 'Add some AmazingFeature'`)
4. Push al branch (`git push origin feature/AmazingFeature`)
5. Abrir Pull Request

---

## 📄 Licencia

MIT License - Ver `LICENSE` file para detalles

---

## 👤 Autor

**Tu Nombre**
- Portfolio: [tu-portfolio.com](https://tu-portfolio.com)
- LinkedIn: [linkedin.com/in/tu-perfil](https://linkedin.com/in/tu-perfil)
- GitHub: [@tu-usuario](https://github.com/tu-usuario)

---

## 💡 Preguntas Frecuentes

**P: ¿Cuántos recursos consume el stack?**
R: ~1.5GB RAM, ~2GB disco. En producción, Prometheus escala según retención configurada.

**P: ¿Puedo usar esto en producción?**
R: Con ajustes sí. Agregar: autenticación robusta, SSL/TLS, persistent storage externo, backup automatizado.

**P: ¿Por qué no usar Datadog/New Relic?**
R: SaaS APM es excelente pero caro. Este stack es:
- ✅ Open-source (gratis)
- ✅ Self-hosted (data privacy)
- ✅ Industry-standard (transferible entre empresas)

**P: ¿Cómo escala a 100+ servicios?**
R: 
- Service discovery (Consul, Kubernetes)
- Prometheus federation
- Thanos para long-term storage
- Grafana organizaciones y folders

---

**⭐ Si este proyecto te resultó útil, considerá darle una estrella!**
```

---

## 🎯 Checklist Final de Implementación

### Antes de la demo
- [ ] `docker compose up -d` exitoso
- [ ] Todos los targets UP en Prometheus
- [ ] Grafana carga con datasource configurado
- [ ] Al menos 1 dashboard visible
- [ ] `/scripts/stress-test.sh` ejecuta sin errores
- [ ] Alertas disparan después de stress test

### Para el README del repo
- [ ] Screenshots de dashboards
- [ ] GIF/video de demo (opcional pero impresiona)
- [ ] Badges de CI/CD si los tenés
- [ ] Diagram arquitectura (draw.io, excalidraw)

### Para presentar a recruiters
- [ ] Slide deck de 3-5 slides (opcional)
  - Slide 1: Problema que resuelve
  - Slide 2: Arquitectura high-level
  - Slide 3: Demo en vivo
  - Slide 4: Métricas de impacto
  - Slide 5: Next steps / learnings
- [ ] Demo script practicado (< 5 min)
- [ ] Prepared answers:
  - "¿Por qué esta stack vs alternativas?"
  - "¿Cómo escalarías a producción?"
  - "¿Qué aprendiste construyendo esto?"

---

## 🎬 Script de Demo (2-3 minutos)

```
[Pantalla: Terminal]
"Les voy a mostrar un stack de monitoreo de infraestructura 
que implementé usando las mismas herramientas que Netflix."

$ docker compose up -d
$ docker compose ps
"Con un comando levanto 5 servicios: la API de demo, 
Prometheus para métricas, Grafana para visualización, 
y Alertmanager para alertas."

[Pantalla: Navegador → Grafana]
"Acá tengo dashboards pre-configurados. Este muestra 
containers en tiempo real: CPU, memoria, network."

[Pantalla: Terminal]
$ ./scripts/stress-test.sh
"Ahora simulo carga real: CPU intensivo, memory leak, 
queries lentas."

[Pantalla: Grafana Dashboard]
"Ven cómo el spike aparece instantáneamente. En 2 minutos 
va a disparar una alerta automática."

[Pantalla: Alertmanager]
"Y acá está la alerta: HighCPUUsage. En producción esto 
mandaría notificación a Slack o PagerDuty."

[Cierre]
"Todo el código está en GitHub. Es 100% reproducible 
y declarativo. Lo diseñé para ser fácil de extender 
a escenarios reales con Kubernetes o microservicios."
```

---

¡Éxito con tu proyecto! 🚀
