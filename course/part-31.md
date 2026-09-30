# Part 31: Observability — Metrics, Tracing, และ Logging

## Module 3: Advanced Level — การทำงานระดับสูง

---

## ขั้นตอนที่ 483: Prometheus Metrics Collection

```bash
#!/bin/bash
# prometheus_manager.sh - จัดการ Prometheus

PROMETHEUS_URL="${PROMETHEUS_URL:-http://localhost:9090}"
ALERTMANAGER_URL="${ALERTMANAGER_URL:-http://localhost:9093}"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# Query Prometheus
prometheus_query() {
    local query="$1"
    local time="${2:-}"
    
    local url="${PROMETHEUS_URL}/api/v1/query"
    local params="query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${query}'))")"
    
    [[ -n "$time" ]] && params+="&time=${time}"
    
    curl -sf "${url}?${params}" 2>/dev/null | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
status = data.get('status', 'unknown')
if status != 'success':
    print(f'Error: {status}')
    sys.exit(1)

results = data.get('data', {}).get('result', [])
for r in results:
    metric = r.get('metric', {})
    value = r.get('value', ['', ''])
    labels = ', '.join(f'{k}=\"{v}\"' for k,v in metric.items() if k != '__name__')
    name = metric.get('__name__', 'unknown')
    print(f'{name}{{{labels}}} = {value[1]}')
" 2>/dev/null
}

# Range query
prometheus_range() {
    local query="$1"
    local start="${2:-$(date -d '1 hour ago' +%s)}"
    local end="${3:-$(date +%s)}"
    local step="${4:-60}"
    
    local url="${PROMETHEUS_URL}/api/v1/query_range"
    local params
    params="query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${query}'))")"
    params+="&start=${start}&end=${end}&step=${step}"
    
    curl -sf "${url}?${params}" 2>/dev/null | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
results = data.get('data', {}).get('result', [])
for r in results:
    metric = r.get('metric', {})
    values = r.get('values', [])
    labels = metric.get('instance', metric.get('__name__', 'unknown'))
    print(f'--- {labels} ---')
    # แสดงแค่ค่าล่าสุด
    if values:
        print(f'  Latest: {values[-1][1]} at {values[-1][0]}')
        avg = sum(float(v[1]) for v in values if v[1] != 'NaN') / max(len(values), 1)
        print(f'  Average: {avg:.3f}')
" 2>/dev/null
}

# Dashboard queries
show_system_dashboard() {
    echo "=== System Metrics Dashboard ==="
    echo "Time: $(date)"
    echo ""
    
    # CPU usage
    echo "=== CPU Usage ==="
    prometheus_query '100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)'
    
    echo ""
    echo "=== Memory Usage ==="
    prometheus_query '(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100'
    
    echo ""
    echo "=== Disk Usage ==="
    prometheus_query '(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100'
    
    echo ""
    echo "=== Network Traffic (bytes/s) ==="
    prometheus_query 'rate(node_network_receive_bytes_total[5m])'
    
    echo ""
    echo "=== HTTP Request Rate ==="
    prometheus_query 'sum(rate(http_requests_total[5m])) by (service)'
    
    echo ""
    echo "=== HTTP Error Rate ==="
    prometheus_query 'sum(rate(http_requests_total{status=~"5.."}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) * 100'
}

# ตรวจสอบ alerts
check_alerts() {
    echo "=== Active Alerts ==="
    
    curl -sf "${ALERTMANAGER_URL}/api/v2/alerts" 2>/dev/null | \
    python3 -c "
import json, sys
alerts = json.load(sys.stdin)
if not alerts:
    print('  No active alerts')
else:
    print(f'  Total: {len(alerts)} alerts')
    print()
    for alert in alerts:
        labels = alert.get('labels', {})
        annotations = alert.get('annotations', {})
        status = alert.get('status', {})
        
        name = labels.get('alertname', 'Unknown')
        severity = labels.get('severity', 'unknown')
        state = status.get('state', 'unknown')
        
        sev_icons = {'critical': '🔴', 'warning': '⚠️', 'info': 'ℹ️'}
        icon = sev_icons.get(severity, '?')
        
        print(f'{icon} {name} ({severity}) - {state}')
        print(f'   Summary: {annotations.get(\"summary\", \"N/A\")}')
        print()
" 2>/dev/null || echo "  Cannot reach Alertmanager"
}

# สร้าง Prometheus config
generate_prometheus_config() {
    local output_file="${1:-/tmp/prometheus.yml}"
    
    cat > "$output_file" << 'YAML'
# prometheus.yml - Prometheus Configuration

global:
  scrape_interval:     15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'production'
    environment: 'production'

# Alerting
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Rules
rule_files:
  - "rules/*.yml"
  - "alerts/*.yml"

# Scrape configs
scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter
  - job_name: 'node'
    static_configs:
      - targets:
          - 'node-01:9100'
          - 'node-02:9100'
          - 'node-03:9100'
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: instance

  # Kubernetes pods
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  # Kubernetes services
  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2

  # MySQL Exporter
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql-exporter:9104']

  # Redis Exporter
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  # Blackbox Exporter (HTTP probes)
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://api.example.com/health
          - https://www.example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
YAML
    
    echo "✓ สร้าง Prometheus config: ${output_file}"
}

# สร้าง alert rules
generate_alert_rules() {
    local output_file="${1:-/tmp/alerts.yml}"
    
    cat > "$output_file" << 'YAML'
# alerts.yml - Prometheus Alert Rules

groups:
  - name: infrastructure
    interval: 30s
    rules:
      # CPU
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}"
      
      - alert: CriticalCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanizePercentage }}"
      
      # Memory
      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanizePercentage }}"
      
      # Disk
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "{{ $labels.mountpoint }} has {{ $value | humanizePercentage }} free"
      
      - alert: DiskSpaceCritical
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical disk space on {{ $labels.instance }}"
          description: "Only {{ $value | humanizePercentage }} disk space remaining"
  
  - name: applications
    rules:
      # HTTP errors
      - alert: HighHTTPErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) * 100 > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High HTTP error rate for {{ $labels.service }}"
          description: "{{ $value | humanizePercentage }} error rate"
      
      # Latency
      - alert: HighLatency
        expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency for {{ $labels.service }}"
          description: "P99 latency is {{ $value }}s"
      
      # Pod restarts
      - alert: FrequentPodRestarts
        expr: increase(kube_pod_container_status_restarts_total[30m]) > 3
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} restarting frequently"
          description: "{{ $value }} restarts in last 30 minutes"
      
      # PVC space
      - alert: PersistentVolumeFillingUp
        expr: kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes < 0.1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PVC {{ $labels.persistentvolumeclaim }} filling up"
          description: "Only {{ $value | humanizePercentage }} available"
YAML
    
    echo "✓ สร้าง alert rules: ${output_file}"
}

# Custom metrics exporter
custom_metrics_exporter() {
    local port="${1:-9101}"
    
    echo "=== Starting Custom Metrics Exporter (port: ${port}) ==="
    
    # สร้าง metrics collector script
    local metrics_script="/tmp/custom_metrics_$$.sh"
    
    cat > "$metrics_script" << 'SCRIPT'
#!/bin/bash
# รวบรวม custom metrics

collect_metrics() {
    local timestamp=$(date +%s)
    
    # System metrics
    local load_avg
    load_avg=$(awk '{print $1}' /proc/loadavg)
    
    local memory_used
    memory_used=$(free -b | awk '/Mem:/ {print $3}')
    
    local memory_total
    memory_total=$(free -b | awk '/Mem:/ {print $2}')
    
    # Process metrics
    local process_count
    process_count=$(ps aux | wc -l)
    
    # Output Prometheus format
    cat << METRICS
# HELP system_load_average System load average (1 minute)
# TYPE system_load_average gauge
system_load_average ${load_avg}

# HELP system_memory_used_bytes Memory currently in use
# TYPE system_memory_used_bytes gauge
system_memory_used_bytes ${memory_used}

# HELP system_memory_total_bytes Total system memory
# TYPE system_memory_total_bytes gauge
system_memory_total_bytes ${memory_total}

# HELP system_process_count Total number of processes
# TYPE system_process_count gauge
system_process_count ${process_count}

# HELP custom_timestamp_seconds Timestamp of metrics collection
# TYPE custom_timestamp_seconds gauge
custom_timestamp_seconds ${timestamp}
METRICS
}

# HTTP server สำหรับ metrics
serve_metrics() {
    local port="$1"
    
    while true; do
        # รับ HTTP request
        echo "Waiting for connection on port ${port}..."
        
        {
            read request
            while IFS= read -r line; do
                [[ -z "$line" ]] && break
            done
            
            # Output metrics
            local metrics
            metrics=$(collect_metrics)
            local length=${#metrics}
            
            printf "HTTP/1.1 200 OK\r\n"
            printf "Content-Type: text/plain; version=0.0.4; charset=utf-8\r\n"
            printf "Content-Length: %d\r\n" "$length"
            printf "\r\n"
            printf "%s" "$metrics"
        } | nc -l -p "$port" -q 1
    done
}

serve_metrics "$1"
SCRIPT
    
    chmod +x "$metrics_script"
    echo "Starting metrics server on :${port}..."
    bash "$metrics_script" "$port" &
    local pid=$!
    
    echo "✓ Custom metrics exporter started (PID: ${pid})"
    echo "  Test: curl http://localhost:${port}/metrics"
    echo "  Stop: kill ${pid}"
}

# Main
case "${1:-}" in
    query)      prometheus_query "${2}" "${3:-}" ;;
    range)      prometheus_range "${2}" "${3:-}" "${4:-}" "${5:-60}" ;;
    dashboard)  show_system_dashboard ;;
    alerts)     check_alerts ;;
    config)     generate_prometheus_config "${2:-/tmp/prometheus.yml}" ;;
    rules)      generate_alert_rules "${2:-/tmp/alerts.yml}" ;;
    exporter)   custom_metrics_exporter "${2:-9101}" ;;
    *)
        echo "Usage: $0 {query|range|dashboard|alerts|config|rules|exporter} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 484: Distributed Tracing ด้วย Jaeger/Zipkin

```bash
#!/bin/bash
# distributed_tracing.sh - จัดการ Distributed Tracing

