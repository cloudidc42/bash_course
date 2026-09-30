# Part 53: Advanced Observability and AIOps (ขั้นตอนที่ 561-564)

## Observability ระดับ World-class และ AI-powered Operations

---

## ขั้นตอนที่ 561: Advanced Prometheus and Thanos

```bash
#!/bin/bash
# prometheus-thanos-advanced.sh
# Advanced Prometheus: Long-term Storage with Thanos, Federation, High Availability

set -euo pipefail

NAMESPACE="${NAMESPACE:-monitoring}"
THANOS_VERSION="${THANOS_VERSION:-0.32.0}"
S3_BUCKET="${S3_BUCKET:-thanos-metrics}"
AWS_REGION="${AWS_REGION:-us-east-1}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Thanos Sidecar Pattern ====================
setup_thanos_sidecar() {
    log "Setting up Thanos Sidecar for long-term storage..."
    
    # Thanos object store config
    cat <<EOF | kubectl create secret generic thanos-objstore-config \
        --namespace "${NAMESPACE}" \
        --from-literal=objstore.yml="$(cat)" \
        --dry-run=client -o yaml | kubectl apply -f -
type: S3
config:
  bucket: ${S3_BUCKET}
  region: ${AWS_REGION}
  endpoint: s3.amazonaws.com
  sse_config:
    type: SSE-S3
EOF

    # Add Thanos sidecar to Prometheus via ServiceMonitor
    cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: thanos-prometheus
  namespace: monitoring
spec:
  replicas: 2
  replicaExternalLabelName: prometheus_replica
  prometheusExternalLabelName: cluster
  externalLabels:
    cluster: production
    region: us-east-1
  
  retention: 2h
  retentionSize: 50GB
  
  thanos:
    baseImage: quay.io/thanos/thanos
    version: v0.32.0
    objectStorageConfig:
      key: objstore.yml
      name: thanos-objstore-config
    readyTimeout: 10m
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 1Gi
  
  alerting:
    alertmanagers:
    - namespace: monitoring
      name: alertmanager-operated
      port: web
  
  ruleNamespaceSelector: {}
  ruleSelector:
    matchLabels:
      prometheus: kube-prometheus
      role: alert-rules
  
  serviceMonitorSelector: {}
  podMonitorSelector: {}
  
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: gp3
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 100Gi
EOF

    log "Thanos sidecar configured"
}

# ==================== Thanos Components ====================
setup_thanos_components() {
    log "Setting up Thanos query stack..."
    
    # Thanos Querier (multi-cluster query)
    cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: thanos-query
  template:
    metadata:
      labels:
        app: thanos-query
    spec:
      containers:
      - name: thanos-query
        image: quay.io/thanos/thanos:v0.32.0
        args:
        - query
        - --log.level=info
        - --log.format=json
        - --query.replica-label=prometheus_replica
        - --query.replica-label=rule_replica
        # Sidecars
        - --store=dnssrv+_grpc._tcp.prometheus-operated.monitoring.svc.cluster.local
        # Remote stores
        - --store=thanos-store-gateway.monitoring:10901
        - --store=thanos-ruler.monitoring:10901
        # Cross-cluster (via ClusterMesh or VPC peering)
        - --store=dr-thanos-sidecar.dr-monitoring:10901
        ports:
        - containerPort: 10902
          name: http
        - containerPort: 10901
          name: grpc
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-store-gateway
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: thanos-store-gateway
  template:
    metadata:
      labels:
        app: thanos-store-gateway
    spec:
      containers:
      - name: thanos-store-gateway
        image: quay.io/thanos/thanos:v0.32.0
        args:
        - store
        - --log.level=info
        - --data-dir=/var/thanos/store
        - --objstore.config-file=/etc/thanos/objstore.yml
        - --sync-block-duration=3m
        - --block-sync-concurrency=20
        - --chunk-pool-size=2GB
        - --index-cache.config-file=/etc/thanos/index-cache.yaml
        volumeMounts:
        - name: objstore-config
          mountPath: /etc/thanos
        - name: data
          mountPath: /var/thanos/store
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 2000m
            memory: 8Gi
      volumes:
      - name: objstore-config
        secret:
          secretName: thanos-objstore-config
      - name: data
        emptyDir: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-compactor
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: thanos-compactor
  template:
    metadata:
      labels:
        app: thanos-compactor
    spec:
      containers:
      - name: thanos-compactor
        image: quay.io/thanos/thanos:v0.32.0
        args:
        - compact
        - --log.level=info
        - --data-dir=/var/thanos/compact
        - --objstore.config-file=/etc/thanos/objstore.yml
        - --retention.resolution-raw=30d
        - --retention.resolution-5m=90d
        - --retention.resolution-1h=1y
        - --compact.concurrency=4
        - --wait
        - --wait-interval=3m
        volumeMounts:
        - name: objstore-config
          mountPath: /etc/thanos
        resources:
          requests:
            cpu: 1000m
            memory: 4Gi
      volumes:
      - name: objstore-config
        secret:
          secretName: thanos-objstore-config
EOF

    log "Thanos components deployed"
}

# ==================== Advanced Recording Rules ====================
create_recording_rules() {
    log "Creating advanced recording rules..."
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: platform-recording-rules
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
  # Pre-compute expensive queries
  - name: recording.requests
    interval: 30s
    rules:
    - record: namespace:http_requests:rate5m
      expr: sum by (namespace) (rate(http_requests_total[5m]))
    
    - record: namespace:http_errors:ratio_rate5m
      expr: |
        sum by (namespace) (rate(http_requests_total{code=~"5.."}[5m]))
        /
        sum by (namespace) (rate(http_requests_total[5m]))
    
    - record: namespace_service:http_request_duration_seconds:p99
      expr: |
        histogram_quantile(0.99, 
          sum by (namespace, service, le) 
          (rate(http_request_duration_seconds_bucket[5m])))
    
    - record: cluster:node_cpu:ratio_rate5m
      expr: |
        1 - avg by (cluster) (
          irate(node_cpu_seconds_total{mode="idle"}[5m])
        )
    
    - record: cluster:container_cpu_usage:ratio
      expr: |
        sum by (cluster, namespace) (
          rate(container_cpu_usage_seconds_total{container!=""}[5m])
        )
        /
        sum by (cluster, namespace) (
          kube_pod_container_resource_requests{resource="cpu"}
        )
  
  # Business metrics recording
  - name: recording.business
    interval: 60s
    rules:
    - record: business:orders:rate1m
      expr: sum(rate(checkout_orders_total{status="completed"}[1m]))
    
    - record: business:revenue:rate1h
      expr: sum(increase(checkout_revenue_total[1h]))
    
    - record: business:active_users:sum5m
      expr: count(count by (user_id) (http_requests_total{authenticated="true"}[5m]))
EOF

    log "Recording rules created"
}

# ==================== Grafana Advanced Dashboards ====================
setup_grafana_advanced() {
    log "Setting up advanced Grafana dashboards..."
    
    # Install Grafana plugins
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-plugins
  namespace: monitoring
  labels:
    grafana_plugin: "true"
data:
  plugins.yaml: |
    plugins:
    - grafana-clock-panel
    - grafana-piechart-panel
    - grafana-worldmap-panel
    - grafana-polystat-panel
    - natel-discrete-panel
    - grafana-oncall-app
    - grafana-k6-app
    - yesoreyeram-infinity-datasource
    - isovalent-hubble-datasource
EOF

    # Grafana configuration
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-extra-config
  namespace: monitoring
data:
  grafana.ini: |
    [analytics]
    reporting_enabled = false
    check_for_updates = false
    
    [security]
    admin_user = admin
    cookie_secure = true
    cookie_samesite = lax
    strict_transport_security = true
    
    [auth.generic_oauth]
    enabled = true
    name = Company SSO
    allow_sign_up = true
    client_id = grafana
    client_secret = ${GRAFANA_OAUTH_SECRET}
    scopes = openid email profile groups
    auth_url = https://auth.company.com/auth
    token_url = https://auth.company.com/token
    api_url = https://auth.company.com/userinfo
    role_attribute_path = contains(groups[*], 'platform-team') && 'Admin' || 'Viewer'
    
    [dashboards]
    min_refresh_interval = 5s
    
    [feature_toggles]
    enable = traceToMetrics publicDashboards lokiLogs
    
    [unified_alerting]
    enabled = true
    
    [smtp]
    enabled = true
    host = smtp.company.com:587
    user = grafana@company.com
    password = ${SMTP_PASSWORD}
    from_address = grafana@company.com
    from_name = Grafana Alerts
EOF

    log "Grafana advanced config applied"
}

# ==================== Prometheus Federation ====================
setup_federation() {
    log "Setting up Prometheus federation for multi-cluster..."
    
    # Global Prometheus that federates from all clusters
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 60s
      evaluation_interval: 60s
      external_labels:
        cluster: global
    
    scrape_configs:
    # Federate from production
    - job_name: federate-production
      scrape_interval: 30s
      honor_labels: true
      metrics_path: /federate
      params:
        match[]:
        - '{__name__=~"recording:.*"}'
        - '{__name__=~"business:.*"}'
        - '{job=~"kubernetes-.*"}'
      static_configs:
      - targets:
        - prometheus.monitoring.production.svc:9090
    
    # Federate from staging
    - job_name: federate-staging
      scrape_interval: 60s
      honor_labels: true
      metrics_path: /federate
      params:
        match[]:
        - '{__name__=~"recording:.*"}'
      static_configs:
      - targets:
        - prometheus.monitoring.staging.svc:9090
    
    # Federate from DR
    - job_name: federate-dr
      scrape_interval: 60s
      honor_labels: true
      metrics_path: /federate
      params:
        match[]:
        - '{__name__=~"recording:.*"}'
        - '{job="kubernetes-nodes"}'
      static_configs:
      - targets:
        - prometheus.monitoring.dr.svc:9090
EOF

    log "Federation configured"
}

main() {
    case "${1:-all}" in
        sidecar)  setup_thanos_sidecar ;;
        thanos)   setup_thanos_components ;;
        rules)    create_recording_rules ;;
        grafana)  setup_grafana_advanced ;;
        federate) setup_federation ;;
        all)
            setup_thanos_sidecar
            setup_thanos_components
            create_recording_rules
            setup_grafana_advanced
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 562: Loki Stack — Log Management at Scale

```bash
#!/bin/bash
# loki-enterprise.sh
# Loki Enterprise: Distributed Log Aggregation, LogQL, Alerting

