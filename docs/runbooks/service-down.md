# Runbook: ServiceDown

**Alert**: `ServiceDown`  
**Severity**: critical  
**Threshold**: API returns no successful responses for 2+ minutes

---

## Symptoms

- Grafana: "Service Availability" panel at 0%
- Prometheus: `up{job="todo-app-api"} == 0`
- All API requests returning 502/503/504
- Slack alert in `#critical`

---

## Immediate Actions

```bash
# 1. Check pod status immediately
kubectl get pods -n todo-app

# 2. Check if the service endpoint is reachable
kubectl get svc -n todo-app
kubectl get httproute -n todo-app

# 3. Try hitting health endpoint directly
kubectl port-forward svc/todo-app-api 3000:3000 -n todo-app &
curl -s http://localhost:3000/api/health | jq .
```

---

## Root Cause Investigation

### 1. All pods down / crashing

```bash
kubectl describe pods -n todo-app | grep -E 'Status|Reason|Message'
kubectl get events -n todo-app --sort-by='.lastTimestamp'
```

→ See [crashloop-backoff.md](crashloop-backoff.md) or [pod-oom-kill.md](pod-oom-kill.md)

### 2. Deployment scaled to 0

```bash
kubectl get deployment -n todo-app
# Check READY column — should be 1/1 or 2/2
```

If replicas are 0:
```bash
kubectl scale deployment todo-app-api --replicas=1 -n todo-app
# Then investigate why it was scaled down
```

Check HPA:
```bash
kubectl get hpa -n todo-app
cat helm/api/values.yaml | grep -A10 autoscaling
```

### 3. Gateway routing broken

```bash
kubectl get httproute -n todo-app -o yaml
kubectl describe gateway -n todo-app
kubectl get pods -n kube-system | grep traefik
```

Check `helm/umbrella/templates/gateway.yaml` — if the HTTPRoute was recently changed, revert.

### 4. Wrong image tag

```bash
kubectl get deployment todo-app-api -n todo-app -o jsonpath='{.spec.template.spec.containers[0].image}'
```

If the image tag doesn't exist in GHCR, the pod can't start. Check:
```bash
kubectl describe pod <pod-name> -n todo-app | grep -A5 "Failed to pull"
```

Fix: update `image.tag` in `helm/api/values.yaml` to a valid tag from GHCR.

---

## Fix Checklist

- [ ] Confirm pods are running: `kubectl get pods -n todo-app`
- [ ] If CrashLoop: follow [crashloop-backoff.md](crashloop-backoff.md)
- [ ] If OOMKilled: follow [pod-oom-kill.md](pod-oom-kill.md)
- [ ] If scaled to 0: scale up and investigate cause
- [ ] If gateway issue: check HTTPRoute and Traefik logs
- [ ] If bad image: update `image.tag` in `helm/api/values.yaml`
- [ ] Confirm `/api/health` returns 200 before closing incident
