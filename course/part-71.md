# Part 71: AIOps — Predictive Autoscaling, Autonomous Incident Resolution และ ML-Driven Observability

## ขั้นตอนที่ 631: AIOps Platform — Predictive Autoscaling with Prophet

### `aiops-predictive-scaling.sh`

```bash
#!/bin/bash
# AIOps: ML-based predictive autoscaling using Prophet + custom Kubernetes controller

cat > aiops/predictive_scaler.py << 'PYTHON'
import asyncio
import logging
from datetime import datetime, timedelta
from dataclasses import dataclass
from typing import Optional
import numpy as np
import pandas as pd
from prophet import Prophet
from prometheus_api_client import PrometheusConnect
from kubernetes import client as k8s, config as k8s_config
import json

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(message)s')


@dataclass
class ScalingDecision:
    deployment: str
    namespace: str
    current_replicas: int
    recommended_replicas: int
    predicted_load: float
    confidence: float
    reason: str
    horizon_minutes: int = 30


class PredictiveScaler:
    """
    Forecast workload 30 minutes ahead using Facebook Prophet,
    then pre-scale deployments before traffic arrives.
    """
    MIN_REPLICAS = 1
    MAX_REPLICAS = 50
    LOOKAHEAD_MINUTES = 30
    TRAINING_DAYS = 14
    CPU_TARGET_UTILIZATION = 0.70   # scale such that predicted load / replicas = 70% CPU
    SCALE_THRESHOLD_UP   = 0.10    # scale up if recommended > current by >10%
    SCALE_THRESHOLD_DOWN = 0.20    # scale down only if recommended < current by >20% (hysteresis)

    def __init__(self, prometheus_url: str, namespace: str):
        self.prom = PrometheusConnect(url=prometheus_url, disable_ssl=False)
        self.namespace = namespace
        k8s_config.load_incluster_config()
        self._apps_v1 = k8s.AppsV1Api()
        self._models: dict[str, Prophet] = {}

    async def run(self):
        while True:
            try:
                deployments = await self._list_scaler_enabled_deployments()
                decisions = await asyncio.gather(
                    *[self._evaluate_deployment(d) for d in deployments],
                    return_exceptions=True,
                )
                for decision in decisions:
                    if isinstance(decision, Exception):
                        logger.error(f"Scaling evaluation failed: {decision}")
                    elif decision and self._should_scale(decision):
                        await self._apply_scaling(decision)
            except Exception as e:
                logger.error(f"Scaler loop error: {e}")
            await asyncio.sleep(300)  # run every 5 minutes

    async def _list_scaler_enabled_deployments(self) -> list[str]:
        deployments = self._apps_v1.list_namespaced_deployment(self.namespace)
        enabled = []
        for d in deployments.items:
            annotations = d.metadata.annotations or {}
            if annotations.get("aiops.io/predictive-scaling") == "true":
                enabled.append(d.metadata.name)
        return enabled

    async def _evaluate_deployment(self, deployment_name: str) -> Optional[ScalingDecision]:
        # Fetch historical CPU usage
        df = await self._fetch_metric_history(
            deployment=deployment_name,
            metric=f'avg(rate(container_cpu_usage_seconds_total{{namespace="{self.namespace}",pod=~"{deployment_name}-.*"}}[5m]))',
        )

        if len(df) < 288:  # need at least 1 day of data (288 × 5min)
            logger.warning(f"Insufficient history for {deployment_name}: {len(df)} points")
            return None

        # Train or retrain Prophet model
        model = await self._get_or_train_model(deployment_name, df)

        # Forecast next 30 minutes
        future = model.make_future_dataframe(periods=6, freq='5T')  # 6 × 5min
        forecast = model.predict(future)

        # Extract the prediction for LOOKAHEAD_MINUTES from now
        target_time = datetime.utcnow() + timedelta(minutes=self.LOOKAHEAD_MINUTES)
        forecast['ds'] = pd.to_datetime(forecast['ds'])
        closest = forecast.iloc[(forecast['ds'] - target_time).abs().argmin()]

        predicted_cpu = max(0, float(closest['yhat']))
        uncertainty = float(closest['yhat_upper']) - float(closest['yhat_lower'])
        confidence = max(0, 1 - uncertainty / (predicted_cpu + 1e-6))

        # Get current replica count
        dep = self._apps_v1.read_namespaced_deployment(deployment_name, self.namespace)
        current_replicas = dep.spec.replicas or 1

        # Calculate recommended replicas: predicted_cpu / (target_cpu_per_replica)
        cpu_per_replica = self.CPU_TARGET_UTILIZATION
        recommended = int(np.ceil(predicted_cpu / cpu_per_replica))
        recommended = max(self.MIN_REPLICAS, min(self.MAX_REPLICAS, recommended))

        return ScalingDecision(
            deployment=deployment_name,
            namespace=self.namespace,
            current_replicas=current_replicas,
            recommended_replicas=recommended,
            predicted_load=predicted_cpu,
            confidence=confidence,
            reason=f"Prophet forecast: {predicted_cpu:.3f} CPU in {self.LOOKAHEAD_MINUTES}min "
                   f"(confidence={confidence:.2f})",
            horizon_minutes=self.LOOKAHEAD_MINUTES,
        )

    async def _fetch_metric_history(self, deployment: str, metric: str) -> pd.DataFrame:
        end = datetime.utcnow()
        start = end - timedelta(days=self.TRAINING_DAYS)
        result = self.prom.custom_query_range(
            query=metric,
            start_time=start,
            end_time=end,
            step="5m",
        )
        if not result:
            return pd.DataFrame()

        values = result[0]["values"]
        df = pd.DataFrame(values, columns=["ds", "y"])
        df["ds"] = pd.to_datetime(df["ds"].astype(float), unit="s")
        df["y"]  = df["y"].astype(float)
        return df

    async def _get_or_train_model(self, name: str, df: pd.DataFrame) -> Prophet:
        if name not in self._models or self._should_retrain(name):
            logger.info(f"Training Prophet model for {name} on {len(df)} points")
            model = Prophet(
                changepoint_prior_scale=0.05,
                seasonality_prior_scale=10,
                daily_seasonality=True,
                weekly_seasonality=True,
                interval_width=0.80,
            )
            # Add custom seasonality for peak hours (09:00-22:00 UTC)
            model.add_seasonality(name="peak_hours", period=1, fourier_order=5)
            model.fit(df)
            self._models[name] = model
            self._models[f"_trained_{name}"] = datetime.utcnow()
        return self._models[name]

    def _should_retrain(self, name: str) -> bool:
        key = f"_trained_{name}"
        if key not in self._models:
            return True
        return datetime.utcnow() - self._models[key] > timedelta(hours=6)

    def _should_scale(self, d: ScalingDecision) -> bool:
        if d.confidence < 0.5:
            logger.info(f"Skipping {d.deployment}: low confidence {d.confidence:.2f}")
            return False
        if d.recommended_replicas > d.current_replicas:
            return (d.recommended_replicas - d.current_replicas) / d.current_replicas >= self.SCALE_THRESHOLD_UP
        else:
            return (d.current_replicas - d.recommended_replicas) / d.current_replicas >= self.SCALE_THRESHOLD_DOWN

    async def _apply_scaling(self, d: ScalingDecision):
        logger.info(
            f"Scaling {d.deployment}: {d.current_replicas} → {d.recommended_replicas} "
            f"({d.reason})"
        )
        patch = {"spec": {"replicas": d.recommended_replicas}}
        self._apps_v1.patch_namespaced_deployment(
            name=d.deployment,
            namespace=d.namespace,
            body=patch,
        )
        # Emit scaling event
        v1 = k8s.CoreV1Api()
        v1.create_namespaced_event(
            namespace=d.namespace,
            body=k8s.V1Event(
                metadata=k8s.V1ObjectMeta(
                    name=f"predictive-scale-{d.deployment}-{int(datetime.utcnow().timestamp())}",
                    namespace=d.namespace,
                ),
                involved_object=k8s.V1ObjectReference(
                    kind="Deployment",
                    name=d.deployment,
                    namespace=d.namespace,
                ),
                reason="PredictiveScale",
                message=d.reason,
                type="Normal",
                event_time=datetime.utcnow().isoformat() + "Z",
                action="Scale",
                reporting_component="predictive-scaler",
                reporting_instance="predictive-scaler-0",
            ),
        )


if __name__ == "__main__":
    scaler = PredictiveScaler(
        prometheus_url="http://prometheus:9090",
        namespace="payment-prod",
    )
    asyncio.run(scaler.run())
PYTHON

# Kubernetes deployment for the scaler
cat > aiops/predictive-scaler-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: predictive-scaler
  namespace: platform-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: predictive-scaler
  template:
    metadata:
      labels:
        app: predictive-scaler
    spec:
      serviceAccountName: predictive-scaler-sa
      containers:
        - name: scaler
          image: gcr.io/myproject/predictive-scaler:latest
          env:
            - name: PROMETHEUS_URL
              value: "http://prometheus.monitoring:9090"
            - name: TARGET_NAMESPACE
              value: "payment-prod"
          resources:
            requests:
              cpu: "500m"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: predictive-scaler-sa
  namespace: platform-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: predictive-scaler-role
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: predictive-scaler-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: predictive-scaler-role
subjects:
  - kind: ServiceAccount
    name: predictive-scaler-sa
    namespace: platform-system
EOF
kubectl apply -f aiops/

echo "Predictive autoscaler deployed"
```