set -euo pipefail

NAMESPACE="${NAMESPACE:-observability}"
LOKI_VERSION="${LOKI_VERSION:-5.36.0}"
S3_BUCKET="${S3_BUCKET:-loki-logs}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Loki Distributed Mode ====================
install_loki_distributed() {
    log "Installing Loki in distributed mode..."
    
    cat <<EOF > /tmp/loki-values.yaml
loki:
  schemaConfig:
    configs:
    - from: 2024-01-01
      store: tsdb
      object_store: s3
      schema: v13
      index:
        prefix: loki_index_
        period: 24h
  storageConfig:
    tsdb_shipper:
      active_index_directory: /var/loki/index
      cache_location: /var/loki/index_cache
      cache_ttl: 168h
    aws:
      s3: s3://${AWS_REGION:-us-east-1}/${S3_BUCKET}
      s3forcepathstyle: false
      region: ${AWS_REGION:-us-east-1}
  
  auth_enabled: true
  
  commonConfig:
    replication_factor: 3
    ring:
      kvstore:
        store: memberlist
  
  querier:
    max_concurrent: 20
    engine:
      max_look_back_period: 30s
  
  queryScheduler:
    max_outstanding_requests_per_tenant: 100
  
  limits_config:
    max_query_lookback: 336h
    max_query_parallelism: 32
    max_streams_per_user: 100000
    ingestion_rate_mb: 100
    ingestion_burst_size_mb: 200
    per_stream_rate_limit: 10MB
    per_stream_rate_limit_burst: 50MB
    retention_period: 744h

serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/LokiRole

write:
  replicas: 3
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi

read:
  replicas: 3
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi

backend:
  replicas: 3
  persistence:
    size: 100Gi
    storageClass: gp3

gateway:
  enabled: true
  replicas: 2
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
    - host: loki.company.com
      paths:
      - path: /

monitoring:
  dashboards:
    enabled: true
  rules:
    enabled: true
  serviceMonitor:
    enabled: true
EOF

    helm repo add grafana https://grafana.github.io/helm-charts
    helm upgrade --install loki grafana/loki \
        --namespace "${NAMESPACE}" \
        --create-namespace \
        -f /tmp/loki-values.yaml \
        --wait
    
    log "Loki distributed installed"
}

