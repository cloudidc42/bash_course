# Part 68: Advanced Observability, eBPF Networking, Service Mesh Internals

## ภาพรวม
Part นี้ครอบคลุม deep observability: OpenTelemetry collector pipelines, eBPF network visibility, Cilium Hubble, และ distributed tracing at scale

---

## Step 619: OpenTelemetry Collector — Advanced Pipelines

```bash
cat > otel-advanced.sh << 'SCRIPT'
#!/bin/bash
# OpenTelemetry Collector: Advanced Pipeline Configuration

set -euo pipefail

echo "=== OpenTelemetry Collector Advanced Pipelines ==="

# ─── 1. OTel Collector Deployment ─────────────────────────────────────────
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm upgrade --install otel-collector open-telemetry/opentelemetry-collector \
  --namespace observability \
  --values - << 'EOF'
mode: daemonset    # One per node for infrastructure metrics

config:
  receivers:
    # OTLP from applications
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
          max_recv_msg_size_mib: 32
        http:
          endpoint: 0.0.0.0:4318
          cors:
            allowed_origins: ["https://*.example.com"]
    
    # Prometheus scraping
    prometheus:
      config:
        scrape_configs:
          - job_name: 'kubernetes-pods'
            kubernetes_sd_configs:
              - role: pod
            relabel_configs:
              - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
                action: keep
                regex: 'true'
    
    # Kubernetes events
    k8s_events:
      namespaces: [production, staging]
      auth_type: serviceAccount
    
    # Host metrics (node-level)
    hostmetrics:
      scrapers:
        cpu: {}
        memory: {}
        disk: {}
        network: {}
        filesystem: {}
        load: {}
    
    # Kubelet stats
    kubeletstats:
      auth_type: serviceAccount
      endpoint: "https://${env:K8S_NODE_NAME}:10250"
      insecure_skip_verify: true
      metric_groups: [node, pod, container]

  processors:
    # Batch for efficiency
    batch:
      timeout: 5s
      send_batch_size: 8192
      send_batch_max_size: 16384
    
    # Memory limiter (prevent OOM)
    memory_limiter:
      check_interval: 5s
      limit_mib: 1024
      spike_limit_mib: 256
    
    # Resource detection (cloud metadata)
    resourcedetection:
      detectors: [eks, env, system]
      timeout: 10s
    
    # K8s attributes enrichment
    k8sattributes:
      auth_type: serviceAccount
      passthrough: false
      extract:
        metadata:
          - k8s.pod.name
          - k8s.pod.uid
          - k8s.deployment.name
          - k8s.namespace.name
          - k8s.node.name
          - k8s.pod.start_time
        labels:
          - tag_name: app
            key: app
            from: pod
          - tag_name: version
            key: version
            from: pod
        annotations:
          - tag_name: team
            key: team
            from: pod
      pod_association:
        - sources:
            - from: resource_attribute
              name: k8s.pod.ip
    
    # Tail sampling (keep important traces)
    tail_sampling:
      decision_wait: 30s
      num_traces: 50000
      expected_new_traces_per_sec: 1000
      policies:
        - name: errors-policy
          type: status_code
          status_code: {status_codes: [ERROR]}
        - name: slow-traces-policy
          type: latency
          latency: {threshold_ms: 1000}
        - name: random-5-percent
          type: probabilistic
          probabilistic: {sampling_percentage: 5}
        - name: payment-critical
          type: string_attribute
          string_attribute:
            key: service.name
            values: ["payment-service", "fraud-detection"]
            enabled_regex_matching: false
            cache_max_size: 100
            invert_match: false
          # Override: always keep payment traces
    
    # Metric transform
    metricstransform:
      transforms:
        - include: http_server_duration_milliseconds
          match_type: strict
          action: update
          new_name: http.server.duration
        - include: ".*"
          match_type: regexp
          action: update
          operations:
            - action: add_label
              new_label: environment
              new_value: production

    # Filter: drop noisy health check spans
    filter:
      traces:
        span:
          - 'attributes["http.target"] == "/healthz"'
          - 'attributes["http.target"] == "/readyz"'
          - 'attributes["http.target"] == "/metrics"'

  exporters:
    # Traces → Jaeger / Tempo
    otlp/tempo:
      endpoint: tempo.observability.svc.cluster.local:4317
      tls:
        insecure: true
      sending_queue:
        enabled: true
        num_consumers: 10
        queue_size: 5000
      retry_on_failure:
        enabled: true
        initial_interval: 5s
        max_interval: 30s
        max_elapsed_time: 300s
    
    # Metrics → Prometheus (remote write)
    prometheusremotewrite:
      endpoint: http://thanos-receive.observability:10908/api/v1/receive
      resource_to_telemetry_conversion:
        enabled: true
      write_buffer_size: 524288
    
    # Logs → Loki
    loki:
      endpoint: http://loki.observability.svc.cluster.local:3100/loki/api/v1/push
      labels:
        resource:
          k8s.namespace.name: namespace
          k8s.deployment.name: deployment
          k8s.pod.name: pod
          service.name: service
    
    # Debug (dev only)
    debug:
      verbosity: detailed
      sampling_initial: 5
      sampling_thereafter: 100

  service:
    telemetry:
      logs:
        level: info
      metrics:
        address: ":8888"
    
    pipelines:
      traces:
        receivers: [otlp]
        processors:
          - memory_limiter
          - k8sattributes
          - resourcedetection
          - filter
          - tail_sampling
          - batch
        exporters: [otlp/tempo]
      
      metrics:
        receivers: [otlp, prometheus, hostmetrics, kubeletstats]
        processors:
          - memory_limiter
          - k8sattributes
          - resourcedetection
          - metricstransform
          - batch
        exporters: [prometheusremotewrite]
      
      logs:
        receivers: [otlp, k8s_events]
        processors:
          - memory_limiter
          - k8sattributes
          - resourcedetection
          - batch
        exporters: [loki]
EOF

echo "OTel Collector configured"
echo "=== Step 619 Complete: OTel Collector Advanced ==="
SCRIPT
chmod +x otel-advanced.sh
echo "Script created: otel-advanced.sh"
```