JAEGER_URL="${JAEGER_URL:-http://localhost:16686}"
ZIPKIN_URL="${ZIPKIN_URL:-http://localhost:9411}"

# Query Jaeger traces
query_jaeger_traces() {
    local service="$1"
    local operation="${2:-}"
    local limit="${3:-20}"
    local lookback="${4:-1h}"
    
    echo "=== Jaeger Traces: ${service} ==="
    
    local params="service=${service}&limit=${limit}&lookback=${lookback}"
    [[ -n "$operation" ]] && params+="&operation=${operation}"
    
    curl -sf "${JAEGER_URL}/api/traces?${params}" 2>/dev/null | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
traces = data.get('data', [])
print(f'Found: {len(traces)} traces')
print()
for trace in traces[:10]:
    trace_id = trace.get('traceID', 'unknown')
    spans = trace.get('spans', [])
    
    if not spans:
        continue
    
    # หา root span
    root = next((s for s in spans if not s.get('references')), spans[0])
    duration = root.get('duration', 0) / 1000  # microseconds to ms
    start_time = root.get('startTime', 0) / 1000000  # microseconds to seconds
    
    errors = sum(1 for s in spans 
                 for tag in s.get('tags', []) 
                 if tag.get('key') == 'error' and tag.get('value') == True)
    
    status = '✗' if errors > 0 else '✓'
    print(f'{status} TraceID: {trace_id[:12]}...')
    print(f'   Spans: {len(spans)} | Duration: {duration:.2f}ms | Errors: {errors}')
    print()
" 2>/dev/null || echo "Cannot connect to Jaeger"
}

