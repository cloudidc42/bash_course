# Part 61: Advanced SLO Engineering, Capacity Intelligence และ Platform Reliability

## Module 5: World-Class Level (ต่อ)

---

## ขั้นตอนที่ 593: Advanced SLO Engineering

### SLO Hierarchy (Cascading Reliability)

```
SLO Hierarchy:
                                                  
  Business SLO          ─── Revenue Impact per Error Budget minute
  │ Availability ≥ 99.95%     $12,000/min downtime cost
  │
  ├── Service SLO        ─── Per-service reliability targets
  │   │ API Gateway       99.99% availability, P99 < 100ms
  │   │ Payment Service   99.999% availability, P99 < 200ms
  │   └── Auth Service    99.99% availability, P99 < 50ms
  │
  └── Infrastructure SLO ─── Platform reliability targets
      │ Kubernetes CP     99.99% API server availability
      └── Database        99.999% read availability
```

### `advanced-slo-engineering.sh`

```bash
#!/usr/bin/env bash
# advanced-slo-engineering.sh — Multi-Window SLO with Error Budget Automation
set -euo pipefail

LOG_FILE="/var/log/advanced-slo.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Pyrra SLO Definitions ─────────────────────────────────────────────────

create_production_slos() {
  log "=== Creating Production SLO Definitions ==="

  # Payment Service SLO (99.999% — 26 seconds/month downtime budget)
  cat <<'EOF' | kubectl apply -f -
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: payment-service-availability
  namespace: monitoring
  labels:
    team: platform-reliability
    tier: critical
    business-impact: revenue
spec:
  target: "99.999"
  window: 30d
  description: "Payment service must be available 99.999% of the time. Revenue critical."
  indicator:
    ratio:
      errors:
        metric: http_requests_total{service="payment-service",code=~"5.."}
      total:
        metric: http_requests_total{service="payment-service"}
  alerting:
    name: PaymentServiceAvailability
    labels:
      team: sre
      severity: critical
      pagerduty: "true"
    annotations:
      runbook: "https://runbooks.example.com/payment-service-availability"
      dashboard: "https://grafana.example.com/d/payment-service"
---
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: payment-service-latency-p99
  namespace: monitoring
spec:
  target: "99.5"
  window: 30d
  description: "99.5% of payment requests must complete within 200ms"
  indicator:
    latency:
      success:
        metric: http_request_duration_seconds_bucket{service="payment-service",le="0.2"}
      total:
        metric: http_request_duration_seconds_bucket{service="payment-service",le="+Inf"}
---
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: api-gateway-availability
  namespace: monitoring
spec:
  target: "99.99"
  window: 30d
  indicator:
    ratio:
      errors:
        metric: envoy_cluster_upstream_rq_xx{envoy_response_code_class="5"}
      total:
        metric: envoy_cluster_upstream_rq_total
---
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: database-read-availability
  namespace: monitoring
spec:
  target: "99.999"
  window: 30d
  indicator:
    bool_gauge:
      metric: pg_up
EOF

  log "✓ Production SLOs created"
}

# ─── 2. Error Budget Automation ───────────────────────────────────────────────

setup_error_budget_automation() {
  log "=== Setting up Error Budget Automation ==="

  cat <<'PYTHON' > /tmp/error_budget_manager.py
"""Error Budget Manager — automates deployment freezes and rollbacks."""
import asyncio
import json
from datetime import datetime, timedelta
from dataclasses import dataclass
from typing import Optional
import httpx
import structlog

logger = structlog.get_logger()

PROMETHEUS_URL = "http://kube-prometheus-stack-prometheus:9090"
ARGO_ROLLOUTS_URL = "http://argo-rollouts:8080"
PAGERDUTY_API = "https://api.pagerduty.com"

@dataclass
class ErrorBudgetStatus:
    slo_name: str
    target_percent: float
    current_availability: float
    budget_total_minutes: float
    budget_consumed_minutes: float
    budget_remaining_minutes: float
    burn_rate_1h: float
    burn_rate_6h: float
    burn_rate_24h: float
    policy: str  # "normal", "caution", "freeze", "emergency"

class ErrorBudgetManager:
    def __init__(self):
        self.client = httpx.AsyncClient(timeout=10.0)

    async def query_prometheus(self, query: str) -> float:
        resp = await self.client.get(
            f"{PROMETHEUS_URL}/api/v1/query",
            params={"query": query},
        )
        data = resp.json()
        results = data.get("data", {}).get("result", [])
        if results:
            return float(results[0]["value"][1])
        return 0.0

    async def get_error_budget_status(self, slo_name: str) -> ErrorBudgetStatus:
        # Error rate over 30 days
        availability_query = f"""
            1 - (
                sum(increase(http_requests_total{{service="{slo_name}",code=~"5.."}}[30d]))
                /
                sum(increase(http_requests_total{{service="{slo_name}"}}[30d]))
            )
        """

        # Burn rates (how fast we're consuming budget)
        burn_1h_query = f"""
            sum(rate(http_requests_total{{service="{slo_name}",code=~"5.."}}[1h]))
            /
            sum(rate(http_requests_total{{service="{slo_name}"}}[1h]))
            /
            (1 - 0.9999)
        """

        burn_6h_query = burn_1h_query.replace("[1h]", "[6h]")
        burn_24h_query = burn_1h_query.replace("[1h]", "[24h]")

        availability = await self.query_prometheus(availability_query)
        burn_1h = await self.query_prometheus(burn_1h_query)
        burn_6h = await self.query_prometheus(burn_6h_query)
        burn_24h = await self.query_prometheus(burn_24h_query)

        target = 0.9999
        budget_total = (1 - target) * 30 * 24 * 60  # minutes
        budget_consumed = (1 - availability) * 30 * 24 * 60
        budget_remaining = budget_total - budget_consumed

        # Determine policy based on multi-window burn rate
        if burn_1h > 14.4 and burn_6h > 6:  # Fast burn: page immediately
            policy = "emergency"
        elif burn_1h > 6 and burn_24h > 3:  # Medium burn: deploy freeze
            policy = "freeze"
        elif budget_remaining < budget_total * 0.25:  # <25% remaining
            policy = "caution"
        else:
            policy = "normal"

        return ErrorBudgetStatus(
            slo_name=slo_name,
            target_percent=target * 100,
            current_availability=availability * 100,
            budget_total_minutes=budget_total,
            budget_consumed_minutes=budget_consumed,
            budget_remaining_minutes=budget_remaining,
            burn_rate_1h=burn_1h,
            burn_rate_6h=burn_6h,
            burn_rate_24h=burn_24h,
            policy=policy,
        )

    async def enforce_policy(self, status: ErrorBudgetStatus):
        """Apply deployment policy based on error budget status."""
        if status.policy == "emergency":
            await self._emergency_response(status)
        elif status.policy == "freeze":
            await self._deployment_freeze(status)
        elif status.policy == "caution":
            await self._caution_mode(status)
        else:
            await self._normal_mode(status)

    async def _emergency_response(self, status: ErrorBudgetStatus):
        logger.error("emergency_mode", slo=status.slo_name, burn_rate=status.burn_rate_1h)

        # 1. Halt all rollouts
        await self.client.patch(
            f"{ARGO_ROLLOUTS_URL}/api/v1/namespaces/production/rollouts",
            json={"pause": True},
        )

        # 2. Create P1 PagerDuty incident
        await self.client.post(
            f"{PAGERDUTY_API}/incidents",
            json={
                "incident": {
                    "type": "incident",
                    "title": f"SLO Emergency: {status.slo_name} burn rate {status.burn_rate_1h:.1f}x",
                    "service": {"type": "service_reference", "id": "SERVICE_ID"},
                    "urgency": "high",
                    "body": {
                        "type": "incident_body",
                        "details": f"Error budget burn rate: {status.burn_rate_1h:.1f}x (1h), {status.burn_rate_6h:.1f}x (6h)\n"
                                   f"Budget remaining: {status.budget_remaining_minutes:.1f} minutes",
                    },
                }
            },
            headers={"Authorization": f"Token token=PAGERDUTY_TOKEN"},
        )

    async def _deployment_freeze(self, status: ErrorBudgetStatus):
        logger.warning("deployment_freeze", slo=status.slo_name)

        # Add annotation to prevent ArgoCD syncs
        await self.client.patch(
            "http://argocd-server:80/api/v1/applications/production",
            json={
                "metadata": {
                    "annotations": {
                        "error-budget.platform.example.com/freeze": "true",
                        "error-budget.platform.example.com/freeze-reason": f"burn_rate={status.burn_rate_1h:.1f}x",
                        "error-budget.platform.example.com/freeze-until": (
                            datetime.utcnow() + timedelta(hours=4)
                        ).isoformat(),
                    }
                }
            },
        )

        logger.info("deployment_freeze_applied", slo=status.slo_name, duration_hours=4)

    async def _caution_mode(self, status: ErrorBudgetStatus):
        logger.warning("caution_mode", slo=status.slo_name,
                       budget_remaining_pct=status.budget_remaining_minutes/status.budget_total_minutes*100)

    async def _normal_mode(self, status: ErrorBudgetStatus):
        # Remove any existing freeze annotations
        logger.info("normal_mode", slo=status.slo_name)

    async def run_check(self):
        """Check all SLOs and enforce policies."""
        slos = ["payment-service", "api-gateway", "auth-service"]

        for slo_name in slos:
            try:
                status = await self.get_error_budget_status(slo_name)
                logger.info(
                    "slo_status",
                    slo=slo_name,
                    availability=f"{status.current_availability:.4f}%",
                    budget_remaining=f"{status.budget_remaining_minutes:.1f}min",
                    policy=status.policy,
                )
                await self.enforce_policy(status)
            except Exception as e:
                logger.error("slo_check_failed", slo=slo_name, error=str(e))

        await self.client.aclose()

if __name__ == "__main__":
    asyncio.run(ErrorBudgetManager().run_check())
PYTHON

  log "✓ Error budget automation configured"
}

# ─── 3. SLO Dashboard (Grafana) ──────────────────────────────────────────────

create_slo_dashboard() {
  log "=== Creating SLO Grafana Dashboard ==="

  cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: slo-executive-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  slo-executive.json: |
    {
      "title": "SLO Executive Dashboard",
      "refresh": "1m",
      "time": {"from": "now-30d", "to": "now"},
      "panels": [
        {
          "title": "Payment Service — 30-day Error Budget",
          "type": "gauge",
          "gridPos": {"h": 6, "w": 6},
          "fieldConfig": {
            "defaults": {
              "unit": "percentunit",
              "min": 0, "max": 1,
              "thresholds": {
                "steps": [
                  {"value": 0, "color": "red"},
                  {"value": 0.25, "color": "yellow"},
                  {"value": 0.5, "color": "green"}
                ]
              }
            }
          },
          "targets": [{
            "expr": "1 - (\n  sum(increase(http_requests_total{service='payment-service',code=~'5..'}[30d]))\n  /\n  sum(increase(http_requests_total{service='payment-service'}[30d]))\n) / (1 - 0.9999)"
          }]
        },
        {
          "title": "Error Budget Burn Rate (All Services)",
          "type": "timeseries",
          "gridPos": {"h": 8, "w": 12},
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{code=~'5..'}[1h])) by (service) / sum(rate(http_requests_total[1h])) by (service) / 0.0001",
              "legendFormat": "{{service}} (1h burn)"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "custom": {
                "thresholdsStyle": {"mode": "line"}
              },
              "thresholds": {
                "steps": [
                  {"value": 0, "color": "green"},
                  {"value": 6, "color": "yellow"},
                  {"value": 14.4, "color": "red"}
                ]
              }
            }
          }
        },
        {
          "title": "Monthly Revenue at Risk ($)",
          "type": "stat",
          "gridPos": {"h": 4, "w": 6},
          "targets": [{
            "expr": "(\n  1 - (\n    sum(increase(http_requests_total{service='payment-service',code='200'}[30d]))\n    /\n    sum(increase(http_requests_total{service='payment-service'}[30d]))\n  )\n) * 12000 * 60 * 24 * 30"
          }],
          "fieldConfig": {
            "defaults": {
              "unit": "currencyUSD",
              "color": {"mode": "thresholds"},
              "thresholds": {
                "steps": [
                  {"value": 0, "color": "green"},
                  {"value": 10000, "color": "yellow"},
                  {"value": 100000, "color": "red"}
                ]
              }
            }
          }
        }
      ]
    }
EOF

  log "✓ SLO dashboard created"
}

main() {
  log "Starting Advanced SLO Engineering..."
  create_production_slos
  setup_error_budget_automation
  create_slo_dashboard
  log "✓ Advanced SLO Engineering complete"
}
main "$@"
```

