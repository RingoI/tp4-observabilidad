# TP4 - Observabilidad: GameCloud en Kubernetes

Administración Avanzada de Redes y Servidores (UNNOBA).
Stack de observabilidad sobre minikube con Prometheus, Grafana, Loki y Grafana Alloy,
usado para monitorear la API de ranking de GameCloud y diagnosticar fallas.

## Estructura

| Carpeta | Contenido |
|---|---|
| `app/` | API de ranking (provista por la cátedra) |
| `charts/gamecloud-ranking/` | Chart de Helm de la API, con el `servicemonitor.yaml` agregado |
| `observabilidad/` | Archivos de valores de kube-prometheus-stack, Loki y Alloy |
| `dashboards/` | Dashboard de Grafana exportado como JSON |
| `escenarios/` | Scripts de fallas (provistos) |

## Requisitos

Docker, minikube, kubectl y Helm. Al menos 12 GB de RAM en el equipo.

## Reproducir la práctica

### 1. Entorno

```bash
minikube start --cpus 4 --memory 8192 --driver docker
kubectl get nodes

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create namespace observabilidad
kubectl create namespace gamecloud
```

### 2. Prometheus y Grafana

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version 92.11.0 -n observabilidad \
  -f observabilidad/values-kube-prometheus-stack.yaml --wait --timeout 10m
```

### 3. Loki y Alloy

```bash
helm upgrade --install loki grafana/loki --version 7.5.0 -n observabilidad \
  -f observabilidad/values-loki.yaml --wait --timeout 10m

helm upgrade --install alloy grafana/alloy --version 1.14.2 -n observabilidad \
  -f observabilidad/values-alloy.yaml --wait --timeout 10m
```

### 4. API y ServiceMonitor

```bash
minikube image build -t gamecloud-ranking:1.0 app/

helm upgrade --install gamecloud-ranking charts/gamecloud-ranking -n gamecloud --wait
```

El chart incluye `templates/servicemonitor.yaml`, con el label `release: kube-prometheus-stack`
para que el Prometheus instalado lo seleccione.

### 5. Acceso a Prometheus y Grafana

```bash
kubectl port-forward -n observabilidad svc/kube-prometheus-stack-prometheus 9090:9090
kubectl port-forward -n observabilidad svc/kube-prometheus-stack-grafana 3001:80
```

Usuario de Grafana: `admin`. La contraseña:

```bash
kubectl get secret -n observabilidad kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

El datasource de Loki queda configurado desde el archivo de valores.
Para cargar el dashboard: Dashboards > New > Import, y subir `dashboards/gamecloud-dashboard.json`.

### 6. Escenarios de falla

Activar un escenario por vez, esperar 1 o 2 minutos y volver a la normalidad antes del siguiente:

```bash
bash escenarios/1_errores.sh
bash escenarios/normal.sh

bash escenarios/2_latencia.sh
bash escenarios/normal.sh

bash escenarios/3_crashloop.sh
bash escenarios/normal.sh

bash escenarios/4_puerto_service.sh
bash escenarios/normal.sh
```

## Decisiones de configuración

- Alertmanager deshabilitado.
- Controller-manager, scheduler y etcd deshabilitados en kube-prometheus-stack: en minikube
  exponen sus métricas solo en 127.0.0.1 del nodo y Prometheus no puede scrapearlos.
- Loki en modo SingleBinary, almacenamiento en filesystem y sin multi-tenancy.
- Alloy agrega los labels `namespace`, `pod`, `container` y `app`, y para la API extrae `level` del JSON.