# สร้าง trace span ด้วย OpenTelemetry format
create_trace_span() {
    local service_name="$1"
    local operation_name="$2"
    local parent_span_id="${3:-}"
    
    local trace_id
    trace_id=$(python3 -c "import secrets; print(secrets.token_hex(16))" 2>/dev/null || \
               cat /proc/sys/kernel/random/uuid | tr -d '-')
    
    local span_id
    span_id=$(python3 -c "import secrets; print(secrets.token_hex(8))" 2>/dev/null || \
              echo "$RANDOM$RANDOM")
    
    local start_time=$(date +%s%N)
    local start_time_us=$((start_time / 1000))
    
    # รัน operation
    echo "TRACE_ID=${trace_id}" 
    echo "SPAN_ID=${span_id}"
    
    # ส่ง span ไป Jaeger via HTTP
    local span_json
    span_json=$(cat << EOF
[{
    "traceID": "${trace_id}",
    "spanID": "${span_id}",
    "operationName": "${operation_name}",
    "startTime": ${start_time_us},
    "duration": 1000,
    "tags": [
        {"key": "service.name", "type": "string", "value": "${service_name}"},
        {"key": "component", "type": "string", "value": "bash"}
    ],
    "logs": [],
    "processID": "p1",
    "process": {
        "serviceName": "${service_name}",
        "tags": [
            {"key": "hostname", "type": "string", "value": "$(hostname)"}
        ]
    },
    "references": ${parent_span_id:+[{"refType": "CHILD_OF", "traceID": "${trace_id}", "spanID": "${parent_span_id}"}]:-[]}
}]
EOF
)
    
    echo "Trace created:"
    echo "  TraceID: ${trace_id}"
    echo "  SpanID:  ${span_id}"
    echo "  Service: ${service_name}"
    echo "  Op:      ${operation_name}"
}

# Bash OpenTelemetry wrapper
otlp_trace() {
    local service_name="$1"
    local operation_name="$2"
    shift 2
    local command="$*"
    
    # สร้าง trace context
    local trace_id span_id
    trace_id=$(cat /dev/urandom | tr -dc 'a-f0-9' | head -c 32)
    span_id=$(cat /dev/urandom | tr -dc 'a-f0-9' | head -c 16)
    
    # Export trace context สำหรับ children
    export TRACEPARENT="00-${trace_id}-${span_id}-01"
    export OTEL_SERVICE_NAME="$service_name"
    
    local start_ns=$(date +%s%N)
    local status="ok"
    
    # รัน command
    if eval "$command"; then
        status="ok"
    else
        status="error"
    fi
    
    local end_ns=$(date +%s%N)
    local duration_ms=$(( (end_ns - start_ns) / 1000000 ))
    
    echo ""
    echo "Trace completed:"
    echo "  TraceID:   ${trace_id}"
    echo "  SpanID:    ${span_id}"
    echo "  Operation: ${operation_name}"
    echo "  Duration:  ${duration_ms}ms"
    echo "  Status:    ${status}"
    
    # ส่งไป OTLP collector ถ้ามี
    local otlp_endpoint="${OTLP_ENDPOINT:-http://localhost:4318}"
    
    local span_data
    span_data=$(cat << EOF
{
    "resourceSpans": [{
        "resource": {
            "attributes": [
                {"key": "service.name", "value": {"stringValue": "${service_name}"}},
                {"key": "telemetry.sdk.name", "value": {"stringValue": "bash-otel"}}
            ]
        },
        "scopeSpans": [{
            "scope": {"name": "bash"},
            "spans": [{
                "traceId": "${trace_id}",
                "spanId": "${span_id}",
                "name": "${operation_name}",
                "kind": 1,
                "startTimeUnixNano": ${start_ns},
                "endTimeUnixNano": ${end_ns},
                "status": {"code": $([ "$status" == "ok" ] && echo 1 || echo 2)},
                "attributes": [
                    {"key": "bash.command", "value": {"stringValue": "${command}"}}
                ]
            }]
        }]
    }]
}
EOF
)
    
    curl -sf \
        -X POST \
        -H "Content-Type: application/json" \
        "${otlp_endpoint}/v1/traces" \
        -d "$span_data" &>/dev/null || true
}

