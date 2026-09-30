# Part 51: SRE Advanced Practices and Incident Management (ขั้นตอนที่ 553-556)

## Site Reliability Engineering ระดับ Enterprise

---

## ขั้นตอนที่ 553: SRE Observability Stack — Advanced

```bash
#!/bin/bash
# sre-observability-advanced.sh
# Advanced SRE Observability: SLO/SLI/Error Budget, Distributed Tracing, Continuous Profiling

set -euo pipefail

NAMESPACE="${NAMESPACE:-observability}"
PROMETHEUS_VERSION="${PROMETHEUS_VERSION:-2.47.0}"
TEMPO_VERSION="${TEMPO_VERSION:-2.3.0}"
PARCA_VERSION="${PARCA_VERSION:-0.18.0}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== SLO Configuration with Pyrra ====================
setup_slo_framework() {
    log "Setting up SLO framework with Pyrra..."
    
    # Install Pyrra (SLO framework)
    helm repo add pyrra https://pyrra-dev.github.io/pyrra/
    helm upgrade --install pyrra pyrra/pyrra \
        --namespace "${NAMESPACE}" \
        --create-namespace \
        --set replicas=2 \
        --wait
    
    # Define SLOs for critical services
    cat <<'EOF' | kubectl apply -f -
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: api-gateway-availability
  namespace: production
  labels:
    team: platform
    tier: critical
spec:
  target: "99.9"
  window: 30d
  description: "API Gateway must be available 99.9% of the time"
  alerting:
    name: APIGatewayHighErrorBudgetBurn
    annotations:
      message: "High error budget burn rate for API Gateway availability SLO"
    labels:
      severity: warning
  indicator:
    ratio:
      errors:
        metric: http_requests_total{job="api-gateway",code=~"5.."}
      total:
        metric: http_requests_total{job="api-gateway"}
EOF

    cat <<'EOF' | kubectl apply -f -
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: api-gateway-latency
  namespace: production
spec:
  target: "99"
  window: 30d
  description: "99% of API requests must complete within 200ms"
  indicator:
    latency:
      success:
        metric: http_request_duration_seconds_bucket{job="api-gateway",le="0.2"}
      total:
        metric: http_request_duration_seconds_count{job="api-gateway"}
EOF

    cat <<'EOF' | kubectl apply -f -
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: checkout-success-rate
  namespace: production
spec:
  target: "99.5"
  window: 7d
  description: "Checkout service must process 99.5% of orders successfully"
  indicator:
    ratio:
      errors:
        metric: checkout_orders_total{status="failed"}
      total:
        metric: checkout_orders_total
EOF

    log "SLO framework configured"
}

# ==================== Error Budget Monitoring ====================
setup_error_budget_monitoring() {
    log "Setting up Error Budget monitoring and alerting..."
    
    # Prometheus rules for error budget
    cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: error-budget-rules
  namespace: monitoring
spec:
  groups:
  - name: error-budget
    interval: 30s
    rules:
    # Error rate
    - record: job:http_errors:ratio_rate5m
      expr: |
        sum(rate(http_requests_total{code=~"5.."}[5m])) by (job)
        /
        sum(rate(http_requests_total[5m])) by (job)
    
    # Error budget burn rate alerts (multi-window, multi-burn-rate)
    - alert: ErrorBudgetBurnHighFast
      expr: |
        (
          job:http_errors:ratio_rate5m{job="api-gateway"} > (14.4 * (1 - 0.999))
        )
        and
        (
          job:http_errors:ratio_rate1h{job="api-gateway"} > (14.4 * (1 - 0.999))
        )
      for: 2m
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "High fast error budget burn rate"
        description: "API Gateway burning error budget at 14.4x rate. Will exhaust 1h budget."
        runbook: "https://wiki.company.com/runbooks/api-gateway-high-error-rate"
    
    - alert: ErrorBudgetBurnHighSlow
      expr: |
        (
          job:http_errors:ratio_rate30m{job="api-gateway"} > (6 * (1 - 0.999))
        )
        and
        (
          job:http_errors:ratio_rate6h{job="api-gateway"} > (6 * (1 - 0.999))
        )
      for: 15m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "Elevated error budget burn rate"
        description: "API Gateway burning error budget at 6x rate over 6h window"
    
    # Error budget remaining
    - record: error_budget_remaining:30d
      expr: |
        1 - (
          sum_over_time(job:http_errors:ratio_rate5m{job="api-gateway"}[30d])
          /
          count_over_time(job:http_errors:ratio_rate5m{job="api-gateway"}[30d])
        ) / (1 - 0.999)
EOF

    log "Error budget monitoring configured"
}

# ==================== Distributed Tracing with Tempo ====================
setup_distributed_tracing() {
    log "Setting up Grafana Tempo for distributed tracing..."
    
    # Install Tempo
    cat <<EOF > /tmp/tempo-values.yaml
tempo:
  multitenancy_enabled: true
  storage:
    trace:
      backend: s3
      s3:
        bucket: tempo-traces
        region: ${AWS_REGION:-us-east-1}
        endpoint: s3.amazonaws.com
  receiver:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
    jaeger:
      protocols:
        thrift_http:
          endpoint: 0.0.0.0:14268
        thrift_compact:
          endpoint: 0.0.0.0:6831
  query_frontend:
    search:
      default_result_limit: 20
      max_result_limit: 1000

distributor:
  replicas: 2

ingester:
  replicas: 2
  config:
    replication_factor: 3

compactor:
  replicas: 1

querier:
  replicas: 2

queryFrontend:
  replicas: 1

metricsGenerator:
  enabled: true
  replicas: 1
  config:
    storage:
      path: /var/tempo/wal
      remote_write:
      - url: http://prometheus-operated.monitoring:9090/api/v1/write

serviceGraph:
  enabled: true

traceql:
  enabled: true
EOF

    helm repo add grafana https://grafana.github.io/helm-charts
    helm upgrade --install tempo grafana/tempo-distributed \
        --namespace "${NAMESPACE}" \
        -f /tmp/tempo-values.yaml \
        --wait
    
    log "Grafana Tempo installed"
}

# ==================== OpenTelemetry Collector ====================
setup_otel_collector() {
    log "Setting up OpenTelemetry Collector..."
    
    helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
    
    cat <<'EOF' > /tmp/otel-collector-values.yaml
mode: daemonset

config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
    prometheus:
      config:
        scrape_configs:
        - job_name: otel-collector
          static_configs:
          - targets: [localhost:8888]
    k8s_cluster:
      auth_type: serviceAccount
    filelog:
      include:
      - /var/log/pods/*/*/*.log
      start_at: end
      include_file_path: true
      include_file_name: false
      operators:
      - type: router
        id: get-format
        routes:
        - output: parser-docker
          expr: 'body matches "^\\{"'
        - output: parser-crio
          expr: 'body matches "^[^ Z]+ "'
        default: parser-containerd
      - type: json_parser
        id: parser-docker
        output: extract_metadata_from_filepath
      - type: extract_metadata_from_filepath
        id: extract_metadata_from_filepath
        parse_from: attributes["log.file.path"]
        regex: '^.*\/(?P<namespace>[^_]+)_(?P<pod_name>[^_]+)_(?P<uid>[a-f0-9\-]{36})\/(?P<container_name>[^\._]+)\/(?P<restart_count>\d+)\.log$'
    
  processors:
    memory_limiter:
      check_interval: 1s
      limit_mib: 4096
      spike_limit_mib: 800
    batch:
      send_batch_size: 10000
      timeout: 10s
    resourcedetection:
      detectors: [env, eks, ec2]
      timeout: 2s
    k8sattributes:
      auth_type: serviceAccount
      passthrough: false
      filter:
        node_from_env_var: KUBE_NODE_NAME
      extract:
        metadata:
        - k8s.pod.name
        - k8s.pod.uid
        - k8s.deployment.name
        - k8s.namespace.name
        - k8s.node.name
        - k8s.pod.start_time
        labels:
        - tag_name: service.name
          key: app
          from: pod
        - tag_name: team
          key: team
          from: pod
    transform:
      error_mode: ignore
      metric_statements:
      - context: metric
        statements:
        - set(description, "") where description == "Deprecated."
    
  exporters:
    otlp/tempo:
      endpoint: tempo-distributor.observability:4317
      tls:
        insecure: true
    prometheusremotewrite:
      endpoint: http://prometheus-operated.monitoring:9090/api/v1/write
    loki:
      endpoint: http://loki-gateway.observability/loki/api/v1/push
    debug:
      verbosity: detailed
      sampling_initial: 5
      sampling_thereafter: 200
    
  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, k8sattributes, batch]
        exporters: [otlp/tempo]
      metrics:
        receivers: [otlp, prometheus]
        processors: [memory_limiter, resourcedetection, k8sattributes, batch]
        exporters: [prometheusremotewrite]
      logs:
        receivers: [filelog]
        processors: [memory_limiter, k8sattributes, batch]
        exporters: [loki]
EOF

    helm upgrade --install otel-collector \
        open-telemetry/opentelemetry-collector \
        --namespace "${NAMESPACE}" \
        -f /tmp/otel-collector-values.yaml \
        --wait
    
    log "OpenTelemetry Collector deployed"
}

# ==================== Continuous Profiling with Parca ====================
setup_continuous_profiling() {
    log "Setting up Parca for continuous profiling..."
    
    # Install Parca
    kubectl create namespace "${NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: parca
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: parca
  template:
    metadata:
      labels:
        app: parca
    spec:
      containers:
      - name: parca
        image: ghcr.io/parca-dev/parca:latest
        args:
        - /parca
        - --config-path=/etc/parca/parca.yaml
        - --storage-active-memory=4294967296
        ports:
        - containerPort: 7070
        volumeMounts:
        - name: config
          mountPath: /etc/parca
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 2000m
            memory: 8Gi
      volumes:
      - name: config
        configMap:
          name: parca-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: parca-config
  namespace: observability
data:
  parca.yaml: |
    object_storage:
      bucket:
        type: S3
        config:
          bucket: parca-profiles
          region: us-east-1
    scrape_configs:
    - job_name: kubernetes-pods
      scrape_interval: 10s
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_parca_dev_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_parca_dev_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: ${__meta_kubernetes_pod_ip}:${1}
EOF

    # Install Parca Agent (eBPF profiler on each node)
    cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: parca-agent
  namespace: observability
spec:
  selector:
    matchLabels:
      app: parca-agent
  template:
    metadata:
      labels:
        app: parca-agent
    spec:
      hostPID: true
      hostNetwork: true
      tolerations:
      - operator: Exists
      containers:
      - name: parca-agent
        image: ghcr.io/parca-dev/parca-agent:latest
        args:
        - /bin/parca-agent
        - --node=$(NODE_NAME)
        - --http-address=:7071
        - --remote-store-address=parca.observability:7070
        - --remote-store-insecure
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          privileged: true
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 2Gi
        volumeMounts:
        - name: proc
          mountPath: /proc
          readOnly: true
        - name: sys
          mountPath: /sys
          readOnly: true
        - name: debugfs
          mountPath: /sys/kernel/debug
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: debugfs
        hostPath:
          path: /sys/kernel/debug
EOF

    log "Parca continuous profiling configured"
}

# ==================== Grafana Dashboards ====================
setup_sre_dashboards() {
    log "Setting up SRE dashboards in Grafana..."
    
    # SLO Dashboard
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: slo-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "true"
data:
  slo-dashboard.json: |
    {
      "title": "SLO / Error Budget Dashboard",
      "tags": ["slo", "sre"],
      "panels": [
        {
          "title": "Error Budget Remaining",
          "type": "gauge",
          "datasource": "Prometheus",
          "targets": [
            {
              "expr": "error_budget_remaining:30d * 100",
              "legendFormat": "Budget Remaining %"
            }
          ],
          "options": {
            "thresholds": {
              "steps": [
                {"color": "red", "value": 0},
                {"color": "yellow", "value": 25},
                {"color": "green", "value": 50}
              ]
            }
          }
        },
        {
          "title": "Request Success Rate (30d)",
          "type": "stat",
          "datasource": "Prometheus",
          "targets": [
            {
              "expr": "1 - job:http_errors:ratio_rate30m{job='api-gateway'}",
              "legendFormat": "Success Rate"
            }
          ]
        },
        {
          "title": "Error Budget Burn Rate",
          "type": "timeseries",
          "datasource": "Prometheus",
          "targets": [
            {
              "expr": "job:http_errors:ratio_rate5m{job='api-gateway'} / (1 - 0.999)",
              "legendFormat": "Burn Rate (5m)"
            },
            {
              "expr": "job:http_errors:ratio_rate1h{job='api-gateway'} / (1 - 0.999)",
              "legendFormat": "Burn Rate (1h)"
            }
          ],
          "thresholds": [
            {"value": 14.4, "color": "red"},
            {"value": 6, "color": "yellow"},
            {"value": 1, "color": "green"}
          ]
        }
      ]
    }
EOF

    log "SRE dashboards configured"
}

main() {
    case "${1:-all}" in
        slo)       setup_slo_framework ;;
        budget)    setup_error_budget_monitoring ;;
        tempo)     setup_distributed_tracing ;;
        otel)      setup_otel_collector ;;
        profiling) setup_continuous_profiling ;;
        dashboard) setup_sre_dashboards ;;
        all)
            setup_slo_framework
            setup_error_budget_monitoring
            setup_distributed_tracing
            setup_otel_collector
            setup_continuous_profiling
            setup_sre_dashboards
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 554: Incident Management Platform

```bash
#!/bin/bash
# incident-management.sh
# Enterprise Incident Management: PagerDuty, Runbooks, Post-mortems