# ==================== Promtail Advanced Config ====================
setup_promtail_advanced() {
    log "Configuring advanced Promtail pipeline..."
    
    cat <<'EOF' > /tmp/promtail-values.yaml
config:
  clients:
  - url: http://loki-gateway.observability/loki/api/v1/push
    tenant_id: production
    batchwait: 1s
    batchsize: 1048576
    external_labels:
      cluster: production
      region: us-east-1
  
  snippets:
    extraScrapeConfigs: |
      # Application logs with JSON parsing
      - job_name: application-logs
        kubernetes_sd_configs:
        - role: pod
        pipeline_stages:
        - docker: {}
        - match:
            selector: '{app=~".+"}'
            stages:
            - json:
                expressions:
                  level: level
                  msg: message
                  trace_id: trace_id
                  span_id: span_id
                  user_id: user_id
                  duration_ms: duration_ms
            - labels:
                level:
                trace_id:
            - metrics:
                log_lines_total:
                  type: Counter
                  description: Total log lines
                  labels:
                    level:
                    namespace:
                  config:
                    match_all: true
                    action: inc
                request_duration_seconds:
                  type: Histogram
                  description: Log duration
                  source: duration_ms
                  config:
                    buckets: [50, 100, 250, 500, 1000, 2500, 5000]
            - output:
                source: msg
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace]
          target_label: namespace
        - source_labels: [__meta_kubernetes_pod_name]
          target_label: pod
        - source_labels: [__meta_kubernetes_pod_label_app]
          target_label: app
        - source_labels: [__meta_kubernetes_pod_node_name]
          target_label: node
        - source_labels: [__meta_kubernetes_pod_container_name]
          target_label: container
      
      # Audit logs
      - job_name: audit-logs
        static_configs:
        - targets: [localhost]
          labels:
            job: k8s-audit
            __path__: /var/log/kube-apiserver-audit.log
        pipeline_stages:
        - json:
            expressions:
              user: user.username
              verb: verb
              resource: objectRef.resource
              namespace: objectRef.namespace
              response_code: responseStatus.code
        - labels:
            user:
            verb:
            resource:
            response_code:
        - drop:
            source: verb
            expression: "get|list|watch"
            drop_counter_reason: readonly_filtered
EOF

    helm upgrade --install promtail grafana/promtail \
        --namespace "${NAMESPACE}" \
        -f /tmp/promtail-values.yaml \
        --wait
    
    log "Promtail configured"
}

# ==================== LogQL Advanced Queries ====================
demonstrate_logql() {
    log "Demonstrating advanced LogQL queries..."
    
    cat <<'LOGQL'
# === ADVANCED LOGQL EXAMPLES ===

# 1. Error rate by service (last 5m)
sum by (app) (
  rate({namespace="production"} |= "ERROR" [5m])
)

# 2. Top 10 slowest requests (JSON parsing)
topk(10,
  avg by (app, path) (
    {namespace="production"} 
    | json 
    | duration_ms > 1000
    | unwrap duration_ms
    | avg_over_time [5m]
  )
)

# 3. Trace ID correlation
{namespace="production"} 
| json 
| trace_id = "abc123def456"
| line_format "{{.timestamp}} [{{.level}}] {{.msg}}"

# 4. Security events - failed auth (last 1h)
count_over_time(
  {job="k8s-audit"} 
  | json 
  | verb = "create" 
  | response_code >= 401 [1h]
)

# 5. Log volume by namespace
sum by (namespace) (
  rate({cluster="production"}[5m])
)

# 6. Pattern-based anomaly detection
{namespace="production"} 
| pattern `<_> level=ERROR <msg>` 
| line_format "{{.msg}}"
| decolorize

# 7. Parse nginx access log
{job="nginx-ingress"} 
| regexp `(?P<remote_addr>\S+) - \S+ \[(?P<time_local>[^\]]+)\] "(?P<request>[^"]+)" (?P<status>\d+) (?P<body_bytes_sent>\d+) "[^"]*" "(?P<http_user_agent>[^"]+)"` 
| status >= 500 
| line_format "{{.remote_addr}} {{.status}} {{.request}}"

# 8. Metric from logs (extract and aggregate)
sum by (app) (
  rate(
    {namespace="production"} 
    | json duration_ms="duration_ms"
    | unwrap duration_ms [5m]
  )
)

# 9. LogQL alerting rule (in AlertManager)
# Alert when error rate > 10/min for 5 minutes
sum by (app) (
  rate({namespace="production"} |= "ERROR" [5m])
) > 10

# 10. Multi-line log merging
{namespace="production"} 
| multiline firstline="^\d{4}-\d{2}-\d{2}" 
| json 
| level = "ERROR"
LOGQL

    log "LogQL examples displayed"
}

