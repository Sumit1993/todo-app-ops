# Ops Configuration

This directory contains Kubernetes Helm configurations for the Todo application.

## Structure
- `helm/`: Contains Helm chart for the application
  - `ingress/`: Main Helm chart
    - `templates/`: Contains Kubernetes template files
    - `values.yaml`: Default configuration values
    - `Chart.yaml`: Chart metadata

## Prerequisites
### Setting up NGINX Ingress Controller
Detailed instructions can be found at: https://docs.rancherdesktop.io/how-to-guides/setup-NGINX-Ingress-Controller/

1. Deploy the NGINX ingress controller via helm or kubectl:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.2/deploy/static/provider/cloud/deploy.yaml
```
```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

2. Verify the installation:
```bash
kubectl get pods -n ingress-nginx
```

## Application Setup
1. Add the following to your hosts file (`C:\Windows\System32\drivers\etc\hosts`):
```
127.0.0.1 nextjs.local
```

2. Install the Helm chart:
```bash
helm install ingress ./helm/ingress --namespace todo-app
```

3. To upgrade the deployment:
```bash
helm upgrade ingress ./helm/ingress  --namespace todo-app
```

4. To uninstall:
```bash
helm uninstall ingress --namespace todo-app
```

## Troubleshooting
- If ingress is not working, ensure NGINX Ingress Controller is running:
```bash
kubectl get pods -n ingress-nginx
```
- Check ingress status:
```bash
kubectl get ingress -n ingress
```
- View ingress logs:
```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

# Todo App Operations Guide

## Monitoring Setup

### Prerequisites
- Kubernetes cluster
- Helm v3.x installed
- kubectl configured with cluster access

### Installing Prometheus Stack

The monitoring setup uses the kube-prometheus-stack which includes:
- Prometheus (metrics database)
- Alertmanager (handling alerts)
- Grafana (visualization)
- Various exporters for Kubernetes monitoring

```bash
# Navigate to the monitoring chart directory
cd helm/monitoring

# Update Helm dependencies
helm dependency update

# Install the monitoring stack
helm install monitoring . --create-namespace --namespace monitoring

# Verify the installation
kubectl get pods -n monitoring
```

To upgrade the monitoring stack:
```bash
helm upgrade monitoring . -n monitoring --namespace monitoring
```

To uninstall:
```bash
helm uninstall monitoring -n monitoring --namespace monitoring
```

### Monitoring Components

#### ServiceMonitor
The ServiceMonitor is configured to scrape metrics from the Todo App API:
- Endpoint: `/metrics`
- Scrape interval: 15 seconds
- Target service labels: `app: todo-app-api`
- Port: `http`

#### Prometheus Rules
The following alerts are configured:

1. **High Error Rate Alert**
   - Triggers when error rate exceeds 10% for 5 minutes
   - Severity: Critical

2. **High Memory Usage Alert**
   - Triggers when memory usage exceeds 1.5GB for 5 minutes
   - Severity: Warning

3. **High Request Latency Alert**
   - Triggers when 95th percentile latency exceeds 2 seconds
   - Severity: Warning

4. **High Request Rate Alert**
   - Triggers when request rate exceeds 100 requests/second
   - Severity: Warning

5. **Service Down Alert**
   - Triggers when service is unreachable for 1 minute
   - Severity: Critical

### Accessing Monitoring Tools

```bash
# Port forward Prometheus UI
 kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring

# Port forward Grafana
kubectl port-forward svc/monitoring-grafana 3000:80
# Default Grafana credentials:
# username: admin
# password: prom-operator
```

### Metrics Available
The Todo App API exposes the following metrics:
- `http_requests_total`: Total number of HTTP requests
- Default Node.js metrics (memory, CPU, etc.)
- Process metrics for memory leak detection

### Verifying the Setup

1. Check if ServiceMonitor is working:
```bash
kubectl get servicemonitors
```

2. Verify Prometheus targets:
- Access Prometheus UI at `localhost:9090`
- Go to Status -> Targets
- Look for `todo-app-api` endpoints

3. Check if rules are loaded:
- In Prometheus UI, go to Status -> Rules
- Look for `todo-app-api.rules` group

### Troubleshooting

1. If metrics are not being scraped:
- Verify the service labels match ServiceMonitor selector
- Check if metrics endpoint is accessible
- Verify port names in the service definition

2. If alerts are not firing:
- Check PromQL expressions in Prometheus UI
- Verify alert rules are loaded
- Check Alertmanager configuration

### Best Practices

1. **Alert Thresholds**
   - Adjust thresholds based on your application's normal behavior
   - Monitor false positives and tune accordingly

2. **Memory Management**
   - Keep an eye on memory usage trends
   - Investigate any sustained increases in memory usage

3. **Response Time**
   - Monitor latency patterns during peak hours
   - Set up different thresholds for different endpoints if needed