set -euo pipefail

PAGERDUTY_TOKEN="${PAGERDUTY_TOKEN:-}"
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"
NAMESPACE="${NAMESPACE:-incident-management}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] [$1] $2"; }

# ==================== Incident Lifecycle Management ====================
declare -A INCIDENT_STATE

create_incident() {
    local TITLE="${1}"
    local SEVERITY="${2:-p3}"
    local SERVICE="${3:-unknown}"
    local DESCRIPTION="${4:-}"
    
    local INCIDENT_ID="INC-$(date +%Y%m%d)-$(openssl rand -hex 3 | tr '[:lower:]' '[:upper:]')"
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    # Create PagerDuty incident
    local PD_RESPONSE=""
    if [[ -n "${PAGERDUTY_TOKEN}" ]]; then
        PD_RESPONSE=$(curl -s -X POST \
            "https://api.pagerduty.com/incidents" \
            -H "Authorization: Token token=${PAGERDUTY_TOKEN}" \
            -H "Accept: application/vnd.pagerduty+json;version=2" \
            -H "Content-Type: application/json" \
            -d "{
                \"incident\": {
                    \"type\": \"incident\",
                    \"title\": \"[${SEVERITY^^}] ${TITLE}\",
                    \"service\": {
                        \"id\": \"${SERVICE}\",
                        \"type\": \"service_reference\"
                    },
                    \"urgency\": $([ \"${SEVERITY}\" == \"p1\" ] && echo '\"high\"' || echo '\"low\"'),
                    \"body\": {
                        \"type\": \"incident_body\",
                        \"details\": \"${DESCRIPTION}\"
                    }
                }
            }")
        local PD_ID=$(echo "${PD_RESPONSE}" | python3 -c "import json,sys; print(json.load(sys.stdin)['incident']['id'])" 2>/dev/null)
    fi
    
    # Post to Slack
    if [[ -n "${SLACK_WEBHOOK}" ]]; then
        local SEVERITY_COLOR="good"
        case "${SEVERITY}" in
            p1) SEVERITY_COLOR="#FF0000" ;;
            p2) SEVERITY_COLOR="#FF6600" ;;
            p3) SEVERITY_COLOR="#FFCC00" ;;
            *) SEVERITY_COLOR="#36a64f" ;;
        esac
        
        curl -s -X POST "${SLACK_WEBHOOK}" \
            -H "Content-Type: application/json" \
            -d "{
                \"attachments\": [{
                    \"color\": \"${SEVERITY_COLOR}\",
                    \"title\": \"🚨 Incident Created: ${TITLE}\",
                    \"text\": \"*ID:* ${INCIDENT_ID}\\n*Severity:* ${SEVERITY^^}\\n*Service:* ${SERVICE}\\n*Description:* ${DESCRIPTION}\",
                    \"footer\": \"Incident Management | $(date -u)\",
                    \"actions\": [
                        {\"type\": \"button\", \"text\": \"View Runbook\", \"url\": \"https://wiki.company.com/runbooks/${SERVICE}\"},
                        {\"type\": \"button\", \"text\": \"View Dashboard\", \"url\": \"https://grafana.company.com/d/incidents\"}
                    ]
                }]
            }"
    fi
    
    # Store incident state
    cat <<EOF > "/tmp/incidents/${INCIDENT_ID}.json"
{
    "id": "${INCIDENT_ID}",
    "title": "${TITLE}",
    "severity": "${SEVERITY}",
    "service": "${SERVICE}",
    "status": "open",
    "created_at": "${TIMESTAMP}",
    "timeline": [
        {"time": "${TIMESTAMP}", "event": "Incident created", "author": "system"}
    ],
    "pagerduty_id": "${PD_ID:-}",
    "commander": "",
    "responders": [],
    "affected_systems": ["${SERVICE}"],
    "customer_impact": "unknown",
    "root_cause": "",
    "resolution": ""
}
EOF

    mkdir -p /tmp/incidents
    echo "${INCIDENT_ID}"
    log "CREATED" "Incident ${INCIDENT_ID}: ${TITLE} [${SEVERITY}]"
}

