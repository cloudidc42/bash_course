# Part 57: Multi-Cloud Strategy และ Edge Computing

## Module 5: World-Class Level (ต่อ)

---

## ขั้นตอนที่ 577: Multi-Cloud Federation และ Unified Control Plane

### แนวคิด Multi-Cloud Architecture

```
Multi-Cloud Federation Architecture:
┌─────────────────────────────────────────────────────────────┐
│                    Unified Control Plane                      │
│              (Anthos / Azure Arc / Crossplane)               │
└──────────────────────┬──────────────────────────────────────┘
           ┌───────────┼───────────┐
           ▼           ▼           ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │   AWS    │ │  Azure   │ │   GCP    │
    │  EKS     │ │  AKS     │ │  GKE     │
    │  RDS     │ │  CosmosDB│ │  Spanner │
    │  S3      │ │  Blob    │ │  GCS     │
    └──────────┘ └──────────┘ └──────────┘
           │           │           │
           └───────────┼───────────┘
                       ▼
             ┌─────────────────┐
             │  Service Mesh   │
             │  (Istio + MCS)  │
             └─────────────────┘
```

### `multi-cloud-federation.sh`

```bash
#!/usr/bin/env bash
# multi-cloud-federation.sh — Unified Multi-Cloud Control Plane
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_FILE="/var/log/multi-cloud-federation.log"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Crossplane Multi-Cloud Configuration ───────────────────────────────────

setup_crossplane_multicloud() {
  log "=== Setting up Crossplane Multi-Cloud Federation ==="

  # Install Crossplane with multi-cloud providers
  helm repo add crossplane-stable https://charts.crossplane.io/stable
  helm repo update

  helm upgrade --install crossplane crossplane-stable/crossplane \
    --namespace crossplane-system \
    --create-namespace \
    --set args='{--enable-external-secret-stores}' \
    --wait

  # ─── AWS Provider ───────────────────────────────────────────────────────────
  cat <<'EOF' | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws:v0.47.0
  controllerConfigRef:
    name: aws-config
---
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: aws-config
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/crossplane-aws
spec:
  podSecurityContext:
    fsGroup: 2000
---
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: aws-production
spec:
  credentials:
    source: IRSA
EOF

  # ─── Azure Provider ─────────────────────────────────────────────────────────
  cat <<'EOF' | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure
spec:
  package: xpkg.upbound.io/upbound/provider-azure:v0.38.0
---
apiVersion: azure.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: azure-production
spec:
  credentials:
    source: SystemAssignedManagedIdentity
EOF

  # ─── GCP Provider ───────────────────────────────────────────────────────────
  cat <<'EOF' | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp
spec:
  package: xpkg.upbound.io/upbound/provider-gcp:v0.37.0
---
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: gcp-production
spec:
  projectID: my-gcp-project
  credentials:
    source: InjectedIdentity
EOF

  log "✓ Crossplane multi-cloud providers configured"
}

# ─── 2. Multi-Cloud XRD (Cross-Resource Definition) ───────────────────────────

create_multicloud_xrd() {
  log "=== Creating Multi-Cloud XRD ==="

  cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xmulticlouddatabases.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: XMultiCloudDatabase
    plural: xmulticlouddatabases
  claimNames:
    kind: MultiCloudDatabase
    plural: multiclouddatabases
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - parameters
              properties:
                parameters:
                  type: object
                  required:
                    - provider
                    - engine
                    - size
                  properties:
                    provider:
                      type: string
                      enum: ["aws", "azure", "gcp"]
                    engine:
                      type: string
                      enum: ["postgresql", "mysql", "mongodb"]
                    size:
                      type: string
                      enum: ["small", "medium", "large"]
                    region:
                      type: string
                      default: "us-east-1"
                    backupEnabled:
                      type: boolean
                      default: true
            status:
              type: object
              properties:
                endpoint:
                  type: string
                port:
                  type: integer
                ready:
                  type: boolean
EOF

  # ─── Composition for AWS RDS ─────────────────────────────────────────────────
  cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xmulticlouddatabases-aws
  labels:
    provider: aws
    engine: postgresql
spec:
  compositeTypeRef:
    apiVersion: platform.example.com/v1alpha1
    kind: XMultiCloudDatabase
  mode: Pipeline
  pipeline:
    - step: render-aws-rds
      functionRef:
        name: function-go-templating
      input:
        apiVersion: gotemplating.fn.crossplane.io/v1beta1
        kind: GoTemplate
        source: Inline
        inline:
          template: |
            apiVersion: rds.aws.upbound.io/v1beta1
            kind: Instance
            metadata:
              name: {{ .observed.composite.resource.metadata.name }}-rds
              annotations:
                gotemplating.fn.crossplane.io/composition-resource-name: rds-instance
            spec:
              forProvider:
                region: {{ .observed.composite.resource.spec.parameters.region }}
                dbInstanceClass: {{ if eq .observed.composite.resource.spec.parameters.size "large" }}db.r6g.2xlarge{{ else if eq .observed.composite.resource.spec.parameters.size "medium" }}db.r6g.large{{ else }}db.t3.medium{{ end }}
                engine: {{ .observed.composite.resource.spec.parameters.engine }}
                engineVersion: "15.4"
                allocatedStorage: {{ if eq .observed.composite.resource.spec.parameters.size "large" }}500{{ else if eq .observed.composite.resource.spec.parameters.size "medium" }}100{{ else }}20{{ end }}
                multiAz: true
                skipFinalSnapshot: false
                backupRetentionPeriod: {{ if .observed.composite.resource.spec.parameters.backupEnabled }}7{{ else }}0{{ end }}
                storageEncrypted: true
                deletionProtection: true
                publiclyAccessible: false
              providerConfigRef:
                name: aws-production
EOF

  # ─── Composition for Azure PostgreSQL ────────────────────────────────────────
  cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xmulticlouddatabases-azure
  labels:
    provider: azure
    engine: postgresql
spec:
  compositeTypeRef:
    apiVersion: platform.example.com/v1alpha1
    kind: XMultiCloudDatabase
  resources:
    - name: azure-postgresql
      base:
        apiVersion: dbforpostgresql.azure.upbound.io/v1beta1
        kind: FlexibleServer
        spec:
          forProvider:
            location: East US
            version: "15"
            skuName: GP_Standard_D4s_v3
            storageMb: 131072
            highAvailability:
              - mode: ZoneRedundant
            backupRetentionDays: 7
            geoRedundantBackupEnabled: true
            administratorLogin: pgadmin
            administratorPasswordSecretRef:
              namespace: crossplane-system
              name: azure-pg-password
              key: password
          providerConfigRef:
            name: azure-production
      patches:
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.location
          transforms:
            - type: map
              map:
                us-east-1: East US
                eu-west-1: West Europe
                ap-southeast-1: Southeast Asia
EOF

  log "✓ Multi-cloud XRD and Compositions created"
}

# ─── 3. Multi-Cloud Service Mesh (Istio Multi-Cluster) ────────────────────────

setup_multicluster_istio() {
  log "=== Setting up Istio Multi-Cluster Service Mesh ==="

  # Cluster 1: AWS EKS (Primary)
  cat <<'EOF' > /tmp/istio-aws-primary.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-primary
  namespace: istio-system
spec:
  profile: default
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: aws-production
      network: aws-network
      externalIstiod: false
  meshConfig:
    defaultConfig:
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: aws-network
        enabled: true
        k8s:
          service:
            type: LoadBalancer
            ports:
              - port: 15021
                targetPort: 15021
                name: status-port
              - port: 15443
                targetPort: 15443
                name: tls
              - port: 15012
                targetPort: 15012
                name: tls-istiod
              - port: 15017
                targetPort: 15017
                name: tls-webhook
EOF

  # Cluster 2: GCP GKE (Secondary)
  cat <<'EOF' > /tmp/istio-gcp-secondary.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-secondary
  namespace: istio-system
spec:
  profile: default
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: gcp-production
      network: gcp-network
      remotePilotAddress: ${ISTIOD_REMOTE_ENDPOINT}
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label:
          istio: eastwestgateway
          topology.istio.io/network: gcp-network
        enabled: true
        k8s:
          service:
            type: LoadBalancer
EOF

  # Multi-Cluster Service Entry
  cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: cross-cloud-service
  namespace: production
spec:
  hosts:
    - payment-service.production.svc.cluster.local
  location: MESH_INTERNAL
  ports:
    - name: grpc
      number: 9090
      protocol: GRPC
  resolution: STATIC
  workloadSelector:
    labels:
      app: payment-service
  endpoints:
    - address: ${AWS_EKS_ENDPOINT}
      network: aws-network
      ports:
        grpc: 9090
      labels:
        region: us-east-1
        cloud: aws
    - address: ${GCP_GKE_ENDPOINT}
      network: gcp-network
      ports:
        grpc: 9090
      labels:
        region: us-central1
        cloud: gcp
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-multicloud
  namespace: production
spec:
  host: payment-service.production.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      http:
        http2MaxRequests: 1000
      tcp:
        maxConnections: 100
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 60s
  subsets:
    - name: aws
      labels:
        cloud: aws
    - name: gcp
      labels:
        cloud: gcp
EOF

  log "✓ Istio multi-cluster service mesh configured"
}

# ─── 4. Multi-Cloud Observability ─────────────────────────────────────────────

setup_multicloud_observability() {
  log "=== Setting up Multi-Cloud Observability ==="

  # Global Prometheus Federation
  cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: global-prometheus
  namespace: monitoring
spec:
  replicas: 2
  retention: 90d
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 500Gi
  additionalScrapeConfigs:
    name: additional-scrape-configs
    key: prometheus-additional.yaml
  remoteRead:
    - url: http://aws-thanos-query:9090/api/v1/read
      name: aws-production
      requiredMatchers:
        cloud: aws
    - url: http://gcp-thanos-query:9090/api/v1/read
      name: gcp-production
      requiredMatchers:
        cloud: gcp
    - url: http://azure-thanos-query:9090/api/v1/read
      name: azure-production
      requiredMatchers:
        cloud: azure
EOF

  # Multi-Cloud Cost Optimization Dashboard
  cat <<'PYTHON' > /tmp/multicloud-cost-analyzer.py
#!/usr/bin/env python3
"""Multi-Cloud Cost Analyzer — compares resource costs across AWS/Azure/GCP."""

import json
import boto3
from datetime import datetime, timedelta

class MultiCloudCostAnalyzer:
    def __init__(self):
        self.aws_client = boto3.client('ce', region_name='us-east-1')
        # Azure and GCP clients initialized similarly via SDKs

    def get_aws_costs(self, days: int = 30) -> dict:
        end = datetime.now().strftime('%Y-%m-%d')
        start = (datetime.now() - timedelta(days=days)).strftime('%Y-%m-%d')

        response = self.aws_client.get_cost_and_usage(
            TimePeriod={'Start': start, 'End': end},
            Granularity='MONTHLY',
            Metrics=['BlendedCost', 'UnblendedCost'],
            GroupBy=[
                {'Type': 'DIMENSION', 'Key': 'SERVICE'},
                {'Type': 'TAG', 'Key': 'Environment'},
            ]
        )

        costs = {}
        for result in response['ResultsByTime']:
            for group in result['Groups']:
                service = group['Keys'][0]
                env = group['Keys'][1]
                cost = float(group['Metrics']['BlendedCost']['Amount'])
                costs[f"{service}/{env}"] = cost

        return costs

    def calculate_savings_opportunities(self) -> list:
        opportunities = []

        # Check for underutilized resources
        ec2 = boto3.client('ec2')
        cw = boto3.client('cloudwatch')

        instances = ec2.describe_instances(
            Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
        )['Reservations']

        for reservation in instances:
            for instance in reservation['Instances']:
                instance_id = instance['InstanceId']

                # Get CPU utilization
                metrics = cw.get_metric_statistics(
                    Namespace='AWS/EC2',
                    MetricName='CPUUtilization',
                    Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                    StartTime=datetime.now() - timedelta(days=14),
                    EndTime=datetime.now(),
                    Period=3600,
                    Statistics=['Average']
                )

                if metrics['Datapoints']:
                    avg_cpu = sum(d['Average'] for d in metrics['Datapoints']) / len(metrics['Datapoints'])
                    if avg_cpu < 10:
                        opportunities.append({
                            'resource': instance_id,
                            'type': 'underutilized-ec2',
                            'avg_cpu': avg_cpu,
                            'recommendation': 'Downsize or terminate instance',
                            'estimated_savings': self._estimate_savings(instance)
                        })

        return opportunities

    def _estimate_savings(self, instance: dict) -> float:
        # Simplified pricing lookup
        pricing = {
            'm5.xlarge': 0.192,
            'm5.2xlarge': 0.384,
            'm5.4xlarge': 0.768,
            'r5.xlarge': 0.252,
        }
        instance_type = instance.get('InstanceType', 'm5.xlarge')
        hourly_cost = pricing.get(instance_type, 0.10)
        return hourly_cost * 24 * 30 * 0.5  # 50% savings estimate

    def generate_report(self) -> dict:
        aws_costs = self.get_aws_costs()
        savings_opportunities = self.calculate_savings_opportunities()

        total_potential_savings = sum(o['estimated_savings'] for o in savings_opportunities)

        return {
            'timestamp': datetime.now().isoformat(),
            'aws_monthly_costs': aws_costs,
            'savings_opportunities': savings_opportunities,
            'total_potential_savings_monthly': f"${total_potential_savings:.2f}",
            'recommendations': [
                "Enable AWS Savings Plans for 20-30% discount",
                "Use Spot instances for non-critical workloads",
                "Enable S3 Intelligent-Tiering",
                "Right-size underutilized instances",
                "Delete unattached EBS volumes",
            ]
        }

if __name__ == '__main__':
    analyzer = MultiCloudCostAnalyzer()
    report = analyzer.generate_report()
    print(json.dumps(report, indent=2))
PYTHON

  log "✓ Multi-cloud observability configured"
}

# ─── Main ──────────────────────────────────────────────────────────────────────

main() {
  log "Starting Multi-Cloud Federation setup..."
  setup_crossplane_multicloud
  create_multicloud_xrd
  setup_multicluster_istio
  setup_multicloud_observability
  log "✓ Multi-Cloud Federation complete"
}

main "$@"
```