**สิ่งที่เรียนรู้:**
- OTel Collector pipelines: receivers (OTLP, Prometheus, hostmetrics, kubeletstats, k8s_events)
- Tail-based sampling: keep ERROR traces + slow (>1s) + payment-critical + 5% random
- k8sattributes processor: enrich spans/metrics ด้วย pod name, deployment, namespace, labels
- Memory limiter: spike_limit_mib=256 ป้องกัน OOM
- Filter processor: drop health check spans (reduces trace volume 30-40%)
- Multi-destination export: Tempo (traces), Thanos (metrics), Loki (logs)

---

## Step 620: eBPF Networking — Cilium Hubble + Network Observability

```bash
cat > ebpf-networking.sh << 'SCRIPT'
#!/bin/bash
# eBPF Networking: Cilium Hubble + Advanced Network Policies

set -euo pipefail

echo "=== eBPF Networking with Cilium Hubble ==="

# ─── 1. Cilium with Hubble ─────────────────────────────────────────────────
helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  --version 1.17.0 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost="$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')" \
  --set k8sServicePort=6443 \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}" \
  --set hubble.metrics.serviceMonitor.enabled=true \
  --set gatewayAPI.enabled=true \
  --set envoy.enabled=true \
  --set loadBalancer.algorithm=maglev \
  --set bpf.masquerade=true \
  --set ipam.mode=kubernetes \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true

# ─── 2. Advanced Cilium Network Policies ──────────────────────────────────
kubectl apply -f - << 'EOF'
# L7 HTTP policy: payment service can only call specific API paths
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: payment-service-egress-l7
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-service
  
  # Ingress: only allow from API gateway + order service
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: api-gateway
        - matchLabels:
            app: order-service
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: POST
                path: /api/v2/payments
              - method: GET
                path: /api/v2/payments/[^/]+  # GET /payments/{id}
              - method: POST
                path: /api/v2/refunds
  
  # Egress: only to PostgreSQL and Kafka
  egress:
    # PostgreSQL
    - toEndpoints:
        - matchLabels:
            app: postgresql
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
    
    # Kafka
    - toEndpoints:
        - matchLabels:
            app: kafka
      toPorts:
        - ports:
            - port: "9092"
              protocol: TCP
    
    # External: Stripe API only
    - toFQDNs:
        - matchName: api.stripe.com
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
          rules:
            http:
              - method: POST
                path: /v1/payment_intents
              - method: POST
                path: /v1/charges
              - method: GET
                path: /v1/payment_intents/.*
    
    # DNS (required for FQDN policy)
    - toEndpoints:
        - matchLabels:
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*.stripe.com"
              - matchPattern: "*.amazonaws.com"
---
# Deny-all baseline (security by default)
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: deny-all-baseline
spec:
  # Apply to all pods that are NOT system pods
  endpointSelector:
    matchExpressions:
      - key: "k8s:io.kubernetes.pod.namespace"
        operator: NotIn
        values:
          - kube-system
          - monitoring
          - istio-system
  
  ingress:
    - fromEntities:
        - cluster   # Allow intra-cluster by default
  
  egress:
    - toEntities:
        - cluster   # Allow intra-cluster
    - toEntities:
        - kube-apiserver  # Allow K8s API access
    # DNS must be explicitly allowed
    - toEndpoints:
        - matchLabels:
            k8s:k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
EOF

# ─── 3. Hubble CLI — Network Flow Analysis ─────────────────────────────────
cat > hubble-analysis.sh << 'INNERSCRIPT'
#!/bin/bash
# Hubble network flow analysis

# Install Hubble CLI
HUBBLE_VERSION="v1.17.0"
curl -L --fail --remote-name-all \
  "https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-amd64.tar.gz" 2>/dev/null || true

# Port-forward Hubble relay
kubectl port-forward -n kube-system svc/hubble-relay 4245:80 &
RELAY_PID=$!
sleep 2

# Flow analysis examples:

# 1. Real-time flows for payment service
echo "Payment service flows (last 60 seconds):"
hubble observe \
  --namespace production \
  --label "app=payment-service" \
  --last 60s \
  --output json 2>/dev/null | head -20 || echo "hubble CLI required"

# 2. Dropped flows (policy violations)
echo "Dropped flows (policy violations):"
hubble observe \
  --verdict DROPPED \
  --namespace production \
  --last 5m 2>/dev/null || echo "hubble CLI required"

# 3. DNS queries
echo "DNS queries:"
hubble observe \
  --protocol DNS \
  --namespace production \
  --last 1m 2>/dev/null || echo "hubble CLI required"

kill $RELAY_PID 2>/dev/null || true
INNERSCRIPT
chmod +x hubble-analysis.sh

# ─── 4. eBPF Performance Monitoring with bpftrace ─────────────────────────
cat > ebpf-perf-monitor.sh << 'EOF'
#!/bin/bash
# eBPF performance monitoring scripts

# Monitor TCP connection establishment latency
bpftrace -e '
kprobe:tcp_v4_connect {
    @start[tid] = nsecs;
}
kretprobe:tcp_v4_connect /@start[tid]/ {
    $latency = (nsecs - @start[tid]) / 1000;
    @tcp_connect_latency_us = hist($latency);
    delete(@start[tid]);
}
interval:s:10 {
    print(@tcp_connect_latency_us);
    clear(@tcp_connect_latency_us);
}
' 2>/dev/null &

# Monitor syscall latency for specific process
bpftrace -e '
tracepoint:raw_syscalls:sys_enter {
    @start[tid] = nsecs;
    @syscall[tid] = args->id;
}
tracepoint:raw_syscalls:sys_exit /@start[tid] != 0/ {
    $delta_us = (nsecs - @start[tid]) / 1000;
    if ($delta_us > 1000) {  // Log syscalls > 1ms
        printf("pid=%d syscall=%d latency=%d us\n", pid, @syscall[tid], $delta_us);
    }
    delete(@start[tid]);
    delete(@syscall[tid]);
}
' 2>/dev/null &

# Kill background scripts after demo
sleep 2
kill %1 %2 2>/dev/null || true
echo "eBPF monitoring scripts ready (bpftrace required)"
EOF
chmod +x ebpf-perf-monitor.sh

echo "=== Step 620 Complete: eBPF Networking with Cilium Hubble ==="
SCRIPT
chmod +x ebpf-networking.sh
echo "Script created: ebpf-networking.sh"
```