# ==================== Incident Update ====================
update_incident() {
    local INCIDENT_ID="${1}"
    local STATUS="${2}"
    local UPDATE_MSG="${3}"
    
    local INCIDENT_FILE="/tmp/incidents/${INCIDENT_ID}.json"
    
    if [[ ! -f "${INCIDENT_FILE}" ]]; then
        log "ERROR" "Incident ${INCIDENT_ID} not found"
        return 1
    fi
    
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    # Update incident file
    python3 - <<EOF
import json, sys
with open('${INCIDENT_FILE}') as f:
    incident = json.load(f)

incident['status'] = '${STATUS}'
incident['timeline'].append({
    'time': '${TIMESTAMP}',
    'event': '${UPDATE_MSG}',
    'author': 'responder'
})

if '${STATUS}' == 'resolved':
    incident['resolved_at'] = '${TIMESTAMP}'

with open('${INCIDENT_FILE}', 'w') as f:
    json.dump(incident, f, indent=2)
    
print(json.dumps(incident, indent=2))
EOF
    
    # Notify Slack
    if [[ -n "${SLACK_WEBHOOK}" ]]; then
        curl -s -X POST "${SLACK_WEBHOOK}" \
            -H "Content-Type: application/json" \
            -d "{
                \"text\": \"📝 *Incident ${INCIDENT_ID} Update*\\nStatus: ${STATUS^^}\\n${UPDATE_MSG}\"
            }" > /dev/null
    fi
    
    log "UPDATED" "Incident ${INCIDENT_ID}: ${STATUS} - ${UPDATE_MSG}"
}