# ==================== Log Alerting ====================
setup_log_alerting() {
    log "Setting up log-based alerting..."
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: loki-log-alerts
  namespace: monitoring
spec:
  groups:
  - name: log.alerts
    rules:
    - alert: HighErrorLogRate
      expr: |
        sum by (namespace, app) (
          rate({namespace=~"production|staging"} |= "ERROR" [5m])
        ) > 100
      for: 5m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "High error log rate in {{ $labels.app }}"
        description: "{{ $labels.app }} in {{ $labels.namespace }} has {{ $value }} ERROR logs/s"
        runbook: "https://wiki.company.com/runbooks/high-error-rate"
    
    - alert: OOMKilledDetected
      expr: |
        count_over_time(
          {cluster="production"} |= "OOMKilled" [5m]
        ) > 0
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: "OOMKill detected in production"
    
    - alert: PanicLogDetected
      expr: |
        count_over_time(
          {namespace="production"} |~ "(?i)panic|fatal" [2m]
        ) > 0
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: "Panic/Fatal log detected"
    
    - alert: AuditSensitiveOperation
      expr: |
        count_over_time(
          {job="k8s-audit"} 
          | json 
          | verb = "delete" 
          | objectRef_resource = "secrets" [5m]
        ) > 0
      for: 0m
      labels:
        severity: warning
        team: security
      annotations:
        summary: "Secret deletion detected in K8s audit"
EOF

    log "Log alerting configured"
}

main() {
    case "${1:-all}" in
        install)  install_loki_distributed ;;
        promtail) setup_promtail_advanced ;;
        logql)    demonstrate_logql ;;
        alerts)   setup_log_alerting ;;
        all)
            install_loki_distributed
            setup_promtail_advanced
            setup_log_alerting
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 563: AIOps — Machine Learning for Operations