# Zipkin integration
send_zipkin_span() {
    local service_name="$1"
    local span_name="$2"
    local duration_us="${3:-1000}"
    
    local trace_id
    trace_id=$(cat /dev/urandom | tr -dc 'a-f0-9' | head -c 16)
    
    local span_id
    span_id=$(cat /dev/urandom | tr -dc 'a-f0-9' | head -c 16)
    
    local timestamp=$(( $(date +%s%N) / 1000 ))  # microseconds
    
    local span_data
    span_data=$(cat << EOF
[{
    "traceId": "${trace_id}",
    "id": "${span_id}",
    "name": "${span_name}",
    "timestamp": ${timestamp},
    "duration": ${duration_us},
    "localEndpoint": {
        "serviceName": "${service_name}",
        "ipv4": "$(hostname -I | awk '{print $1}')"
    },
    "tags": {
        "component": "bash",
        "peer.service": "${service_name}"
    }
}]
EOF
)
    
    if curl -sf \
        -X POST \
        -H "Content-Type: application/json" \
        "${ZIPKIN_URL}/api/v2/spans" \
        -d "$span_data" &>/dev/null; then
        echo "✓ Span sent to Zipkin"
    else
        echo "✗ Failed to send to Zipkin"
    fi
}

# Main
case "${1:-}" in
    query)     query_jaeger_traces "${2}" "${3:-}" "${4:-20}" "${5:-1h}" ;;
    span)      create_trace_span "${2}" "${3}" "${4:-}" ;;
    trace)     otlp_trace "${2}" "${3}" "${@:4}" ;;
    zipkin)    send_zipkin_span "${2}" "${3}" "${4:-1000}" ;;
    *)
        echo "Usage: $0 {query|span|trace|zipkin} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 485: Grafana Dashboard Management