---

## ขั้นตอนที่ 632: Autonomous Incident Resolution Engine

### `autonomous-incident-resolver.sh`

```bash
#!/bin/bash
# Autonomous Incident Resolution: Alert correlation → RCA → Auto-remediation

cat > aiops/incident_resolver.py << 'PYTHON'
import asyncio
import logging
import time
import json
import re
from dataclasses import dataclass, field
from typing import Optional
from enum import Enum
import httpx
import anthropic  # Claude API for RCA reasoning

logger = logging.getLogger(__name__)


class Severity(str, Enum):
    P1 = "P1"  # Critical
    P2 = "P2"  # High
    P3 = "P3"  # Medium
    P4 = "P4"  # Low


class RemediationAction(str, Enum):
    RESTART_POD          = "restart_pod"
    SCALE_UP             = "scale_up"
    ROLLBACK_DEPLOYMENT  = "rollback_deployment"
    CLEAR_CACHE          = "clear_cache"
    CIRCUIT_BREAK        = "circuit_break"
    REROUTE_TRAFFIC      = "reroute_traffic"
    DRAIN_NODE           = "drain_node"
    ESCALATE_HUMAN       = "escalate_human"


@dataclass
class Alert:
    name: str
    severity: Severity
    service: str
    labels: dict
    annotations: dict
    fired_at: float = field(default_factory=time.time)
    value: float = 0.0


@dataclass
class Incident:
    id: str
    alerts: list[Alert]
    service: str
    severity: Severity
    created_at: float = field(default_factory=time.time)
    rca: str = ""
    actions_taken: list[str] = field(default_factory=list)
    resolved: bool = False
    resolution_time: Optional[float] = None


class AutonomousIncidentResolver:
    """
    Correlates alerts → generates RCA with Claude API → applies safe remediations.
    """
    CORRELATION_WINDOW = 300     # 5 minutes
    AUTO_REMEDIATION_THRESHOLD = Severity.P2  # Only auto-remediate P2 and below
    MAX_AUTO_ACTIONS_PER_INCIDENT = 3

    # Known safe remediations per alert pattern (no human required)
    SAFE_REMEDIATIONS: dict[str, list[RemediationAction]] = {
        "OOMKilled":              [RemediationAction.RESTART_POD, RemediationAction.SCALE_UP],
        "CrashLoopBackOff":       [RemediationAction.RESTART_POD],
        "HighMemoryUsage":        [RemediationAction.CLEAR_CACHE, RemediationAction.SCALE_UP],
        "HighCPUUsage":           [RemediationAction.SCALE_UP],
        "ConnectionPoolExhausted":[RemediationAction.RESTART_POD, RemediationAction.SCALE_UP],
        "HighErrorRate":          [RemediationAction.CIRCUIT_BREAK, RemediationAction.REROUTE_TRAFFIC],
        "PodNotReady":            [RemediationAction.RESTART_POD],
        "NodeDiskPressure":       [RemediationAction.DRAIN_NODE],
        "DeploymentRollout":      [RemediationAction.ROLLBACK_DEPLOYMENT],
    }

    def __init__(self, prometheus_url: str, k8s_namespace: str, slack_webhook: str):
        self.prometheus_url = prometheus_url
        self.namespace = k8s_namespace
        self.slack_webhook = slack_webhook
        self._claude = anthropic.Anthropic()
        self._active_incidents: dict[str, Incident] = {}
        self._alert_buffer: list[Alert] = []
        self._http = httpx.AsyncClient(timeout=30)

    async def handle_alert(self, alert_payload: dict):
        for a in alert_payload.get("alerts", []):
            if a.get("status") != "firing":
                continue
            alert = Alert(
                name=a["labels"].get("alertname", "Unknown"),
                severity=Severity(a["labels"].get("severity", "P3")),
                service=a["labels"].get("service", a["labels"].get("job", "unknown")),
                labels=a["labels"],
                annotations=a.get("annotations", {}),
                value=float(a.get("value", 0)),
            )
            self._alert_buffer.append(alert)
            logger.info(f"Buffered alert: {alert.name} [{alert.severity}] service={alert.service}")

        # Correlate after buffer accumulation
        await self._correlate_and_resolve()

    async def _correlate_and_resolve(self):
        now = time.time()
        # Group alerts by service within correlation window
        recent = [a for a in self._alert_buffer if now - a.fired_at < self.CORRELATION_WINDOW]
        service_groups: dict[str, list[Alert]] = {}
        for alert in recent:
            service_groups.setdefault(alert.service, []).append(alert)

        for service, alerts in service_groups.items():
            incident_key = f"{service}:{sorted([a.name for a in alerts])[0]}"
            if incident_key in self._active_incidents:
                # Enrich existing incident
                self._active_incidents[incident_key].alerts.extend(alerts)
                continue

            severity = max(alerts, key=lambda a: ["P4","P3","P2","P1"].index(a.severity)).severity
            incident = Incident(
                id=f"INC-{int(now)}-{service[:8]}",
                alerts=alerts,
                service=service,
                severity=severity,
            )
            self._active_incidents[incident_key] = incident

            logger.info(f"New incident: {incident.id} ({severity}) for {service} "
                        f"with {len(alerts)} alerts")

            await self._process_incident(incident)

    async def _process_incident(self, incident: Incident):
        # Step 1: Gather context
        context = await self._gather_context(incident)

        # Step 2: RCA with Claude
        incident.rca = await self._generate_rca(incident, context)
        logger.info(f"[{incident.id}] RCA: {incident.rca[:200]}")

        # Step 3: Determine remediation
        actions = self._determine_remediations(incident)

        # Step 4: Apply safe auto-remediations
        if incident.severity in (Severity.P3, Severity.P4) or \
           (incident.severity == Severity.P2 and len(actions) <= self.MAX_AUTO_ACTIONS_PER_INCIDENT):
            for action in actions[:self.MAX_AUTO_ACTIONS_PER_INCIDENT]:
                success = await self._execute_action(incident, action)
                incident.actions_taken.append(f"{action}: {'OK' if success else 'FAILED'}")
        else:
            incident.actions_taken.append("Escalated to on-call team (P1 incident)")

        # Step 5: Notify
        await self._notify_slack(incident)

    async def _gather_context(self, incident: Incident) -> dict:
        queries = {
            "error_rate": f'rate(http_requests_total{{service="{incident.service}",status=~"5.."}}[5m])',
            "latency_p99": f'histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{{service="{incident.service}"}}[5m]))',
            "pod_restarts": f'increase(kube_pod_container_status_restarts_total{{namespace="{self.namespace}",pod=~"{incident.service}-.*"}}[10m])',
            "memory_usage": f'container_memory_working_set_bytes{{namespace="{self.namespace}",pod=~"{incident.service}-.*"}}',
        }

        context = {}
        for name, query in queries.items():
            try:
                resp = await self._http.get(
                    f"{self.prometheus_url}/api/v1/query",
                    params={"query": query},
                )
                data = resp.json()
                if data["data"]["result"]:
                    context[name] = float(data["data"]["result"][0]["value"][1])
            except Exception as e:
                logger.warning(f"Prometheus query failed ({name}): {e}")

        return context

    async def _generate_rca(self, incident: Incident, context: dict) -> str:
        alert_summary = "\n".join(
            f"- {a.name} ({a.severity}): {a.annotations.get('description', '')}"
            for a in incident.alerts
        )
        context_summary = json.dumps(context, indent=2)

        message = self._claude.messages.create(
            model="claude-opus-5-5",
            max_tokens=1024,
            messages=[{
                "role": "user",
                "content": f"""You are an expert SRE performing root cause analysis.

Service: {incident.service}
Incident: {incident.id} (Severity: {incident.severity})

Active Alerts:
{alert_summary}

Current Metrics:
{context_summary}

Provide a concise root cause analysis (2-3 sentences) and the most likely single root cause.
Format: ROOT_CAUSE: <one sentence> | ANALYSIS: <2 sentences>"""
            }]
        )
        return message.content[0].text

    def _determine_remediations(self, incident: Incident) -> list[RemediationAction]:
        actions: set[RemediationAction] = set()
        for alert in incident.alerts:
            for pattern, remediation_list in self.SAFE_REMEDIATIONS.items():
                if pattern.lower() in alert.name.lower():
                    actions.update(remediation_list)

        if not actions or incident.severity == Severity.P1:
            actions.add(RemediationAction.ESCALATE_HUMAN)

        # Prioritize: restart before scale
        ordered = [RemediationAction.ESCALATE_HUMAN,
                   RemediationAction.ROLLBACK_DEPLOYMENT,
                   RemediationAction.CIRCUIT_BREAK,
                   RemediationAction.REROUTE_TRAFFIC,
                   RemediationAction.DRAIN_NODE,
                   RemediationAction.RESTART_POD,
                   RemediationAction.SCALE_UP,
                   RemediationAction.CLEAR_CACHE]
        return [a for a in ordered if a in actions]

    async def _execute_action(self, incident: Incident, action: RemediationAction) -> bool:
        logger.info(f"[{incident.id}] Executing: {action}")
        try:
            if action == RemediationAction.RESTART_POD:
                return await self._restart_pods(incident.service)
            elif action == RemediationAction.SCALE_UP:
                return await self._scale_up(incident.service, delta=2)
            elif action == RemediationAction.ROLLBACK_DEPLOYMENT:
                return await self._rollback(incident.service)
            elif action == RemediationAction.CLEAR_CACHE:
                return await self._clear_cache(incident.service)
            elif action == RemediationAction.CIRCUIT_BREAK:
                return await self._set_circuit_breaker(incident.service, open=True)
            elif action == RemediationAction.ESCALATE_HUMAN:
                return await self._page_oncall(incident)
        except Exception as e:
            logger.error(f"[{incident.id}] Action {action} failed: {e}")
            return False
        return True

    async def _restart_pods(self, service: str) -> bool:
        # Rolling restart via deployment annotation
        resp = await self._http.patch(
            f"http://kubernetes.default/apis/apps/v1/namespaces/{self.namespace}/deployments/{service}",
            json={"spec": {"template": {"metadata": {"annotations":
                {"kubectl.kubernetes.io/restartedAt": time.strftime("%Y-%m-%dT%H:%M:%SZ")}
            }}}},
            headers={"Content-Type": "application/strategic-merge-patch+json"},
        )
        return resp.status_code < 300

    async def _scale_up(self, service: str, delta: int) -> bool:
        from kubernetes import client as k8s, config as k8s_config
        k8s_config.load_incluster_config()
        apps = k8s.AppsV1Api()
        dep = apps.read_namespaced_deployment(service, self.namespace)
        new_replicas = min(50, (dep.spec.replicas or 1) + delta)
        apps.patch_namespaced_deployment(service, self.namespace, {"spec": {"replicas": new_replicas}})
        return True

    async def _rollback(self, service: str) -> bool:
        resp = await self._http.post(
            f"http://kubernetes.default/apis/apps/v1/namespaces/{self.namespace}/deployments/{service}/rollback",
            json={"apiVersion": "apps/v1", "name": service, "rollbackTo": {"revision": 0}},
        )
        return resp.status_code < 300

    async def _clear_cache(self, service: str) -> bool:
        # Emit a cache-clear event that the service handles via its ConfigMap watch
        import subprocess
        result = subprocess.run([
            "kubectl", "annotate", "deployment", service,
            f"cache-cleared-at={time.strftime('%Y%m%d%H%M%S')}",
            "-n", self.namespace, "--overwrite"
        ])
        return result.returncode == 0

    async def _set_circuit_breaker(self, service: str, open: bool) -> bool:
        # Patch Istio VirtualService to abort 100% traffic (open breaker)
        vs_patch = {
            "spec": {"http": [{"fault": {"abort": {"percentage": {"value": 0 if not open else 100},
                                                    "httpStatus": 503}}}]}
        }
        resp = await self._http.patch(
            f"http://kubernetes.default/apis/networking.istio.io/v1beta1/namespaces/{self.namespace}/virtualservices/{service}",
            json=vs_patch,
        )
        return resp.status_code < 300

    async def _page_oncall(self, incident: Incident) -> bool:
        payload = {
            "routing_key": "PAGERDUTY_ROUTING_KEY",
            "event_action": "trigger",
            "dedup_key": incident.id,
            "payload": {
                "summary": f"[{incident.severity}] {incident.service}: {incident.id}",
                "source": incident.service,
                "severity": incident.severity.lower(),
                "custom_details": {"rca": incident.rca, "alerts": len(incident.alerts)},
            },
        }
        resp = await self._http.post("https://events.pagerduty.com/v2/enqueue", json=payload)
        return resp.status_code == 202

    async def _notify_slack(self, incident: Incident):
        color = {"P1": "#ff0000", "P2": "#ff8800", "P3": "#ffcc00", "P4": "#00cc00"}[incident.severity]
        payload = {
            "attachments": [{
                "color": color,
                "title": f"[{incident.severity}] {incident.id}: {incident.service}",
                "text": incident.rca,
                "fields": [
                    {"title": "Alerts", "value": str(len(incident.alerts)), "short": True},
                    {"title": "Actions", "value": "\n".join(incident.actions_taken) or "None", "short": False},
                ],
                "footer": "AIOps Autonomous Resolver",
                "ts": int(incident.created_at),
            }]
        }
        await self._http.post(self.slack_webhook, json=payload)
PYTHON

echo "Autonomous incident resolver complete"
```