---

## ขั้นตอนที่ 578: Edge Computing Architecture

### Edge Computing Stack

```
Edge Computing Architecture:
┌─────────────────────────────────────────────┐
│              Cloud (Central)                  │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│    │  EKS    │  │  GKE    │  │  AKS    │    │
│    └────┬────┘  └────┬────┘  └────┬────┘    │
└─────────┼────────────┼────────────┼──────────┘
          │            │            │
     ┌────┴────┐  ┌────┴────┐  ┌────┴────┐
     │Regional │  │Regional │  │Regional │
     │  Edge   │  │  Edge   │  │  Edge   │
     │  (K3s)  │  │  (K3s)  │  │  (K3s)  │
     └────┬────┘  └─────────┘  └─────────┘
          │
    ┌─────┴──────────────────────────┐
    │         Far Edge               │
    │  ┌──────────┐  ┌──────────┐   │
    │  │ MicroVM  │  │ WASM RT  │   │
    │  │(Firecracker)│(Wasmtime)│   │
    │  └──────────┘  └──────────┘   │
    └────────────────────────────────┘
```

### `edge-computing.sh`

```bash
#!/usr/bin/env bash
# edge-computing.sh — K3s Edge Clusters and Cloudflare Workers
set -euo pipefail

LOG_FILE="/var/log/edge-computing.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. K3s Edge Cluster Setup ───────────────────────────────────────────────

setup_k3s_edge_cluster() {
  local CLUSTER_NAME="${1:-edge-cluster-01}"
  local SERVER_IP="${2:-192.168.1.100}"

  log "=== Setting up K3s Edge Cluster: $CLUSTER_NAME ==="

  # Install K3s server (minimal footprint)
  curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
    --cluster-init \
    --disable traefik \
    --disable servicelb \
    --disable coredns \
    --disable local-storage \
    --kubelet-arg='kube-reserved=cpu=200m,memory=200Mi' \
    --kubelet-arg='system-reserved=cpu=200m,memory=200Mi' \
    --kubelet-arg='eviction-hard=memory.available<100Mi' \
    --tls-san ${SERVER_IP} \
    --node-label='node.kubernetes.io/edge=true' \
    --node-label='topology.kubernetes.io/zone=edge-us-east-1a'" \
  sh -

  # Get join token
  K3S_TOKEN=$(cat /var/lib/rancher/k3s/server/node-token)
  export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

  # Install lightweight CNI (Flannel replacement with Cilium)
  helm repo add cilium https://helm.cilium.io/
  helm upgrade --install cilium cilium/cilium \
    --namespace kube-system \
    --set k8sServiceHost="${SERVER_IP}" \
    --set k8sServicePort=6443 \
    --set image.pullPolicy=IfNotPresent \
    --set ipam.mode=cluster-pool \
    --set ipam.operator.clusterPoolIPv4PodCIDR="10.42.0.0/16" \
    --set tunnel=disabled \
    --set nativeRoutingCIDR="192.168.0.0/16" \
    --set resources.requests.cpu=50m \
    --set resources.requests.memory=64Mi

  log "✓ K3s edge cluster $CLUSTER_NAME initialized"
  echo "Join token: $K3S_TOKEN"
}

# ─── 2. Edge Workload Deployment ──────────────────────────────────────────────

deploy_edge_workloads() {
  log "=== Deploying Edge Workloads ==="

  # Edge AI Inference Service
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edge-ai-inference
  namespace: edge-workloads
  labels:
    app: edge-ai-inference
    edge.tier: far-edge
spec:
  replicas: 1
  selector:
    matchLabels:
      app: edge-ai-inference
  template:
    metadata:
      labels:
        app: edge-ai-inference
    spec:
      nodeSelector:
        node.kubernetes.io/edge: "true"
      tolerations:
        - key: node.kubernetes.io/edge
          operator: Exists
          effect: NoSchedule
      containers:
        - name: inference
          image: ghcr.io/ollama/ollama:0.1.26
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 4Gi
          ports:
            - containerPort: 11434
          env:
            - name: OLLAMA_MODELS
              value: /models
            - name: OLLAMA_NUM_PARALLEL
              value: "2"
          volumeMounts:
            - name: models
              mountPath: /models
      initContainers:
        - name: model-downloader
          image: curlimages/curl:8.4.0
          command:
            - sh
            - -c
            - |
              curl -L https://ollama.ai/api/pull \
                -d '{"name":"phi3:mini"}' \
                -H "Content-Type: application/json" \
                -o /dev/null
          volumeMounts:
            - name: models
              mountPath: /models
      volumes:
        - name: models
          persistentVolumeClaim:
            claimName: edge-models-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: edge-ai-inference
  namespace: edge-workloads
spec:
  selector:
    app: edge-ai-inference
  ports:
    - port: 11434
      targetPort: 11434
  type: ClusterIP
---
# Edge Cache Layer
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: edge-cache
  namespace: edge-workloads
spec:
  selector:
    matchLabels:
      app: edge-cache
  template:
    metadata:
      labels:
        app: edge-cache
    spec:
      nodeSelector:
        node.kubernetes.io/edge: "true"
      containers:
        - name: varnish
          image: varnish:7.4
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 1Gi
          ports:
            - containerPort: 6081
          volumeMounts:
            - name: vcl-config
              mountPath: /etc/varnish
          args:
            - -F
            - -f
            - /etc/varnish/default.vcl
            - -s
            - malloc,256m
      volumes:
        - name: vcl-config
          configMap:
            name: varnish-vcl
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: varnish-vcl
  namespace: edge-workloads
data:
  default.vcl: |
    vcl 4.1;

    backend default {
      .host = "origin-service";
      .port = "8080";
      .probe = {
        .url = "/health";
        .interval = 5s;
        .timeout = 2s;
        .window = 5;
        .threshold = 3;
      }
    }

    sub vcl_recv {
      if (req.method == "PURGE") {
        if (client.ip != "127.0.0.1") {
          return(synth(405, "Not allowed."));
        }
        return(purge);
      }
      if (req.url ~ "\.(css|js|png|jpg|gif|ico|woff2)$") {
        unset req.http.Cookie;
        return(hash);
      }
    }

    sub vcl_backend_response {
      if (bereq.url ~ "\.(css|js|png|jpg|gif|ico|woff2)$") {
        set beresp.ttl = 30d;
        set beresp.grace = 1d;
      } else {
        set beresp.ttl = 5m;
        set beresp.grace = 30s;
      }
    }

    sub vcl_deliver {
      if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
      } else {
        set resp.http.X-Cache = "MISS";
      }
    }
EOF

  log "✓ Edge workloads deployed"
}

# ─── 3. Cloudflare Workers Edge Functions ─────────────────────────────────────

setup_cloudflare_workers() {
  log "=== Setting up Cloudflare Workers ==="

  # Cloudflare Worker: Smart Router
  cat <<'EOF' > /tmp/edge-smart-router.js
// edge-smart-router.js — Cloudflare Worker for intelligent routing
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const cf = request.cf;

    // ── Geo-based routing ─────────────────────────────────────────────────────
    const region = cf?.region || 'US';
    const continent = cf?.continent || 'NA';

    let backendUrl;
    switch (continent) {
      case 'EU':
        backendUrl = env.EU_BACKEND;
        break;
      case 'AS':
        backendUrl = env.APAC_BACKEND;
        break;
      default:
        backendUrl = env.US_BACKEND;
    }

    // ── A/B Testing (10% traffic to canary) ──────────────────────────────────
    const abCookie = request.headers.get('Cookie')?.match(/ab_group=(\w+)/)?.[1];
    let group = abCookie;

    if (!group) {
      group = Math.random() < 0.1 ? 'canary' : 'stable';
    }

    if (group === 'canary') {
      backendUrl = env.CANARY_BACKEND;
    }

    // ── Cache check ──────────────────────────────────────────────────────────
    const cacheKey = new Request(url.toString(), request);
    const cache = caches.default;
    let response = await cache.match(cacheKey);

    if (!response) {
      // ── Rate limiting ────────────────────────────────────────────────────
      const ip = request.headers.get('CF-Connecting-IP');
      const rateLimitKey = `rate:${ip}`;
      const { success } = await env.RATE_LIMITER.limit({ key: rateLimitKey });

      if (!success) {
        return new Response('Too Many Requests', {
          status: 429,
          headers: {
            'Retry-After': '60',
            'X-RateLimit-Limit': '100',
          },
        });
      }

      // ── Fetch from backend ───────────────────────────────────────────────
      const backendRequest = new Request(backendUrl + url.pathname + url.search, {
        method: request.method,
        headers: request.headers,
        body: request.body,
      });

      response = await fetch(backendRequest);

      // ── Cache successful GET responses ───────────────────────────────────
      if (request.method === 'GET' && response.status === 200) {
        const cloned = response.clone();
        ctx.waitUntil(cache.put(cacheKey, cloned));
      }
    }

    // ── Add edge headers ─────────────────────────────────────────────────────
    const newResponse = new Response(response.body, response);
    newResponse.headers.set('X-Edge-Region', region);
    newResponse.headers.set('X-AB-Group', group);
    newResponse.headers.set('X-Served-By', 'cloudflare-edge');

    // ── Set AB cookie if not present ──────────────────────────────────────────
    if (!abCookie) {
      newResponse.headers.append('Set-Cookie',
        `ab_group=${group}; Path=/; Max-Age=86400; SameSite=Lax`);
    }

    return newResponse;
  }
};
EOF

  # Cloudflare Worker: Edge Authentication
  cat <<'EOF' > /tmp/edge-auth.js
// edge-auth.js — JWT verification at the edge
import { verify } from '@tsndr/cloudflare-worker-jwt';

export default {
  async fetch(request, env) {
    const authHeader = request.headers.get('Authorization');

    if (!authHeader?.startsWith('Bearer ')) {
      return new Response(JSON.stringify({ error: 'Missing authorization' }), {
        status: 401,
        headers: { 'Content-Type': 'application/json' },
      });
    }

    const token = authHeader.slice(7);

    try {
      const isValid = await verify(token, env.JWT_SECRET);
      if (!isValid) {
        return new Response(JSON.stringify({ error: 'Invalid token' }), {
          status: 401,
          headers: { 'Content-Type': 'application/json' },
        });
      }

      // Decode and pass user info to origin
      const [, payload] = token.split('.');
      const decoded = JSON.parse(atob(payload));

      const newRequest = new Request(request, {
        headers: new Headers({
          ...Object.fromEntries(request.headers),
          'X-User-ID': decoded.sub,
          'X-User-Role': decoded.role,
          'X-User-Email': decoded.email,
        }),
      });

      return fetch(newRequest);

    } catch (err) {
      return new Response(JSON.stringify({ error: 'Token verification failed' }), {
        status: 401,
        headers: { 'Content-Type': 'application/json' },
      });
    }
  }
};
EOF

  # Wrangler config for deployment
  cat <<'EOF' > /tmp/wrangler.toml
name = "edge-platform"
main = "src/index.js"
compatibility_date = "2024-01-01"

[vars]
ENVIRONMENT = "production"

[[kv_namespaces]]
binding = "CACHE"
id = "your-kv-namespace-id"

[[r2_buckets]]
binding = "ASSETS"
bucket_name = "edge-assets"

[rate_limiting]
[[rate_limiting.rules]]
namespace = "RATE_LIMITER"
simple = { limit = 100, period = 60 }

[env.production]
workers_dev = false
route = { pattern = "api.example.com/*", zone_name = "example.com" }
EOF

  log "✓ Cloudflare Workers edge functions configured"
}

# ─── 4. AWS Outposts Edge Deployment ─────────────────────────────────────────

setup_aws_outposts() {
  log "=== Setting up AWS Outposts Edge ==="

  # EKS on Outposts Node Group
  cat <<'EOF' > /tmp/outposts-nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: outposts-cluster
  region: us-east-1

outpost:
  controlPlaneOutpostARN: arn:aws:outposts:us-east-1:123456789012:outpost/op-0123456789abcdef0
  controlPlaneInstanceType: m5.xlarge

managedNodeGroups:
  - name: outposts-workers
    instanceType: m5.2xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    outpostARN: arn:aws:outposts:us-east-1:123456789012:outpost/op-0123456789abcdef0
    privateNetworking: true
    labels:
      node.kubernetes.io/outpost: "true"
      topology.kubernetes.io/zone: us-east-1-nyc-1a
    taints:
      - key: outpost
        value: "true"
        effect: NoSchedule
    volumeSize: 100
    volumeType: gp3
    iam:
      withAddonPolicies:
        ebs: true
        efs: true
        fsx: true

addons:
  - name: vpc-cni
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    configurationValues: |
      node:
        tolerations:
          - key: outpost
            operator: Equal
            value: "true"
            effect: NoSchedule
EOF

  # Local Zone Storage Class (gp2 for Outposts)
  cat <<'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: outposts-local-storage
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  throughput: "500"
  iops: "3000"
  encrypted: "true"
allowVolumeExpansion: true
allowedTopologies:
  - matchLabelExpressions:
      - key: topology.kubernetes.io/zone
        values:
          - us-east-1-nyc-1a
EOF

  log "✓ AWS Outposts edge deployment configured"
}

# ─── 5. Edge Observability (Lightweight) ─────────────────────────────────────

setup_edge_observability() {
  log "=== Setting up Edge Observability ==="

  # Lightweight Prometheus for Edge (VictoriaMetrics Agent)
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: vmagent
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: vmagent
  template:
    metadata:
      labels:
        app: vmagent
    spec:
      nodeSelector:
        node.kubernetes.io/edge: "true"
      containers:
        - name: vmagent
          image: victoriametrics/vmagent:v1.95.1
          args:
            - -promscrape.config=/config/scrape.yaml
            - -remoteWrite.url=https://central-metrics.example.com/api/v1/write
            - -remoteWrite.basicAuth.username=vmagent
            - -remoteWrite.basicAuth.passwordFile=/secrets/password
            - -remoteWrite.queues=4
            - -remoteWrite.maxBlockSize=8MB
            - -memory.allowedPercent=60
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
          volumeMounts:
            - name: config
              mountPath: /config
            - name: secrets
              mountPath: /secrets
      volumes:
        - name: config
          configMap:
            name: vmagent-config
        - name: secrets
          secret:
            secretName: vmagent-secrets
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: vmagent-config
  namespace: monitoring
data:
  scrape.yaml: |
    global:
      scrape_interval: 30s

    scrape_configs:
      - job_name: edge-ai-inference
        static_configs:
          - targets: ['edge-ai-inference:11434']
        metrics_path: /metrics

      - job_name: edge-cache
        static_configs:
          - targets: ['edge-cache:6082']

      - job_name: node-exporter
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - source_labels: [__address__]
            regex: (.+):(.+)
            target_label: __address__
            replacement: ${1}:9100
EOF

  log "✓ Edge observability configured"
}

# ─── Main ──────────────────────────────────────────────────────────────────────

main() {
  log "Starting Edge Computing setup..."
  setup_k3s_edge_cluster "edge-us-east-01" "10.0.1.100"
  deploy_edge_workloads
  setup_cloudflare_workers
  setup_aws_outposts
  setup_edge_observability
  log "✓ Edge Computing stack complete"
}

main "$@"
```

