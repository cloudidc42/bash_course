# Part 38: Advanced Monitoring และ Observability Patterns

## Module 3: Advanced Level - Advanced Observability

---

## ขั้นตอนที่ 510: Advanced Prometheus Patterns

```bash
#!/bin/bash
# advanced-prometheus.sh - Advanced Prometheus Monitoring Patterns

set -euo pipefail

PROMETHEUS_URL="${PROMETHEUS_URL:-http://localhost:9090}"
ALERTMANAGER_URL="${ALERTMANAGER_URL:-http://localhost:9093}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Create custom recording rules
create_recording_rules() {
    local namespace="${1:-monitoring}"
    local service_name="${2:-my-service}"
    
    log "Creating recording rules for: $service_name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ${service_name}-recording-rules
  namespace: ${namespace}
  labels:
    app: kube-prometheus-stack
    release: prometheus
spec:
  groups:
  - name: ${service_name}.recording
    interval: 30s
    rules:
    # Request rate per endpoint
    - record: job:http_requests:rate5m
      expr: >
        sum(rate(http_requests_total[5m])) by (job, status_code, endpoint)
    
    # Error rate
    - record: job:http_errors:rate5m
      expr: >
        sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (job, endpoint)
        /
        sum(rate(http_requests_total[5m])) by (job, endpoint)
    
    # P99 latency
    - record: job:http_request_duration_seconds:p99
      expr: >
        histogram_quantile(0.99, 
          sum(rate(http_request_duration_seconds_bucket[5m])) by (job, endpoint, le)
        )
    
    # P95 latency
    - record: job:http_request_duration_seconds:p95
      expr: >
        histogram_quantile(0.95,
          sum(rate(http_request_duration_seconds_bucket[5m])) by (job, endpoint, le)
        )
    
    # CPU usage rate
    - record: job:container_cpu_usage:rate5m
      expr: >
        sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace, pod, container)
    
    # Memory usage percentage
    - record: job:container_memory_usage:percentage
      expr: >
        container_memory_working_set_bytes{container!=""}
        /
        container_spec_memory_limit_bytes{container!=""} * 100
    
    # Availability (successful requests / total requests)
    - record: job:service_availability:5m
      expr: >
        1 - (
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)
        )
EOF
    
    success "Recording rules created for: $service_name"
}

# Create alert rules
create_alert_rules() {
    local namespace="${1:-monitoring}"
    local service_name="${2:-my-service}"
    local slo_availability="${3:-99.9}"
    local slo_latency_p99="${4:-500}"
    
    log "Creating alert rules for: $service_name (SLO: ${slo_availability}% availability, ${slo_latency_p99}ms P99)"
    
    local availability_threshold
    availability_threshold=$(echo "scale=4; (100 - $slo_availability) / 100" | bc)
    local latency_threshold
    latency_threshold=$(echo "scale=3; $slo_latency_p99 / 1000" | bc)
    
    cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ${service_name}-alerts
  namespace: ${namespace}
  labels:
    app: kube-prometheus-stack
    release: prometheus
spec:
  groups:
  - name: ${service_name}.availability
    rules:
    - alert: HighErrorRate
      expr: |
        job:http_errors:rate5m{job="${service_name}"} > ${availability_threshold}
      for: 5m
      labels:
        severity: warning
        service: ${service_name}
      annotations:
        summary: "High error rate for {{ \$labels.job }}"
        description: "Error rate is {{ \$value | humanizePercentage }} for {{ \$labels.endpoint }}"
        runbook: https://runbooks.example.com/${service_name}/high-error-rate
    
    - alert: ServiceDown
      expr: |
        up{job="${service_name}"} == 0
      for: 1m
      labels:
        severity: critical
        service: ${service_name}
      annotations:
        summary: "Service {{ \$labels.job }} is down"
        description: "Service has been down for more than 1 minute"
        runbook: https://runbooks.example.com/${service_name}/service-down
    
    - alert: SLOViolation
      expr: |
        job:service_availability:5m{job="${service_name}"} < $(echo "scale=4; $slo_availability / 100" | bc)
      for: 5m
      labels:
        severity: critical
        service: ${service_name}
        slo: availability
      annotations:
        summary: "SLO violation: availability below ${slo_availability}%"
        description: "Service availability is {{ \$value | humanizePercentage }}, below SLO of ${slo_availability}%"
    
  - name: ${service_name}.latency
    rules:
    - alert: HighLatency
      expr: |
        job:http_request_duration_seconds:p99{job="${service_name}"} > ${latency_threshold}
      for: 5m
      labels:
        severity: warning
        service: ${service_name}
      annotations:
        summary: "High P99 latency for {{ \$labels.job }}"
        description: "P99 latency is {{ \$value }}s, threshold is ${latency_threshold}s"
    
  - name: ${service_name}.resources
    rules:
    - alert: HighCPUUsage
      expr: |
        job:container_cpu_usage:rate5m{namespace=~".*${service_name}.*"} > 0.9
      for: 10m
      labels:
        severity: warning
        service: ${service_name}
      annotations:
        summary: "High CPU usage for {{ \$labels.container }}"
        description: "CPU usage is {{ \$value | humanizePercentage }}"
    
    - alert: HighMemoryUsage
      expr: |
        job:container_memory_usage:percentage > 90
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage for {{ \$labels.container }}"
        description: "Memory usage is {{ \$value }}%"
EOF
    
    success "Alert rules created for: $service_name"
}

# Configure Alertmanager routing
configure_alertmanager() {
    local slack_webhook="${1:-}"
    local pagerduty_key="${2:-}"
    local email="${3:-}"
    local namespace="${4:-monitoring}"
    
    log "Configuring Alertmanager routing..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-custom
  namespace: ${namespace}
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      slack_api_url: '${slack_webhook}'
      smtp_smarthost: 'smtp.example.com:587'
      smtp_from: 'alertmanager@example.com'
      smtp_auth_username: 'alertmanager'
      smtp_auth_password: 'password'
    
    templates:
    - '/etc/alertmanager/templates/*.tmpl'
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'slack-notifications'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty-critical'
        continue: true
      - match:
          severity: critical
        receiver: 'slack-critical'
      - match:
          severity: warning
        receiver: 'slack-warnings'
    
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
    
    - name: 'slack-critical'
      slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
        title: ':fire: CRITICAL: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    
    - name: 'slack-warnings'
      slack_configs:
      - channel: '#alerts-warnings'
        send_resolved: true
        title: ':warning: WARNING: {{ .GroupLabels.alertname }}'
    
    - name: 'pagerduty-critical'
      pagerduty_configs:
      - routing_key: '${pagerduty_key}'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
        severity: critical
    
    inhibit_rules:
    - source_match:
        severity: critical
      target_match:
        severity: warning
      equal: ['alertname', 'cluster', 'service']
EOF
    
    success "Alertmanager configured"
}

# Multi-cluster Prometheus federation
setup_prometheus_federation() {
    local parent_namespace="${1:-monitoring}"
    local child_clusters="${2:-cluster1,cluster2}"
    
    log "Setting up Prometheus federation..."
    
    IFS=',' read -ra clusters <<< "$child_clusters"
    
    # Create federation scrape config
    local federation_jobs=""
    for cluster in "${clusters[@]}"; do
        federation_jobs+="
      - job_name: '${cluster}-federation'
        scrape_interval: 15s
        honor_labels: true
        metrics_path: '/federate'
        params:
          'match[]':
          - '{__name__=~\"job:.*\"}'
          - '{__name__=~\"instance:.*\"}'
          - 'up'
          - 'ALERTS'
        static_configs:
        - targets:
          - 'prometheus.${cluster}.svc.cluster.local:9090'
          labels:
            cluster: '${cluster}'"
    done
    
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-federation-config
  namespace: ${parent_namespace}
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: management
        region: us-east-1
    
    scrape_configs:${federation_jobs}
EOF
    
    success "Prometheus federation configured"
}

# Thanos setup for long-term storage
setup_thanos() {
    local namespace="${1:-monitoring}"
    local object_store_bucket="${2:-my-thanos-bucket}"
    local object_store_region="${3:-us-east-1}"
    
    log "Setting up Thanos for long-term storage..."
    
    # Create Thanos object store config
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objstore-config
  namespace: ${namespace}
stringData:
  objstore.yml: |
    type: S3
    config:
      bucket: ${object_store_bucket}
      endpoint: s3.amazonaws.com
      region: ${object_store_region}
      insecure: false
EOF

    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update
    
    helm upgrade --install thanos bitnami/thanos \
        -n "$namespace" \
        --set existingObjstoreSecret=thanos-objstore-config \
        --set query.enabled=true \
        --set query.stores[0]=thanos-storegateway:10901 \
        --set storegateway.enabled=true \
        --set compactor.enabled=true \
        --set compactor.retentionResolutionRaw=30d \
        --set compactor.retentionResolution5m=90d \
        --set compactor.retentionResolution1h=1y \
        --set ruler.enabled=true \
        --wait
    
    success "Thanos setup complete"
}

# Main
case "${1:-help}" in
    recording-rules) create_recording_rules "${2:-monitoring}" "${3:-my-service}" ;;
    alert-rules) create_alert_rules "${2:-monitoring}" "${3:-my-service}" "${4:-99.9}" "${5:-500}" ;;
    alertmanager) configure_alertmanager "${2:-}" "${3:-}" "${4:-}" "${5:-monitoring}" ;;
    federation) setup_prometheus_federation "${2:-monitoring}" "${3:-cluster1,cluster2}" ;;
    thanos) setup_thanos "${2:-monitoring}" "${3:-my-thanos-bucket}" "${4:-us-east-1}" ;;
    *)
        echo "Usage: $0 {recording-rules|alert-rules|alertmanager|federation|thanos}"
        ;;
esac
```