# ==================== Automated Runbook Execution ====================
execute_runbook() {
    local SERVICE="${1}"
    local ISSUE_TYPE="${2}"
    local INCIDENT_ID="${3:-unknown}"
    
    log "RUNBOOK" "Executing runbook for ${SERVICE}/${ISSUE_TYPE}..."
    
    case "${SERVICE}/${ISSUE_TYPE}" in
        "api-gateway/high-error-rate")
            log "RUNBOOK" "Step 1: Check pod status"
            kubectl get pods -n production -l app=api-gateway
            
            log "RUNBOOK" "Step 2: Check recent logs"
            kubectl logs -n production -l app=api-gateway --tail=100 --since=5m 2>/dev/null | \
                grep -E "ERROR|CRITICAL|Exception" | tail -20
            
            log "RUNBOOK" "Step 3: Check resource usage"
            kubectl top pods -n production -l app=api-gateway 2>/dev/null
            
            log "RUNBOOK" "Step 4: Check HPA status"
            kubectl describe hpa api-gateway -n production 2>/dev/null
            
            log "RUNBOOK" "Step 5: Check upstream dependencies"
            kubectl get endpoints -n production | grep -E "backend|database"
            
            log "RUNBOOK" "Step 6: Auto-remediation: Scale up if CPU > 80%"
            CPU_USAGE=$(kubectl top pods -n production -l app=api-gateway 2>/dev/null | \
                awk 'NR>1 {gsub(/m/,"",$2); print $2}' | sort -n | tail -1)
            if [[ "${CPU_USAGE:-0}" -gt 800 ]]; then
                kubectl scale deployment api-gateway -n production --replicas=10
                update_incident "${INCIDENT_ID}" "investigating" "Auto-scaled api-gateway to 10 replicas (CPU: ${CPU_USAGE}m)"
            fi
            ;;
        
        "database/connection-exhausted")
            log "RUNBOOK" "Step 1: Check connection count"
            kubectl exec -n databases postgresql-0 -- psql -U postgres \
                -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state ORDER BY count DESC;"
            
            log "RUNBOOK" "Step 2: Kill idle connections"
            kubectl exec -n databases postgresql-0 -- psql -U postgres \
                -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity \
                    WHERE state = 'idle' AND query_start < NOW() - INTERVAL '10 minutes';"
            
            log "RUNBOOK" "Step 3: Check PgBouncer pool status"
            kubectl exec -n databases pgbouncer-0 -- psql -p 6432 pgbouncer -c "SHOW POOLS;"
            ;;
        
        "kubernetes/node-not-ready")
            local NODE="${4:-unknown}"
            log "RUNBOOK" "Step 1: Get node details"
            kubectl describe node "${NODE}"
            
            log "RUNBOOK" "Step 2: Check node conditions"
            kubectl get node "${NODE}" -o jsonpath='{.status.conditions}' | python3 -c "import json,sys; [print(c['type'], c['status'], c['message'][:80]) for c in json.load(sys.stdin)]"
            
            log "RUNBOOK" "Step 3: Cordon node"
            kubectl cordon "${NODE}"
            update_incident "${INCIDENT_ID}" "investigating" "Node ${NODE} cordoned"
            
            log "RUNBOOK" "Step 4: Drain workloads"
            kubectl drain "${NODE}" \
                --ignore-daemonsets \
                --delete-emptydir-data \
                --force \
                --timeout=300s
            
            log "RUNBOOK" "Step 5: Terminate and replace node (if EC2)"
            local INSTANCE_ID=$(kubectl get node "${NODE}" -o jsonpath='{.spec.providerID}' | cut -d'/' -f5)
            if [[ -n "${INSTANCE_ID}" ]]; then
                aws ec2 terminate-instances --instance-ids "${INSTANCE_ID}"
                update_incident "${INCIDENT_ID}" "investigating" "Node ${NODE} (${INSTANCE_ID}) terminated for replacement"
            fi
            ;;
    esac
    
    log "RUNBOOK" "Runbook execution complete"
}

