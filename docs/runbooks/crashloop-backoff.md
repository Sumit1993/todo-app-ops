# Runbook: CrashLoopBackOff

**Alert**: `PodCrashLooping`  
**Severity**: critical  
**Service**: todo-app (api variants)

---

## Symptoms

- Pod status: `CrashLoopBackOff` — `kubectl get pods -n todo-app`
- Prometheus: `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} > 0`
- Restart count > 3 within 10 minutes
- Service returning 502/503

---

## Immediate Actions

```bash
# See why pod is crashing
kubectl describe pod <pod-name> -n todo-app
kubectl logs <pod-name> -n todo-app --previous

# Check events
kubectl get events -n todo-app --sort-by='.lastTimestamp' | tail -20
```

---

## Root Cause Investigation

CrashLoopBackOff has three common causes. Check in this order:

### 1. Readiness/liveness probe timing mismatch

The most common cause when deploying a **different language** or after a probe config change.

```bash
cat helm/api/values.yaml | grep -A15 readinessProbe
cat helm/api/values.yaml | grep -A15 livenessProbe
```

Required minimum probe delays per language:

| Language | `readinessProbe.initialDelaySeconds` | `livenessProbe.initialDelaySeconds` |
|----------|--------------------------------------|--------------------------------------|
| NestJS | 5s | 15s |
| FastAPI | 3s | 10s |
| Spring Boot | **30s** | **60s** |
| Go | 1s | 5s |

Spring Boot is the most common culprit — JVM startup takes 8–15s for class loading and bean initialization. Using NestJS-calibrated probe timing (5s) will kill the Spring Boot pod before it finishes starting.

**Fix**: Update probe delays in `helm/api/values.yaml` to match the deployed language, then:
```bash
helm upgrade todo-app helm/umbrella -n todo-app -f helm/umbrella/values.yaml
```

### 2. Missing or wrong environment variable

```bash
# Check what env vars the container sees
kubectl exec -it <running-pod> -n todo-app -- env | grep -E 'DATABASE|REDIS|NODE_ENV'

# Compare against expected config
cat helm/api/values.yaml | grep -A30 env
kubectl get configmap -n todo-app
kubectl get secret -n todo-app
```

### 3. Database not reachable at startup

```bash
# Check if the DB pod is ready
kubectl get pods -n data

# Check initContainer logs (Prisma/Flyway migration)
kubectl logs <pod-name> -n todo-app -c migrate
```

---

## Fix Checklist

- [ ] Identify crash reason from `kubectl logs --previous`
- [ ] If probe timing: update `initialDelaySeconds` in `helm/api/values.yaml`
- [ ] If missing env: check `helm/api/values.yaml` env section and secrets
- [ ] If DB issue: verify `postgresql` pod is Running in `data` namespace
- [ ] Apply Helm upgrade and watch pod reach `Running/Ready`