---

## ขั้นตอนที่ 633: Log Intelligence — Anomaly Detection & Pattern Mining

### `log-intelligence.sh`

```bash
#!/bin/bash
# Log Intelligence: ML-based anomaly detection, log clustering, noise reduction

cat > aiops/log_intelligence.py << 'PYTHON'
import asyncio
import json
import logging
import re
import hashlib
from collections import Counter, defaultdict
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import Optional
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.cluster import MiniBatchKMeans
from sklearn.ensemble import IsolationForest
import aiohttp

logger = logging.getLogger(__name__)


@dataclass
class LogEntry:
    timestamp: datetime
    level: str
    service: str
    message: str
    trace_id: str = ""
    span_id: str = ""
    template: str = ""    # extracted log template (variables replaced with <VAR>)
    cluster_id: int = -1
    anomaly_score: float = 0.0


@dataclass
class LogCluster:
    id: int
    template: str
    count: int
    services: set[str]
    first_seen: datetime
    last_seen: datetime
    anomalous: bool = False
    representative: str = ""


class LogTemplateExtractor:
    """
    Drain3-inspired log template extraction.
    Replaces variable tokens (numbers, UUIDs, IPs, paths) with <VAR>.
    """
    VAR_PATTERNS = [
        (re.compile(r'\b[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}\b', re.I), '<UUID>'),
        (re.compile(r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b'), '<IP>'),
        (re.compile(r'\b[0-9]+\b'), '<NUM>'),
        (re.compile(r'"[^"]{4,}"'), '<STR>'),
        (re.compile(r'/[a-zA-Z0-9_/.-]{5,}'), '<PATH>'),
        (re.compile(r'\b[0-9a-f]{32,64}\b'), '<HASH>'),
    ]

    def extract(self, message: str) -> str:
        template = message
        for pattern, replacement in self.VAR_PATTERNS:
            template = pattern.sub(replacement, template)
        # Normalize whitespace
        return ' '.join(template.split())


class LogIntelligenceEngine:
    WINDOW_MINUTES = 5
    CLUSTER_COUNT = 50
    ANOMALY_CONTAMINATION = 0.05
    NOISE_TEMPLATES = {
        "health check",
        "readiness probe",
        "liveness probe",
        "GET /healthz",
        "GET /readyz",
        "GET /metrics",
    }

    def __init__(self, loki_url: str, namespace: str):
        self.loki_url = loki_url
        self.namespace = namespace
        self.template_extractor = LogTemplateExtractor()
        self._vectorizer = TfidfVectorizer(max_features=1000, ngram_range=(1, 2))
        self._kmeans: Optional[MiniBatchKMeans] = None
        self._isolation_forest: Optional[IsolationForest] = None
        self._clusters: dict[int, LogCluster] = {}
        self._template_cache: dict[str, int] = {}  # template → cluster_id
        self._window_counts = defaultdict(list)  # cluster_id → [timestamps]

    async def run(self):
        while True:
            try:
                logs = await self._fetch_recent_logs()
                if logs:
                    processed = self._process_batch(logs)
                    anomalies = self._detect_anomalies(processed)
                    if anomalies:
                        await self._report_anomalies(anomalies)
                        logger.info(f"Detected {len(anomalies)} anomalous log patterns")
            except Exception as e:
                logger.error(f"Log intelligence error: {e}")
            await asyncio.sleep(self.WINDOW_MINUTES * 60)

    async def _fetch_recent_logs(self) -> list[LogEntry]:
        end = datetime.utcnow()
        start = end - timedelta(minutes=self.WINDOW_MINUTES)
        entries = []

        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.loki_url}/loki/api/v1/query_range",
                params={
                    "query": f'{{namespace="{self.namespace}"}}',
                    "start": int(start.timestamp() * 1e9),
                    "end": int(end.timestamp() * 1e9),
                    "limit": 10000,
                    "direction": "forward",
                },
            ) as resp:
                data = await resp.json()

        for stream in data.get("data", {}).get("result", []):
            labels = stream.get("stream", {})
            service = labels.get("app", labels.get("pod", "unknown"))
            for ts_nano, line in stream.get("values", []):
                try:
                    log = json.loads(line)
                    entries.append(LogEntry(
                        timestamp=datetime.fromtimestamp(int(ts_nano) / 1e9),
                        level=log.get("level", "INFO").upper(),
                        service=service,
                        message=log.get("msg", log.get("message", line)),
                        trace_id=log.get("trace_id", ""),
                        span_id=log.get("span_id", ""),
                    ))
                except json.JSONDecodeError:
                    entries.append(LogEntry(
                        timestamp=datetime.fromtimestamp(int(ts_nano) / 1e9),
                        level="INFO",
                        service=service,
                        message=line,
                    ))
        return entries

    def _process_batch(self, entries: list[LogEntry]) -> list[LogEntry]:
        for entry in entries:
            entry.template = self.template_extractor.extract(entry.message)

        # Filter noise
        entries = [
            e for e in entries
            if not any(noise in e.message.lower() for noise in self.NOISE_TEMPLATES)
        ]

        if not entries:
            return []

        templates = [e.template for e in entries]

        # Fit or partial-fit TF-IDF + KMeans
        try:
            vectors = self._vectorizer.fit_transform(templates)
            n_clusters = min(self.CLUSTER_COUNT, len(entries))
            self._kmeans = MiniBatchKMeans(n_clusters=n_clusters, random_state=42, batch_size=256)
            self._kmeans.fit(vectors)

            labels = self._kmeans.predict(vectors)
            for entry, label, vec in zip(entries, labels, vectors.toarray()):
                entry.cluster_id = int(label)
                self._update_cluster(entry, int(label))

            # Isolation Forest on cluster counts per window
            count_features = self._get_count_features()
            if count_features.shape[0] >= 10:
                self._isolation_forest = IsolationForest(
                    contamination=self.ANOMALY_CONTAMINATION,
                    random_state=42,
                )
                scores = self._isolation_forest.fit_predict(count_features)
                # Mark recent window anomalies
                for i, entry in enumerate(entries):
                    if i < len(scores):
                        entry.anomaly_score = float(-scores[i])  # 1=anomaly, -1=normal → invert
        except Exception as e:
            logger.warning(f"Clustering failed: {e}")

        return entries

    def _update_cluster(self, entry: LogEntry, cluster_id: int):
        if cluster_id not in self._clusters:
            self._clusters[cluster_id] = LogCluster(
                id=cluster_id,
                template=entry.template,
                count=0,
                services=set(),
                first_seen=entry.timestamp,
                last_seen=entry.timestamp,
                representative=entry.message,
            )
        cluster = self._clusters[cluster_id]
        cluster.count += 1
        cluster.services.add(entry.service)
        cluster.last_seen = max(cluster.last_seen, entry.timestamp)
        self._window_counts[cluster_id].append(entry.timestamp)

    def _get_count_features(self) -> np.ndarray:
        now = datetime.utcnow()
        features = []
        for cid, times in self._window_counts.items():
            recent = [t for t in times if now - t < timedelta(minutes=self.WINDOW_MINUTES)]
            features.append([cid, len(recent)])
        return np.array(features) if features else np.zeros((0, 2))

    def _detect_anomalies(self, entries: list[LogEntry]) -> list[dict]:
        anomalies = []
        # Anomalous clusters: spike in error logs or new unseen pattern
        cluster_errors = Counter(
            e.cluster_id for e in entries if e.level in ("ERROR", "FATAL")
        )

        for cluster_id, error_count in cluster_errors.items():
            cluster = self._clusters.get(cluster_id)
            if cluster and error_count >= 10:
                anomalies.append({
                    "type": "error_spike",
                    "cluster_id": cluster_id,
                    "template": cluster.template,
                    "error_count": error_count,
                    "services": list(cluster.services),
                    "representative": cluster.representative,
                })

        # New patterns not seen in previous windows
        for entry in entries:
            if entry.anomaly_score > 0.8 and entry.level in ("ERROR", "WARN"):
                anomalies.append({
                    "type": "anomalous_pattern",
                    "template": entry.template,
                    "score": entry.anomaly_score,
                    "service": entry.service,
                    "sample": entry.message[:300],
                })

        return anomalies

    async def _report_anomalies(self, anomalies: list[dict]):
        for anomaly in anomalies:
            logger.warning(f"Log anomaly detected: {json.dumps(anomaly, default=str)}")
            # In production: send to alertmanager / Slack / incident resolver
PYTHON

echo "Log intelligence engine complete"
```