---

## ขั้นตอนที่ 579: Kubernetes Operators — สร้าง Custom Operators

### Operator Pattern

```
Kubernetes Operator Pattern:
┌─────────────────────────────────────────┐
│           Kubernetes API Server          │
│                                          │
│  Custom Resource: DatabaseCluster        │
│  ┌──────────────────────────────────┐   │
│  │  name: prod-postgres              │   │
│  │  spec:                            │   │
│  │    replicas: 3                    │   │
│  │    version: "15"                  │   │
│  │    storage: 100Gi                 │   │
│  └──────────────────────────────────┘   │
└──────────────┬──────────────────────────┘
               │ Watch
               ▼
┌─────────────────────────────────────────┐
│         Custom Controller Loop           │
│                                          │
│  1. Observe current state                │
│  2. Compare with desired state           │
│  3. Take action to reconcile             │
│  4. Update status                        │
└──────────────┬──────────────────────────┘
               │ Create/Update/Delete
               ▼
┌─────────────────────────────────────────┐
│      Managed Resources                   │
│  StatefulSet + Services + PVCs +         │
│  ConfigMaps + Secrets + Ingress          │
└─────────────────────────────────────────┘
```

### `custom-operator.sh`

```bash
#!/usr/bin/env bash
# custom-operator.sh — Build and deploy a production-grade Kubernetes Operator
set -euo pipefail

LOG_FILE="/var/log/custom-operator.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Operator SDK Project Setup ────────────────────────────────────────────

setup_operator_project() {
  log "=== Setting up Operator SDK Project ==="

  # Install Operator SDK
  OPERATOR_SDK_VERSION="v1.33.0"
  curl -LO "https://github.com/operator-framework/operator-sdk/releases/download/${OPERATOR_SDK_VERSION}/operator-sdk_linux_amd64"
  chmod +x operator-sdk_linux_amd64
  mv operator-sdk_linux_amd64 /usr/local/bin/operator-sdk

  # Initialize project
  mkdir -p /workspace/database-operator
  cd /workspace/database-operator

  operator-sdk init \
    --domain=platform.example.com \
    --repo=github.com/example/database-operator \
    --project-name=database-operator

  # Create API
  operator-sdk create api \
    --group=database \
    --version=v1alpha1 \
    --kind=DatabaseCluster \
    --resource=true \
    --controller=true

  log "✓ Operator project initialized"
}

# ─── 2. CRD Type Definition ───────────────────────────────────────────────────

write_crd_types() {
  log "=== Writing CRD Type Definitions ==="

  cat <<'GOCODE' > /workspace/database-operator/api/v1alpha1/databasecluster_types.go
package v1alpha1

import (
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/api/resource"
)

// DatabaseClusterSpec defines the desired state of DatabaseCluster
type DatabaseClusterSpec struct {
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=9
	Replicas int32 `json:"replicas"`

	// +kubebuilder:validation:Enum=postgresql;mysql;redis
	Engine string `json:"engine"`

	// +kubebuilder:validation:Pattern=`^\d+\.\d+$`
	Version string `json:"version"`

	// +kubebuilder:default="10Gi"
	Storage resource.Quantity `json:"storage,omitempty"`

	// +kubebuilder:default="gp3"
	StorageClass string `json:"storageClass,omitempty"`

	// +kubebuilder:validation:Enum=small;medium;large;xlarge
	// +kubebuilder:default=medium
	Size string `json:"size,omitempty"`

	Backup *BackupSpec `json:"backup,omitempty"`

	HighAvailability *HighAvailabilitySpec `json:"highAvailability,omitempty"`

	Resources corev1.ResourceRequirements `json:"resources,omitempty"`

	Parameters map[string]string `json:"parameters,omitempty"`
}

type BackupSpec struct {
	Enabled  bool   `json:"enabled"`
	Schedule string `json:"schedule,omitempty"`
	RetentionDays int32 `json:"retentionDays,omitempty"`
	S3Bucket string `json:"s3Bucket,omitempty"`
}

type HighAvailabilitySpec struct {
	Enabled          bool   `json:"enabled"`
	ReplicationMode  string `json:"replicationMode,omitempty"`
}

// DatabaseClusterStatus defines the observed state of DatabaseCluster
type DatabaseClusterStatus struct {
	// +kubebuilder:validation:Enum=Pending;Initializing;Running;Degraded;Failed
	Phase string `json:"phase,omitempty"`

	ReadyReplicas   int32  `json:"readyReplicas,omitempty"`
	CurrentReplicas int32  `json:"currentReplicas,omitempty"`
	PrimaryEndpoint string `json:"primaryEndpoint,omitempty"`
	ReplicaEndpoint string `json:"replicaEndpoint,omitempty"`

	Conditions []metav1.Condition `json:"conditions,omitempty"`

	LastBackupTime  *metav1.Time `json:"lastBackupTime,omitempty"`
	CurrentVersion  string       `json:"currentVersion,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas
// +kubebuilder:printcolumn:name="Engine",type=string,JSONPath=`.spec.engine`
// +kubebuilder:printcolumn:name="Version",type=string,JSONPath=`.spec.version`
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.status.readyReplicas`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
// +kubebuilder:printcolumn:name="Primary",type=string,JSONPath=`.status.primaryEndpoint`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
type DatabaseCluster struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   DatabaseClusterSpec   `json:"spec,omitempty"`
	Status DatabaseClusterStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type DatabaseClusterList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []DatabaseCluster `json:"items"`
}

func init() {
	SchemeBuilder.Register(&DatabaseCluster{}, &DatabaseClusterList{})
}
GOCODE

  log "✓ CRD types written"
}

# ─── 3. Controller Reconciler ─────────────────────────────────────────────────

write_controller() {
  log "=== Writing Controller Reconciler ==="

  cat <<'GOCODE' > /workspace/database-operator/controllers/databasecluster_controller.go
package controllers

import (
	"context"
	"fmt"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/log"

	databasev1alpha1 "github.com/example/database-operator/api/v1alpha1"
)

const (
	databaseClusterFinalizer = "database.platform.example.com/finalizer"
	requeueAfter             = 30 * time.Second
)

// DatabaseClusterReconciler reconciles a DatabaseCluster object
type DatabaseClusterReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=database.platform.example.com,resources=databaseclusters,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.platform.example.com,resources=databaseclusters/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=database.platform.example.com,resources=databaseclusters/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services;configmaps;secrets;persistentvolumeclaims,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch

func (r *DatabaseClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	// Fetch the DatabaseCluster
	cluster := &databasev1alpha1.DatabaseCluster{}
	if err := r.Get(ctx, req.NamespacedName, cluster); err != nil {
		if errors.IsNotFound(err) {
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, err
	}

	// Handle finalizer
	if cluster.DeletionTimestamp != nil {
		if controllerutil.ContainsFinalizer(cluster, databaseClusterFinalizer) {
			if err := r.finalize(ctx, cluster); err != nil {
				return ctrl.Result{}, err
			}
			controllerutil.RemoveFinalizer(cluster, databaseClusterFinalizer)
			if err := r.Update(ctx, cluster); err != nil {
				return ctrl.Result{}, err
			}
		}
		return ctrl.Result{}, nil
	}

	// Add finalizer
	if !controllerutil.ContainsFinalizer(cluster, databaseClusterFinalizer) {
		controllerutil.AddFinalizer(cluster, databaseClusterFinalizer)
		if err := r.Update(ctx, cluster); err != nil {
			return ctrl.Result{}, err
		}
	}

	// Reconcile all sub-resources
	if err := r.reconcileConfigMap(ctx, cluster); err != nil {
		return ctrl.Result{}, r.setFailedCondition(ctx, cluster, "ConfigMapReconcileFailed", err)
	}

	if err := r.reconcileSecret(ctx, cluster); err != nil {
		return ctrl.Result{}, r.setFailedCondition(ctx, cluster, "SecretReconcileFailed", err)
	}

	if err := r.reconcileStatefulSet(ctx, cluster); err != nil {
		return ctrl.Result{}, r.setFailedCondition(ctx, cluster, "StatefulSetReconcileFailed", err)
	}

	if err := r.reconcileServices(ctx, cluster); err != nil {
		return ctrl.Result{}, r.setFailedCondition(ctx, cluster, "ServiceReconcileFailed", err)
	}

	// Update status
	if err := r.updateStatus(ctx, cluster); err != nil {
		logger.Error(err, "Failed to update status")
		return ctrl.Result{RequeueAfter: requeueAfter}, nil
	}

	return ctrl.Result{RequeueAfter: requeueAfter}, nil
}

func (r *DatabaseClusterReconciler) reconcileStatefulSet(ctx context.Context, cluster *databasev1alpha1.DatabaseCluster) error {
	labels := map[string]string{
		"app.kubernetes.io/name":       cluster.Name,
		"app.kubernetes.io/managed-by": "database-operator",
		"database.platform.example.com/cluster": cluster.Name,
	}

	storageSize := cluster.Spec.Storage
	if storageSize.IsZero() {
		storageSize = resource.MustParse("10Gi")
	}

	sts := &appsv1.StatefulSet{
		ObjectMeta: metav1.ObjectMeta{
			Name:      cluster.Name,
			Namespace: cluster.Namespace,
			Labels:    labels,
		},
		Spec: appsv1.StatefulSetSpec{
			Replicas:    &cluster.Spec.Replicas,
			ServiceName: cluster.Name + "-headless",
			Selector: &metav1.LabelSelector{
				MatchLabels: labels,
			},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: labels,
				},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{
						r.buildDatabaseContainer(cluster),
					},
					InitContainers: []corev1.Container{
						r.buildInitContainer(cluster),
					},
					Affinity: &corev1.Affinity{
						PodAntiAffinity: &corev1.PodAntiAffinity{
							PreferredDuringSchedulingIgnoredDuringExecution: []corev1.WeightedPodAffinityTerm{
								{
									Weight: 100,
									PodAffinityTerm: corev1.PodAffinityTerm{
										LabelSelector: &metav1.LabelSelector{
											MatchLabels: labels,
										},
										TopologyKey: "kubernetes.io/hostname",
									},
								},
							},
						},
					},
				},
			},
			VolumeClaimTemplates: []corev1.PersistentVolumeClaim{
				{
					ObjectMeta: metav1.ObjectMeta{
						Name: "data",
					},
					Spec: corev1.PersistentVolumeClaimSpec{
						AccessModes: []corev1.PersistentVolumeAccessMode{corev1.ReadWriteOnce},
						Resources: corev1.VolumeResourceRequirements{
							Requests: corev1.ResourceList{
								corev1.ResourceStorage: storageSize,
							},
						},
						StorageClassName: &cluster.Spec.StorageClass,
					},
				},
			},
		},
	}

	// Set owner reference for garbage collection
	if err := controllerutil.SetControllerReference(cluster, sts, r.Scheme); err != nil {
		return err
	}

	// Create or update
	existing := &appsv1.StatefulSet{}
	err := r.Get(ctx, types.NamespacedName{Name: sts.Name, Namespace: sts.Namespace}, existing)
	if errors.IsNotFound(err) {
		return r.Create(ctx, sts)
	}
	if err != nil {
		return err
	}

	// Update replica count and image
	existing.Spec.Replicas = sts.Spec.Replicas
	existing.Spec.Template.Spec.Containers = sts.Spec.Template.Spec.Containers
	return r.Update(ctx, existing)
}

func (r *DatabaseClusterReconciler) buildDatabaseContainer(cluster *databasev1alpha1.DatabaseCluster) corev1.Container {
	var image string
	switch cluster.Spec.Engine {
	case "postgresql":
		image = fmt.Sprintf("postgres:%s-alpine", cluster.Spec.Version)
	case "mysql":
		image = fmt.Sprintf("mysql:%s", cluster.Spec.Version)
	case "redis":
		image = fmt.Sprintf("redis:%s-alpine", cluster.Spec.Version)
	default:
		image = "postgres:15-alpine"
	}

	resources := cluster.Spec.Resources
	if resources.Requests == nil {
		sizeMap := map[string]corev1.ResourceList{
			"small":  {corev1.ResourceCPU: resource.MustParse("250m"), corev1.ResourceMemory: resource.MustParse("256Mi")},
			"medium": {corev1.ResourceCPU: resource.MustParse("500m"), corev1.ResourceMemory: resource.MustParse("1Gi")},
			"large":  {corev1.ResourceCPU: resource.MustParse("2"), corev1.ResourceMemory: resource.MustParse("4Gi")},
			"xlarge": {corev1.ResourceCPU: resource.MustParse("4"), corev1.ResourceMemory: resource.MustParse("8Gi")},
		}
		if req, ok := sizeMap[cluster.Spec.Size]; ok {
			resources.Requests = req
			resources.Limits = req
		}
	}

	return corev1.Container{
		Name:      cluster.Spec.Engine,
		Image:     image,
		Resources: resources,
		Ports: []corev1.ContainerPort{
			{ContainerPort: 5432, Name: "db"},
		},
		EnvFrom: []corev1.EnvFromSource{
			{SecretRef: &corev1.SecretEnvSource{LocalObjectReference: corev1.LocalObjectReference{Name: cluster.Name + "-credentials"}}},
		},
		VolumeMounts: []corev1.VolumeMount{
			{Name: "data", MountPath: "/var/lib/postgresql/data"},
			{Name: "config", MountPath: "/etc/postgresql/postgresql.conf", SubPath: "postgresql.conf"},
		},
		ReadinessProbe: &corev1.Probe{
			ProbeHandler: corev1.ProbeHandler{
				Exec: &corev1.ExecAction{
					Command: []string{"pg_isready", "-U", "postgres"},
				},
			},
			InitialDelaySeconds: 10,
			PeriodSeconds:       10,
			FailureThreshold:    6,
		},
		LivenessProbe: &corev1.Probe{
			ProbeHandler: corev1.ProbeHandler{
				Exec: &corev1.ExecAction{
					Command: []string{"pg_isready", "-U", "postgres"},
				},
			},
			InitialDelaySeconds: 30,
			PeriodSeconds:       30,
		},
	}
}

func (r *DatabaseClusterReconciler) buildInitContainer(cluster *databasev1alpha1.DatabaseCluster) corev1.Container {
	return corev1.Container{
		Name:  "init-permissions",
		Image: "busybox:1.36",
		Command: []string{
			"sh", "-c",
			"chown -R 999:999 /var/lib/postgresql/data && chmod 700 /var/lib/postgresql/data",
		},
		VolumeMounts: []corev1.VolumeMount{
			{Name: "data", MountPath: "/var/lib/postgresql/data"},
		},
		SecurityContext: &corev1.SecurityContext{
			RunAsUser: func() *int64 { v := int64(0); return &v }(),
		},
	}
}

func (r *DatabaseClusterReconciler) updateStatus(ctx context.Context, cluster *databasev1alpha1.DatabaseCluster) error {
	sts := &appsv1.StatefulSet{}
	if err := r.Get(ctx, types.NamespacedName{Name: cluster.Name, Namespace: cluster.Namespace}, sts); err != nil {
		return err
	}

	patch := client.MergeFrom(cluster.DeepCopy())

	cluster.Status.ReadyReplicas = sts.Status.ReadyReplicas
	cluster.Status.CurrentReplicas = sts.Status.CurrentReplicas
	cluster.Status.PrimaryEndpoint = fmt.Sprintf("%s.%s.svc.cluster.local:5432", cluster.Name, cluster.Namespace)
	cluster.Status.ReplicaEndpoint = fmt.Sprintf("%s-headless.%s.svc.cluster.local:5432", cluster.Name, cluster.Namespace)

	if sts.Status.ReadyReplicas == cluster.Spec.Replicas {
		cluster.Status.Phase = "Running"
	} else if sts.Status.ReadyReplicas > 0 {
		cluster.Status.Phase = "Degraded"
	} else {
		cluster.Status.Phase = "Initializing"
	}

	return r.Status().Patch(ctx, cluster, patch)
}

func (r *DatabaseClusterReconciler) finalize(ctx context.Context, cluster *databasev1alpha1.DatabaseCluster) error {
	log.FromContext(ctx).Info("Finalizing DatabaseCluster", "name", cluster.Name)
	// Perform cleanup: create final backup, delete external resources
	return nil
}

func (r *DatabaseClusterReconciler) setFailedCondition(ctx context.Context, cluster *databasev1alpha1.DatabaseCluster, reason string, err error) error {
	patch := client.MergeFrom(cluster.DeepCopy())
	cluster.Status.Phase = "Failed"
	r.Status().Patch(ctx, cluster, patch) //nolint
	return err
}

func (r *DatabaseClusterReconciler) reconcileConfigMap(ctx context.Context, cluster *databasev1alpha1.DatabaseCluster) error {
	// Build PostgreSQL config from cluster.Spec.Parameters
	return nil
}

func (r *DatabaseClusterReconciler) reconcileSecret(ctx context.Context, cluster *databasev1alpha1.DatabaseCluster) error {
	// Generate/rotate database credentials
	return nil
}

func (r *DatabaseClusterReconciler) reconcileServices(ctx context.Context, cluster *databasev1alpha1.DatabaseCluster) error {
	// Create primary + headless services
	return nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *DatabaseClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&databasev1alpha1.DatabaseCluster{}).
		Owns(&appsv1.StatefulSet{}).
		Owns(&corev1.Service{}).
		Owns(&corev1.ConfigMap{}).
		Complete(r)
}
GOCODE

  log "✓ Controller reconciler written"
}

# ─── 4. Operator Build and Deploy ─────────────────────────────────────────────

build_and_deploy_operator() {
  log "=== Building and Deploying Operator ==="

  cd /workspace/database-operator

  # Generate CRD manifests
  make generate
  make manifests

  # Build operator image
  make docker-build IMG=ghcr.io/example/database-operator:v0.1.0
  make docker-push IMG=ghcr.io/example/database-operator:v0.1.0

  # Deploy to cluster
  make deploy IMG=ghcr.io/example/database-operator:v0.1.0

  # Verify deployment
  kubectl wait --for=condition=available \
    deployment/database-operator-controller-manager \
    -n database-operator-system \
    --timeout=120s

  log "✓ Operator deployed"
}

# ─── 5. Test DatabaseCluster CR ───────────────────────────────────────────────

test_database_operator() {
  log "=== Testing DatabaseCluster Operator ==="

  # Create test DatabaseCluster
  cat <<'EOF' | kubectl apply -f -
apiVersion: database.platform.example.com/v1alpha1
kind: DatabaseCluster
metadata:
  name: prod-postgres
  namespace: default
spec:
  replicas: 3
  engine: postgresql
  version: "15"
  size: medium
  storage: "50Gi"
  storageClass: gp3
  backup:
    enabled: true
    schedule: "0 2 * * *"
    retentionDays: 30
    s3Bucket: "my-db-backups"
  highAvailability:
    enabled: true
    replicationMode: streaming
  parameters:
    shared_buffers: "256MB"
    max_connections: "200"
    work_mem: "16MB"
EOF

  # Wait for Ready
  kubectl wait --for=jsonpath='{.status.phase}'=Running \
    databasecluster/prod-postgres \
    --timeout=300s

  # Check status
  kubectl get databasecluster prod-postgres -o wide
  kubectl describe databasecluster prod-postgres

  log "✓ DatabaseCluster operator test complete"
}

# ─── Main ──────────────────────────────────────────────────────────────────────

main() {
  log "Building Custom Kubernetes Operator..."
  setup_operator_project
  write_crd_types
  write_controller
  build_and_deploy_operator
  test_database_operator
  log "✓ Custom Operator complete"
}

main "$@"
```