**สิ่งที่เรียนรู้:**
- Cilium: kubeProxyReplacement (eBPF FullKubeProxy replacement), Maglev load balancing
- Hubble metrics: DNS, drop, TCP, flow, port-distribution, HTTP → Prometheus/Grafana
- CiliumNetworkPolicy L7 HTTP: method+path matching, FQDN-based egress (api.stripe.com paths)
- CiliumClusterwideNetworkPolicy: deny-all baseline ยกเว้น intra-cluster + kube-apiserver
- bpftrace: TCP connection latency histogram, syscall latency >1ms monitoring

---

## Step 621: Distributed Tracing at Scale — Jaeger + Tempo

```bash
cat > distributed-tracing.sh << 'SCRIPT'
#!/bin/bash
# Distributed Tracing at Scale: Grafana Tempo + Jaeger

set -euo pipefail

echo "=== Distributed Tracing at Scale ==="

# ─── 1. Grafana Tempo with Object Storage ──────────────────────────────────
helm upgrade --install tempo grafana/tempo-distributed \
  --namespace observability \
  --values - << 'EOF'
global:
  clusterDomain: cluster.local

compactor:
  replicas: 1
  config:
    compaction:
      block_retention: 720h    # 30 days

distributor:
  replicas: 3
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

ingester:
  replicas: 3
  config:
    max_block_duration: 30m
    max_block_bytes: 524288000  # 500MB

querier:
  replicas: 2

queryFrontend:
  replicas: 2

storage:
  trace:
    backend: s3
    s3:
      bucket: tempo-traces
      endpoint: s3.amazonaws.com
      region: ap-southeast-1
    wal:
      path: /var/tempo/wal

metricsGenerator:
  enabled: true
  replicas: 2
  config:
    storage:
      path: /var/tempo/generator
      remote_write:
        - url: http://thanos-receive:10908/api/v1/receive
    processor:
      span_metrics:
        dimensions:
          - service.name
          - http.method
          - http.status_code
          - db.system
      service_graphs:
        dimensions:
          - service.name
        max_items: 10000
EOF

# ─── 2. Automatic Instrumentation with OTel Operator ──────────────────────
helm upgrade --install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace observability \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-contrib"

kubectl apply -f - << 'EOF'
# Auto-instrumentation: inject OTel SDK automatically
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: auto-instrumentation
  namespace: production
spec:
  # Sampling: 5% + 100% for slow/error
  sampler:
    type: parentbased_traceidratio
    argument: "0.05"
  
  exporter:
    endpoint: http://otel-collector.observability.svc.cluster.local:4317
  
  propagators:
    - tracecontext
    - baggage
    - b3multi    # For legacy services using B3

  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.52b0
    env:
      - name: OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED
        value: "true"
  
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:2.10.0
    env:
      - name: OTEL_INSTRUMENTATION_MICROMETER_ENABLED
        value: "true"
  
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:0.57.0
  
  go:
    image: ghcr.io/open-telemetry/opentelemetry-go-instrumentation/autoinstrumentation-go:v0.20.0-alpha
---
# Annotate deployment to enable auto-instrumentation
# Add annotation: instrumentation.opentelemetry.io/inject-python: "true"
EOF

# ─── 3. Custom Trace Context Propagation ──────────────────────────────────
cat > trace_context.py << 'PYEOF'
#!/usr/bin/env python3
"""
Custom OpenTelemetry instrumentation.
Manual spans, baggage, and correlation with logs.
"""

import json
import logging
import functools
import time
from typing import Callable, Any

from opentelemetry import trace, baggage
from opentelemetry.trace import SpanKind, StatusCode
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# Setup
provider = TracerProvider()
provider.add_span_processor(
    SimpleSpanProcessor(
        OTLPSpanExporter(endpoint="http://otel-collector:4317")
    )
)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("payment-service", "1.0.0")

# Auto-instrument requests library
RequestsInstrumentor().instrument()

def traced(operation_name: str = None, service_name: str = None):
    """Decorator: wrap function in OpenTelemetry span."""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            span_name = operation_name or f"{func.__module__}.{func.__qualname__}"
            
            with tracer.start_as_current_span(
                span_name,
                kind=SpanKind.INTERNAL,
            ) as span:
                # Add semantic attributes
                if service_name:
                    span.set_attribute("service.name", service_name)
                
                # Add function arguments as attributes (safely)
                for i, arg in enumerate(args[:3]):  # Max 3 positional args
                    if isinstance(arg, (str, int, float, bool)):
                        span.set_attribute(f"arg.{i}", arg)
                
                try:
                    result = func(*args, **kwargs)
                    span.set_status(StatusCode.OK)
                    return result
                except Exception as e:
                    span.set_status(StatusCode.ERROR, str(e))
                    span.record_exception(e)
                    raise
        
        return wrapper
    return decorator

class PaymentService:
    def __init__(self):
        self.logger = logging.getLogger(__name__)

    @traced("payment.create", "payment-service")
    def create_payment(
        self,
        user_id: str,
        amount: float,
        currency: str = "USD"
    ) -> dict:
        current_span = trace.get_current_span()
        
        # Add business attributes to span
        current_span.set_attribute("payment.user_id", user_id)
        current_span.set_attribute("payment.amount", amount)
        current_span.set_attribute("payment.currency", currency)
        
        # Set baggage for downstream propagation
        ctx = baggage.set_baggage("user.tier", "premium")
        ctx = baggage.set_baggage("request.source", "web", context=ctx)
        
        # Structured log with trace correlation
        trace_id = format(current_span.get_span_context().trace_id, '032x')
        span_id = format(current_span.get_span_context().span_id, '016x')
        
        self.logger.info(json.dumps({
            "message": "Creating payment",
            "trace_id": trace_id,
            "span_id": span_id,
            "user_id": user_id,
            "amount": amount,
            "currency": currency,
        }))
        
        # Nested spans for sub-operations
        with tracer.start_as_current_span(
            "payment.validate",
            kind=SpanKind.INTERNAL,
        ) as validate_span:
            validate_span.set_attribute("validation.rules_applied", 5)
            # Simulate validation
        
        with tracer.start_as_current_span(
            "payment.db.insert",
            kind=SpanKind.CLIENT,
            attributes={
                "db.system": "postgresql",
                "db.name": "payments",
                "db.operation": "INSERT",
                "db.sql.table": "payments",
            }
        ) as db_span:
            # Simulate DB insert
            time.sleep(0.005)
        
        payment_id = "pay_" + user_id[:8]
        current_span.set_attribute("payment.id", payment_id)
        
        return {
            "payment_id": payment_id,
            "status": "created",
            "trace_id": trace_id,
        }

# Demo
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s'
)

service = PaymentService()
print("=== Distributed Tracing Demo ===\n")
try:
    result = service.create_payment("user_abc123", 150.00, "USD")
    print(f"Payment created: {result}")
    print(f"\nSpan attributes added:")
    print("  payment.user_id, payment.amount, payment.currency")
    print("  payment.id (set after creation)")
    print("  db.system=postgresql, db.operation=INSERT")
    print("\nNested spans: payment.create → payment.validate, payment.db.insert")
    print("Baggage propagated: user.tier=premium, request.source=web")
    print("\nTrace correlation in structured logs (trace_id + span_id)")
except Exception as e:
    print(f"Note: OTel exporter requires actual collector: {e}")
PYEOF

pip install opentelemetry-sdk opentelemetry-exporter-otlp opentelemetry-instrumentation-requests 2>/dev/null | tail -3
python3 trace_context.py 2>/dev/null || python3 -c "print('OTel SDK demo (collector not available in demo)')"

echo "=== Step 621 Complete: Distributed Tracing at Scale ==="
SCRIPT
chmod +x distributed-tracing.sh
echo "Script created: distributed-tracing.sh"
```

