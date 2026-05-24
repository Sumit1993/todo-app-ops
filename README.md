# todo-app-ops-k8s

Kubernetes infrastructure for the Todo App — owned by the SRE team.

This repo contains Helm charts, Gateway API configuration, Prometheus alert rules, and on-call runbooks for the Todo App platform running on Kubernetes.

---

## Repository Structure

```
todo-app-ops-k8s/
├── helm/
│   ├── api/            # NestJS API Helm chart
│   ├── ui/             # Next.js UI Helm chart
│   ├── postgresql/     # PostgreSQL Helm chart
│   ├── valkey/         # Valkey (Redis-compatible) Helm chart
│   └── umbrella/       # Umbrella chart — deploys the full stack
├── prometheus/
│   ├── prometheus.yml
│   ├── alertmanager.yml
│   └── rules/
│       └── todo-app-rules.yml
├── docs/
│   └── runbooks/
│       ├── service-down.md
│       ├── high-error-rate.md
│       ├── high-latency.md
│       ├── crashloop-backoff.md
│       └── pod-oom-kill.md
└── scripts/
    └── load-test.js
```

---

## Prerequisites

- [Rancher Desktop](https://rancherdesktop.io/) or any CNCF-conformant cluster
- `kubectl` + `helm` v3
- `pnpm` (for load test scripts)

---

## Deploying the Stack

### Full deployment

```bash
helm upgrade --install todo-app helm/umbrella \
  --namespace todo-app \
  --create-namespace \
  -f helm/umbrella/values.yaml
```

### Per-component upgrades

```bash
# API only
helm upgrade todo-app-api helm/api -n todo-app

# UI only
helm upgrade todo-app-ui helm/ui -n todo-app
```

### Tear down

```bash
helm uninstall todo-app -n todo-app
kubectl delete namespace todo-app
```

---

## Routing

Traffic is handled by [Gateway API](https://gateway-api.sigs.k8s.io/) v1.5 with Traefik as the gateway controller.

```bash
kubectl get httproute -n todo-app
kubectl get gateway -n todo-app
```

The umbrella chart installs a `Gateway` and two `HTTPRoute` resources:
- `/api/*` → `todo-app-api` service
- `/*` → `todo-app-ui` service

---

## Monitoring

### Alert rules

Alert definitions live in `prometheus/rules/todo-app-rules.yml`. Active alerts:

| Alert | Severity | Threshold |
|-------|----------|-----------|
| `ServiceDown` | critical | No successful responses for 2 min |
| `HighTodoAppErrorRate` | warning/critical | 5xx rate > 1% for 1 min |
| `HighRequestLatency` | warning | p95 > 2s for 5 min |
| `PodCrashLooping` | critical | CrashLoopBackOff restart count > 3 in 10 min |
| `PodOOMKilled` | critical | OOMKill event on any pod |

Each alert's `runbookUrl` annotation points to the corresponding file in `docs/runbooks/`.

### Applying alert rule changes

```bash
# Reload Prometheus config (no restart needed)
curl -s -X POST http://localhost:9090/-/reload

# Verify rules loaded
curl -s http://localhost:9090/api/v1/rules | jq '.data.groups[].rules[].name'
```

---

## On-Call Runbooks

| Runbook | Alert |
|---------|-------|
| [service-down.md](docs/runbooks/service-down.md) | `ServiceDown` |
| [high-error-rate.md](docs/runbooks/high-error-rate.md) | `HighTodoAppErrorRate` |
| [high-latency.md](docs/runbooks/high-latency.md) | `HighRequestLatency` |
| [crashloop-backoff.md](docs/runbooks/crashloop-backoff.md) | `PodCrashLooping` |
| [pod-oom-kill.md](docs/runbooks/pod-oom-kill.md) | `PodOOMKilled` |

---

## Helm Values Reference

### API resource tuning (`helm/api/values.yaml`)

```yaml
replicaCount: 1

image:
  repository: ghcr.io/todo-corp/todo-app-api-nestjs
  tag: latest

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

readinessProbe:
  initialDelaySeconds: 5
  periodSeconds: 10

livenessProbe:
  initialDelaySeconds: 15
  periodSeconds: 20
```

> For Spring Boot deployments, `initialDelaySeconds` must be increased to at least 30s (readiness) / 60s (liveness). See [crashloop-backoff.md](docs/runbooks/crashloop-backoff.md).

### Scaling

```bash
# Manual scale-out
kubectl scale deployment todo-app-api --replicas=3 -n todo-app

# Or update values and upgrade
# helm/api/values.yaml → replicaCount: 3
helm upgrade todo-app helm/umbrella -n todo-app -f helm/umbrella/values.yaml
```

---

## Namespace Layout

| Namespace | Contents |
|-----------|----------|
| `todo-app` | API, UI deployments |
| `data` | PostgreSQL, Valkey |
| `monitoring` | Prometheus, Alertmanager, Grafana |

---

## Common Kubectl Commands

```bash
# Pod status
kubectl get pods -n todo-app
kubectl get pods -n data

# Logs
kubectl logs deployment/todo-app-api -n todo-app --since=10m
kubectl logs deployment/todo-app-ui -n todo-app --since=10m

# Shell into pod
kubectl exec -it <pod-name> -n todo-app -- sh

# Resource usage
kubectl top pods -n todo-app
kubectl top nodes

# Events (useful for diagnosing CrashLoop/OOM)
kubectl get events -n todo-app --sort-by='.lastTimestamp'
```

---

## Related Repositories

| Repo | Description |
|------|-------------|
| [todo-app-api-nestjs](https://github.com/todo-corp/todo-app-api-nestjs) | NestJS REST API — backend team |
| [todo-app-ui](https://github.com/todo-corp/todo-app-ui) | Next.js frontend — frontend team |
| [todo-app-ops-docker](https://github.com/todo-corp/todo-app-ops-docker) | Docker Compose staging — SRE team |
| [todo-app-ops-vm](https://github.com/todo-corp/todo-app-ops-vm) | Kamal VM fleet — SRE team |