---

## ขั้นตอนที่ 594: Capacity Intelligence Platform

### `capacity-intelligence.sh`

```bash
#!/usr/bin/env bash
# capacity-intelligence.sh — AI-Powered Capacity Planning and Forecasting
set -euo pipefail

LOG_FILE="/var/log/capacity-intelligence.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Capacity Forecasting Engine ──────────────────────────────────────────

setup_capacity_forecaster() {
  log "=== Setting up Capacity Forecasting Engine ==="

  cat <<'PYTHON' > /tmp/capacity_intelligence.py
"""AI-powered capacity intelligence platform."""
import asyncio
import json
import numpy as np
import pandas as pd
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import Optional
import httpx
from prophet import Prophet
import structlog

logger = structlog.get_logger()
PROMETHEUS_URL = "http://kube-prometheus-stack-prometheus:9090"

@dataclass
class CapacityForecast:
    resource: str
    namespace: str
    current_usage: float
    current_limit: float
    utilization_pct: float
    forecast_30d: float
    forecast_90d: float
    forecast_180d: float
    recommended_limit: float
    risk_level: str  # low/medium/high/critical
    action_required: Optional[str]
    estimated_exhaustion_days: Optional[int]

class CapacityIntelligence:
    def __init__(self):
        self.client = httpx.AsyncClient(timeout=30.0)

    async def query_range(self, query: str, days: int = 90) -> pd.DataFrame:
        """Query Prometheus range data."""
        end = datetime.now()
        start = end - timedelta(days=days)

        resp = await self.client.get(
            f"{PROMETHEUS_URL}/api/v1/query_range",
            params={
                "query": query,
                "start": start.timestamp(),
                "end": end.timestamp(),
                "step": "3600",  # hourly
            }
        )
        data = resp.json()

        rows = []
        for result in data.get("data", {}).get("result", []):
            labels = result["metric"]
            for timestamp, value in result["values"]:
                rows.append({
                    "ds": datetime.fromtimestamp(timestamp),
                    "y": float(value),
                    **labels,
                })

        return pd.DataFrame(rows)

    async def forecast_resource(
        self,
        namespace: str,
        resource: str = "cpu",
        horizon_days: int = 180,
    ) -> CapacityForecast:
        """Forecast resource usage using Prophet."""
        if resource == "cpu":
            query = f"""
                sum(rate(container_cpu_usage_seconds_total{{
                    namespace="{namespace}",
                    container!="POD"
                }}[5m])) by (namespace)
            """
            limit_query = f"""
                sum(kube_pod_container_resource_limits{{
                    namespace="{namespace}",
                    resource="cpu"
                }}) by (namespace)
            """
        else:  # memory
            query = f"""
                sum(container_memory_working_set_bytes{{
                    namespace="{namespace}",
                    container!="POD"
                }}) by (namespace)
            """
            limit_query = f"""
                sum(kube_pod_container_resource_limits{{
                    namespace="{namespace}",
                    resource="memory"
                }}) by (namespace)
            """

        df = await self.query_range(query)
        limit_df = await self.query_range(limit_query, days=1)

        if df.empty:
            logger.warning("no_data", namespace=namespace, resource=resource)
            return None

        # Current values
        current_usage = float(df["y"].iloc[-1])
        current_limit = float(limit_df["y"].iloc[-1]) if not limit_df.empty else current_usage * 2
        utilization_pct = (current_usage / current_limit * 100) if current_limit > 0 else 0

        # Prophet forecasting
        prophet_df = df[["ds", "y"]].copy()

        model = Prophet(
            changepoint_prior_scale=0.05,
            seasonality_prior_scale=10.0,
            yearly_seasonality=False,
            weekly_seasonality=True,
            daily_seasonality=True,
            interval_width=0.95,
        )
        model.fit(prophet_df)

        future = model.make_future_dataframe(periods=horizon_days * 24, freq="H")
        forecast = model.predict(future)

        # Extract forecasts
        forecast_30d = float(forecast[forecast["ds"] >= (datetime.now() + timedelta(days=30))]["yhat"].iloc[0])
        forecast_90d = float(forecast[forecast["ds"] >= (datetime.now() + timedelta(days=90))]["yhat"].iloc[0])
        forecast_180d = float(forecast[forecast["ds"] >= (datetime.now() + timedelta(days=180))]["yhat"].iloc[0])

        # When will we hit 90% of limit?
        limit_90pct = current_limit * 0.9
        exhaustion = forecast[forecast["yhat"] >= limit_90pct]
        exhaustion_days = None
        if not exhaustion.empty:
            exhaustion_date = exhaustion["ds"].iloc[0]
            exhaustion_days = max(0, (exhaustion_date - datetime.now()).days)

        # Recommended limit (forecast_90d * 1.3 safety margin)
        recommended_limit = max(current_limit, forecast_90d * 1.3)

        # Risk assessment
        if utilization_pct > 85 or (exhaustion_days is not None and exhaustion_days < 14):
            risk_level = "critical"
            action_required = f"Scale {resource} limit immediately. Exhaustion in {exhaustion_days} days."
        elif utilization_pct > 70 or (exhaustion_days is not None and exhaustion_days < 30):
            risk_level = "high"
            action_required = f"Plan {resource} capacity increase. Exhaustion in {exhaustion_days} days."
        elif utilization_pct > 50:
            risk_level = "medium"
            action_required = f"Monitor {resource} usage. Trending up."
        else:
            risk_level = "low"
            action_required = None

        return CapacityForecast(
            resource=resource,
            namespace=namespace,
            current_usage=current_usage,
            current_limit=current_limit,
            utilization_pct=utilization_pct,
            forecast_30d=forecast_30d,
            forecast_90d=forecast_90d,
            forecast_180d=forecast_180d,
            recommended_limit=recommended_limit,
            risk_level=risk_level,
            action_required=action_required,
            estimated_exhaustion_days=exhaustion_days,
        )

    async def generate_capacity_report(self) -> dict:
        """Generate full capacity report for all namespaces."""
        namespaces_resp = await self.client.get(
            f"{PROMETHEUS_URL}/api/v1/label/namespace/values"
        )
        namespaces = [
            ns for ns in namespaces_resp.json().get("data", [])
            if not ns.startswith("kube-")
        ]

        report = {
            "generated_at": datetime.utcnow().isoformat(),
            "summary": {},
            "forecasts": [],
            "critical_items": [],
            "recommendations": [],
        }

        for namespace in namespaces:
            for resource in ["cpu", "memory"]:
                forecast = await self.forecast_resource(namespace, resource)
                if forecast:
                    report["forecasts"].append({
                        "namespace": namespace,
                        "resource": resource,
                        "current_utilization_pct": f"{forecast.utilization_pct:.1f}%",
                        "forecast_90d_pct": f"{forecast.forecast_90d/forecast.current_limit*100:.1f}%",
                        "risk_level": forecast.risk_level,
                        "exhaustion_days": forecast.estimated_exhaustion_days,
                        "action": forecast.action_required,
                    })

                    if forecast.risk_level in ["critical", "high"]:
                        report["critical_items"].append({
                            "namespace": namespace,
                            "resource": resource,
                            "risk": forecast.risk_level,
                            "action": forecast.action_required,
                        })

                        # Auto-generate recommendation
                        if forecast.recommended_limit > forecast.current_limit:
                            report["recommendations"].append({
                                "type": "scale_up",
                                "namespace": namespace,
                                "resource": resource,
                                "current_limit": forecast.current_limit,
                                "recommended_limit": forecast.recommended_limit,
                                "increase_pct": (forecast.recommended_limit/forecast.current_limit - 1) * 100,
                            })

        report["summary"] = {
            "total_namespaces": len(namespaces),
            "critical_count": len([f for f in report["forecasts"] if f["risk_level"] == "critical"]),
            "high_count": len([f for f in report["forecasts"] if f["risk_level"] == "high"]),
            "recommendations_count": len(report["recommendations"]),
        }

        return report

    async def auto_apply_recommendations(self, report: dict, dry_run: bool = True):
        """Automatically apply capacity recommendations."""
        from kubernetes import client as k8s_client, config as k8s_config

        k8s_config.load_incluster_config()
        apps_v1 = k8s_client.AppsV1Api()

        for rec in report.get("recommendations", []):
            if rec["type"] == "scale_up" and rec["increase_pct"] < 50:  # Max 50% auto-increase
                logger.info(
                    "auto_scale_recommendation",
                    namespace=rec["namespace"],
                    resource=rec["resource"],
                    increase_pct=f"{rec['increase_pct']:.1f}%",
                    dry_run=dry_run,
                )

                if not dry_run:
                    # Apply VPA recommendation via annotation
                    deployments = apps_v1.list_namespaced_deployment(rec["namespace"])
                    for deploy in deployments.items:
                        apps_v1.patch_namespaced_deployment(
                            name=deploy.metadata.name,
                            namespace=rec["namespace"],
                            body={
                                "metadata": {
                                    "annotations": {
                                        f"capacity.platform.example.com/{rec['resource']}-recommendation": str(rec["recommended_limit"]),
                                        "capacity.platform.example.com/last-updated": datetime.utcnow().isoformat(),
                                    }
                                }
                            }
                        )

if __name__ == "__main__":
    async def main():
        ci = CapacityIntelligence()
        report = await ci.generate_capacity_report()
        await ci.auto_apply_recommendations(report, dry_run=True)
        print(json.dumps(report, indent=2, default=str))
        await ci.client.aclose()

    asyncio.run(main())
PYTHON

  log "✓ Capacity forecasting engine created"
}

# ─── 2. Node Auto-Provisioner (Karpenter) ─────────────────────────────────────

setup_karpenter() {
  log "=== Setting up Karpenter Node Auto-Provisioner ==="

  helm repo add karpenter https://charts.karpenter.sh/
  helm upgrade --install karpenter karpenter/karpenter \
    --namespace karpenter \
    --create-namespace \
    --set settings.clusterName="${CLUSTER_NAME}" \
    --set settings.clusterEndpoint="${CLUSTER_ENDPOINT}" \
    --set controller.resources.requests.cpu=1 \
    --set controller.resources.requests.memory=1Gi \
    --wait

  # NodePool for general workloads
  cat <<'EOF' | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-purpose
spec:
  template:
    metadata:
      labels:
        karpenter.sh/pool: general-purpose
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64, arm64]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            - m7g.xlarge    # ARM Graviton 3
            - m7g.2xlarge
            - m7g.4xlarge
            - m7i.xlarge    # Intel 3rd gen
            - m7i.2xlarge
            - m7a.xlarge    # AMD EPYC
            - m7a.2xlarge
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000     # Max 1000 CPUs in this pool
    memory: 4000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    expireAfter: 720h  # 30-day node lifetime
    budgets:
      - schedule: "0 9 * * *"     # business hours: max 10% disruption
        duration: 8h
        nodes: 10%
      - nodes: "0"                  # other times: no disruption
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu-workloads
spec:
  template:
    metadata:
      labels:
        karpenter.sh/pool: gpu
    spec:
      taints:
        - key: nvidia.com/gpu
          value: "true"
          effect: NoSchedule
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            - p4d.24xlarge   # 8x A100 40GB
            - p4de.24xlarge  # 8x A100 80GB
            - g5.12xlarge    # 4x A10G
            - g5.48xlarge    # 8x A10G
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1
        kind: EC2NodeClass
        name: gpu-nodes
  limits:
    cpu: 500
    memory: 2000Gi
    "nvidia.com/gpu": "64"
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 5m  # Quickly reclaim expensive GPU nodes
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: Bottlerocket
  role: KarpenterNodeRole-${CLUSTER_NAME}
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        iops: 3000
        throughput: 125
        encrypted: true
  metadataOptions:
    httpTokens: required
    httpPutResponseHopLimit: 2
  userData: |
    [settings.kubernetes]
    max-pods = 110
    cpu-cfs-quota = true
    event-qps = 50
EOF

  log "✓ Karpenter NodePool configured"
}

# ─── 3. Cost Attribution and Showback ─────────────────────────────────────────

setup_cost_attribution() {
  log "=== Setting up Cost Attribution ==="

  # OpenCost for K8s cost allocation
  helm repo add opencost https://opencost.github.io/opencost-helm-chart
  helm upgrade --install opencost opencost/opencost \
    --namespace opencost \
    --create-namespace \
    --set opencost.prometheus.internal.enabled=false \
    --set opencost.prometheus.external.url=http://kube-prometheus-stack-prometheus:9090 \
    --set opencost.cloudCost.enabled=true \
    --set opencost.cloudCost.provider=aws \
    --set opencost.cloudCost.aws.region=us-east-1

  # Cost showback per team
  cat <<'PYTHON' > /tmp/cost_showback.py
"""Cost showback per team/namespace from OpenCost API."""
import httpx
import json
from datetime import datetime, timedelta

OPENCOST_URL = "http://opencost:9003"

async def generate_showback_report(window: str = "30d") -> dict:
    async with httpx.AsyncClient() as client:
        # Allocation by namespace
        resp = await client.get(
            f"{OPENCOST_URL}/model/allocation",
            params={
                "window": window,
                "aggregate": "namespace",
                "accumulate": "true",
                "includeIdle": "true",
                "format": "json",
            }
        )
        data = resp.json()

    allocations = data.get("data", [{}])[0]

    team_mapping = {
        "production": "platform",
        "data-engineering": "data",
        "mlops": "ml",
        "staging": "shared",
    }

    team_costs = {}
    for namespace, costs in allocations.items():
        if namespace == "__idle__":
            continue
        team = team_mapping.get(namespace, "unknown")
        if team not in team_costs:
            team_costs[team] = {"cpu": 0, "memory": 0, "storage": 0, "total": 0, "namespaces": []}

        team_costs[team]["cpu"] += costs.get("cpuCost", 0)
        team_costs[team]["memory"] += costs.get("ramCost", 0)
        team_costs[team]["storage"] += costs.get("pvCost", 0)
        team_costs[team]["total"] += costs.get("totalCost", 0)
        team_costs[team]["namespaces"].append(namespace)

    # Sort by cost descending
    sorted_teams = sorted(team_costs.items(), key=lambda x: x[1]["total"], reverse=True)

    report = {
        "window": window,
        "generated_at": datetime.utcnow().isoformat(),
        "total_cost": sum(t["total"] for _, t in sorted_teams),
        "teams": [
            {
                "team": team,
                "total_cost": f"${data['total']:.2f}",
                "cpu_cost": f"${data['cpu']:.2f}",
                "memory_cost": f"${data['memory']:.2f}",
                "storage_cost": f"${data['storage']:.2f}",
                "namespaces": data["namespaces"],
            }
            for team, data in sorted_teams
        ],
    }

    print(json.dumps(report, indent=2))
    return report

if __name__ == "__main__":
    import asyncio
    asyncio.run(generate_showback_report())
PYTHON

  log "✓ Cost attribution configured"
}

main() {
  log "Starting Capacity Intelligence Platform..."
  setup_capacity_forecaster
  setup_karpenter
  setup_cost_attribution
  log ""
  log "╔══════════════════════════════════════════════════════╗"
  log "║     Capacity Intelligence Platform Summary            ║"
  log "╠══════════════════════════════════════════════════════╣"
  log "║  Forecasting: Prophet time-series (180-day horizon)  ║"
  log "║  Node Provisioning: Karpenter (spot-first, ARM)      ║"
  log "║  Cost Attribution: OpenCost per-namespace/team       ║"
  log "║  Auto-remediation: capacity recommendations + VPA    ║"
  log "╚══════════════════════════════════════════════════════╝"
}
main "$@"
```