```bash
#!/bin/bash
# aiops-platform.sh
# AIOps: Anomaly Detection, Predictive Alerting, Root Cause Analysis with ML

set -euo pipefail

NAMESPACE="${NAMESPACE:-aiops}"
PROMETHEUS_URL="${PROMETHEUS_URL:-http://prometheus-operated.monitoring:9090}"
GRAFANA_URL="${GRAFANA_URL:-http://grafana.monitoring:3000}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Anomaly Detection with Isolation Forest ====================
deploy_anomaly_detector() {
    log "Deploying ML Anomaly Detection service..."
    
    # Python anomaly detection service
    cat <<'PYEOF' > /tmp/anomaly_detector.py
"""
AIOps Anomaly Detection Service
Uses Isolation Forest + Prophet for time-series anomaly detection
"""
import os
import json
import time
import logging
from datetime import datetime, timedelta
import urllib.request
import urllib.parse

import numpy as np
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import StandardScaler
try:
    from prophet import Prophet
    PROPHET_AVAILABLE = True
except ImportError:
    PROPHET_AVAILABLE = False

logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(message)s')
logger = logging.getLogger(__name__)

PROMETHEUS_URL = os.getenv('PROMETHEUS_URL', 'http://prometheus-operated.monitoring:9090')
ANOMALY_THRESHOLD = float(os.getenv('ANOMALY_THRESHOLD', '-0.1'))  # Isolation Forest score
LOOKBACK_HOURS = int(os.getenv('LOOKBACK_HOURS', '24'))
CHECK_INTERVAL = int(os.getenv('CHECK_INTERVAL', '300'))  # 5 minutes


def query_prometheus(query: str, start: datetime, end: datetime, step: str = '5m') -> list:
    """Query Prometheus range API"""
    params = urllib.parse.urlencode({
        'query': query,
        'start': start.isoformat() + 'Z',
        'end': end.isoformat() + 'Z',
        'step': step
    })
    url = f"{PROMETHEUS_URL}/api/v1/query_range?{params}"
    try:
        with urllib.request.urlopen(url, timeout=30) as resp:
            data = json.loads(resp.read())
            return data.get('data', {}).get('result', [])
    except Exception as e:
        logger.error(f"Prometheus query failed: {e}")
        return []


def detect_anomalies_isolation_forest(metric_name: str, values: list) -> dict:
    """Detect anomalies using Isolation Forest"""
    if len(values) < 10:
        return {'anomalies': [], 'score': 0}
    
    X = np.array(values).reshape(-1, 1)
    scaler = StandardScaler()
    X_scaled = scaler.fit_transform(X)
    
    model = IsolationForest(
        contamination=0.05,  # Expected 5% anomaly rate
        random_state=42,
        n_estimators=100
    )
    
    scores = model.fit_predict(X_scaled)
    decision_scores = model.decision_function(X_scaled)
    
    anomaly_indices = np.where(scores == -1)[0]
    
    return {
        'metric': metric_name,
        'anomalies': anomaly_indices.tolist(),
        'scores': decision_scores.tolist(),
        'anomaly_count': len(anomaly_indices),
        'anomaly_rate': len(anomaly_indices) / len(values)
    }


def forecast_with_prophet(metric_name: str, timestamps: list, values: list) -> dict:
    """Forecast metric using Facebook Prophet"""
    if not PROPHET_AVAILABLE or len(values) < 100:
        return {'forecast': [], 'anomalies': []}
    
    import pandas as pd
    
    df = pd.DataFrame({
        'ds': pd.to_datetime(timestamps, unit='s'),
        'y': values
    })
    
    model = Prophet(
        changepoint_prior_scale=0.05,
        seasonality_mode='multiplicative',
        interval_width=0.95
    )
    model.add_seasonality(name='hourly', period=1/24, fourier_order=5)
    model.fit(df)
    
    # Forecast next 2 hours
    future = model.make_future_dataframe(periods=24, freq='5min')
    forecast = model.predict(future)
    
    # Find anomalies: actual values outside prediction interval
    merged = df.merge(
        forecast[['ds', 'yhat', 'yhat_lower', 'yhat_upper']],
        on='ds',
        how='left'
    )
    anomalies = merged[
        (merged['y'] < merged['yhat_lower']) | 
        (merged['y'] > merged['yhat_upper'])
    ]
    
    return {
        'metric': metric_name,
        'forecast_next_2h': forecast.tail(24)[['ds', 'yhat', 'yhat_lower', 'yhat_upper']].to_dict('records'),
        'anomalies': anomalies.to_dict('records'),
        'anomaly_count': len(anomalies)
    }


def analyze_metrics():
    """Main analysis loop"""
    end = datetime.utcnow()
    start = end - timedelta(hours=LOOKBACK_HOURS)
    
    metrics_to_monitor = [
        ('http_request_rate', 'sum(rate(http_requests_total[5m]))'),
        ('error_rate', 'sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))'),
        ('p99_latency', 'histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))'),
        ('cpu_usage', 'avg(rate(container_cpu_usage_seconds_total{container!=""}[5m]))'),
        ('memory_usage', 'avg(container_memory_working_set_bytes{container!=""} / container_spec_memory_limit_bytes{container!=""})'),
    ]
    
    anomaly_report = {
        'timestamp': datetime.utcnow().isoformat(),
        'metrics_analyzed': len(metrics_to_monitor),
        'results': []
    }
    
    for metric_name, query in metrics_to_monitor:
        results = query_prometheus(query, start, end)
        
        for result in results:
            values_with_ts = result.get('values', [])
            timestamps = [float(v[0]) for v in values_with_ts]
            values = [float(v[1]) for v in values_with_ts if v[1] != 'NaN']
            
            if not values:
                continue
            
            # Isolation Forest detection
            isolation_result = detect_anomalies_isolation_forest(metric_name, values)
            
            # Prophet forecast (if enough data)
            prophet_result = forecast_with_prophet(metric_name, timestamps, values)
            
            if isolation_result['anomaly_count'] > 0:
                logger.warning(
                    f"ANOMALY DETECTED: {metric_name} - "
                    f"{isolation_result['anomaly_count']} anomalies "
                    f"({isolation_result['anomaly_rate']:.1%} of data)"
                )
                
                anomaly_report['results'].append({
                    'metric': metric_name,
                    'isolation_forest': isolation_result,
                    'prophet': prophet_result,
                    'severity': 'high' if isolation_result['anomaly_rate'] > 0.1 else 'medium'
                })
    
    return anomaly_report


def main():
    logger.info("AIOps Anomaly Detection Service started")
    
    while True:
        try:
            report = analyze_metrics()
            
            anomaly_count = len(report['results'])
            if anomaly_count > 0:
                logger.warning(f"Found {anomaly_count} metrics with anomalies")
                print(json.dumps(report, indent=2, default=str))
            else:
                logger.info(f"All {report['metrics_analyzed']} metrics normal")
            
        except Exception as e:
            logger.error(f"Analysis failed: {e}", exc_info=True)
        
        time.sleep(CHECK_INTERVAL)


if __name__ == '__main__':
    main()
PYEOF

    # Kubernetes deployment
    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: anomaly-detector
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: anomaly-detector
  template:
    metadata:
      labels:
        app: anomaly-detector
    spec:
      serviceAccountName: anomaly-detector
      containers:
      - name: anomaly-detector
        image: python:3.11-slim
        command: [python, /app/anomaly_detector.py]
        env:
        - name: PROMETHEUS_URL
          value: "${PROMETHEUS_URL}"
        - name: ANOMALY_THRESHOLD
          value: "-0.1"
        - name: LOOKBACK_HOURS
          value: "24"
        - name: CHECK_INTERVAL
          value: "300"
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        volumeMounts:
        - name: app
          mountPath: /app
      initContainers:
      - name: install-deps
        image: python:3.11-slim
        command: [pip, install, scikit-learn, prophet, numpy, pandas]
      volumes:
      - name: app
        configMap:
          name: anomaly-detector-code
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: anomaly-detector-code
  namespace: ${NAMESPACE}
data:
  anomaly_detector.py: |
$(cat /tmp/anomaly_detector.py | sed 's/^/    /')
EOF

    log "Anomaly detector deployed"
}

# ==================== Auto Root Cause Analysis ====================
setup_rca_engine() {
    log "Setting up Root Cause Analysis engine..."
    
    cat <<'EOF' > /tmp/rca_engine.py
"""
Automated Root Cause Analysis Engine
Correlates metrics, logs, and events to identify root causes
"""
import json
import re
from datetime import datetime, timedelta
from dataclasses import dataclass
from typing import Optional

@dataclass
class IncidentSignal:
    timestamp: datetime
    source: str
    signal_type: str
    severity: str
    message: str
    labels: dict
    value: Optional[float] = None


class RCAEngine:
    """Rule-based + correlation RCA engine"""
    
    # Causal rules: if A happens, likely cause is B
    CAUSAL_RULES = [
        {
            'symptom': r'high error rate',
            'causes': ['database connection exhausted', 'upstream service down', 'deployment rollout'],
            'queries': {
                'database_conn': 'pg_stat_activity_count > 90',
                'upstream_health': 'up{job="backend-service"} == 0',
                'recent_deploy': 'kube_deployment_status_observed_generation rate > 0'
            }
        },
        {
            'symptom': r'high latency',
            'causes': ['cpu throttling', 'memory pressure', 'network saturation', 'database slow queries'],
            'queries': {
                'cpu_throttle': 'rate(container_cpu_cfs_throttled_seconds_total[5m]) > 0.1',
                'memory_pressure': 'container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.9',
                'network': 'rate(node_network_receive_drop_total[5m]) > 10'
            }
        },
        {
            'symptom': r'pod(s?) (crash|restart)',
            'causes': ['oom kill', 'liveness probe failure', 'image pull error', 'config error'],
            'queries': {
                'oom': 'kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} > 0',
                'restarts': 'increase(kube_pod_container_status_restarts_total[5m]) > 0'
            }
        },
        {
            'symptom': r'node not ready',
            'causes': ['disk pressure', 'memory pressure', 'network partition', 'kubelet crash'],
            'queries': {
                'disk_pressure': 'node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.1',
                'memory': 'node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1'
            }
        }
    ]
    
    def __init__(self, prometheus_url: str):
        self.prometheus_url = prometheus_url
        self.signals = []
        self.correlations = []
    
    def add_signal(self, signal: IncidentSignal):
        self.signals.append(signal)
    
    def correlate_signals(self, window_minutes: int = 15) -> list:
        """Find temporally correlated signals"""
        correlations = []
        
        for i, s1 in enumerate(self.signals):
            for j, s2 in enumerate(self.signals[i+1:], i+1):
                time_diff = abs((s1.timestamp - s2.timestamp).total_seconds())
                if time_diff <= window_minutes * 60:
                    correlations.append({
                        'signal_1': s1,
                        'signal_2': s2,
                        'time_diff_seconds': time_diff,
                        'likely_related': time_diff < 120
                    })
        
        return correlations
    
    def apply_causal_rules(self, incident_description: str) -> list:
        """Apply causal rules to identify likely causes"""
        likely_causes = []
        
        for rule in self.CAUSAL_RULES:
            if re.search(rule['symptom'], incident_description, re.IGNORECASE):
                likely_causes.extend([
                    {
                        'cause': cause,
                        'verification_query': rule['queries'].get(cause.replace(' ', '_'), ''),
                        'confidence': 'high' if idx == 0 else 'medium'
                    }
                    for idx, cause in enumerate(rule['causes'])
                ])
        
        return likely_causes
    
    def generate_rca_report(self, incident_description: str) -> dict:
        """Generate comprehensive RCA report"""
        correlations = self.correlate_signals()
        likely_causes = self.apply_causal_rules(incident_description)
        
        # Sort by time proximity
        close_correlations = [c for c in correlations if c['likely_related']]
        
        report = {
            'incident': incident_description,
            'analyzed_at': datetime.utcnow().isoformat(),
            'signals_analyzed': len(self.signals),
            'temporal_correlations': len(close_correlations),
            'likely_root_causes': likely_causes[:3],  # Top 3
            'correlated_events': [
                {
                    'event_1': c['signal_1'].message,
                    'event_2': c['signal_2'].message,
                    'time_gap': f"{c['time_diff_seconds']:.0f}s"
                }
                for c in close_correlations[:5]
            ],
            'timeline': sorted([
                {
                    'time': s.timestamp.isoformat(),
                    'source': s.source,
                    'event': s.message,
                    'severity': s.severity
                }
                for s in self.signals
            ], key=lambda x: x['time']),
            'recommendations': self._generate_recommendations(likely_causes)
        }
        
        return report
    
    def _generate_recommendations(self, causes: list) -> list:
        recommendations_map = {
            'database connection exhausted': [
                'Check pg_stat_activity for idle connections',
                'Increase PgBouncer pool size',
                'Add connection timeout to application'
            ],
            'cpu throttling': [
                'Increase CPU limits in pod spec',
                'Profile application for CPU hotspots',
                'Enable CPU burst with LimitRange'
            ],
            'oom kill': [
                'Increase memory limits',
                'Profile for memory leaks',
                'Add memory limit to 2x of typical usage'
            ]
        }
        
        recs = []
        for cause_info in causes[:2]:
            cause = cause_info.get('cause', '')
            for pattern, rec_list in recommendations_map.items():
                if pattern.lower() in cause.lower():
                    recs.extend(rec_list)
        
        return list(set(recs))[:5]


# Demo
if __name__ == '__main__':
    engine = RCAEngine('http://prometheus:9090')
    
    # Simulate incident signals
    base_time = datetime.utcnow() - timedelta(minutes=30)
    
    engine.add_signal(IncidentSignal(
        timestamp=base_time,
        source='prometheus',
        signal_type='alert',
        severity='warning',
        message='ErrorBudgetBurnHighFast firing',
        labels={'service': 'api-gateway'},
        value=15.4
    ))
    
    engine.add_signal(IncidentSignal(
        timestamp=base_time + timedelta(minutes=2),
        source='kubernetes',
        signal_type='event',
        severity='warning',
        message='Pod api-gateway-xxx restarted 3 times',
        labels={'namespace': 'production', 'pod': 'api-gateway-xxx'}
    ))
    
    engine.add_signal(IncidentSignal(
        timestamp=base_time + timedelta(minutes=5),
        source='loki',
        signal_type='log',
        severity='error',
        message='database connection pool exhausted: too many clients',
        labels={'app': 'api-gateway', 'level': 'error'}
    ))
    
    report = engine.generate_rca_report('high error rate in api-gateway')
    print(json.dumps(report, indent=2))
EOF

    python3 /tmp/rca_engine.py
    log "RCA engine demonstrated"
}

# ==================== Predictive Autoscaling ====================
setup_predictive_autoscaling() {
    log "Setting up KEDA with predictive scaling..."
    
    # KEDA Installation
    helm repo add kedacore https://kedacore.github.io/charts
    helm upgrade --install keda kedacore/keda \
        --namespace keda \
        --create-namespace \
        --wait
    
    # ScaledObject with ML-predicted schedule
    cat <<'EOF' | kubectl apply -f -
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-gateway-predictive-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: api-gateway
  minReplicaCount: 2
  maxReplicaCount: 50
  pollingInterval: 30
  cooldownPeriod: 120
  
  triggers:
  # Current CPU
  - type: cpu
    metadata:
      type: AverageValue
      value: "70"
  
  # Queue depth
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-operated.monitoring:9090
      metricName: http_requests_queue_depth
      threshold: "100"
      query: |
        sum(rate(http_requests_total{namespace="production",app="api-gateway"}[1m])) 
        / 
        scalar(kube_deployment_spec_replicas{namespace="production",deployment="api-gateway"})
  
  # Cron-based predictive scaling (peak hours)
  - type: cron
    metadata:
      timezone: Asia/Bangkok
      start: "0 8 * * 1-5"   # Weekday morning ramp
      end: "0 9 * * 1-5"
      desiredReplicas: "15"
  - type: cron
    metadata:
      timezone: Asia/Bangkok
      start: "0 9 * * 1-5"   # Business hours
      end: "0 18 * * 1-5"
      desiredReplicas: "20"
  - type: cron
    metadata:
      timezone: Asia/Bangkok
      start: "0 12 * * 1-5"  # Lunch peak
      end: "0 13 * * 1-5"
      desiredReplicas: "30"
  
  # External event (Black Friday, promotions)
  - type: redis
    metadata:
      address: redis-master.databases:6379
      listName: scale-events
      listLength: "1"
EOF

    log "Predictive autoscaling configured"
}

main() {
    case "${1:-all}" in
        anomaly)    deploy_anomaly_detector ;;
        rca)        setup_rca_engine ;;
        autoscale)  setup_predictive_autoscaling ;;
        all)
            deploy_anomaly_detector
            setup_rca_engine
            setup_predictive_autoscaling
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 564: OpenTelemetry Complete — Traces, Metrics, Logs

```bash
#!/bin/bash
# opentelemetry-complete.sh
# Complete OpenTelemetry Implementation: Auto-instrumentation, eBPF, OTLP