---

## ขั้นตอนที่ 511: Distributed Tracing Advanced

```bash
#!/bin/bash
# distributed-tracing-advanced.sh - Advanced Distributed Tracing

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Setup Jaeger with Elasticsearch backend
setup_jaeger_production() {
    local namespace="${1:-observability}"
    local es_url="${2:-http://elasticsearch:9200}"
    local storage_type="${3:-elasticsearch}"
    
    log "Setting up Jaeger with $storage_type backend..."
    
    # Install Jaeger Operator
    kubectl create namespace "$namespace" --dry-run=client -o yaml | kubectl apply -f -
    
    kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/latest/download/jaeger-operator.yaml \
        -n "$namespace" 2>/dev/null || true
    
    # Wait for operator
    kubectl wait deployment jaeger-operator \
        -n "$namespace" \
        --for=condition=Available \
        --timeout=120s 2>/dev/null || log "Operator may need more time"
    
    # Create Jaeger instance
    cat <<EOF | kubectl apply -f -
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: ${namespace}
spec:
  strategy: production
  storage:
    type: ${storage_type}
    elasticsearch:
      serverUrls: ${es_url}
      indexPrefix: my-app
      username: elastic
      password: changeme
  collector:
    replicas: 3
    resources:
      limits:
        cpu: "500m"
        memory: "256Mi"
      requests:
        cpu: "100m"
        memory: "128Mi"
  query:
    replicas: 2
    metricsStorage:
      type: prometheus
    options:
      prometheus:
        server-url: http://prometheus:9090
  agent:
    strategy: sidecar
  ingress:
    enabled: true
    hosts:
    - jaeger.example.com
    tls:
    - secretName: jaeger-tls
      hosts:
      - jaeger.example.com
EOF
    
    success "Jaeger production setup complete"
}

# Create OpenTelemetry Collector
setup_otel_collector() {
    local namespace="${1:-observability}"
    local jaeger_endpoint="${2:-http://jaeger-collector:14268/api/traces}"
    local prometheus_port="${3:-8889}"
    
    log "Setting up OpenTelemetry Collector..."
    
    # Install OTel Operator
    kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml 2>/dev/null || true
    
    # Create OTel Collector config
    cat <<EOF | kubectl apply -f -
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: ${namespace}
spec:
  mode: deployment
  replicas: 2
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      zipkin:
        endpoint: 0.0.0.0:9411
      jaeger:
        protocols:
          grpc:
            endpoint: 0.0.0.0:14250
          thrift_http:
            endpoint: 0.0.0.0:14268
      prometheus:
        config:
          scrape_configs:
          - job_name: 'otel-collector'
            static_configs:
            - targets: ['localhost:8888']
    
    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
      memory_limiter:
        limit_mib: 512
        check_interval: 5s
      resource:
        attributes:
        - key: service.cluster
          value: "my-cluster"
          action: upsert
      tail_sampling:
        decision_wait: 10s
        num_traces: 100
        expected_new_traces_per_sec: 10
        policies:
        - name: error-in-policy
          type: status_code
          status_code: {status_codes: [ERROR]}
        - name: slow-traces-policy
          type: latency
          latency: {threshold_ms: 1000}
        - name: probabilistic-policy
          type: probabilistic
          probabilistic: {sampling_percentage: 10}
    
    exporters:
      jaeger:
        endpoint: ${jaeger_endpoint}
        tls:
          insecure: true
      prometheus:
        endpoint: "0.0.0.0:${prometheus_port}"
      otlp:
        endpoint: "http://tempo:4317"
        tls:
          insecure: true
      logging:
        loglevel: info
    
    extensions:
      health_check:
        endpoint: 0.0.0.0:13133
      pprof:
        endpoint: 0.0.0.0:1777
      zpages:
        endpoint: 0.0.0.0:55679
    
    service:
      extensions: [health_check, pprof, zpages]
      pipelines:
        traces:
          receivers: [otlp, zipkin, jaeger]
          processors: [memory_limiter, batch, resource, tail_sampling]
          exporters: [jaeger, otlp, logging]
        metrics:
          receivers: [otlp, prometheus]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
        logs:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [logging]
EOF
    
    success "OpenTelemetry Collector deployed"
}

# Auto-instrument applications with OTel
auto_instrument_application() {
    local namespace="${1:-default}"
    local language="${2:-nodejs}"  # nodejs, python, java, dotnet
    local app_label="${3:-my-app}"
    
    log "Auto-instrumenting $language application: $app_label"
    
    # Create Instrumentation resource
    cat <<EOF | kubectl apply -f -
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: ${app_label}-instrumentation
  namespace: ${namespace}
spec:
  exporter:
    endpoint: http://otel-collector:4317
  propagators:
  - tracecontext
  - baggage
  - b3
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"
  $(case "$language" in
    nodejs)
      cat <<NODEEOF
  nodejs:
    env:
    - name: OTEL_EXPORTER_OTLP_ENDPOINT
      value: http://otel-collector:4317
    - name: OTEL_METRICS_EXPORTER
      value: prometheus
    - name: OTEL_LOGS_EXPORTER
      value: otlp
NODEEOF
      ;;
    python)
      cat <<PYEOF
  python:
    env:
    - name: OTEL_EXPORTER_OTLP_ENDPOINT
      value: http://otel-collector:4317
    - name: OTEL_METRICS_EXPORTER
      value: prometheus
PYEOF
      ;;
    java)
      cat <<JAVAEOF
  java:
    env:
    - name: OTEL_EXPORTER_OTLP_ENDPOINT
      value: http://otel-collector:4317
    - name: OTEL_METRICS_EXPORTER
      value: prometheus
    - name: OTEL_JAVAAGENT_DEBUG
      value: "false"
JAVAEOF
      ;;
  esac)
EOF

    # Annotate namespace or specific deployments
    kubectl annotate namespace "$namespace" \
        "instrumentation.opentelemetry.io/inject-${language}=true" \
        --overwrite 2>/dev/null || \
    kubectl annotate deployment -n "$namespace" \
        -l "app=$app_label" \
        "instrumentation.opentelemetry.io/inject-${language}=true" \
        --overwrite
    
    success "Auto-instrumentation configured for $language apps in $namespace"
}

# Trace analysis
analyze_traces() {
    local jaeger_url="${1:-http://localhost:16686}"
    local service="${2:-my-service}"
    local lookback="${3:-1h}"
    local limit="${4:-20}"
    
    log "Analyzing traces for: $service (last $lookback)"
    
    # Query Jaeger API
    local response
    response=$(curl -s "${jaeger_url}/api/traces?service=${service}&lookback=${lookback}&limit=${limit}")
    
    if [[ -z "$response" ]]; then
        error "No response from Jaeger"
        return 1
    fi
    
    echo "=== Trace Summary ==="
    echo "$response" | jq -r '
        .data[] | 
        "TraceID: \(.traceID) | Duration: \(.spans[0].duration / 1000)ms | Spans: \(.spans | length) | Root: \(.spans[0].operationName)"
    ' 2>/dev/null | head -20
    
    echo ""
    echo "=== Error Traces ==="
    echo "$response" | jq -r '
        .data[] | 
        select(.spans[] | .tags[] | select(.key == "error" and .value == true)) |
        "TraceID: \(.traceID) | Operation: \(.spans[0].operationName)"
    ' 2>/dev/null | head -10
    
    echo ""
    echo "=== Slow Traces (top 5) ==="
    echo "$response" | jq -r '
        [.data[] | {traceID: .traceID, duration: .spans[0].duration, op: .spans[0].operationName}] |
        sort_by(-.duration) |
        .[0:5][] |
        "TraceID: \(.traceID) | Duration: \(.duration / 1000)ms | Op: \(.op)"
    ' 2>/dev/null
}

# Setup Tempo for traces
setup_tempo() {
    local namespace="${1:-observability}"
    local s3_bucket="${2:-my-tempo-bucket}"
    local region="${3:-us-east-1}"
    
    log "Setting up Grafana Tempo..."
    
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update
    
    cat <<EOF > /tmp/tempo-values.yaml
tempo:
  storage:
    trace:
      backend: s3
      s3:
        bucket: ${s3_bucket}
        endpoint: s3.amazonaws.com
        region: ${region}
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: "0.0.0.0:4317"
        http:
          endpoint: "0.0.0.0:4318"
    zipkin:
      endpoint: "0.0.0.0:9411"
    jaeger:
      protocols:
        thrift_http:
          endpoint: "0.0.0.0:14268"
        grpc:
          endpoint: "0.0.0.0:14250"
  querier:
    max_concurrent_queries: 20
  query_frontend:
    max_retries: 2
serviceMonitor:
  enabled: true
EOF
    
    helm upgrade --install tempo grafana/tempo-distributed \
        -n "$namespace" \
        --create-namespace \
        -f /tmp/tempo-values.yaml \
        --wait
    
    success "Tempo installed in namespace: $namespace"
}

# Main
case "${1:-help}" in
    jaeger) setup_jaeger_production "${2:-observability}" "${3:-http://elasticsearch:9200}" "${4:-elasticsearch}" ;;
    otel-collector) setup_otel_collector "${2:-observability}" "${3:-}" "${4:-8889}" ;;
    auto-instrument) auto_instrument_application "${2:-default}" "${3:-nodejs}" "${4:-my-app}" ;;
    analyze) analyze_traces "${2:-http://localhost:16686}" "${3:-my-service}" "${4:-1h}" "${5:-20}" ;;
    tempo) setup_tempo "${2:-observability}" "${3:-my-tempo-bucket}" "${4:-us-east-1}" ;;
    *)
        echo "Usage: $0 {jaeger|otel-collector|auto-instrument|analyze|tempo}"
        ;;
esac
```