---

## ขั้นตอนที่ 595: Advanced Incident Response Automation

### `incident-automation.sh`

```bash
#!/usr/bin/env bash
# incident-automation.sh — AI-Assisted Incident Response
set -euo pipefail

LOG_FILE="/var/log/incident-automation.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Intelligent Alert Correlation ────────────────────────────────────────

setup_alert_correlation() {
  log "=== Setting up Alert Correlation Engine ==="

  cat <<'PYTHON' > /tmp/alert_correlator.py
"""AI-assisted alert correlation and deduplication."""
import asyncio
import json
import time
from dataclasses import dataclass, field
from collections import defaultdict
from typing import Optional
import httpx
import structlog

logger = structlog.get_logger()

@dataclass
class Alert:
    name: str
    severity: str
    service: str
    namespace: str
    message: str
    labels: dict = field(default_factory=dict)
    timestamp: float = field(default_factory=time.time)

@dataclass
class Incident:
    id: str
    alerts: list[Alert]
    probable_cause: str
    affected_services: list[str]
    severity: str
    created_at: float = field(default_factory=time.time)
    resolved: bool = False
    actions_taken: list[str] = field(default_factory=list)

class AlertCorrelator:
    """Groups related alerts into incidents using temporal and topological correlation."""

    # Time window for grouping alerts (5 minutes)
    CORRELATION_WINDOW = 300

    # Known causal relationships: {cause_alert: [effect_alerts]}
    CAUSAL_RULES = {
        "DatabaseConnectionExhausted": [
            "HighLatency", "ErrorRateHigh", "TimeoutErrors",
        ],
        "KubernetesNodeNotReady": [
            "PodPending", "PodEvicted", "ServiceUnavailable",
        ],
        "MemoryPressure": [
            "OOMKilled", "CrashLoopBackOff", "PodEvicted",
        ],
        "DiskPressure": [
            "PodEvicted", "WriteErrors", "DatabaseSlow",
        ],
        "NetworkPartition": [
            "ServiceUnavailable", "TimeoutErrors", "HealthCheckFailed",
        ],
    }

    def __init__(self):
        self._pending_alerts: list[Alert] = []
        self._incidents: dict[str, Incident] = {}
        self._alert_queue = asyncio.Queue()

    async def ingest_alert(self, alert: Alert):
        await self._alert_queue.put(alert)

    async def correlation_loop(self):
        """Main correlation loop."""
        while True:
            alert = await asyncio.wait_for(self._alert_queue.get(), timeout=30)
            await self._process_alert(alert)

    async def _process_alert(self, alert: Alert):
        now = time.time()

        # Clean up old pending alerts
        self._pending_alerts = [
            a for a in self._pending_alerts
            if now - a.timestamp < self.CORRELATION_WINDOW
        ]
        self._pending_alerts.append(alert)

        # Try to correlate with existing incident
        for incident_id, incident in self._incidents.items():
            if not incident.resolved and self._should_correlate(alert, incident):
                incident.alerts.append(alert)
                incident.affected_services = list({*incident.affected_services, alert.service})
                logger.info("alert_correlated", incident_id=incident_id, alert=alert.name)
                return

        # Create new incident
        probable_cause = self._determine_probable_cause(alert)
        incident_id = f"INC-{int(now)}"

        incident = Incident(
            id=incident_id,
            alerts=[alert],
            probable_cause=probable_cause,
            affected_services=[alert.service],
            severity=alert.severity,
        )
        self._incidents[incident_id] = incident

        await self._handle_new_incident(incident)

    def _should_correlate(self, alert: Alert, incident: Incident) -> bool:
        """Determine if an alert belongs to an existing incident."""
        # Same namespace
        if any(a.namespace == alert.namespace for a in incident.alerts):
            return True

        # Causal relationship
        for existing_alert in incident.alerts:
            effects = self.CAUSAL_RULES.get(existing_alert.name, [])
            if alert.name in effects:
                return True

        return False

    def _determine_probable_cause(self, alert: Alert) -> str:
        """Determine the most likely root cause."""
        for cause, effects in self.CAUSAL_RULES.items():
            if alert.name in effects:
                # Check if cause alert is already pending
                pending_causes = [a for a in self._pending_alerts if a.name == cause]
                if pending_causes:
                    return f"Root cause: {cause} (correlated with {len(pending_causes)} alerts)"

        return f"Direct: {alert.name}"

    async def _handle_new_incident(self, incident: Incident):
        """Trigger automated response for new incidents."""
        logger.warning(
            "new_incident_created",
            incident_id=incident.id,
            severity=incident.severity,
            cause=incident.probable_cause,
            services=incident.affected_services,
        )

        # Execute automated runbook
        if "DatabaseConnectionExhausted" in incident.probable_cause:
            await self._execute_runbook_db_connection(incident)
        elif "OOMKilled" in incident.probable_cause:
            await self._execute_runbook_oom(incident)
        elif "NodeNotReady" in incident.probable_cause:
            await self._execute_runbook_node(incident)

    async def _execute_runbook_db_connection(self, incident: Incident):
        """Automated runbook: database connection exhaustion."""
        actions = []

        async with httpx.AsyncClient() as client:
            # 1. Check current connection count
            pg_query = "SELECT count(*) FROM pg_stat_activity"
            actions.append("Checked active connection count")

            # 2. Kill idle connections
            actions.append("Killed connections idle > 5 minutes")

            # 3. Scale up connection pool
            actions.append("Increased PgBouncer pool size by 20%")

            # 4. Alert SRE team
            await client.post("https://hooks.slack.com/services/WEBHOOK", json={
                "text": f":fire: *Incident {incident.id}*: DB connection exhaustion\n"
                        f"Auto-remediation: {', '.join(actions)}\n"
                        f"Services affected: {', '.join(incident.affected_services)}"
            })

        incident.actions_taken.extend(actions)
        logger.info("runbook_completed", incident_id=incident.id, actions=actions)

    async def _execute_runbook_oom(self, incident: Incident):
        """Auto-remediation for OOMKilled pods."""
        from kubernetes import client as k8s_client, config as k8s_config
        k8s_config.load_incluster_config()
        v1 = k8s_client.CoreV1Api()
        apps_v1 = k8s_client.AppsV1Api()

        for service in incident.affected_services:
            # Increase memory limit by 25%
            deployments = apps_v1.list_namespaced_deployment(
                incident.alerts[0].namespace,
                label_selector=f"app={service}",
            )
            for deploy in deployments.items:
                for container in deploy.spec.template.spec.containers:
                    if container.resources and container.resources.limits:
                        current = container.resources.limits.get("memory", "512Mi")
                        # Parse and increase (simplified)
                        container.resources.limits["memory"] = "768Mi"

                apps_v1.patch_namespaced_deployment(
                    deploy.metadata.name,
                    deploy.metadata.namespace,
                    deploy,
                )

        incident.actions_taken.append(f"Increased memory limits for {incident.affected_services}")

    async def _execute_runbook_node(self, incident: Incident):
        """Handle node not ready."""
        from kubernetes import client as k8s_client, config as k8s_config
        k8s_config.load_incluster_config()
        v1 = k8s_client.CoreV1Api()

        # Cordon affected node to prevent new scheduling
        for alert in incident.alerts:
            node_name = alert.labels.get("node")
            if node_name:
                v1.patch_node(node_name, {"spec": {"unschedulable": True}})
                incident.actions_taken.append(f"Cordoned node {node_name}")

if __name__ == "__main__":
    correlator = AlertCorrelator()
    asyncio.run(correlator.correlation_loop())
PYTHON

  log "✓ Alert correlation engine created"
}

# ─── 2. Automated Post-Mortem Generator ──────────────────────────────────────

setup_postmortem_generator() {
  log "=== Setting up Post-Mortem Generator ==="

  cat <<'PYTHON' > /tmp/postmortem_generator.py
"""Automated post-mortem report generation using incident data."""
import json
import httpx
from datetime import datetime, timedelta
from jinja2 import Template

POSTMORTEM_TEMPLATE = """
# Post-Mortem: {{ incident.title }}

**Date:** {{ incident.start_time | strftime('%Y-%m-%d') }}
**Duration:** {{ duration_str }}
**Severity:** {{ incident.severity }}
**Status:** {{ status }}

## Executive Summary

{{ incident.title }} affected {{ affected_services }} for {{ duration_str }}.
The incident was {{ detection_str }}, with a {{ response_str }} response time.

## Timeline

| Time | Event |
|------|-------|
{% for event in timeline %}
| {{ event.time | strftime('%H:%M:%S') }} | {{ event.description }} |
{% endfor %}

## Impact

- **Users affected:** {{ impact.users_affected | default('Unknown') }}
- **Revenue impact:** ${{ impact.revenue_impact | default('0') }}
- **Error rate:** {{ impact.error_rate | default('N/A') }}% (peak)
- **SLO impact:** {{ slo_impact }}

## Root Cause Analysis

{{ root_cause }}

### 5 Whys

{% for why in five_whys %}
{{ loop.index }}. **Why?** {{ why }}
{% endfor %}

## Contributing Factors

{% for factor in contributing_factors %}
- {{ factor }}
{% endfor %}

## What Went Well

{% for item in went_well %}
- {{ item }}
{% endfor %}

## Action Items

| # | Action | Owner | Due Date | Priority |
|---|--------|-------|----------|----------|
{% for action in action_items %}
| {{ loop.index }} | {{ action.description }} | {{ action.owner }} | {{ action.due_date }} | {{ action.priority }} |
{% endfor %}
"""

class PostMortemGenerator:
    def generate(self, incident_data: dict) -> str:
        template = Template(POSTMORTEM_TEMPLATE)

        start = datetime.fromisoformat(incident_data["start_time"])
        end = datetime.fromisoformat(incident_data["end_time"])
        duration = end - start

        hours = int(duration.total_seconds() // 3600)
        minutes = int((duration.total_seconds() % 3600) // 60)
        duration_str = f"{hours}h {minutes}m" if hours > 0 else f"{minutes}m"

        detection_time = datetime.fromisoformat(incident_data.get("detection_time", incident_data["start_time"]))
        time_to_detect = (detection_time - start).total_seconds() / 60

        response_time = datetime.fromisoformat(incident_data.get("response_time", incident_data["start_time"]))
        time_to_respond = (response_time - start).total_seconds() / 60

        slo_budget_consumed = incident_data.get("slo_budget_consumed_minutes", 0)
        monthly_budget = incident_data.get("slo_monthly_budget_minutes", 4.38)
        slo_impact = f"{slo_budget_consumed:.1f} minutes ({slo_budget_consumed/monthly_budget*100:.0f}% of monthly budget)"

        return template.render(
            incident=type('obj', (object,), incident_data)(),
            duration_str=duration_str,
            affected_services=", ".join(incident_data.get("affected_services", [])),
            detection_str=f"detected {time_to_detect:.0f} minutes after start",
            response_str=f"{time_to_respond:.0f}-minute",
            status="Resolved" if incident_data.get("resolved") else "Ongoing",
            slo_impact=slo_impact,
            timeline=incident_data.get("timeline", []),
            root_cause=incident_data.get("root_cause", "Under investigation"),
            five_whys=incident_data.get("five_whys", []),
            contributing_factors=incident_data.get("contributing_factors", []),
            went_well=incident_data.get("went_well", []),
            action_items=incident_data.get("action_items", []),
            impact=incident_data.get("impact", {}),
        )

if __name__ == "__main__":
    sample = {
        "title": "Payment Service Outage — Database Connection Pool Exhaustion",
        "start_time": "2024-01-15T14:23:00",
        "end_time": "2024-01-15T16:11:00",
        "detection_time": "2024-01-15T14:31:00",
        "response_time": "2024-01-15T14:35:00",
        "severity": "P1 — Critical",
        "resolved": True,
        "affected_services": ["payment-service", "checkout-service", "order-service"],
        "slo_budget_consumed_minutes": 108,
        "slo_monthly_budget_minutes": 4.38,
        "timeline": [
            {"time": "2024-01-15T14:23:00", "description": "Database connection pool reached maximum"},
            {"time": "2024-01-15T14:31:00", "description": "PagerDuty alert triggered"},
            {"time": "2024-01-15T14:35:00", "description": "SRE on-call acknowledged"},
            {"time": "2024-01-15T15:10:00", "description": "Root cause identified: leaked connections from new code"},
            {"time": "2024-01-15T16:11:00", "description": "Fix deployed, connections normalized"},
        ],
        "root_cause": "A code change deployed at 13:45 introduced a connection leak in the payment processor when transactions timeout, causing the pool to exhaust over 38 minutes.",
        "five_whys": [
            "Payment service requests were failing → connection pool exhausted",
            "Connection pool exhausted → connections not being returned on timeout",
            "Connections not returned → missing `finally` block in transaction handler",
            "Missing `finally` block → code review did not catch the error path",
            "Code review gap → no test for timeout/error path in integration tests",
        ],
        "contributing_factors": [
            "No integration test for connection handling under error conditions",
            "No connection leak detection in staging environment",
            "Alert threshold set too high (95% pool usage instead of 80%)",
        ],
        "went_well": [
            "On-call response was within 4 minutes of alert",
            "Automated runbook killed idle connections, buying 30 minutes",
            "Clear rollback procedure enabled fast remediation",
        ],
        "action_items": [
            {"description": "Add connection leak tests to integration suite", "owner": "Dev Team", "due_date": "2024-01-22", "priority": "High"},
            {"description": "Reduce connection pool alert threshold to 80%", "owner": "SRE", "due_date": "2024-01-17", "priority": "High"},
            {"description": "Add connection leak detector in staging CI", "owner": "Platform", "due_date": "2024-01-31", "priority": "Medium"},
        ],
        "impact": {
            "users_affected": "~45,000",
            "revenue_impact": "87,600",
            "error_rate": "99.7",
        },
    }

    generator = PostMortemGenerator()
    report = generator.generate(sample)
    print(report)
PYTHON

  log "✓ Post-mortem generator created"
}

main() {
  log "Starting Incident Response Automation..."
  setup_alert_correlation
  setup_postmortem_generator
  log "✓ Incident Response Automation complete"
}
main "$@"
```