# ==================== Post-Mortem Generator ====================
generate_postmortem() {
    local INCIDENT_ID="${1}"
    local OUTPUT_DIR="${2:-/tmp/postmortems}"
    
    mkdir -p "${OUTPUT_DIR}"
    
    local INCIDENT_FILE="/tmp/incidents/${INCIDENT_ID}.json"
    if [[ ! -f "${INCIDENT_FILE}" ]]; then
        log "ERROR" "Incident ${INCIDENT_ID} not found"
        return 1
    fi
    
    # Generate post-mortem template
    python3 - <<'PYEOF'
import json, sys
from datetime import datetime

with open(sys.argv[1]) as f:
    incident = json.load(f)

created = datetime.fromisoformat(incident['created_at'].replace('Z', '+00:00'))
resolved = datetime.fromisoformat(incident.get('resolved_at', incident['created_at']).replace('Z', '+00:00'))
duration = resolved - created

postmortem = f"""# Post-Mortem: {incident['title']}

**Incident ID:** {incident['id']}
**Severity:** {incident['severity'].upper()}
**Date:** {created.strftime('%Y-%m-%d')}
**Duration:** {str(duration).split('.')[0]}
**Status:** {incident['status'].upper()}
**Service(s) Affected:** {', '.join(incident['affected_systems'])}

---

## Executive Summary

*Brief description of what happened, impact, and resolution.*

> TODO: Fill in executive summary

---

## Impact

| Metric | Value |
|--------|-------|
| Duration | {str(duration).split('.')[0]} |
| Affected Services | {', '.join(incident['affected_systems'])} |
| Customer Impact | {incident.get('customer_impact', 'TBD')} |
| Error Budget Consumed | TBD% |

---

## Timeline

| Time (UTC) | Event |
|------------|-------|
"""
for event in incident.get('timeline', []):
    t = datetime.fromisoformat(event['time'].replace('Z', '+00:00'))
    postmortem += f"| {t.strftime('%H:%M:%S')} | {event['event']} |\n"

postmortem += """
---

## Root Cause Analysis

### Contributing Factors

1. **Primary Cause:** *What was the direct technical cause?*
2. **Contributing Factor 1:** *What conditions made this possible?*
3. **Contributing Factor 2:** *What amplified the impact?*

### 5 Whys Analysis

1. **Why 1:** The service was returning 500 errors
   - **Because:** The database connection pool was exhausted
2. **Why 2:** The connection pool was exhausted
   - **Because:** Connection leak in the new deployment
3. **Why 3:** There was a connection leak
   - **Because:** Missing connection cleanup in error handler
4. **Why 4:** The error handler lacked cleanup
   - **Because:** Test coverage did not include error paths
5. **Why 5:** Tests didn't cover error paths
   - **Because:** No requirement for error path coverage > 80%

---

## Resolution

*Describe what was done to resolve the incident.*

> TODO: Fill in resolution steps

---

## Action Items

| Priority | Action | Owner | Due Date | Status |
|----------|--------|-------|----------|--------|
| P1 | Fix connection leak in error handler | Backend Team | 2 days | Open |
| P1 | Add integration test for error paths | QA Team | 1 week | Open |
| P2 | Add connection pool exhaustion alert | SRE Team | 1 week | Open |
| P2 | Implement circuit breaker for DB connections | Architecture | 2 weeks | Open |
| P3 | Review and improve runbook | SRE Team | 1 month | Open |
| P3 | Add chaos engineering test for DB failure | SRE Team | 1 month | Open |

---

## What Went Well

- Incident detected within 2 minutes via automated alerting
- On-call team responded within 5 minutes
- Runbook provided clear remediation steps
- Communication to stakeholders was timely

## What Could Be Improved

- Alert was too noisy — fired 47 times in 30 minutes
- Runbook lacked step for connection leak scenario
- Post-incident capacity was insufficient during recovery

---

## Metrics

| Metric | Target | Actual |
|--------|--------|--------|
| MTTD (Mean Time to Detect) | < 5 min | TBD |
| MTTR (Mean Time to Respond) | < 15 min | TBD |
| MTTF (Mean Time to Fix) | < 60 min | TBD |
| Error Budget Consumed | < 5% | TBD% |

---

## References

- [Incident Dashboard](https://grafana.company.com/d/incidents)
- [Alert Rules](https://grafana.company.com/alerting)
- [Runbook](https://wiki.company.com/runbooks/{service})

---

*Post-mortem completed by: [Name]*
*Reviewed by: [Manager/SRE Lead]*
*Date completed: {datetime.now().strftime('%Y-%m-%d')}*
"""

output_path = sys.argv[2]
with open(output_path, 'w') as f:
    f.write(postmortem)
print(f"Post-mortem saved to: {output_path}")
PYEOF
    
    log "POSTMORTEM" "Post-mortem template generated: ${OUTPUT_DIR}/${INCIDENT_ID}-postmortem.md"
}

# ==================== On-Call Management ====================
setup_oncall_rotation() {
    log "INFO" "Setting up on-call rotation..."
    
    # Create Grafana OnCall schedule via API
    local GRAFANA_URL="${GRAFANA_URL:-http://grafana.monitoring:3000}"
    
    # Create schedule
    curl -s -X POST \
        "${GRAFANA_URL}/api/oncall/api/v1/schedules/" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${GRAFANA_API_KEY:-}" \
        -d '{
            "name": "Platform Team On-Call",
            "type": "calendar",
            "time_zone": "UTC",
            "slack": {
                "channel_id": "C1234567890",
                "user_group_id": "S1234567890"
            },
            "notify_oncall_shift_freq": "each",
            "notify_empty_oncall": "all",
            "notify_shift_change_to_slack": true
        }' 2>/dev/null || echo "Grafana OnCall API unavailable (expected in non-Grafana Cloud)"
    
    log "INFO" "On-call rotation configured"
}

main() {
    mkdir -p /tmp/incidents
    
    case "${1:-help}" in
        create)
            create_incident "${2}" "${3:-p3}" "${4:-unknown}" "${5:-}"
            ;;
        update)
            update_incident "${2}" "${3}" "${4}"
            ;;
        runbook)
            execute_runbook "${2}" "${3}" "${4:-unknown}"
            ;;
        postmortem)
            generate_postmortem "${2}" "${3:-/tmp/postmortems}"
            ;;
        oncall)
            setup_oncall_rotation
            ;;
        demo)
            INC_ID=$(create_incident "API Gateway High Error Rate" "p1" "api-gateway" "Error rate > 10% for 5 minutes")
            update_incident "${INC_ID}" "investigating" "Runbook execution started"
            execute_runbook "api-gateway" "high-error-rate" "${INC_ID}"
            update_incident "${INC_ID}" "resolved" "Scaled deployment from 3 to 10 replicas. Error rate normalized."
            generate_postmortem "${INC_ID}" "/tmp/postmortems"
            ;;
        *)
            echo "Usage: $0 {create|update|runbook|postmortem|oncall|demo}"
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 555: Chaos Engineering Platform