---

## ขั้นตอนที่ 512: Log Management Advanced - Loki Stack

```bash
#!/bin/bash
# log-management-loki.sh - Advanced Log Management with Loki

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Install Loki stack
setup_loki_stack() {
    local namespace="${1:-monitoring}"
    local s3_bucket="${2:-my-loki-bucket}"
    local region="${3:-us-east-1}"
    local retention_days="${4:-30}"
    
    log "Setting up Loki stack..."
    
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update
    
    cat <<EOF > /tmp/loki-values.yaml
loki:
  schemaConfig:
    configs:
    - from: 2024-01-01
      store: tsdb
      object_store: s3
      schema: v12
      index:
        prefix: loki_index_
        period: 24h
  storage:
    type: s3
    s3:
      bucketnames: ${s3_bucket}
      region: ${region}
  compactor:
    retention_enabled: true
    retention_delete_delay: 2h
    retention_delete_worker_count: 150
  limits_config:
    retention_period: ${retention_days}d
    max_cache_freshness_per_query: 10m
    max_query_length: 0h
    max_query_parallelism: 32
    reject_old_samples: true
    reject_old_samples_max_age: 168h
  query_scheduler:
    max_outstanding_requests_per_tenant: 32768
  querier:
    max_concurrent: 16
  frontend:
    max_outstanding_per_tenant: 2048
  ruler:
    enable_api: true
    enable_alertmanager_v2: true
    alertmanager_url: http://alertmanager:9093

promtail:
  config:
    clients:
    - url: http://loki:3100/loki/api/v1/push
    scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      pipeline_stages:
      - cri: {}
      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'
          max_wait_time: 3s
      - json:
          expressions:
            level: level
            message: message
            timestamp: timestamp
      - labels:
          level:
      - timestamp:
          source: timestamp
          format: RFC3339Nano
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_container_name]
        target_label: container
      - source_labels: [__meta_kubernetes_pod_node_name]
        target_label: node

grafana:
  enabled: true
  sidecar:
    datasources:
      enabled: true
  additionalDataSources:
  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      maxLines: 1000
      derivedFields:
      - datasourceName: Jaeger
        matcherRegex: "(?:traceID|trace_id)=(\\w+)"
        name: TraceID
        url: "\$\${__value.raw}"
        datasourceUid: jaeger
EOF
    
    helm upgrade --install loki grafana/loki-stack \
        -n "$namespace" \
        --create-namespace \
        -f /tmp/loki-values.yaml \
        --wait
    
    success "Loki stack installed"
}

# Create Loki alert rules
create_loki_alerts() {
    local namespace="${1:-monitoring}"
    local service_name="${2:-my-service}"
    
    log "Creating Loki alert rules for: $service_name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: ${service_name}-loki-rules
  namespace: ${namespace}
  labels:
    loki_rule: "true"
data:
  ${service_name}-alerts.yaml: |
    groups:
    - name: ${service_name}.logs
      rules:
      - alert: HighErrorLogRate
        expr: |
          sum(rate({app="${service_name}", level="error"}[5m])) > 10
        for: 5m
        labels:
          severity: warning
          service: ${service_name}
        annotations:
          summary: "High error log rate for ${service_name}"
          description: "Error log rate is {{ \$value }} per second"
      
      - alert: CriticalLogDetected
        expr: |
          count_over_time({app="${service_name}"} |= "CRITICAL" [1m]) > 0
        for: 0m
        labels:
          severity: critical
          service: ${service_name}
        annotations:
          summary: "CRITICAL log detected in ${service_name}"
      
      - alert: PanicDetected
        expr: |
          count_over_time({app="${service_name}"} |~ "panic|PANIC|fatal|FATAL" [1m]) > 0
        for: 0m
        labels:
          severity: critical
          service: ${service_name}
        annotations:
          summary: "Panic/Fatal log detected in ${service_name}"
      
      - alert: DatabaseErrorLogs
        expr: |
          sum(rate({app="${service_name}"} |= "database error" [5m])) > 5
        for: 5m
        labels:
          severity: warning
          service: ${service_name}
        annotations:
          summary: "Database errors in ${service_name}"
EOF
    
    success "Loki alert rules created for: $service_name"
}

# LogQL query helpers
run_logql_query() {
    local loki_url="${1:-http://localhost:3100}"
    local query="${2:-}"
    local since="${3:-1h}"
    local limit="${4:-100}"
    
    if [[ -z "$query" ]]; then
        error "LogQL query required"
        echo ""
        echo "Example queries:"
        echo "  Error logs: {app=\"my-app\",level=\"error\"}"
        echo "  Slow requests: {app=\"my-app\"} | json | duration > 1s"
        echo "  Count errors: sum(rate({app=\"my-app\",level=\"error\"}[5m]))"
        return 1
    fi
    
    log "Running LogQL query..."
    
    # Range query
    local start_time
    start_time=$(date -d "$since ago" +%s%N 2>/dev/null || \
                 date -v "-${since}" +%s%N 2>/dev/null || \
                 echo "$(($(date +%s) - 3600))000000000")
    
    curl -s "${loki_url}/loki/api/v1/query_range" \
        --data-urlencode "query=$query" \
        --data-urlencode "start=$start_time" \
        --data-urlencode "end=$(date +%s%N)" \
        --data-urlencode "limit=$limit" | \
    jq -r '.data.result[] | .stream | to_entries[] | "\(.key)=\(.value)"' 2>/dev/null || \
    echo "No results or Loki not available"
}

# Log aggregation dashboard
create_log_dashboard() {
    local grafana_url="${1:-http://localhost:3000}"
    local grafana_user="${2:-admin}"
    local grafana_pass="${3:-admin}"
    local service_name="${4:-my-service}"
    
    log "Creating log dashboard for: $service_name"
    
    local dashboard
    dashboard=$(cat <<EOF
{
  "dashboard": {
    "title": "${service_name} - Logs Dashboard",
    "tags": ["logs", "${service_name}"],
    "panels": [
      {
        "type": "logs",
        "title": "Application Logs",
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 0},
        "datasource": "Loki",
        "targets": [{
          "expr": "{app=\"${service_name}\"}",
          "refId": "A"
        }],
        "options": {
          "showLabels": true,
          "showTime": true,
          "wrapLogMessage": true,
          "dedupStrategy": "none"
        }
      },
      {
        "type": "timeseries",
        "title": "Log Rate by Level",
        "gridPos": {"h": 6, "w": 12, "x": 0, "y": 8},
        "datasource": "Loki",
        "targets": [
          {
            "expr": "sum(rate({app=\"${service_name}\", level=\"error\"}[5m]))",
            "legendFormat": "errors",
            "refId": "A"
          },
          {
            "expr": "sum(rate({app=\"${service_name}\", level=\"warn\"}[5m]))",
            "legendFormat": "warnings",
            "refId": "B"
          },
          {
            "expr": "sum(rate({app=\"${service_name}\", level=\"info\"}[5m]))",
            "legendFormat": "info",
            "refId": "C"
          }
        ]
      }
    ]
  },
  "overwrite": true
}
EOF
)
    
    curl -s -X POST \
        -H "Content-Type: application/json" \
        -u "${grafana_user}:${grafana_pass}" \
        "${grafana_url}/api/dashboards/db" \
        -d "$dashboard" | jq '.url // .message'
    
    success "Log dashboard created for: $service_name"
}

# Main
case "${1:-help}" in
    setup) setup_loki_stack "${2:-monitoring}" "${3:-my-loki-bucket}" "${4:-us-east-1}" "${5:-30}" ;;
    alerts) create_loki_alerts "${2:-monitoring}" "${3:-my-service}" ;;
    query) run_logql_query "${2:-http://localhost:3100}" "${3:-}" "${4:-1h}" "${5:-100}" ;;
    dashboard) create_log_dashboard "${2:-http://localhost:3000}" "${3:-admin}" "${4:-admin}" "${5:-my-service}" ;;
    *)
        echo "Usage: $0 {setup|alerts|query|dashboard}"
        ;;
esac
```

---

## สรุป Part 38

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | เครื่องมือ | ขั้นตอน |
|--------|-----------|---------|
| Advanced Prometheus | Recording Rules, Alertmanager, Thanos | 510 |
| Distributed Tracing Advanced | Jaeger Production, OTel Collector, Tempo | 511 |
| Log Management Advanced | Loki Stack, LogQL, Log Alerts | 512 |

**เทคโนโลยีที่ใช้:**
- Prometheus Operator: Recording/Alert rules CRD
- Thanos: Long-term metric storage
- Jaeger/Tempo: Distributed tracing
- OpenTelemetry Operator: Auto-instrumentation
- Loki: Log aggregation
- Grafana: Unified observability

ขั้นตอนต่อไป: Part 39 - Performance Engineering และ Load Testing