set -euo pipefail

NAMESPACE="${NAMESPACE:-production}"
OTEL_NAMESPACE="${OTEL_NAMESPACE:-observability}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== OpenTelemetry Operator ====================
install_otel_operator() {
    log "Installing OpenTelemetry Operator..."
    
    kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
    
    kubectl wait deployment opentelemetry-operator-controller-manager \
        --namespace opentelemetry-operator-system \
        --for condition=Available \
        --timeout 300s
    
    log "OpenTelemetry Operator installed"
}

# ==================== Auto-Instrumentation ====================
setup_auto_instrumentation() {
    log "Setting up OpenTelemetry Auto-Instrumentation..."
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: platform-instrumentation
  namespace: production
spec:
  exporter:
    endpoint: http://otel-collector.observability:4317
  
  propagators:
  - tracecontext
  - baggage
  - b3
  
  sampler:
    type: parentbased_traceidratio
    argument: "0.1"   # Sample 10% in production
  
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
    env:
    - name: OTEL_INSTRUMENTATION_MESSAGING_EXPERIMENTAL_RECEIVE_TELEMETRY_ENABLED
      value: "true"
    - name: OTEL_LOGS_EXPORTER
      value: otlp
  
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest
    env:
    - name: OTEL_NODE_ENABLED_INSTRUMENTATIONS
      value: http,grpc,express,pg,redis,aws-sdk
  
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
    env:
    - name: OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED
      value: "true"
    - name: OTEL_PYTHON_LOG_CORRELATION
      value: "true"
  
  go:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-go:latest
EOF

    # Annotate deployments for auto-instrumentation
    for DEPLOY in api-gateway backend-service checkout-service; do
        kubectl annotate deployment "${DEPLOY}" -n "${NAMESPACE}" \
            "instrumentation.opentelemetry.io/inject-python=true" \
            "instrumentation.opentelemetry.io/container-names=${DEPLOY}" \
            2>/dev/null || true
    done
    
    log "Auto-instrumentation configured"
}

