# Operations Setup

This directory contains configurations for the Todo application infrastructure and monitoring.

## Directory Structure
```
todo-app-ops/
├── helm/                    # Kubernetes configurations
│   └── todo-app/           # Main application Helm chart
├── prometheus/             # Local Prometheus setup
│   ├── prometheus.yml.template      # Prometheus configuration template
│   ├── alertmanager.yml.template    # Alertmanager configuration template
│   ├── .env                        # Environment variables (gitignored)
│   └── rules/              # Alert rules
│       └── todo-app.yml    # Application-specific alert rules
└── README.md
```

## Prerequisites
1. [Rancher Desktop](https://rancherdesktop.io/) for local Kubernetes
2. [Prometheus](https://prometheus.io/download/) for metrics collection
3. [Alertmanager](https://prometheus.io/download/#alertmanager) for alert management

## Kubernetes Setup

### 1. NGINX Ingress Controller
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.watchIngressWithoutClass=true
```

### 2. Application Deployment
```bash
# Add to hosts file (C:\Windows\System32\drivers\etc\hosts):
# 127.0.0.1 nextjs.local

helm install todo-app ./helm/todo-app --namespace todo-app --create-namespace
```

## Monitoring Setup

### 1. Prometheus Configuration
1. Extract Prometheus to `prometheus/` directory
2. Create `.env` file in the prometheus directory:
   ```bash
   PROMETHEUS_TARGET_URL=your-target-url
   ALERTMANAGER_WEBHOOK_URL=your-webhook-url
   ```
3. Generate configuration files from templates:
   ```powershell
   cd prometheus
   # Using envsubst (if available)
   envsubst < prometheus.yml.template > prometheus.yml
   envsubst < alertmanager.yml.template > alertmanager.yml
   
   # OR using sed (alternative)
   sed "s/\${PROMETHEUS_TARGET_URL}/$PROMETHEUS_TARGET_URL/g" prometheus.yml.template > prometheus.yml
   sed "s/\${ALERTMANAGER_WEBHOOK_URL}/$ALERTMANAGER_WEBHOOK_URL/g" alertmanager.yml.template > alertmanager.yml
   ```
4. Start Prometheus:
```powershell
cd prometheus
.\prometheus.exe --config.file=prometheus.yml
```

### 2. Alertmanager
1. Extract Alertmanager to `prometheus/` directory
2. Start Alertmanager:
```powershell
cd prometheus
.\alertmanager.exe --config.file=alertmanager.yml
```

## Accessing Services
- Application: http://nextjs.local
- Prometheus: http://localhost:9090
- Alertmanager: http://localhost:9093

## Common Commands

### Kubernetes
```bash
# Update application
helm upgrade todo-app ./helm/todo-app -n todo-app

# Uninstall application
helm uninstall todo-app -n todo-app

# Check ingress status
kubectl get ingress -n todo-app
```

### Monitoring
```powershell
# Reload Prometheus config
Invoke-RestMethod -Method POST http://localhost:9090/-/reload

# Reload Alertmanager config
Invoke-RestMethod -Method POST http://localhost:9093/-/reload

# View Prometheus targets
Start-Process "http://localhost:9090/targets"

# View active alerts
Start-Process "http://localhost:9090/alerts"
```

## Security Notes
- Never commit the actual `.env` file or generated configuration files
- Keep your webhook URLs and target URLs secure
- Regularly rotate sensitive credentials and URLs
- Use HTTPS for all external communications
- For production, use environment variables or a secrets management system:
  - HashiCorp Vault
  - AWS Secrets Manager
  - Kubernetes Secrets
  - Or your preferred secrets management solution

## Troubleshooting
- If ingress is not working, check NGINX controller:
  ```bash
  kubectl get pods -n ingress-nginx
  ```
- If metrics are missing, verify Prometheus targets:
  ```bash
  curl http://localhost:9090/api/v1/targets
  ```
- For alert issues, check Alertmanager status:
  ```bash
  curl http://localhost:9093/api/v1/status
  ```