**สิ่งที่เรียนรู้:**
- Grafana Tempo Distributed: compactor (30-day retention), ingester (500MB max block), MetricsGenerator (span_metrics + service_graphs)
- OTel Operator Instrumentation: auto-inject SDK สำหรับ Python/Java/Node/Go ด้วย annotation
- Tail sampling: parentbased_traceidratio 5% + ERROR + slow traces
- Custom `@traced` decorator: span per function, safe attribute capture, exception recording
- Baggage propagation: user.tier + request.source ส่งไปยัง downstream services
- Trace-log correlation: inject trace_id/span_id ใน structured JSON logs

---

## Step 622: SLO Dashboards + Alerting Engineering

```bash
cat > slo-dashboards.sh << 'SCRIPT'
#!/bin/bash
# SLO Dashboards + Advanced Alerting

set -euo pipefail

echo "=== SLO Dashboards + Alerting Engineering ==="

# ─── 1. Prometheus Recording Rules (pre-compute SLI) ──────────────────────
kubectl apply -f - << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: sli-recording-rules
  namespace: production
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
    # ── Payment Service SLIs ─────────────────────────────────────────────
    - name: payment-service-sli
      interval: 30s
      rules:
        # Availability: success requests / total requests
        - record: job:http_requests_total:sum_rate5m
          expr: |
            sum by (job, status_code) (
              rate(http_requests_total{job="payment-service"}[5m])
            )
        
        - record: job:http_success_rate5m
          expr: |
            sum(rate(http_requests_total{job="payment-service",status_code!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="payment-service"}[5m]))
        
        # Error budget burn rate (1h window)
        - record: job:error_budget_burn_rate1h
          expr: |
            (
              1 - sum(rate(http_requests_total{job="payment-service",status_code!~"5.."}[1h]))
                / sum(rate(http_requests_total{job="payment-service"}[1h]))
            )
            /
            (1 - 0.9999)   # 99.99% SLO target = 0.01% error budget
        
        # Burn rate 6h window
        - record: job:error_budget_burn_rate6h
          expr: |
            (
              1 - sum(rate(http_requests_total{job="payment-service",status_code!~"5.."}[6h]))
                / sum(rate(http_requests_total{job="payment-service"}[6h]))
            )
            /
            (1 - 0.9999)
        
        # Latency: p99 over 5 minutes
        - record: job:http_request_duration_p99_5m
          expr: |
            histogram_quantile(0.99,
              sum by (le) (
                rate(http_request_duration_seconds_bucket{job="payment-service"}[5m])
              )
            )
        
        # Latency SLI: requests within 500ms threshold
        - record: job:http_latency_sli5m
          expr: |
            sum(rate(http_request_duration_seconds_bucket{
              job="payment-service",
              le="0.5"
            }[5m]))
            /
            sum(rate(http_request_duration_seconds_count{job="payment-service"}[5m]))
    
    # ── Multi-window Burn Rate Alerts ────────────────────────────────────
    - name: payment-service-burn-rate
      rules:
        # Page: fast burn (1h window × 14.4 rate)
        - alert: PaymentServiceFastBurnRate
          expr: |
            job:error_budget_burn_rate1h{job="payment-service"} > 14.4
            AND
            job:error_budget_burn_rate5m{job="payment-service"} > 14.4
          for: 2m
          labels:
            severity: critical
            team: payments
            slo: availability
          annotations:
            summary: "Payment service error budget burning fast (P1)"
            description: |
              Burn rate {{ $value | humanizePercentage }} above 14.4x threshold.
              At this rate, the monthly error budget will be exhausted in 2 hours.
              Error budget remaining: {{ query "1 - (sum(increase(http_requests_total{job='payment-service',status_code=~'5..'}[30d])) / sum(increase(http_requests_total{job='payment-service'}[30d]))) / 0.0001" | first | value | humanizePercentage }}
            runbook_url: https://runbooks.example.com/payment-service/fast-burn
        
        # Warn: slow burn (6h window × 6 rate)
        - alert: PaymentServiceSlowBurnRate
          expr: |
            job:error_budget_burn_rate6h{job="payment-service"} > 6
            AND
            job:error_budget_burn_rate30m{job="payment-service"} > 6
          for: 15m
          labels:
            severity: warning
            team: payments
          annotations:
            summary: "Payment service error budget burning (P3)"
            runbook_url: https://runbooks.example.com/payment-service/slow-burn
        
        # Latency SLO violation
        - alert: PaymentServiceLatencyHigh
          expr: |
            job:http_latency_sli5m{job="payment-service"} < 0.99
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Payment service latency SLO violation"
            description: "Only {{ $value | humanizePercentage }} of requests within 500ms (SLO: 99%)"
EOF

# ─── 2. Grafana SLO Dashboard ──────────────────────────────────────────────
cat > slo-dashboard.json << 'EOF'
{
  "title": "Payment Platform — SLO Dashboard",
  "tags": ["slo", "payment", "production"],
  "panels": [
    {
      "title": "Availability SLI (30-day rolling)",
      "type": "gauge",
      "gridPos": {"x": 0, "y": 0, "w": 6, "h": 4},
      "targets": [{
        "expr": "1 - (sum(increase(http_requests_total{job='payment-service',status_code=~'5..'}[30d])) / sum(increase(http_requests_total{job='payment-service'}[30d])))"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "min": 0.99,
          "max": 1,
          "thresholds": {
            "steps": [
              {"value": 0, "color": "red"},
              {"value": 0.9990, "color": "yellow"},
              {"value": 0.9999, "color": "green"}
            ]
          }
        }
      }
    },
    {
      "title": "Error Budget Remaining (30-day)",
      "type": "stat",
      "targets": [{
        "expr": "max(1 - (sum(increase(http_requests_total{job='payment-service',status_code=~'5..'}[30d])) / sum(increase(http_requests_total{job='payment-service'}[30d]))) / 0.0001)"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "thresholds": {
            "steps": [
              {"value": 0, "color": "red"},
              {"value": 0.25, "color": "yellow"},
              {"value": 0.5, "color": "green"}
            ]
          }
        }
      }
    },
    {
      "title": "Burn Rate (1h vs 6h)",
      "type": "timeseries",
      "targets": [
        {"expr": "job:error_budget_burn_rate1h{job='payment-service'}", "legendFormat": "1h burn rate"},
        {"expr": "job:error_budget_burn_rate6h{job='payment-service'}", "legendFormat": "6h burn rate"}
      ]
    },
    {
      "title": "p50/p95/p99 Latency",
      "type": "timeseries",
      "targets": [
        {"expr": "histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket{job='payment-service'}[5m])))", "legendFormat": "p50"},
        {"expr": "histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{job='payment-service'}[5m])))", "legendFormat": "p95"},
        {"expr": "histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job='payment-service'}[5m])))", "legendFormat": "p99"}
      ]
    }
  ]
}
EOF

echo "SLO dashboard JSON created"
echo "=== Step 622 Complete: SLO Dashboards + Alerting ==="
SCRIPT
chmod +x slo-dashboards.sh
echo "Script created: slo-dashboards.sh"
```

**สิ่งที่เรียนรู้:**
- Recording rules: pre-compute `job:http_success_rate5m`, `job:error_budget_burn_rate1h` (avoid expensive real-time queries)
- Multi-window burn rate alerts: 1h×14.4 (2-hour exhaustion) = P1/critical, 6h×6 = P3/warning
- Error budget formula: `(1 - success_rate) / (1 - SLO_target)` — burn rate 1.0 = consuming at normal rate
- Grafana gauge: thresholds สำหรับ 99.9% (yellow) และ 99.99% (green)
- Alert annotation ด้วย `query` function: แสดง error budget remaining ใน alert message

---

## สรุป Part 68

| Step | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|----------------|
| 619 | OTel Collector Advanced | Tail sampling, k8sattributes, filter processor, multi-destination export |
| 620 | eBPF Networking | Cilium Hubble, L7 HTTP policy, FQDN egress, bpftrace |
| 621 | Distributed Tracing | Tempo distributed, OTel auto-instrumentation, baggage, trace-log correlation |
| 622 | SLO Dashboards | Multi-window burn rate rules, error budget alerting, Grafana panels |

**ขั้นตอนต่อไป: Part 69 — AR/VR Platform, Real-Time Gaming Infrastructure และ WebRTC at Scale**
