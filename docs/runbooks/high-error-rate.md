# Runbook: HighTodoAppErrorRate

**Alert**: `HighTodoAppErrorRate`  
**Severity**: warning → critical  
**Threshold**: 5xx error rate > 1% for 1+ minute

---

## Symptoms

- Grafana: "Error Rate" panel above 1%
- Prometheus: `rate(http_requests_total{status=~"5.."}[1m]) / rate(http_requests_total[1m]) > 0.01`
- Users reporting failed requests

---

## Immediate Actions

```bash
# Check pod health
kubectl get pods -n todo-app

# Check recent error logs
kubectl logs deployment/todo-app-api -n todo-app | grep -E 'ERROR|error|500' | tail -30

# Check if DB is healthy
kubectl exec -it <db-pod> -n data -- psql -U todo -d tododb -c 'SELECT 1'
```

---

## Root Cause Investigation

### 1. Application errors (most common)

Check logs for patterns:
```bash
kubectl logs deployment/todo-app-api -n todo-app --since=10m | grep -c '"level":"error"'
```

Common error patterns:
- `PrismaClientKnownRequestError` → database issue (see below)
- `ECONNREFUSED redis://` → Valkey/Redis unreachable
- `Cannot read properties of undefined` → unhandled null in code path (code bug)
- `TimeoutError` → upstream or DB timeout

### 2. Database connection pool exhausted

```bash
# Check Valkey and PostgreSQL pods
kubectl get pods -n data

# Check connection pool metrics in Grafana
# Panel: "Database Connection Pool" → active vs max
```

If pool is exhausted, check `helm/api/values.yaml` for `DATABASE_POOL_SIZE` env var.
Default Prisma pool is 10. Under load this may need to be 20–50.

### 3. Rate limiter misconfigured

```bash
# Check if errors are 429 (rate limit) being counted as 5xx by a misconfigured middleware
kubectl logs deployment/todo-app-api -n todo-app | grep '"statusCode":429' | wc -l
```

### 4. Dependency outage

```bash
# Check all pods in the namespace
kubectl get pods -n todo-app -n data -n monitoring

# External dependencies (if any)
kubectl get httproute -n todo-app
```

---

## Fix Checklist

- [ ] Identify error type from logs (DB, cache, code, config)
- [ ] If DB: verify `postgresql` pod is healthy, check connection pool
- [ ] If cache: verify `valkey` pod is healthy
- [ ] If code bug: check recent commits in `todo-app-api-nestjs` repo
- [ ] If rate limiter: review `RATE_LIMIT_*` env vars in `helm/api/values.yaml`
- [ ] After fix: confirm error rate drops below 0.1% in Grafana