```bash
#!/bin/bash
# chaos-engineering.sh
# Chaos Engineering: Chaos Mesh, Litmus Chaos, Chaos Monkey

set -euo pipefail

CHAOS_NAMESPACE="${CHAOS_NAMESPACE:-chaos-testing}"
CHAOS_MESH_VERSION="${CHAOS_MESH_VERSION:-2.6.0}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Chaos Mesh Installation ====================
install_chaos_mesh() {
    log "Installing Chaos Mesh..."
    
    helm repo add chaos-mesh https://charts.chaos-mesh.org
    helm repo update
    
    helm upgrade --install chaos-mesh chaos-mesh/chaos-mesh \
        --namespace "${CHAOS_NAMESPACE}" \
        --create-namespace \
        --version "${CHAOS_MESH_VERSION}" \
        --set chaosDaemon.runtime=containerd \
        --set chaosDaemon.socketPath=/run/containerd/containerd.sock \
        --set dashboard.securityMode=false \
        --wait
    
    log "Chaos Mesh installed"
}

# ==================== Chaos Experiments ====================
run_network_chaos() {
    log "Running Network Chaos experiments..."
    
    # Network delay experiment
    cat <<'EOF' | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: api-gateway-network-delay
  namespace: production
spec:
  action: delay
  mode: all
  selector:
    namespaces:
    - production
    labelSelectors:
      app: api-gateway
  delay:
    latency: 100ms
    correlation: "25"
    jitter: 10ms
  duration: 5m
  scheduler:
    cron: "@every 1h"
EOF

    # Network partition experiment
    cat <<'EOF' | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: backend-network-partition
  namespace: production
spec:
  action: partition
  mode: one
  selector:
    namespaces:
    - production
    labelSelectors:
      app: backend-service
  direction: both
  target:
    mode: all
    selector:
      namespaces:
      - databases
  duration: 30s
EOF

    # Bandwidth throttle
    cat <<'EOF' | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: bandwidth-throttle
  namespace: production
spec:
  action: bandwidth
  mode: all
  selector:
    namespaces:
    - production
    labelSelectors:
      tier: frontend
  bandwidth:
    rate: 1mbps
    limit: 100mb
    buffer: 10000
  duration: 10m
EOF

    log "Network chaos experiments created"
}

run_pod_chaos() {
    log "Running Pod Chaos experiments..."
    
    # Pod failure
    cat <<'EOF' | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: random-pod-failure
  namespace: production
spec:
  action: pod-failure
  mode: fixed-percent
  value: "20"
  selector:
    namespaces:
    - production
    labelSelectors:
      tier: backend
  duration: 2m
  scheduler:
    cron: "0 */4 * * *"
EOF

    # Container kill
    cat <<'EOF' | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: sidecar-container-kill
  namespace: production
spec:
  action: container-kill
  mode: one
  selector:
    namespaces:
    - production
    labelSelectors:
      app: api-gateway
  containerNames:
  - istio-proxy
  duration: 30s
EOF

    log "Pod chaos experiments created"
}

run_stress_chaos() {
    log "Running Stress Chaos experiments..."
    
    # CPU stress
    cat <<'EOF' | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress-test
  namespace: production
spec:
  mode: one
  selector:
    namespaces:
    - production
    labelSelectors:
      app: backend-service
  stressors:
    cpu:
      workers: 4
      load: 80
  duration: 5m
EOF

    # Memory stress
    cat <<'EOF' | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: memory-stress-test
  namespace: production
spec:
  mode: one
  selector:
    namespaces:
    - production
    labelSelectors:
      app: worker-service
  stressors:
    memory:
      workers: 2
      size: 512Mi
  duration: 3m
EOF

    log "Stress chaos experiments created"
}

run_io_chaos() {
    log "Running IO Chaos experiments..."
    
    # IO delay
    cat <<'EOF' | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: IOChaos
metadata:
  name: db-io-delay
  namespace: databases
spec:
  action: latency
  mode: all
  selector:
    namespaces:
    - databases
    labelSelectors:
      app: postgresql
  volumePath: /var/lib/postgresql/data
  path: "**"
  delay: 50ms
  percent: 50
  duration: 5m
EOF

    log "IO chaos experiments created"
}

# ==================== Chaos Workflow (Game Day) ====================
run_game_day() {
    log "Running Chaos Engineering Game Day..."
    
    local GAME_DAY_NAME="${1:-gameday-$(date +%Y%m%d)}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: Workflow
metadata:
  name: ${GAME_DAY_NAME}
  namespace: ${CHAOS_NAMESPACE}
spec:
  entry: chaos-sequence
  templates:
  - name: chaos-sequence
    templateType: Serial
    deadline: 2h
    children:
    - network-test
    - wait-recovery-1
    - pod-failure-test
    - wait-recovery-2
    - stress-test
    - final-validation
  
  - name: network-test
    templateType: NetworkChaos
    deadline: 10m
    networkChaos:
      action: delay
      mode: all
      selector:
        namespaces: [production]
        labelSelectors:
          tier: backend
      delay:
        latency: 200ms
        jitter: 50ms
  
  - name: wait-recovery-1
    templateType: Suspend
    deadline: 5m
  
  - name: pod-failure-test
    templateType: PodChaos
    deadline: 5m
    podChaos:
      action: pod-failure
      mode: fixed
      value: "2"
      selector:
        namespaces: [production]
        labelSelectors:
          app: api-gateway
  
  - name: wait-recovery-2
    templateType: Suspend
    deadline: 5m
  
  - name: stress-test
    templateType: StressChaos
    deadline: 10m
    stressChaos:
      mode: all
      selector:
        namespaces: [production]
        labelSelectors:
          tier: compute
      stressors:
        cpu:
          workers: 2
          load: 70
  
  - name: final-validation
    templateType: Task
    task:
      container:
        name: validation
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          echo "Validating system recovery..."
          ATTEMPTS=0
          while [ \$ATTEMPTS -lt 10 ]; do
            STATUS=\$(curl -s -o /dev/null -w "%{http_code}" https://api.company.com/health)
            if [ "\$STATUS" = "200" ]; then
              echo "System recovered successfully!"
              exit 0
            fi
            sleep 30
            ATTEMPTS=\$((ATTEMPTS+1))
          done
          echo "System failed to recover!"
          exit 1
EOF

    log "Game Day workflow started: ${GAME_DAY_NAME}"
    
    # Monitor
    kubectl get workflow "${GAME_DAY_NAME}" -n "${CHAOS_NAMESPACE}" -w &
    local WATCH_PID=$!
    sleep 30
    kill ${WATCH_PID} 2>/dev/null || true
}

# ==================== Steady State Hypothesis ====================
verify_steady_state() {
    log "Verifying Steady State Hypothesis..."
    
    local PASS=0
    local FAIL=0
    
    # 1. API availability > 99.9%
    local API_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "https://api.company.com/health" 2>/dev/null || echo "000")
    if [[ "${API_STATUS}" == "200" ]]; then
        log "PASS" "API Gateway is healthy (${API_STATUS})"
        ((PASS++))
    else
        log "FAIL" "API Gateway unhealthy (${API_STATUS})"
        ((FAIL++))
    fi
    
    # 2. All pods running
    local NOT_RUNNING=$(kubectl get pods -n production --field-selector='status.phase!=Running' --no-headers 2>/dev/null | wc -l)
    if [[ "${NOT_RUNNING}" -eq 0 ]]; then
        log "PASS" "All pods running"
        ((PASS++))
    else
        log "FAIL" "${NOT_RUNNING} pods not running"
        ((FAIL++))
    fi
    
    # 3. Error rate < 1%
    local ERROR_RATE=$(curl -s "http://prometheus.monitoring:9090/api/v1/query" \
        --data-urlencode 'query=sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))' \
        2>/dev/null | python3 -c "import json,sys; r=json.load(sys.stdin); print(float(r['data']['result'][0]['value'][1]) if r['data']['result'] else 0)" 2>/dev/null || echo "0")
    if (( $(echo "${ERROR_RATE} < 0.01" | bc -l 2>/dev/null || echo 0) )); then
        log "PASS" "Error rate acceptable: ${ERROR_RATE}"
        ((PASS++))
    else
        log "FAIL" "Error rate too high: ${ERROR_RATE}"
        ((FAIL++))
    fi
    
    echo ""
    echo "=== STEADY STATE: PASS=${PASS}, FAIL=${FAIL} ==="
    [[ ${FAIL} -eq 0 ]] && echo "STATUS: STABLE ✓" || echo "STATUS: UNSTABLE ✗"
}

main() {
    case "${1:-help}" in
        install)   install_chaos_mesh ;;
        network)   run_network_chaos ;;
        pod)       run_pod_chaos ;;
        stress)    run_stress_chaos ;;
        io)        run_io_chaos ;;
        gameday)   run_game_day "${2:-}" ;;
        verify)    verify_steady_state ;;
        all)
            install_chaos_mesh
            verify_steady_state
            ;;
        *)
            echo "Usage: $0 {install|network|pod|stress|io|gameday|verify|all}"
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 556: Advanced Capacity Planning

```bash
#!/bin/bash
# capacity-planning.sh
# Advanced Capacity Planning: Forecasting, Right-sizing, Load Testing

