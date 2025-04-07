# Monitoring Setup

This directory contains Helm charts for deploying the monitoring stack for the Todo App.

## Components

- **Prometheus**: Metrics collection and alerting
- **Grafana**: Visualization and dashboards
- **AlertManager**: Alert routing and notifications
- **Loki**: Log aggregation (configured as a data source)

## Configuration

### Alertmanager Webhook Integration

The monitoring stack is configured to send alerts to a webhook endpoint. By default, it sends to `http://host.docker.internal:3001/api/webhook`, which can be a Postman Mock Server.

#### Setting up Postman Mock Server for Alerts

1. Open Postman and create a new Collection named "Todo App Alerts"
2. Add a new POST request to the collection with the path `/api/webhook`
3. Save the request
4. Click the 3 dots next to the collection name and select "Mock Collection"
5. Create a new mock server with a name (e.g., "Todo App Alert Receiver")
6. After creation, Postman will show you the mock server URL (e.g., `https://xxxxxxxx.mock.pstmn.io`)
8. Update the webhook URL in values.yaml:
   ```yaml
   alertmanager:
     config:
       receivers:
       - name: 'postman-webhook'
         webhook_configs:
         - url: 'https://xxxxxxxx.mock.pstmn.io'
   ```

#### Testing the Webhook

1. Deploy the monitoring stack with Helm:
   ```bash
   helm upgrade --install monitoring . -n monitoring
   ```
2. Check if the test alert is firing:
   ```bash
   kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring
   ```
3. Open http://localhost:9090/alerts in your browser
4. You should see a "TestAlert" that's always firing
5. Check your Postman mock server for received alerts
6. You can view the alert payload in the Postman console

## Health Checks Monitoring

The monitoring stack is configured to monitor the health endpoint of the Todo App API:

1. Health metrics are scraped from `/health` endpoint
2. A dedicated service monitor is configured for health checks
3. The dashboard includes a health status panel
4. Alerts are configured for health check failures

## Dashboards

The monitoring stack comes with pre-configured dashboards:

- **Todo App Overview**: Basic dashboard with request rate metrics
- **Todo App Dashboard**: Comprehensive dashboard with the following panels:
  - HTTP Request Rate by method and path
  - HTTP Response Time by path
  - Application Logs (from Loki)
  - API Health Status
  - HTTP Status Codes
  - Error Rate (%)

## Alert Rules

The following alert rules are configured:

- **HighTodoAppErrorRate**: Triggers when error rate is above 1% for more than 1 minute
- **HighMemoryUsage**: Triggers when memory usage is above 200MB for more than 2 minutes
- **HighRequestLatency**: Triggers when average request latency is above 0.5 seconds for more than 1 minute
- **LowRequestRate**: Triggers when request rate is below 0.1 requests per second for more than 5 minutes
- **ServiceDown**: Triggers when the Todo API service is down for more than 1 minute
- **SlowResponseTime**: Triggers when 95th percentile of response time is above 2 seconds for more than 5 minutes
- **TestAlert**: Always firing alert for testing webhook integration

## External Services Configuration

The monitoring stack is configured to monitor external services:

1. **Todo App API**: Deployed on Render or other cloud service
   - Metrics endpoint: `/api/metrics`
   - Health endpoint: `/health`
   - Update the hostname in `values.yaml` under `externalTodoApi.host` 