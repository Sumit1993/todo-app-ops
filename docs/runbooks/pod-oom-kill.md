# Runbook: PodOOMKilled

**Alert**: `PodOOMKilled`  
**Severity**: critical  
**Service**: todo-app (api, ui)

---

## Symptoms

- Pod status shows `OOMKilled` — `kubectl get pods -n todo-app`
- Restart count increasing in Grafana: "Pod Restart Count" panel
- 502/503 errors during restart window

---

## Immediate Actions

```bash
# Confirm OOMKill reason
kubectl describe pod <pod-name> -n todo-app | grep -A5 "Last State"

# Check restart count
kubectl get pods -n todo-app

# View recent logs before crash
kubectl logs <pod-name> -n todo-app --previous | tail -50
```

---

## Root Cause Investigation

### 1. Check current memory limits

```bash
cat helm/api/values.yaml | grep -A10 resources
```

Minimum viable limits per language:

| Language | Minimum `limits.memory` | Why |
|----------|------------------------|-----|
| NestJS (Node.js) | `256Mi` | V8 heap needs headroom |
| FastAPI (Python) | `128Mi` | Per-worker baseline |
| Spring Boot (Java) | `512Mi` | JVM heap + Metaspace |
| Go (Gin) | `64Mi` | Goroutine stacks |

### 2. Compare against observed usage

Check Grafana → "Todo App — Resource Usage" → "Container Memory RSS".
If usage was approaching the limit before the kill, the limit needs to be raised.

### 3. Verify language-specific memory flags

For **Java**: confirm `JAVA_OPTS` in `helm/api/values.yaml` or the app Dockerfile includes:
```
-XX:+UseContainerSupport -XX:MaxRAMPercentage=75
```
Without `UseContainerSupport`, JVM reads host RAM (24GB on OCI), allocates 6GB heap, and OOMKills immediately.

For **Node.js**: confirm `--max-old-space-size=192` in the start command if limit is 256Mi.

---

## Fix

Update `helm/api/values.yaml`:

```yaml
resources:
  requests:
    memory: 128Mi
  limits:
    memory: 256Mi   # increase this value
```

Apply:
```bash
helm upgrade todo-app helm/umbrella -n todo-app -f helm/umbrella/values.yaml
```

Monitor for 10 minutes — confirm pod stays `Running` with restart count stable.

---

## Escalation

If OOMKill continues after limit increase, suspect a memory leak. Engage the backend team with:
- Heap dump (Node: `--heapsnapshot-signal=SIGUSR2`)
- Go pprof output: `curl http://<pod-ip>:6060/debug/pprof/heap`