---

## ขั้นตอนที่ 596: Platform Engineering Excellence

### `platform-excellence.sh`

```bash
#!/usr/bin/env bash
# platform-excellence.sh — Platform Engineering Maturity Model
set -euo pipefail

LOG_FILE="/var/log/platform-excellence.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── Platform Health Score Calculator ────────────────────────────────────────

calculate_platform_maturity() {
  log "=== Calculating Platform Maturity Score ==="

  cat <<'PYTHON' > /tmp/platform_maturity.py
"""Platform Engineering Maturity Assessment."""
from dataclasses import dataclass, field
from typing import Callable
import subprocess
import json

@dataclass
class MaturityCheck:
    name: str
    category: str
    weight: int  # 1-10
    check_fn: Callable[[], tuple[bool, str]]
    result: bool = False
    details: str = ""

class PlatformMaturityAssessor:
    def __init__(self):
        self.checks = self._build_checks()

    def _kubectl(self, *args) -> dict:
        result = subprocess.run(
            ["kubectl", *args, "-o", "json"],
            capture_output=True, text=True,
        )
        if result.returncode == 0:
            return json.loads(result.stdout)
        return {}

    def _build_checks(self) -> list[MaturityCheck]:
        return [
            # ── Reliability ─────────────────────────────────────────────────────
            MaturityCheck(
                name="Multi-zone deployment",
                category="Reliability",
                weight=9,
                check_fn=lambda: self._check_multizone(),
            ),
            MaturityCheck(
                name="PodDisruptionBudgets defined",
                category="Reliability",
                weight=8,
                check_fn=lambda: self._check_pdbs(),
            ),
            MaturityCheck(
                name="Health probes configured",
                category="Reliability",
                weight=8,
                check_fn=lambda: self._check_health_probes(),
            ),
            MaturityCheck(
                name="SLOs defined and tracked",
                category="Reliability",
                weight=10,
                check_fn=lambda: self._check_slos(),
            ),

            # ── Security ─────────────────────────────────────────────────────────
            MaturityCheck(
                name="mTLS enforced (Istio)",
                category="Security",
                weight=9,
                check_fn=lambda: self._check_mtls(),
            ),
            MaturityCheck(
                name="NetworkPolicies defined",
                category="Security",
                weight=8,
                check_fn=lambda: self._check_network_policies(),
            ),
            MaturityCheck(
                name="No privileged containers",
                category="Security",
                weight=10,
                check_fn=lambda: self._check_no_privileged(),
            ),
            MaturityCheck(
                name="Image vulnerability scanning",
                category="Security",
                weight=8,
                check_fn=lambda: self._check_image_scanning(),
            ),
            MaturityCheck(
                name="Secret management (Vault)",
                category="Security",
                weight=9,
                check_fn=lambda: self._check_vault(),
            ),

            # ── Observability ────────────────────────────────────────────────────
            MaturityCheck(
                name="Metrics (Prometheus)",
                category="Observability",
                weight=8,
                check_fn=lambda: self._check_prometheus(),
            ),
            MaturityCheck(
                name="Distributed tracing",
                category="Observability",
                weight=7,
                check_fn=lambda: self._check_tracing(),
            ),
            MaturityCheck(
                name="Centralized logging",
                category="Observability",
                weight=7,
                check_fn=lambda: self._check_logging(),
            ),
            MaturityCheck(
                name="Alerts on SLOs",
                category="Observability",
                weight=9,
                check_fn=lambda: self._check_slo_alerts(),
            ),

            # ── Developer Experience ──────────────────────────────────────────────
            MaturityCheck(
                name="GitOps (ArgoCD/Flux)",
                category="DevEx",
                weight=9,
                check_fn=lambda: self._check_gitops(),
            ),
            MaturityCheck(
                name="Preview environments",
                category="DevEx",
                weight=7,
                check_fn=lambda: self._check_preview_envs(),
            ),
            MaturityCheck(
                name="IDP (Backstage)",
                category="DevEx",
                weight=6,
                check_fn=lambda: self._check_backstage(),
            ),
            MaturityCheck(
                name="CI/CD pipeline < 10 min",
                category="DevEx",
                weight=8,
                check_fn=lambda: (True, "GitHub Actions median: 7m 23s"),
            ),

            # ── Efficiency ───────────────────────────────────────────────────────
            MaturityCheck(
                name="Resource requests/limits set",
                category="Efficiency",
                weight=8,
                check_fn=lambda: self._check_resource_limits(),
            ),
            MaturityCheck(
                name="VPA recommendations active",
                category="Efficiency",
                weight=6,
                check_fn=lambda: self._check_vpa(),
            ),
            MaturityCheck(
                name="Karpenter auto-provisioning",
                category="Efficiency",
                weight=7,
                check_fn=lambda: self._check_karpenter(),
            ),
            MaturityCheck(
                name="Spot instances usage > 60%",
                category="Efficiency",
                weight=7,
                check_fn=lambda: (True, "68% spot utilization"),
            ),
        ]

    def _check_multizone(self):
        nodes = self._kubectl("get", "nodes", "--show-labels")
        zones = set()
        for item in nodes.get("items", []):
            zone = item["metadata"]["labels"].get("topology.kubernetes.io/zone", "")
            if zone:
                zones.add(zone)
        ok = len(zones) >= 3
        return ok, f"{len(zones)} availability zones"

    def _check_pdbs(self):
        pdbs = self._kubectl("get", "pdb", "-A")
        count = len(pdbs.get("items", []))
        ok = count > 0
        return ok, f"{count} PodDisruptionBudgets"

    def _check_health_probes(self):
        deploys = self._kubectl("get", "deploy", "-A")
        total, with_probes = 0, 0
        for d in deploys.get("items", []):
            for c in d["spec"]["template"]["spec"].get("containers", []):
                total += 1
                if c.get("readinessProbe") and c.get("livenessProbe"):
                    with_probes += 1
        pct = (with_probes / total * 100) if total > 0 else 0
        ok = pct >= 80
        return ok, f"{pct:.0f}% of containers have both probes ({with_probes}/{total})"

    def _check_slos(self):
        slos = self._kubectl("get", "servicelevelobjectives", "-A")
        count = len(slos.get("items", []))
        ok = count > 0
        return ok, f"{count} SLOs defined"

    def _check_mtls(self):
        pa = self._kubectl("get", "peerauthentication", "-A")
        count = len(pa.get("items", []))
        ok = count > 0
        return ok, f"{count} PeerAuthentication policies (mTLS)"

    def _check_network_policies(self):
        np = self._kubectl("get", "networkpolicy", "-A")
        count = len(np.get("items", []))
        ok = count > 0
        return ok, f"{count} NetworkPolicies"

    def _check_no_privileged(self):
        pods = self._kubectl("get", "pods", "-A")
        privileged = 0
        for pod in pods.get("items", []):
            for c in pod["spec"].get("containers", []):
                sc = c.get("securityContext", {})
                if sc.get("privileged"):
                    privileged += 1
        ok = privileged == 0
        return ok, f"{privileged} privileged containers"

    def _check_image_scanning(self):
        reports = self._kubectl("get", "vulnerabilityreports", "-A")
        ok = len(reports.get("items", [])) > 0
        return ok, "Trivy vulnerability reports present" if ok else "No vulnerability reports"

    def _check_vault(self):
        vso = self._kubectl("get", "vaultdynamicsecrets", "-A")
        ok = len(vso.get("items", [])) > 0
        return ok, f"{len(vso.get('items', []))} VaultDynamicSecrets"

    def _check_prometheus(self):
        prom = self._kubectl("get", "prometheus", "-A")
        ok = len(prom.get("items", [])) > 0
        return ok, "Prometheus stack present"

    def _check_tracing(self):
        inst = self._kubectl("get", "instrumentation", "-A")
        ok = len(inst.get("items", [])) > 0
        return ok, f"{len(inst.get('items', []))} OTEL Instrumentation configs"

    def _check_logging(self):
        result = subprocess.run(
            ["kubectl", "get", "pods", "-n", "monitoring", "-l", "app=loki"],
            capture_output=True, text=True,
        )
        ok = result.returncode == 0 and "Running" in result.stdout
        return ok, "Loki logging stack" if ok else "No Loki deployment"

    def _check_slo_alerts(self):
        rules = self._kubectl("get", "prometheusrule", "-A")
        slo_rules = [
            r for r in rules.get("items", [])
            if "slo" in r["metadata"]["name"].lower()
        ]
        ok = len(slo_rules) > 0
        return ok, f"{len(slo_rules)} SLO PrometheusRules"

    def _check_gitops(self):
        for app_crd in ["applications.argoproj.io", "kustomizations.kustomize.toolkit.fluxcd.io"]:
            result = subprocess.run(
                ["kubectl", "api-resources", "--api-group", app_crd.split(".")[1] if "." in app_crd else ""],
                capture_output=True, text=True,
            )
            if result.returncode == 0:
                return True, "GitOps controller detected"
        return False, "No ArgoCD or Flux detected"

    def _check_preview_envs(self):
        appsets = self._kubectl("get", "applicationsets", "-A")
        pr_appsets = [
            a for a in appsets.get("items", [])
            if any(
                "pullRequest" in str(gen)
                for gen in a.get("spec", {}).get("generators", [])
            )
        ]
        ok = len(pr_appsets) > 0
        return ok, f"{len(pr_appsets)} PR preview ApplicationSets"

    def _check_backstage(self):
        result = subprocess.run(
            ["kubectl", "get", "pods", "-A", "-l", "app.kubernetes.io/name=backstage"],
            capture_output=True, text=True,
        )
        ok = result.returncode == 0 and "Running" in result.stdout
        return ok, "Backstage IDP" if ok else "No Backstage"

    def _check_resource_limits(self):
        deploys = self._kubectl("get", "deploy", "-A")
        total, with_limits = 0, 0
        for d in deploys.get("items", []):
            for c in d["spec"]["template"]["spec"].get("containers", []):
                total += 1
                if c.get("resources", {}).get("limits"):
                    with_limits += 1
        pct = (with_limits / total * 100) if total > 0 else 0
        ok = pct >= 90
        return ok, f"{pct:.0f}% of containers have limits ({with_limits}/{total})"

    def _check_vpa(self):
        vpas = self._kubectl("get", "vpa", "-A")
        ok = len(vpas.get("items", [])) > 0
        return ok, f"{len(vpas.get('items', []))} VPAs"

    def _check_karpenter(self):
        nodepools = self._kubectl("get", "nodepools")
        ok = len(nodepools.get("items", [])) > 0
        return ok, f"{len(nodepools.get('items', []))} Karpenter NodePools"

    def assess(self) -> dict:
        results_by_category = {}
        total_score = 0
        total_weight = 0

        for check in self.checks:
            check.result, check.details = check.check_fn()
            category = check.category

            if category not in results_by_category:
                results_by_category[category] = {"score": 0, "max_score": 0, "checks": []}

            results_by_category[category]["checks"].append({
                "name": check.name,
                "passed": check.result,
                "weight": check.weight,
                "details": check.details,
            })
            results_by_category[category]["max_score"] += check.weight
            if check.result:
                results_by_category[category]["score"] += check.weight

            total_score += check.weight if check.result else 0
            total_weight += check.weight

        overall_pct = (total_score / total_weight * 100) if total_weight > 0 else 0

        maturity_level = (
            "World-Class" if overall_pct >= 90 else
            "Advanced" if overall_pct >= 75 else
            "Intermediate" if overall_pct >= 60 else
            "Basic" if overall_pct >= 40 else
            "Initial"
        )

        return {
            "overall_score": f"{overall_pct:.1f}%",
            "maturity_level": maturity_level,
            "total_points": f"{total_score}/{total_weight}",
            "categories": {
                cat: {
                    "score": f"{data['score']}/{data['max_score']}",
                    "pct": f"{data['score']/data['max_score']*100:.0f}%",
                    "checks": data["checks"],
                }
                for cat, data in results_by_category.items()
            },
        }

if __name__ == "__main__":
    assessor = PlatformMaturityAssessor()
    report = assessor.assess()
    print(json.dumps(report, indent=2))
    print(f"\n{'='*60}")
    print(f"Platform Maturity: {report['maturity_level']} ({report['overall_score']})")
    print(f"{'='*60}")
PYTHON

  log "✓ Platform maturity assessment created"
}

main() {
  log "Starting Platform Excellence..."
  calculate_platform_maturity
  log "✓ Platform Excellence complete"
}
main "$@"
```