---

## ขั้นตอนที่ 634: ML Model Serving Pipeline — Canary + A/B Testing

### `ml-serving-pipeline.sh`

```bash
#!/bin/bash
# ML Model Serving: Canary deployment, A/B testing, shadow mode, drift detection

cat > mlops/model_serving.py << 'PYTHON'
import asyncio
import hashlib
import json
import logging
import random
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Optional
from fastapi import FastAPI, Request, HTTPException, BackgroundTasks
from fastapi.responses import JSONResponse
import httpx
import numpy as np
from prometheus_client import Counter, Histogram, Gauge, generate_latest
from starlette.responses import Response

logger = logging.getLogger(__name__)

app = FastAPI(title="ML Model Router", version="1.0.0")

# Metrics
REQUEST_COUNTER = Counter("model_requests_total", "Total requests", ["model", "version", "status"])
PREDICTION_LATENCY = Histogram("model_prediction_seconds", "Prediction latency", ["model", "version"],
                               buckets=[.005,.01,.025,.05,.1,.25,.5,1,2.5,5])
SHADOW_DIVERGENCE = Gauge("model_shadow_divergence", "Shadow vs production output divergence", ["model"])


class RoutingStrategy(str, Enum):
    CANARY          = "canary"       # % of traffic to new version
    AB_TEST         = "ab_test"      # user-cohort-based split
    SHADOW          = "shadow"       # send to both, return primary response
    CHAMPION_CHALLENGER = "champion_challenger"  # gradually shift based on metric


@dataclass
class ModelVersion:
    name: str
    version: str
    endpoint: str
    weight: float = 1.0  # relative traffic weight
    metadata: dict = field(default_factory=dict)


@dataclass
class RouterConfig:
    model_name: str
    strategy: RoutingStrategy
    primary: ModelVersion
    candidates: list[ModelVersion] = field(default_factory=list)
    shadow: Optional[ModelVersion] = None
    experiment_id: str = ""
    created_at: float = field(default_factory=time.time)


class ModelRouter:
    def __init__(self):
        self._configs: dict[str, RouterConfig] = {}
        self._shadow_results: dict[str, list] = {}
        self._http = httpx.AsyncClient(timeout=10)
        self._ab_assignments: dict[str, str] = {}  # user_id → model_version

    def register(self, config: RouterConfig):
        self._configs[config.model_name] = config
        logger.info(f"Registered router for {config.model_name} (strategy={config.strategy})")

    async def predict(self, model_name: str, payload: dict, user_id: str = "") -> dict:
        config = self._configs.get(model_name)
        if not config:
            raise HTTPException(status_code=404, detail=f"Model {model_name} not found")

        start = time.perf_counter()
        target = self._select_version(config, user_id)

        try:
            response = await self._call_model(target, payload)
            latency = time.perf_counter() - start
            PREDICTION_LATENCY.labels(model=model_name, version=target.version).observe(latency)
            REQUEST_COUNTER.labels(model=model_name, version=target.version, status="ok").inc()

            # Shadow mode: also call shadow model asynchronously
            if config.strategy == RoutingStrategy.SHADOW and config.shadow:
                asyncio.create_task(
                    self._shadow_call(model_name, config.shadow, payload, response)
                )

            return {
                "prediction": response,
                "model_version": target.version,
                "latency_ms": round(latency * 1000, 2),
                "experiment_id": config.experiment_id,
            }
        except Exception as e:
            REQUEST_COUNTER.labels(model=model_name, version=target.version, status="error").inc()
            raise HTTPException(status_code=502, detail=str(e))

    def _select_version(self, config: RouterConfig, user_id: str) -> ModelVersion:
        if config.strategy == RoutingStrategy.CANARY:
            return self._canary_select(config)
        elif config.strategy == RoutingStrategy.AB_TEST:
            return self._ab_select(config, user_id)
        elif config.strategy == RoutingStrategy.CHAMPION_CHALLENGER:
            return self._weighted_select(config)
        else:
            return config.primary

    def _canary_select(self, config: RouterConfig) -> ModelVersion:
        total_candidate_weight = sum(c.weight for c in config.candidates)
        if total_candidate_weight == 0:
            return config.primary
        r = random.random()
        if r < total_candidate_weight / (total_candidate_weight + config.primary.weight):
            # Select among candidates weighted
            weights = [c.weight for c in config.candidates]
            return random.choices(config.candidates, weights=weights, k=1)[0]
        return config.primary

    def _ab_select(self, config: RouterConfig, user_id: str) -> ModelVersion:
        if not user_id:
            return config.primary
        # Deterministic assignment based on user_id hash
        cache_key = f"{config.experiment_id}:{user_id}"
        if cache_key not in self._ab_assignments:
            bucket = int(hashlib.md5(cache_key.encode()).hexdigest(), 16) % 100
            all_versions = [config.primary] + config.candidates
            weights = [v.weight for v in all_versions]
            total = sum(weights)
            cumulative = 0
            assigned = config.primary
            for version, weight in zip(all_versions, weights):
                cumulative += (weight / total) * 100
                if bucket < cumulative:
                    assigned = version
                    break
            self._ab_assignments[cache_key] = assigned.version
        target_version = self._ab_assignments[cache_key]
        all_versions = [config.primary] + config.candidates
        return next((v for v in all_versions if v.version == target_version), config.primary)

    def _weighted_select(self, config: RouterConfig) -> ModelVersion:
        all_versions = [config.primary] + config.candidates
        weights = [v.weight for v in all_versions]
        return random.choices(all_versions, weights=weights, k=1)[0]

    async def _call_model(self, version: ModelVersion, payload: dict) -> Any:
        resp = await self._http.post(f"{version.endpoint}/predict", json=payload)
        resp.raise_for_status()
        return resp.json()

    async def _shadow_call(
        self, model_name: str, shadow: ModelVersion, payload: dict, primary_response: Any
    ):
        try:
            shadow_response = await self._call_model(shadow, payload)
            # Measure divergence between primary and shadow outputs
            divergence = self._compute_divergence(primary_response, shadow_response)
            SHADOW_DIVERGENCE.labels(model=model_name).set(divergence)
            self._shadow_results.setdefault(model_name, []).append({
                "primary": primary_response,
                "shadow": shadow_response,
                "divergence": divergence,
                "ts": time.time(),
            })
            # Keep only last 1000 shadow results
            self._shadow_results[model_name] = self._shadow_results[model_name][-1000:]
        except Exception as e:
            logger.warning(f"Shadow call failed for {model_name}: {e}")

    def _compute_divergence(self, a: Any, b: Any) -> float:
        if isinstance(a, dict) and isinstance(b, dict):
            a_vals = np.array(list(a.get("scores", [0])), dtype=float)
            b_vals = np.array(list(b.get("scores", [0])), dtype=float)
            if len(a_vals) == len(b_vals) > 0:
                return float(np.mean(np.abs(a_vals - b_vals)))
        return 0.0

    def get_shadow_analysis(self, model_name: str) -> dict:
        results = self._shadow_results.get(model_name, [])
        if not results:
            return {"count": 0, "mean_divergence": 0}
        divergences = [r["divergence"] for r in results]
        return {
            "count": len(divergences),
            "mean_divergence": float(np.mean(divergences)),
            "p95_divergence": float(np.percentile(divergences, 95)),
            "recommendation": "promote" if np.mean(divergences) < 0.05 else "investigate",
        }


router = ModelRouter()


@app.post("/v1/models/{model_name}/predict")
async def predict(model_name: str, request: Request, background_tasks: BackgroundTasks):
    payload = await request.json()
    user_id = request.headers.get("X-User-ID", "")
    return await router.predict(model_name, payload, user_id)


@app.get("/v1/models/{model_name}/shadow-analysis")
async def shadow_analysis(model_name: str):
    return router.get_shadow_analysis(model_name)


@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type="text/plain")


@app.on_event("startup")
async def startup():
    # Example: Route fraud detection model with 10% canary
    router.register(RouterConfig(
        model_name="fraud-detection",
        strategy=RoutingStrategy.CANARY,
        primary=ModelVersion("fraud-detection", "v2.1.0", "http://fraud-v2-1:8080", weight=0.9),
        candidates=[
            ModelVersion("fraud-detection", "v2.2.0", "http://fraud-v2-2:8080", weight=0.1),
        ],
        experiment_id="exp-fraud-2024-q1",
    ))

    # Shadow: test new recommendation model in parallel
    router.register(RouterConfig(
        model_name="recommendation",
        strategy=RoutingStrategy.SHADOW,
        primary=ModelVersion("recommendation", "v1.5.0", "http://rec-v1-5:8080"),
        shadow=ModelVersion("recommendation", "v2.0.0", "http://rec-v2-0:8080"),
    ))
PYTHON

echo "ML serving pipeline complete"
```