```bash
#!/bin/bash
# grafana_manager.sh - จัดการ Grafana

GRAFANA_URL="${GRAFANA_URL:-http://localhost:3000}"
GRAFANA_USER="${GRAFANA_USER:-admin}"
GRAFANA_PASS="${GRAFANA_PASS:-admin}"

# เรียก Grafana API
grafana_api() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local curl_args=(
        -s
        -X "$method"
        -u "${GRAFANA_USER}:${GRAFANA_PASS}"
        -H "Content-Type: application/json"
    )
    
    [[ -n "$data" ]] && curl_args+=(-d "$data")
    
    curl "${curl_args[@]}" "${GRAFANA_URL}/api${endpoint}"
}

# แสดงรายการ dashboards
list_dashboards() {
    local folder="${1:-}"
    
    echo "=== Grafana Dashboards ==="
    
    grafana_api "GET" "/search?type=dash-db${folder:+&folderIds=${folder}}" | \
    python3 -c "
import json, sys
dashboards = json.load(sys.stdin)
print(f'Total: {len(dashboards)} dashboards')
print()
for d in dashboards:
    print(f'[{d.get(\"id\", \"?\")}] {d[\"title\"]}')
    print(f'   URL: {d.get(\"url\", \"N/A\")}')
    print(f'   Tags: {d.get(\"tags\", [])}')
    print()
" 2>/dev/null
}

# สร้าง dashboard
create_dashboard() {
    local title="$1"
    local data_source="${2:-Prometheus}"
    
    echo "=== Creating Dashboard: ${title} ==="
    
    local dashboard_json
    dashboard_json=$(cat << JSON
{
    "dashboard": {
        "title": "${title}",
        "tags": ["generated", "bash"],
        "timezone": "Asia/Bangkok",
        "refresh": "30s",
        "time": {
            "from": "now-1h",
            "to": "now"
        },
        "panels": [
            {
                "id": 1,
                "title": "CPU Usage",
                "type": "timeseries",
                "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
                "datasource": "${data_source}",
                "targets": [{
                    "expr": "100 - (avg(rate(node_cpu_seconds_total{mode=\\"idle\\"}[5m])) * 100)",
                    "legendFormat": "CPU %"
                }],
                "fieldConfig": {
                    "defaults": {
                        "unit": "percent",
                        "min": 0,
                        "max": 100,
                        "thresholds": {
                            "mode": "absolute",
                            "steps": [
                                {"color": "green", "value": null},
                                {"color": "yellow", "value": 70},
                                {"color": "red", "value": 85}
                            ]
                        }
                    }
                }
            },
            {
                "id": 2,
                "title": "Memory Usage",
                "type": "gauge",
                "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
                "datasource": "${data_source}",
                "targets": [{
                    "expr": "(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100",
                    "legendFormat": "Memory %"
                }],
                "fieldConfig": {
                    "defaults": {
                        "unit": "percent",
                        "min": 0,
                        "max": 100,
                        "thresholds": {
                            "mode": "absolute",
                            "steps": [
                                {"color": "green", "value": null},
                                {"color": "yellow", "value": 70},
                                {"color": "red", "value": 85}
                            ]
                        }
                    }
                }
            },
            {
                "id": 3,
                "title": "HTTP Request Rate",
                "type": "timeseries",
                "gridPos": {"h": 8, "w": 24, "x": 0, "y": 8},
                "datasource": "${data_source}",
                "targets": [
                    {
                        "expr": "sum(rate(http_requests_total{status=~\\"2..\\"}[5m])) by (service)",
                        "legendFormat": "{{service}} - Success"
                    },
                    {
                        "expr": "sum(rate(http_requests_total{status=~\\"5..\\"}[5m])) by (service)",
                        "legendFormat": "{{service}} - Error"
                    }
                ],
                "fieldConfig": {
                    "defaults": {"unit": "reqps"}
                }
            }
        ]
    },
    "overwrite": false,
    "folderId": 0
}
JSON
)
    
    grafana_api "POST" "/dashboards/db" "$dashboard_json" | \
    python3 -c "
import json, sys
result = json.load(sys.stdin)
status = result.get('status', 'unknown')
if status == 'success':
    print(f'✓ Dashboard created: {result.get(\"slug\", \"N/A\")}')
    print(f'   URL: ${GRAFANA_URL}{result.get(\"url\", \"\")}')
else:
    print(f'✗ Failed: {result}')
" 2>/dev/null
}

# Export dashboard
export_dashboard() {
    local uid="$1"
    local output_file="${2:-/tmp/dashboard-${uid}.json}"
    
    echo "=== Exporting Dashboard: ${uid} ==="
    
    grafana_api "GET" "/dashboards/uid/${uid}" | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
dashboard = data.get('dashboard', {})
print(json.dumps(dashboard, indent=2))
" > "$output_file" 2>/dev/null
    
    echo "✓ Dashboard exported: ${output_file}"
}

# Import dashboard
import_dashboard() {
    local dashboard_file="$1"
    local folder_id="${2:-0}"
    
    echo "=== Importing Dashboard: ${dashboard_file} ==="
    
    if [[ ! -f "$dashboard_file" ]]; then
        echo "ERROR: File not found: ${dashboard_file}"
        return 1
    fi
    
    local payload
    payload=$(python3 -c "
import json, sys
with open('${dashboard_file}') as f:
    dashboard = json.load(f)

# Reset ID for import
dashboard.pop('id', None)
dashboard.pop('uid', None)
dashboard['title'] = dashboard.get('title', 'Imported Dashboard') + ' (Imported)'

print(json.dumps({
    'dashboard': dashboard,
    'overwrite': True,
    'folderId': ${folder_id}
}))
" 2>/dev/null)
    
    grafana_api "POST" "/dashboards/db" "$payload" | \
    python3 -c "
import json, sys
result = json.load(sys.stdin)
if result.get('status') == 'success':
    print(f'✓ Imported: {result.get(\"slug\")}')
else:
    print(f'Result: {result}')
" 2>/dev/null
}

# จัดการ data sources
manage_datasource() {
    local action="$1"
    
    case "$action" in
        list)
            grafana_api "GET" "/datasources" | \
            python3 -c "
import json, sys
sources = json.load(sys.stdin)
for ds in sources:
    print(f'[{ds[\"id\"]}] {ds[\"name\"]} ({ds[\"type\"]}) - {\"default\" if ds.get(\"isDefault\") else \"\"}')" 2>/dev/null
            ;;
        
        add-prometheus)
            local prom_url="${2:-http://prometheus:9090}"
            local name="${3:-Prometheus}"
            
            grafana_api "POST" "/datasources" "{
                \"name\": \"${name}\",
                \"type\": \"prometheus\",
                \"url\": \"${prom_url}\",
                \"access\": \"proxy\",
                \"isDefault\": true
            }" | python3 -c "import json,sys; d=json.load(sys.stdin); print('✓' if 'datasource' in d else '✗', d.get('message', ''))" 2>/dev/null
            ;;
    esac
}

# Main
case "${1:-}" in
    list)       list_dashboards "${2:-}" ;;
    create)     create_dashboard "${2}" "${3:-Prometheus}" ;;
    export)     export_dashboard "${2}" "${3:-}" ;;
    import)     import_dashboard "${2}" "${3:-0}" ;;
    datasource) manage_datasource "${2}" "${@:3}" ;;
    *)
        echo "Usage: $0 {list|create|export|import|datasource} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 486: ELK Stack Integration

```bash
#!/bin/bash
# elk_manager.sh - จัดการ ELK Stack

ELASTICSEARCH_URL="${ELASTICSEARCH_URL:-http://localhost:9200}"
KIBANA_URL="${KIBANA_URL:-http://localhost:5601}"
LOGSTASH_HOST="${LOGSTASH_HOST:-localhost}"
LOGSTASH_PORT="${LOGSTASH_PORT:-5044}"