---

## ขั้นตอนที่ 580: Data Engineering at Scale

### Modern Data Stack Architecture

```
Modern Data Stack at Scale:
┌─────────────────────────────────────────────────────────────┐
│                     Data Sources                             │
│  PostgreSQL │ MongoDB │ Kafka │ S3 │ REST APIs │ IoT Sensors │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Ingestion Layer                           │
│         Apache Kafka (MSK) + Debezium CDC                    │
│              Airbyte (Batch Connectors)                       │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                   Processing Layer                            │
│    Apache Flink (Streaming) │ Apache Spark (Batch)           │
│          dbt (SQL Transformations)                            │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Storage Layer                              │
│  Data Lakehouse (Apache Iceberg on S3) │ ClickHouse (OLAP)   │
│              Apache Hudi (Upserts)                            │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                  Serving Layer                                │
│   Trino (SQL Engine) │ Superset (BI) │ FastAPI (Data APIs)   │
└─────────────────────────────────────────────────────────────┘
```

### `data-engineering-platform.sh`

```bash
#!/usr/bin/env bash
# data-engineering-platform.sh — Modern Data Stack on Kubernetes
set -euo pipefail

LOG_FILE="/var/log/data-engineering.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Apache Flink Streaming Platform ──────────────────────────────────────

setup_flink_streaming() {
  log "=== Setting up Apache Flink Streaming ==="

  helm repo add flink-operator-helm https://downloads.apache.org/flink/flink-kubernetes-operator-1.7.0/
  helm upgrade --install flink-kubernetes-operator \
    flink-operator-helm/flink-kubernetes-operator \
    --namespace flink-system \
    --create-namespace \
    --set webhook.create=true

  # Flink Application: Real-time CDC Processing
  cat <<'EOF' | kubectl apply -f -
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: cdc-streaming-job
  namespace: data-engineering
spec:
  image: flink:1.18-java17
  flinkVersion: v1_18
  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "4"
    parallelism.default: "8"
    state.backend: rocksdb
    state.backend.incremental: "true"
    state.checkpoints.dir: s3://data-lakehouse/checkpoints
    state.savepoints.dir: s3://data-lakehouse/savepoints
    execution.checkpointing.interval: "60000"
    execution.checkpointing.mode: EXACTLY_ONCE
    execution.checkpointing.min-pause: "30000"
    execution.checkpointing.timeout: "120000"
    restart-strategy: exponential-delay
    restart-strategy.exponential-delay.initial-backoff: "1 s"
    restart-strategy.exponential-delay.max-backoff: "5 min"
    metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter
    metrics.reporter.prom.port: "9249"
  serviceAccount: flink-service-account
  jobManager:
    resource:
      memory: "2Gi"
      cpu: 1
    replicas: 2  # HA mode
  taskManager:
    resource:
      memory: "8Gi"
      cpu: 4
    replicas: 8
  job:
    jarURI: s3://data-lakehouse/jars/cdc-streaming-job.jar
    parallelism: 8
    upgradeMode: savepoint
    args:
      - --source-kafka-bootstrap=kafka-cluster:9092
      - --source-topics=postgres-cdc.*
      - --sink-iceberg-catalog=glue
      - --sink-iceberg-database=data_lakehouse
      - --s3-bucket=data-lakehouse
EOF

  log "✓ Apache Flink streaming configured"
}

# ─── 2. Apache Iceberg Data Lakehouse ─────────────────────────────────────────

setup_iceberg_lakehouse() {
  log "=== Setting up Apache Iceberg Data Lakehouse ==="

  # Iceberg REST Catalog (Gravitino)
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iceberg-rest-catalog
  namespace: data-engineering
spec:
  replicas: 3
  selector:
    matchLabels:
      app: iceberg-rest-catalog
  template:
    metadata:
      labels:
        app: iceberg-rest-catalog
    spec:
      containers:
        - name: catalog
          image: apache/gravitino:0.5.0
          ports:
            - containerPort: 8090
          env:
            - name: GRAVITINO_HOME
              value: /gravitino
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2
              memory: 4Gi
          volumeMounts:
            - name: config
              mountPath: /gravitino/conf
      volumes:
        - name: config
          configMap:
            name: gravitino-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: gravitino-config
  namespace: data-engineering
data:
  gravitino.conf: |
    gravitino.server.webserver.host = 0.0.0.0
    gravitino.server.webserver.httpPort = 8090
    gravitino.entity.store = kv
    gravitino.entity.store.kv = RocksDB
    gravitino.entity.store.kv.rocksdb.path = /tmp/gravitino-store

    # S3 catalog storage
    gravitino.catalog.providers = hadoop
    gravitino.hadoop.s3.endpoint = https://s3.amazonaws.com
    gravitino.hadoop.s3.access-key-id = ${AWS_ACCESS_KEY_ID}
    gravitino.hadoop.s3.secret-access-key = ${AWS_SECRET_ACCESS_KEY}
EOF

  # dbt Project for Iceberg Transformations
  cat <<'EOF' > /tmp/dbt_project.yml
name: data_lakehouse
version: '1.0.0'
config-version: 2

profile: data_lakehouse

model-paths: ["models"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]

target-path: "target"
clean-targets: ["target", "dbt_packages"]

models:
  data_lakehouse:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: table
      +schema: intermediate
      +file_format: iceberg
      +location_root: s3://data-lakehouse/intermediate
    marts:
      +materialized: table
      +schema: marts
      +file_format: iceberg
      +location_root: s3://data-lakehouse/marts
      +incremental_strategy: merge
      +unique_key: id
EOF

  # dbt Model: Customer 360
  cat <<'EOF' > /tmp/customer_360.sql
{{
  config(
    materialized='incremental',
    unique_key='customer_id',
    on_schema_change='sync_all_columns',
    partition_by={
      "field": "event_date",
      "data_type": "date",
      "granularity": "month"
    }
  )
}}

with
customer_events as (
  select
    customer_id,
    event_type,
    event_timestamp::date as event_date,
    properties,
    row_number() over (
      partition by customer_id, event_type::date
      order by event_timestamp desc
    ) as rn
  from {{ ref('stg_events') }}
  {% if is_incremental() %}
    where event_timestamp > (select max(updated_at) from {{ this }})
  {% endif %}
),

customer_profile as (
  select * from {{ ref('stg_customers') }}
),

customer_orders as (
  select
    customer_id,
    count(*) as total_orders,
    sum(amount_usd) as lifetime_value,
    max(created_at) as last_order_date,
    avg(amount_usd) as avg_order_value
  from {{ ref('stg_orders') }}
  group by 1
),

final as (
  select
    p.customer_id,
    p.email,
    p.name,
    p.country,
    p.signup_date,
    o.total_orders,
    o.lifetime_value,
    o.last_order_date,
    o.avg_order_value,
    case
      when o.lifetime_value > 10000 then 'VIP'
      when o.lifetime_value > 1000 then 'Premium'
      when o.total_orders > 5 then 'Regular'
      else 'New'
    end as customer_tier,
    count(e.event_type) as events_last_30d,
    max(e.event_date) as last_activity_date,
    current_timestamp as updated_at
  from customer_profile p
  left join customer_orders o using (customer_id)
  left join customer_events e using (customer_id)
  group by 1, 2, 3, 4, 5, 6, 7, 8, 9
)

select * from final
EOF

  log "✓ Apache Iceberg lakehouse configured"
}

# ─── 3. ClickHouse OLAP Analytics ─────────────────────────────────────────────

setup_clickhouse_olap() {
  log "=== Setting up ClickHouse OLAP ==="

  helm repo add clickhouse https://docs.altinity.com/clickhouse-operator/
  helm upgrade --install clickhouse-operator \
    clickhouse/altinity-clickhouse-operator \
    --namespace clickhouse-system \
    --create-namespace

  cat <<'EOF' | kubectl apply -f -
apiVersion: clickhouse.altinity.com/v1
kind: ClickHouseInstallation
metadata:
  name: analytics-cluster
  namespace: data-engineering
spec:
  configuration:
    clusters:
      - name: analytics
        layout:
          shardsCount: 3
          replicasCount: 2
    settings:
      max_concurrent_queries: 200
      max_memory_usage: 40000000000
      max_threads: 16
      use_uncompressed_cache: 1
      uncompressed_cache_size: 8589934592
      mark_cache_size: 5368709120
      merge_tree/max_bytes_to_merge_at_max_space_in_pool: 161061273600
    users:
      analytics/password: "${CLICKHOUSE_ANALYTICS_PASSWORD}"
      analytics/networks/ip:
        - "::/0"
      analytics/profile: analytics
    profiles:
      analytics:
        max_memory_usage: 20000000000
        max_rows_to_read: 1000000000
        max_bytes_to_read: 100000000000
        readonly: 0
  templates:
    podTemplates:
      - name: pod-template
        spec:
          containers:
            - name: clickhouse
              resources:
                requests:
                  cpu: "4"
                  memory: "16Gi"
                limits:
                  cpu: "16"
                  memory: "64Gi"
    volumeClaimTemplates:
      - name: storage
        spec:
          storageClassName: fast-nvme
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 2Ti
EOF

  # Create analytical tables
  cat <<'SQL' > /tmp/clickhouse_schema.sql
-- Events fact table (MergeTree with partitioning)
CREATE TABLE IF NOT EXISTS analytics.events ON CLUSTER analytics
(
    event_id    UUID DEFAULT generateUUIDv4(),
    customer_id UInt64,
    event_type  LowCardinality(String),
    event_date  Date,
    event_time  DateTime64(3, 'UTC'),
    properties  Map(String, String),
    session_id  String,
    country     LowCardinality(String),
    device_type LowCardinality(String),
    amount_usd  Nullable(Float64),
    INDEX idx_customer customer_id TYPE bloom_filter GRANULARITY 4,
    INDEX idx_session session_id TYPE bloom_filter GRANULARITY 4
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/analytics/events',
    '{replica}'
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, customer_id, event_type)
TTL event_date + INTERVAL 2 YEAR
SETTINGS
    index_granularity = 8192,
    min_bytes_for_wide_part = 10485760;

-- Distributed table
CREATE TABLE IF NOT EXISTS analytics.events_dist ON CLUSTER analytics
AS analytics.events
ENGINE = Distributed(analytics, analytics, events, customer_id);

-- Materialized view: Hourly aggregates
CREATE MATERIALIZED VIEW IF NOT EXISTS analytics.events_hourly ON CLUSTER analytics
ENGINE = ReplicatedSummingMergeTree(
    '/clickhouse/tables/{shard}/analytics/events_hourly',
    '{replica}'
)
PARTITION BY toYYYYMM(hour)
ORDER BY (hour, event_type, country)
AS
SELECT
    toStartOfHour(event_time) AS hour,
    event_type,
    country,
    count() AS events_count,
    uniq(customer_id) AS unique_customers,
    sum(amount_usd) AS total_revenue,
    avg(amount_usd) AS avg_revenue
FROM analytics.events
GROUP BY hour, event_type, country;
SQL

  log "✓ ClickHouse OLAP configured"
}

# ─── 4. Trino Query Engine ─────────────────────────────────────────────────────

setup_trino() {
  log "=== Setting up Trino Distributed Query Engine ==="

  helm repo add trino https://trinodb.github.io/charts
  helm upgrade --install trino trino/trino \
    --namespace data-engineering \
    --set coordinator.jvm.maxHeapSize=16G \
    --set worker.jvm.maxHeapSize=32G \
    --set worker.replicas=10 \
    --set additionalCatalogs.iceberg="\
connector.name=iceberg\n\
iceberg.catalog.type=rest\n\
iceberg.rest-catalog.uri=http://iceberg-rest-catalog:8090\n\
fs.native-s3.enabled=true\n\
s3.region=us-east-1" \
    --set additionalCatalogs.clickhouse="\
connector.name=clickhouse\n\
connection-url=jdbc:clickhouse://analytics-cluster:8123/\n\
connection-user=analytics\n\
connection-password=${CLICKHOUSE_PASSWORD}" \
    --set additionalCatalogs.delta="\
connector.name=delta_lake\n\
hive.metastore.uri=thrift://hive-metastore:9083\n\
delta.metadata.cache-ttl=10m" \
    --set "config.query.maxMemoryPerNode=8GB" \
    --set "config.query.maxMemory=80GB"

  log "✓ Trino query engine configured"
}

# ─── 5. Apache Airflow for Orchestration ─────────────────────────────────────

setup_airflow() {
  log "=== Setting up Apache Airflow ==="

  helm repo add apache-airflow https://airflow.apache.org
  helm upgrade --install airflow apache-airflow/airflow \
    --namespace airflow \
    --create-namespace \
    --set executor=KubernetesExecutor \
    --set dags.persistence.enabled=true \
    --set dags.gitSync.enabled=true \
    --set dags.gitSync.repo=https://github.com/example/airflow-dags \
    --set dags.gitSync.branch=main \
    --set webserverSecretKey="${AIRFLOW_SECRET_KEY}" \
    --set "env[0].name=AIRFLOW__CORE__LOAD_EXAMPLES" \
    --set "env[0].value=false" \
    --wait

  # Example DAG: dbt + ClickHouse pipeline
  cat <<'PYTHON' > /tmp/dbt_pipeline_dag.py
"""Data platform pipeline: dbt transformations + ClickHouse sync."""
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator

default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'data_platform_pipeline',
    default_args=default_args,
    description='Daily data platform pipeline',
    schedule_interval='0 3 * * *',
    catchup=False,
    max_active_runs=1,
    tags=['data-engineering', 'dbt', 'clickhouse'],
) as dag:

    # Step 1: dbt run for staging models
    dbt_staging = KubernetesPodOperator(
        task_id='dbt_staging',
        name='dbt-staging',
        namespace='airflow',
        image='ghcr.io/example/dbt-runner:1.7.0',
        cmds=['dbt', 'run', '--select', 'staging', '--target', 'production'],
        env_vars={'DBT_PROFILES_DIR': '/dbt'},
        secrets=[{'secret': 'dbt-profiles', 'deploy_type': 'volume', 'mount_path': '/dbt'}],
        resources={'request_cpu': '500m', 'request_memory': '1Gi', 'limit_cpu': '2', 'limit_memory': '4Gi'},
        is_delete_operator_pod=True,
    )

    # Step 2: dbt test staging
    dbt_test_staging = KubernetesPodOperator(
        task_id='dbt_test_staging',
        name='dbt-test-staging',
        namespace='airflow',
        image='ghcr.io/example/dbt-runner:1.7.0',
        cmds=['dbt', 'test', '--select', 'staging'],
        is_delete_operator_pod=True,
    )

    # Step 3: dbt run marts
    dbt_marts = KubernetesPodOperator(
        task_id='dbt_marts',
        name='dbt-marts',
        namespace='airflow',
        image='ghcr.io/example/dbt-runner:1.7.0',
        cmds=['dbt', 'run', '--select', 'marts'],
        is_delete_operator_pod=True,
    )

    # Step 4: Sync to ClickHouse
    sync_to_clickhouse = KubernetesPodOperator(
        task_id='sync_to_clickhouse',
        name='iceberg-to-clickhouse-sync',
        namespace='airflow',
        image='ghcr.io/example/data-sync:latest',
        cmds=['python', 'sync_iceberg_to_clickhouse.py'],
        env_vars={
            'ICEBERG_CATALOG': 'http://iceberg-rest-catalog:8090',
            'CLICKHOUSE_HOST': 'analytics-cluster:8123',
        },
        is_delete_operator_pod=True,
    )

    # Step 5: Verify data quality
    data_quality_check = SQLExecuteQueryOperator(
        task_id='data_quality_check',
        conn_id='clickhouse_analytics',
        sql="""
            SELECT
                count(*) as total_records,
                countIf(customer_id IS NULL) as null_customer_ids,
                countIf(event_date < today() - 2) as stale_records
            FROM analytics.events_dist
            WHERE event_date = yesterday()
            HAVING null_customer_ids = 0
              AND stale_records = 0
              AND total_records > 10000;
        """,
    )

    dbt_staging >> dbt_test_staging >> dbt_marts >> sync_to_clickhouse >> data_quality_check
PYTHON

  log "✓ Apache Airflow configured"
}

# ─── Main ──────────────────────────────────────────────────────────────────────

main() {
  log "Starting Data Engineering Platform setup..."
  setup_flink_streaming
  setup_iceberg_lakehouse
  setup_clickhouse_olap
  setup_trino
  setup_airflow
  log ""
  log "╔════════════════════════════════════════════════════╗"
  log "║     Data Engineering Platform Summary              ║"
  log "╠════════════════════════════════════════════════════╣"
  log "║  Streaming:  Apache Flink 1.18 (HA, RocksDB)      ║"
  log "║  Lakehouse:  Apache Iceberg + Gravitino Catalog    ║"
  log "║  Transform:  dbt 1.7 (incremental, Iceberg)       ║"
  log "║  OLAP:       ClickHouse (3 shards × 2 replicas)   ║"
  log "║  Query:      Trino (10 workers, multi-catalog)     ║"
  log "║  Orchestrate: Apache Airflow (K8s Executor)        ║"
  log "╚════════════════════════════════════════════════════╝"
}

main "$@"
```

