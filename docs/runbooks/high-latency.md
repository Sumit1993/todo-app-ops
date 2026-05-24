# Runbook: HighRequestLatency

**Alert**: `HighRequestLatency`  
**Severity**: warning  
**Threshold**: p95 response time > 2s for 5+ minutes

---

## Symptoms

- Grafana: "Request Latency p95" above 2000ms
- Users reporting slow responses
- Prometheus: `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2`

---

## Immediate Actions

```bash
# Check if pods are resource-constrained
kubectl top pods -n todo-app

# Check active DB connections
kubectl exec -it <postgresql-pod> -n data -- psql -U todo -d tododb \
  -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# Check error rate (slow requests may be masking errors)
# Grafana → "Error Rate" panel
```

---

## Root Cause Investigation

### 1. CPU throttling

```bash
# Check CPU usage vs limits
kubectl top pods -n todo-app
cat helm/api/values.yaml | grep -A5 resources
```

If CPU usage is near the limit, throttling increases latency. Options:
- Increase `resources.limits.cpu`
- Scale out: increase `replicaCount` in `helm/api/values.yaml`

### 2. Database slow queries

```bash
# Check slow query log
kubectl exec -it <postgresql-pod> -n data -- psql -U todo -d tododb -c "
  SELECT query, calls, mean_exec_time, total_exec_time
  FROM pg_stat_statements
  ORDER BY mean_exec_time DESC
  LIMIT 10;"
```

Missing index on `todos.user_id` or `todos.completed` will cause full table scans.
Check `todo-app-api-nestjs` repo → `prisma/schema.prisma` for index definitions.

### 3. Cache miss storm

If Valkey is restarting or was recently wiped, all requests hit the database.

```bash
kubectl get pods -n data | grep valkey
kubectl logs deployment/valkey -n data | tail -20
```

Check cache hit rate: Grafana → "Cache Hit Rate" panel.

### 4. Event loop blocking (Node.js only)

If the API is NestJS/Node.js and `nodejs_eventloop_lag_seconds > 0.1`:

```bash
kubectl logs deployment/todo-app-api -n todo-app | grep eventloop
```

Event loop lag > 100ms means synchronous work is blocking all requests.
This is a code-level issue — engage backend team with a CPU profile.

### 5. GC pauses (Java only)

If the API is Spring Boot and `jvm_gc_pause_seconds_sum` is high:
- Increase heap: add `-Xmx` in `JAVA_OPTS` env in `helm/api/values.yaml`
- Check Grafana → "JVM Heap Usage" for heap saturation

---

## Fix Checklist

- [ ] Identify which layer is slow: app, DB, cache, or network
- [ ] If CPU: scale out or increase CPU limit in `helm/api/values.yaml`
- [ ] If slow query: add missing index (backend team), or check recent schema migrations
- [ ] If cache miss: verify Valkey is healthy, check TTL config
- [ ] If event loop (Node.js): CPU profile needed — escalate to backend team
- [ ] Apply changes via Helm upgrade
- [ ] Confirm p95 drops below 500ms in Grafana