# Elasticsearch health
es_health() {
    echo "=== Elasticsearch Health ==="
    
    curl -sf "${ELASTICSEARCH_URL}/_cluster/health?pretty" 2>/dev/null | \
    python3 -c "
import json, sys
h = json.load(sys.stdin)
status = h.get('status', 'unknown')
status_icons = {'green': '✓', 'yellow': '⚠', 'red': '✗'}
icon = status_icons.get(status, '?')
print(f'{icon} Cluster: {h.get(\"cluster_name\", \"N/A\")} ({status})')
print(f'   Nodes: {h.get(\"number_of_nodes\", 0)} total, {h.get(\"number_of_data_nodes\", 0)} data')
print(f'   Shards: {h.get(\"active_shards\", 0)} active, {h.get(\"relocating_shards\", 0)} relocating')
print(f'   Unassigned: {h.get(\"unassigned_shards\", 0)}')
" 2>/dev/null || echo "Cannot reach Elasticsearch"
}

# แสดงรายการ indices
list_indices() {
    local filter="${1:-}"
    
    echo "=== Elasticsearch Indices ==="
    
    local url="${ELASTICSEARCH_URL}/_cat/indices${filter:+/${filter}*}?v&s=index"
    
    curl -sf "$url" 2>/dev/null | \
    awk 'NR==1 {print "  " $0} NR>1 {
        health=$1; status=$2; index=$3; docs=$7; size=$9
        icon = (health=="green") ? "✓" : (health=="yellow") ? "⚠" : "✗"
        printf "  %s %-40s docs:%-10s size:%s\n", icon, index, docs, size
    }'
}

# Index document
index_document() {
    local index="$1"
    local doc="$2"
    local id="${3:-}"
    
    local url="${ELASTICSEARCH_URL}/${index}/_doc"
    [[ -n "$id" ]] && url+="/${id}"
    
    local response
    response=$(curl -sf \
        -X POST \
        -H "Content-Type: application/json" \
        "$url" \
        -d "$doc" 2>/dev/null)
    
    echo "$response" | python3 -c "
import json, sys
r = json.load(sys.stdin)
print(f'✓ Indexed: {r.get(\"_id\", \"?\")} in {r.get(\"_index\", \"?\")} (result: {r.get(\"result\", \"?\")})')" 2>/dev/null
}

# Search documents
es_search() {
    local index="$1"
    local query="$2"
    local size="${3:-10}"
    
    echo "=== Search: ${index} ==="
    
    curl -sf \
        -X POST \
        -H "Content-Type: application/json" \
        "${ELASTICSEARCH_URL}/${index}/_search" \
        -d "{
            \"query\": ${query},
            \"size\": ${size},
            \"sort\": [{\"@timestamp\": \"desc\"}]
        }" 2>/dev/null | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
hits = data.get('hits', {})
total = hits.get('total', {}).get('value', 0)
print(f'Total hits: {total}')
print()
for hit in hits.get('hits', []):
    source = hit.get('_source', {})
    print(f'--- {hit[\"_id\"]} ---')
    for k, v in source.items():
        print(f'  {k}: {v}')
    print()
" 2>/dev/null
}

# Aggregate query
es_aggregate() {
    local index="$1"
    local field="$2"
    local size="${3:-10}"
    
    echo "=== Aggregation: ${index}/${field} ==="
    
    curl -sf \
        -X POST \
        -H "Content-Type: application/json" \
        "${ELASTICSEARCH_URL}/${index}/_search" \
        -d "{
            \"size\": 0,
            \"aggs\": {
                \"by_field\": {
                    \"terms\": {
                        \"field\": \"${field}\",
                        \"size\": ${size}
                    }
                }
            }
        }" 2>/dev/null | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
buckets = data.get('aggregations', {}).get('by_field', {}).get('buckets', [])
total = sum(b.get('doc_count', 0) for b in buckets)
for bucket in buckets:
    key = bucket.get('key', 'N/A')
    count = bucket.get('doc_count', 0)
    pct = count / total * 100 if total else 0
    bar = '█' * int(pct / 5)
    print(f'  {key:<30} {count:>8} ({pct:5.1f}%) {bar}')
" 2>/dev/null
}