set -euo pipefail

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Resource Utilization Analysis ====================
analyze_resource_utilization() {
    log "Analyzing resource utilization..."
    
    # Collect CPU/Memory metrics from Prometheus
    local PROM_URL="${PROM_URL:-http://prometheus.monitoring:9090}"
    
    python3 - <<'EOF'
import subprocess, json
from datetime import datetime, timedelta

PROM_URL = "http://prometheus.monitoring:9090"
LOOKBACK_DAYS = 30

queries = {
    "avg_cpu_usage": f'avg by (namespace, pod) (rate(container_cpu_usage_seconds_total{{container!=""}}[5m]))',
    "max_cpu_usage": f'max_over_time(rate(container_cpu_usage_seconds_total{{container!=""}}[5m])[{LOOKBACK_DAYS}d:5m])',
    "avg_memory_usage": f'avg by (namespace, pod) (container_memory_working_set_bytes{{container!=""}})',
    "max_memory_usage": f'max_over_time(container_memory_working_set_bytes{{container!=""}}[{LOOKBACK_DAYS}d:5m])',
    "cpu_request": f'kube_pod_container_resource_requests{{resource="cpu"}}',
    "memory_request": f'kube_pod_container_resource_requests{{resource="memory"}}',
}

import urllib.request, urllib.parse

print("=" * 70)
print(f"CAPACITY UTILIZATION REPORT - Last {LOOKBACK_DAYS} days")
print(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S UTC')}")
print("=" * 70)

for name, query in queries.items():
    url = f"{PROM_URL}/api/v1/query?query={urllib.parse.quote(query)}"
    try:
        with urllib.request.urlopen(url, timeout=5) as resp:
            data = json.loads(resp.read())
            results = data.get('data', {}).get('result', [])
            print(f"\n{name}: {len(results)} results")
    except Exception as e:
        print(f"\n{name}: Error - {e} (Prometheus not available in this environment)")

print("\n--- Recommendations ---")
print("1. Review pods with CPU util < 20% of request → reduce requests")
print("2. Review pods with Memory util > 80% of limit → increase limits")
print("3. Consider VPA for automatic right-sizing")
print("4. Consider cluster-wide resource quotas per namespace")
EOF

    log "Resource analysis complete"
}

# ==================== Right-Sizing Recommendations ====================
generate_rightsizing_report() {
    log "Generating right-sizing recommendations..."
    
    # Use VPA recommendations
    kubectl get vpa -A -o json 2>/dev/null | python3 - <<'EOF' || echo "VPA not available"
import json, sys

vpa_list = json.load(sys.stdin)
items = vpa_list.get('items', [])

print(f"\n{'=' * 70}")
print(f"VPA RIGHT-SIZING RECOMMENDATIONS ({len(items)} VPAs)")
print(f"{'=' * 70}")
print(f"{'Namespace':<20} {'Name':<30} {'Container':<20} {'CPU Req':<10} {'Mem Req':<10}")
print("-" * 90)

for vpa in items:
    ns = vpa['metadata']['namespace']
    name = vpa['metadata']['name']
    recs = vpa.get('status', {}).get('recommendation', {}).get('containerRecommendations', [])
    for rec in recs:
        container = rec['containerName']
        cpu_target = rec.get('target', {}).get('cpu', 'N/A')
        mem_target = rec.get('target', {}).get('memory', 'N/A')
        print(f"{ns:<20} {name:<30} {container:<20} {cpu_target:<10} {mem_target:<10}")
EOF
}

# ==================== Load Testing with k6 ====================
run_load_test() {
    log "Running load test with k6..."
    
    local TARGET_URL="${1:-https://api.company.com}"
    local DURATION="${2:-5m}"
    local VUS="${3:-100}"
    
    cat <<EOF > /tmp/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter, Rate, Trend } from 'k6/metrics';

const errorCount = new Counter('errors');
const successRate = new Rate('success_rate');
const apiDuration = new Trend('api_duration', true);

export const options = {
  stages: [
    { duration: '2m', target: ${VUS} },         // Ramp up
    { duration: '${DURATION}', target: ${VUS} },  // Steady state
    { duration: '2m', target: 0 },                // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
    success_rate: ['rate>0.99'],
  },
};

const BASE_URL = '${TARGET_URL}';

export default function () {
  // Test health endpoint
  const healthRes = http.get(\`\${BASE_URL}/health\`);
  check(healthRes, {
    'health status 200': (r) => r.status === 200,
    'health response < 100ms': (r) => r.timings.duration < 100,
  });

  // Test API endpoints
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer test-token',
  };

  // GET request
  const getRes = http.get(\`\${BASE_URL}/api/v1/products\`, { headers });
  check(getRes, {
    'GET products status 200': (r) => r.status === 200,
    'GET products < 500ms': (r) => r.timings.duration < 500,
  }) ? successRate.add(1) : (errorCount.add(1), successRate.add(0));
  
  apiDuration.add(getRes.timings.duration);

  // POST request
  const payload = JSON.stringify({
    name: 'Test Product',
    price: 99.99,
    inventory: 100,
  });
  
  const postRes = http.post(\`\${BASE_URL}/api/v1/orders\`, payload, { headers });
  check(postRes, {
    'POST order status 201': (r) => r.status === 201 || r.status === 200,
    'POST order < 1s': (r) => r.timings.duration < 1000,
  }) ? successRate.add(1) : (errorCount.add(1), successRate.add(0));

  sleep(Math.random() * 2 + 1);
}

export function handleSummary(data) {
  return {
    '/tmp/load-test-results.json': JSON.stringify(data, null, 2),
    stdout: \`
=== LOAD TEST SUMMARY ===
URL: ${TARGET_URL}
Duration: ${DURATION}
Virtual Users: ${VUS}

HTTP Requests: \${data.metrics.http_reqs?.count || 'N/A'}
Success Rate: \${(data.metrics.success_rate?.values?.rate * 100 || 0).toFixed(2)}%
Error Rate: \${(data.metrics.http_req_failed?.values?.rate * 100 || 0).toFixed(2)}%

Latency:
  p50: \${(data.metrics.http_req_duration?.values?.['p(50)'] || 0).toFixed(2)}ms
  p95: \${(data.metrics.http_req_duration?.values?.['p(95)'] || 0).toFixed(2)}ms
  p99: \${(data.metrics.http_req_duration?.values?.['p(99)'] || 0).toFixed(2)}ms
  max: \${(data.metrics.http_req_duration?.values?.max || 0).toFixed(2)}ms
========================
    \`,
  };
}
EOF

    # Run k6 (if installed)
    if command -v k6 &>/dev/null; then
        k6 run /tmp/load-test.js
    else
        # Run via Docker
        docker run --rm \
            -v /tmp:/tmp \
            grafana/k6:latest \
            run /tmp/load-test.js \
            2>/dev/null || echo "k6 not available. Script saved to /tmp/load-test.js"
    fi
}

# ==================== Capacity Forecast ====================
forecast_capacity() {
    log "Forecasting capacity requirements..."
    
    python3 - <<'EOF'
import json
from datetime import datetime, timedelta
import math

# Simulated growth data (replace with actual Prometheus queries)
growth_rate_monthly = 0.15  # 15% monthly growth
current_metrics = {
    "api_gateway": {"cpu_cores": 8, "memory_gb": 16, "pods": 10, "rps": 5000},
    "backend_service": {"cpu_cores": 16, "memory_gb": 32, "pods": 20, "rps": 8000},
    "database": {"cpu_cores": 32, "memory_gb": 256, "storage_tb": 2.0, "connections": 500},
    "cache": {"memory_gb": 64, "ops_per_sec": 100000},
}

print("=" * 70)
print("CAPACITY FORECAST REPORT")
print(f"Based on {growth_rate_monthly*100:.0f}% monthly growth rate")
print(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
print("=" * 70)

for months in [1, 3, 6, 12]:
    multiplier = (1 + growth_rate_monthly) ** months
    print(f"\n=== FORECAST: +{months} months ({(datetime.now() + timedelta(days=months*30)).strftime('%Y-%m')}) ===")
    
    for service, metrics in current_metrics.items():
        print(f"\n  {service}:")
        for metric, value in metrics.items():
            future = value * multiplier
            unit = ""
            if "cpu" in metric: unit = " cores"
            elif "memory" in metric: unit = " GB"
            elif "storage" in metric: unit = " TB"
            elif "pods" in metric: unit = " pods"
            elif "rps" in metric: unit = " RPS"
            elif "ops" in metric: unit = " ops/s"
            elif "connections" in metric: unit = " conns"
            
            # Add 30% headroom
            with_headroom = future * 1.3
            print(f"    {metric}: {value}{unit} → {future:.1f}{unit} (with headroom: {with_headroom:.1f}{unit})")

print("\n=== RECOMMENDATIONS ===")
print("1. Upgrade EKS node groups in 3 months (16xlarge needed)")
print("2. Scale RDS instance class in 6 months (db.r6g.2xlarge → db.r6g.4xlarge)")
print("3. Expand ElastiCache cluster in 6 months")
print("4. Consider horizontal scaling strategy instead of vertical")
print("5. Review and optimize database queries to reduce growth rate")
EOF
}

main() {
    case "${1:-help}" in
        analyze)    analyze_resource_utilization ;;
        rightsize)  generate_rightsizing_report ;;
        loadtest)   run_load_test "${2:-}" "${3:-5m}" "${4:-100}" ;;
        forecast)   forecast_capacity ;;
        all)
            analyze_resource_utilization
            generate_rightsizing_report
            forecast_capacity
            ;;
        *)
            echo "Usage: $0 {analyze|rightsize|loadtest|forecast|all}"
    esac
}

main "$@"
```

---

## สรุป Part 51

- ✅ **Step 553**: Advanced SRE Observability — SLO/Error Budget (Pyrra), Distributed Tracing (Tempo), OpenTelemetry Collector, Continuous Profiling (Parca)
- ✅ **Step 554**: Incident Management — Lifecycle automation, PagerDuty/Slack integration, Automated runbooks, Post-mortem generator
- ✅ **Step 555**: Chaos Engineering — Chaos Mesh (Network/Pod/Stress/IO chaos), Game Day workflows, Steady State Hypothesis
- ✅ **Step 556**: Capacity Planning — Resource analysis, VPA right-sizing, k6 load testing, Capacity forecasting

**ขั้นตอนต่อไป: Part 52 - Advanced GitOps and Progressive Delivery**