# ==================== eBPF Instrumentation ====================
setup_ebpf_instrumentation() {
    log "Setting up eBPF-based zero-code instrumentation..."
    
    # Grafana Beyla - eBPF auto-instrumentation
    cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: beyla
  namespace: observability
spec:
  selector:
    matchLabels:
      app: beyla
  template:
    metadata:
      labels:
        app: beyla
    spec:
      hostPID: true
      hostNetwork: true
      tolerations:
      - operator: Exists
      containers:
      - name: beyla
        image: grafana/beyla:latest
        env:
        - name: BEYLA_OPEN_PORT
          value: "80,8080,443,8443"
        - name: BEYLA_SERVICE_NAMESPACE
          value: production
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: http://otel-collector.observability:4318
        - name: BEYLA_METRICS_FEATURES
          value: application,application_span,application_service_graph
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: "k8s.cluster.name=production"
        securityContext:
          privileged: true
          readOnlyRootFilesystem: false
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 1Gi
        volumeMounts:
        - name: var-run-beyla
          mountPath: /var/run/beyla
        - name: node-cgroups
          mountPath: /sys/fs/cgroup
      volumes:
      - name: var-run-beyla
        hostPath:
          path: /var/run/beyla
          type: DirectoryOrCreate
      - name: node-cgroups
        hostPath:
          path: /sys/fs/cgroup
EOF

    log "eBPF instrumentation (Beyla) deployed"
}