# Filebeat configuration
generate_filebeat_config() {
    local output_file="${1:-/tmp/filebeat.yml}"
    
    cat > "$output_file" << 'YAML'
# filebeat.yml - Filebeat Configuration

filebeat.inputs:
  # Application logs
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
      - /var/log/nginx/*.log
    fields:
      app: myapp
      environment: production
    fields_under_root: true
    
    # Multiline (Java stack traces)
    multiline.pattern: '^\s'
    multiline.negate: false
    multiline.match: after
    
    # Processors
    processors:
      - add_host_metadata: ~
      - add_cloud_metadata: ~
      - decode_json_fields:
          fields: ["message"]
          target: ""
          overwrite_keys: true

  # Kubernetes container logs
  - type: filestream
    id: kubernetes-containers
    paths:
      - /var/log/containers/*.log
    parsers:
      - container: ~
    prospector.scanner.symlinks: true
    fields:
      log_type: kubernetes
    fields_under_root: true

  # System logs
  - type: log
    paths:
      - /var/log/syslog
      - /var/log/auth.log
    fields:
      log_type: system
    fields_under_root: true

# Processors
processors:
  - add_docker_metadata: ~
  - add_kubernetes_metadata:
      host: ${NODE_NAME}
      default_matchers.enabled: false
      matchers:
        - logs_path:
            logs_path: "/var/log/containers/"
  
  - script:
      lang: javascript
      id: filter_sensitive
      source: >
        function process(event) {
          var msg = event.Get("message");
          if (msg) {
            msg = msg.replace(/password[=:]\S+/gi, "password=[REDACTED]");
            msg = msg.replace(/token[=:]\S+/gi, "token=[REDACTED]");
            event.Put("message", msg);
          }
        }

# Output to Logstash
output.logstash:
  hosts: ["logstash:5044"]
  ssl.certificate_authorities: ["/etc/pki/ca.crt"]
  ssl.certificate: "/etc/pki/client.crt"
  ssl.key: "/etc/pki/client.key"

# Kibana
setup.kibana:
  host: "kibana:5601"
  protocol: "http"

# Monitoring
monitoring.enabled: true
monitoring.elasticsearch:
  hosts: ["elasticsearch:9200"]

# Logging
logging.level: warning
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
YAML
    
    echo "✓ สร้าง Filebeat config: ${output_file}"
}

# Logstash pipeline
generate_logstash_pipeline() {
    local output_file="${1:-/tmp/logstash.conf}"
    
    cat > "$output_file" << 'CONF'
# logstash.conf - Logstash Pipeline Configuration

input {
  # Filebeat input
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/ssl/logstash.crt"
    ssl_key => "/etc/logstash/ssl/logstash.key"
    ssl_certificate_authorities => ["/etc/logstash/ssl/ca.crt"]
    ssl_verify_mode => "force_peer"
  }
  
  # Syslog input
  syslog {
    port => 5140
    type => "syslog"
  }
  
  # TCP input
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  # Parse timestamps
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => { "received_at" => "%{@timestamp}" }
      add_field => { "received_from" => "%{host}" }
    }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
  
  # Parse JSON logs
  if [log_type] == "application" {
    json {
      source => "message"
      target => "parsed"
    }
    
    if "_jsonparsefailure" not in [tags] {
      mutate {
        rename => { "[parsed][level]" => "log_level" }
        rename => { "[parsed][msg]" => "log_message" }
        rename => { "[parsed][time]" => "log_time" }
      }
    }
  }
  
  # Nginx access log
  if [log_type] == "nginx_access" {
    grok {
      match => { "message" => '%{IPORHOST:remote_addr} - %{DATA:remote_user} \[%{HTTPDATE:time_local}\] "%{WORD:request_method} %{URIPATHPARAM:request_uri} HTTP/%{NUMBER:http_version}" %{NUMBER:status} %{NUMBER:body_bytes_sent} "%{DATA:http_referer}" "%{DATA:http_user_agent}" %{NUMBER:request_time}' }
    }
    
    mutate {
      convert => {
        "status" => "integer"
        "body_bytes_sent" => "integer"
        "request_time" => "float"
      }
    }
    
    # GeoIP
    geoip {
      source => "remote_addr"
      target => "geoip"
    }
    
    # User agent
    useragent {
      source => "http_user_agent"
      target => "user_agent"
    }
  }
  
  # Remove sensitive fields
  mutate {
    remove_field => ["password", "token", "secret", "api_key"]
  }
  
  # Add environment tag
  mutate {
    add_field => {
      "environment" => "${ENVIRONMENT:production}"
      "datacenter" => "${DATACENTER:dc1}"
    }
  }
}

output {
  # Main Elasticsearch output
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{[app]}-%{+YYYY.MM.dd}"
    
    # ILM (Index Lifecycle Management)
    ilm_enabled => true
    ilm_rollover_alias => "logs"
    ilm_pattern => "000001"
    ilm_policy => "logs-policy"
    
    # Authentication
    user => "${ES_USER}"
    password => "${ES_PASS}"
    
    # SSL
    ssl => true
    ssl_certificate_verification => true
    cacert => "/etc/logstash/ssl/ca.crt"
    
    # Retry
    retry_on_conflict => 5
  }
  
  # Send errors to separate index
  if [log_level] == "error" or [log_level] == "ERROR" {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "errors-%{+YYYY.MM.dd}"
    }
  }
  
  # Debug output (disabled in production)
  # stdout { codec => rubydebug }
}
CONF
    
    echo "✓ สร้าง Logstash pipeline: ${output_file}"
}

# Main
case "${1:-}" in
    health)     es_health ;;
    indices)    list_indices "${2:-}" ;;
    index)      index_document "${2}" "${3}" "${4:-}" ;;
    search)     es_search "${2}" "${3:-{\"match_all\": {}}}" "${4:-10}" ;;
    agg)        es_aggregate "${2}" "${3}" "${4:-10}" ;;
    filebeat)   generate_filebeat_config "${2:-}" ;;
    logstash)   generate_logstash_pipeline "${2:-}" ;;
    *)
        echo "Usage: $0 {health|indices|index|search|agg|filebeat|logstash} [args]"
        ;;
esac
```

---

## Workshop: Complete Observability Stack

```bash
#!/bin/bash
# observability_workshop.sh - Workshop: Full Observability Setup

set -euo pipefail

NAMESPACE="observability"
KUBECTL="${KUBECTL:-kubectl}"

RED='\033[0;31m'
GREEN='\033[0;32m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
NC='\033[0m'

log()  { echo -e "${BLUE}[$(date +%H:%M:%S)] $*${NC}"; }
ok()   { echo -e "${GREEN}[$(date +%H:%M:%S)] ✓ $*${NC}"; }

# Deploy Prometheus Stack
deploy_prometheus_stack() {
    log "Deploying Prometheus Stack..."
    
    # สร้าง namespace
    $KUBECTL create namespace "$NAMESPACE" --dry-run=client -o yaml | $KUBECTL apply -f -
    
    # Deploy Prometheus (via Helm)
    if command -v helm &>/dev/null; then
        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 2>/dev/null || true
        helm repo update 2>/dev/null || true
        
        helm upgrade --install kube-prometheus-stack \
            prometheus-community/kube-prometheus-stack \
            --namespace "$NAMESPACE" \
            --set prometheus.prometheusSpec.retention=15d \
            --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=10Gi \
            --set grafana.adminPassword=admin123 \
            --wait 2>/dev/null && ok "Prometheus Stack deployed" || \
        log "Helm deployment skipped (demo mode)"
    else
        log "Helm not available, creating minimal manifests..."
        
        cat << YAML | $KUBECTL apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: ${NAMESPACE}
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
YAML
    fi
}

# Deploy Jaeger
deploy_jaeger() {
    log "Deploying Jaeger Tracing..."
    
    cat << YAML | $KUBECTL apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:1.50
        ports:
        - containerPort: 6831
          protocol: UDP
          name: jaeger-compact
        - containerPort: 16686
          name: jaeger-ui
        - containerPort: 14268
          name: jaeger-collector
        - containerPort: 4317
          name: otlp-grpc
        - containerPort: 4318
          name: otlp-http
        env:
        - name: COLLECTOR_OTLP_ENABLED
          value: "true"
        resources:
          limits:
            memory: 512Mi
          requests:
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: ${NAMESPACE}
spec:
  selector:
    app: jaeger
  ports:
  - name: ui
    port: 16686
    targetPort: 16686
  - name: otlp-http
    port: 4318
    targetPort: 4318
YAML
    
    ok "Jaeger deployed"
}

# Deploy EFK Stack
deploy_efk() {
    log "Deploying EFK (Elasticsearch, Fluentd, Kibana)..."
    
    if command -v helm &>/dev/null; then
        helm repo add elastic https://helm.elastic.co 2>/dev/null || true
        
        # Elasticsearch (dev mode - single node)
        helm upgrade --install elasticsearch elastic/elasticsearch \
            --namespace "$NAMESPACE" \
            --set replicas=1 \
            --set minimumMasterNodes=1 \
            --set resources.requests.cpu=100m \
            --set resources.requests.memory=512M \
            2>/dev/null && ok "Elasticsearch deployed" || log "Skipped"
    fi
}

# แสดง observability stack status
show_stack_status() {
    log "Observability Stack Status..."
    
    echo ""
    echo "=== Deployments ==="
    $KUBECTL get deployments -n "$NAMESPACE" 2>/dev/null | \
    awk 'NR==1{print "  " $0} NR>1{
        avail=$4; desired=$2
        icon=(avail==desired && avail!="0") ? "✓" : "⚠"
        print "  " icon " " $0
    }'
    
    echo ""
    echo "=== Services ==="
    $KUBECTL get services -n "$NAMESPACE" 2>/dev/null | grep -v "^NAME" | \
    awk '{print "  " $1 " → " $5}'
    
    echo ""
    echo "=== Access URLs ==="
    echo "  Grafana:     http://localhost:3000 (admin/admin123)"
    echo "  Prometheus:  http://localhost:9090"
    echo "  Jaeger:      http://localhost:16686"
    echo "  Alertmanager: http://localhost:9093"
    echo ""
    echo "  Port-forward commands:"
    echo "  kubectl port-forward svc/grafana 3000:80 -n ${NAMESPACE}"
    echo "  kubectl port-forward svc/jaeger 16686:16686 -n ${NAMESPACE}"
}

# Main
echo -e "${BOLD}${CYAN}"
echo "╔══════════════════════════════════════╗"
echo "║   OBSERVABILITY WORKSHOP             ║"
echo "╚══════════════════════════════════════╝"
echo -e "${NC}"

case "${1:-setup}" in
    setup)
        deploy_prometheus_stack
        deploy_jaeger
        deploy_efk
        show_stack_status
        ;;
    status)
        show_stack_status
        ;;
    cleanup)
        $KUBECTL delete namespace "$NAMESPACE" --ignore-not-found 2>/dev/null || true
        ok "Cleanup complete"
        ;;
    *)
        echo "Usage: $0 {setup|status|cleanup}"
        ;;
esac
```

---

## สรุป Part 31

| หัวข้อ | เนื้อหา |
|--------|---------|
| **Prometheus** | Query, range query, alerts, config generation, alert rules |
| **Custom Metrics** | Prometheus-format metrics exporter ใน Bash |
| **Distributed Tracing** | Jaeger, Zipkin, OpenTelemetry trace spans |
| **Grafana** | Dashboard management, datasource, import/export |
| **ELK Stack** | Elasticsearch, Filebeat, Logstash pipeline |
| **Kibana** | Dashboard setup, search, aggregation |
| **Observability Workshop** | Complete Prometheus+Jaeger+EFK deployment |

**ขั้นตอนต่อไป**: Part 32 - Advanced Security และ Compliance