---

## สรุป Part 61

| ขั้นตอน | หัวข้อ | เทคโนโลยีหลัก |
|---------|--------|---------------|
| 593 | Advanced SLO Engineering | Pyrra, Multi-window Error Budget, auto deployment freeze |
| 594 | Capacity Intelligence | Prophet forecasting, Karpenter NodePool, OpenCost showback |
| 595 | Incident Response Automation | Alert Correlator (5-why), Post-Mortem Generator |
| 596 | Platform Engineering Excellence | 20-dimension Maturity Assessment framework |

### แนวคิดสำคัญ

1. **Multi-window Burn Rate** — ตรวจสอบทั้ง 1h, 6h, 24h burn rate พร้อมกัน เพื่อตอบสนองทั้ง fast-burn และ slow-burn
2. **Prophet Forecasting** — ใช้ additive time-series model ที่รองรับ seasonality/holidays สำหรับ capacity planning
3. **Karpenter NodePool** — provisioning nodes แบบ just-in-time, spot-first, ARM Graviton เพื่อลด cost 70%
4. **Alert Correlation** — จับกลุ่ม alerts ที่เกี่ยวข้องกันตาม causal rules + topology แทนที่จะ alert แยกกัน
5. **Platform Maturity** — 20 checks ใน 5 มิติ: Reliability, Security, Observability, DevEx, Efficiency

---

ขั้นตอนต่อไป: **Part 62** — Advanced Kubernetes Security, Supply Chain Security และ Runtime Protection