# ==================== Service Graph ====================
setup_service_graph() {
    log "Setting up Service Dependency Graph..."
    
    # Grafana service graph panel config
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: service-graph-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "true"
data:
  service-graph.json: |
    {
      "title": "Service Dependency Graph",
      "tags": ["service-graph", "traces"],
      "panels": [
        {
          "title": "Service Map",
          "type": "nodeGraph",
          "datasource": "Tempo",
          "targets": [
            {
              "queryType": "serviceMap",
              "serviceMapQuery": "{namespace=~\"production\"}"
            }
          ]
        },
        {
          "title": "RED Metrics by Service",
          "type": "table",
          "datasource": "Prometheus",
          "targets": [
            {
              "expr": "sum by (client, server) (rate(traces_service_graph_request_total[5m]))",
              "legendFormat": "{{client}} -> {{server}}"
            }
          ]
        }
      ]
    }
EOF

    log "Service graph configured"
}

# ==================== Trace Sampling Strategy ====================
setup_trace_sampling() {
    log "Setting up intelligent trace sampling..."
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: tail-sampling-collector
  namespace: observability
spec:
  mode: deployment
  replicas: 2
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    
    processors:
      # Tail-based sampling - make decisions after full trace
      tail_sampling:
        decision_wait: 10s
        num_traces: 100000
        policies:
        # Always sample errors
        - name: errors-policy
          type: status_code
          status_code:
            status_codes: [ERROR]
        
        # Always sample slow requests (> 1s)
        - name: slow-traces
          type: latency
          latency:
            threshold_ms: 1000
        
        # Sample 100% of checkout (business critical)
        - name: checkout-service
          type: string_attribute
          string_attribute:
            key: service.name
            values: [checkout-service, payment-service]
        
        # Rate limit for other services (10%)
        - name: probabilistic-policy
          type: probabilistic
          probabilistic:
            sampling_percentage: 10
        
        # Composite: AND logic
        - name: composite-policy
          type: composite
          composite:
            max_total_spans_per_second: 10000
            policy_order:
            - errors-policy
            - slow-traces
            - checkout-service
            - probabilistic-policy
      
      batch:
        send_batch_size: 10000
        timeout: 10s
    
    exporters:
      otlp/tempo:
        endpoint: tempo-distributor.observability:4317
        tls:
          insecure: true
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [tail_sampling, batch]
          exporters: [otlp/tempo]
EOF

    log "Intelligent trace sampling configured"
}

main() {
    case "${1:-all}" in
        operator)    install_otel_operator ;;
        autoinst)    setup_auto_instrumentation ;;
        ebpf)        setup_ebpf_instrumentation ;;
        graph)       setup_service_graph ;;
        sampling)    setup_trace_sampling ;;
        all)
            install_otel_operator
            setup_auto_instrumentation
            setup_ebpf_instrumentation
            setup_service_graph
            setup_trace_sampling
            ;;
    esac
}

main "$@"
```

---

## สรุป Part 53

- ✅ **Step 561**: Advanced Prometheus + Thanos — Long-term storage (S3), Querier/Store/Compactor, Recording rules, Grafana advanced config, Federation
- ✅ **Step 562**: Loki Enterprise — Distributed mode, Promtail advanced pipeline (JSON/audit), LogQL examples, Log-based alerting
- ✅ **Step 563**: AIOps — Isolation Forest anomaly detection, Root Cause Analysis engine (causal rules + correlation), KEDA predictive autoscaling
- ✅ **Step 564**: OpenTelemetry Complete — Operator + Auto-instrumentation, eBPF with Beyla, Service Graph, Tail-based sampling

**ขั้นตอนต่อไป: Part 54 - Module 4 Completion + World-class Module 5 Introduction**