---

## สรุป Part 71

Part 71 ครอบคลุม AIOps และ ML-Driven Operations ระดับ production:

| Step | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|--------------|
| 631 | Predictive Autoscaling | Facebook Prophet, Prometheus, K8s custom controller, RBAC |
| 632 | Autonomous Incident Resolution | Claude API (RCA), Prometheus context gathering, auto-remediation |
| 633 | Log Intelligence | TF-IDF + MiniBatchKMeans clustering, Isolation Forest anomaly detection, Drain-style templates |
| 634 | ML Model Serving | Canary/A/B/Shadow/Champion-Challenger routing, divergence measurement |

### Key Concepts ที่เรียนรู้:
- **Prophet Forecasting**: changepoint_prior_scale, weekly+daily seasonality, lookahead prediction
- **K8s Custom Controller**: ClusterRole, in-cluster config, patch deployment replicas
- **Claude API for RCA**: structured prompt engineering for root cause analysis
- **Alert Correlation**: time-window grouping, service-based deduplication, safe remediation catalog
- **Log Clustering**: TF-IDF vectorization, MiniBatchKMeans, Isolation Forest contamination
- **Model Router**: deterministic A/B assignment (MD5 hash), shadow async calls, divergence metrics
- **Canary Deployment**: weighted random selection, experiment tracking, Prometheus metrics

ขั้นตอนต่อไป: **Part 72** — Zero-Trust Security Architecture, mTLS Mesh, Policy as Code