---

## สรุป Part 57

ในส่วนนี้เราได้เรียนรู้:

| ขั้นตอน | หัวข้อ | เทคโนโลยีหลัก |
|---------|--------|---------------|
| 577 | Multi-Cloud Federation | Crossplane Multi-Cloud, Istio Multi-Cluster, Cost Analysis |
| 578 | Edge Computing | K3s Edge, Cloudflare Workers, AWS Outposts, Edge AI |
| 579 | Custom Kubernetes Operators | Operator SDK, Go Controller, CRD Types, Reconciler |
| 580 | Data Engineering at Scale | Flink, Iceberg, ClickHouse, Trino, dbt, Airflow |

### เทคนิคสำคัญที่ได้เรียนรู้

1. **Multi-Cloud XRD** — สร้าง abstraction layer ที่รองรับ AWS/Azure/GCP ผ่าน Crossplane Compositions
2. **K3s Edge** — ลด footprint ด้วยการ disable components ที่ไม่จำเป็น, ใช้ VictoriaMetrics แทน Prometheus เต็ม
3. **Operator Pattern** — Reconcile loop: Observe → Diff → Act → Update Status, ใช้ Finalizer สำหรับ cleanup
4. **Data Lakehouse** — Iceberg format รองรับ ACID transactions, time-travel queries, schema evolution
5. **Cloudflare Workers** — JWT verification, geo-routing, A/B testing, rate limiting ที่ edge (< 1ms latency)

---

ขั้นตอนต่อไป: **Part 58** — Real-time Analytics, Event Streaming Architecture และ Digital Twin Platform
