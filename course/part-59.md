# Part 59: Blockchain Infrastructure, Advanced Service Mesh และ Platform Security

## Module 5: World-Class Level (ต่อ)

---

## ขั้นตอนที่ 585: Blockchain Infrastructure on Kubernetes

### Blockchain Node Architecture

```
Blockchain Infrastructure on K8s:
┌─────────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                           │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Validator   │  │  Full Node   │  │  Archive Node    │  │
│  │  Node (HA)   │  │  (RPC/API)  │  │  (Historical)    │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│          │                 │                   │             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              P2P Network (libp2p)                      │  │
│  └───────────────────────────────────────────────────────┘  │
│          │                 │                   │             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Block Sync  │  │ Transaction  │  │   Smart Contract │  │
│  │  Service     │  │  Pool        │  │   Indexer        │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### `blockchain-infrastructure.sh`

```bash
#!/usr/bin/env bash
# blockchain-infrastructure.sh — Enterprise Blockchain on Kubernetes
set -euo pipefail

LOG_FILE="/var/log/blockchain-infra.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Ethereum Validator Node (Lighthouse) ──────────────────────────────────

setup_ethereum_validator() {
  log "=== Setting up Ethereum Validator Node ==="

  kubectl create namespace ethereum --dry-run=client -o yaml | kubectl apply -f -

  # Execution Client: Geth
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: geth-execution
  namespace: ethereum
spec:
  serviceName: geth-headless
  replicas: 1
  selector:
    matchLabels:
      app: geth
      role: execution
  template:
    metadata:
      labels:
        app: geth
        role: execution
    spec:
      initContainers:
        - name: init-genesis
          image: ethereum/client-go:v1.13.14
          command:
            - sh
            - -c
            - |
              if [ ! -f /data/geth/chaindata/CURRENT ]; then
                echo "Initializing Geth database..."
                geth init --datadir /data /config/genesis.json
              fi
          volumeMounts:
            - name: data
              mountPath: /data
            - name: config
              mountPath: /config
      containers:
        - name: geth
          image: ethereum/client-go:v1.13.14
          command:
            - geth
            - --datadir=/data
            - --http
            - --http.addr=0.0.0.0
            - --http.port=8545
            - --http.api=eth,net,web3,engine,admin
            - --http.corsdomain=*
            - --ws
            - --ws.addr=0.0.0.0
            - --ws.port=8546
            - --ws.api=eth,net,web3,engine
            - --authrpc.addr=0.0.0.0
            - --authrpc.port=8551
            - --authrpc.jwtsecret=/secrets/jwt.hex
            - --metrics
            - --metrics.addr=0.0.0.0
            - --metrics.port=6060
            - --maxpeers=50
            - --cache=4096
          ports:
            - containerPort: 8545
              name: http-rpc
            - containerPort: 8546
              name: ws-rpc
            - containerPort: 8551
              name: auth-rpc
            - containerPort: 30303
              name: p2p
              protocol: TCP
            - containerPort: 30303
              name: p2p-udp
              protocol: UDP
            - containerPort: 6060
              name: metrics
          resources:
            requests:
              cpu: "2"
              memory: 8Gi
            limits:
              cpu: "8"
              memory: 32Gi
          volumeMounts:
            - name: data
              mountPath: /data
            - name: secrets
              mountPath: /secrets
          readinessProbe:
            httpGet:
              path: /
              port: 8545
            initialDelaySeconds: 30
            periodSeconds: 15
      volumes:
        - name: config
          configMap:
            name: ethereum-config
        - name: secrets
          secret:
            secretName: ethereum-secrets
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        storageClassName: fast-nvme
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 2Ti  # Full Ethereum mainnet state
---
# Consensus Client: Lighthouse
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: lighthouse-beacon
  namespace: ethereum
spec:
  serviceName: lighthouse-headless
  replicas: 1
  selector:
    matchLabels:
      app: lighthouse
      role: beacon
  template:
    metadata:
      labels:
        app: lighthouse
        role: beacon
    spec:
      containers:
        - name: lighthouse
          image: sigp/lighthouse:v5.1.3
          command:
            - lighthouse
            - beacon_node
            - --network=mainnet
            - --datadir=/data
            - --http
            - --http-address=0.0.0.0
            - --http-port=5052
            - --execution-endpoint=http://geth-execution:8551
            - --execution-jwt=/secrets/jwt.hex
            - --metrics
            - --metrics-address=0.0.0.0
            - --metrics-port=5054
            - --checkpoint-sync-url=https://mainnet.checkpoint.sigp.io
            - --disable-deposit-contract-sync
            - --target-peers=80
          ports:
            - containerPort: 5052
              name: http-api
            - containerPort: 9000
              name: p2p
            - containerPort: 5054
              name: metrics
          resources:
            requests:
              cpu: "2"
              memory: 8Gi
            limits:
              cpu: "4"
              memory: 16Gi
          volumeMounts:
            - name: data
              mountPath: /data
            - name: secrets
              mountPath: /secrets
      volumes:
        - name: secrets
          secret:
            secretName: ethereum-secrets
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        storageClassName: fast-nvme
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 500Gi
EOF

  log "✓ Ethereum validator node configured"
}

# ─── 2. Hyperledger Fabric Enterprise Network ────────────────────────────────

setup_hyperledger_fabric() {
  log "=== Setting up Hyperledger Fabric Enterprise Network ==="

  # Fabric CA (Certificate Authority)
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fabric-ca
  namespace: hyperledger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fabric-ca
  template:
    metadata:
      labels:
        app: fabric-ca
    spec:
      containers:
        - name: fabric-ca
          image: hyperledger/fabric-ca:2.5
          command:
            - fabric-ca-server
            - start
            - -b
            - admin:adminpw
            - --cfg.affiliations.allowremove
            - --cfg.identities.allowremove
          env:
            - name: FABRIC_CA_HOME
              value: /etc/hyperledger/fabric-ca-server
            - name: FABRIC_CA_SERVER_CA_NAME
              value: ca-org1
            - name: FABRIC_CA_SERVER_TLS_ENABLED
              value: "true"
            - name: FABRIC_CA_SERVER_PORT
              value: "7054"
            - name: FABRIC_CA_SERVER_DB_TYPE
              value: postgres
            - name: FABRIC_CA_SERVER_DB_DATASOURCE
              valueFrom:
                secretKeyRef:
                  name: fabric-ca-db
                  key: datasource
          ports:
            - containerPort: 7054
          volumeMounts:
            - name: ca-data
              mountPath: /etc/hyperledger/fabric-ca-server
      volumes:
        - name: ca-data
          persistentVolumeClaim:
            claimName: fabric-ca-pvc
EOF

  # Orderer (RAFT Consensus)
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: orderer
  namespace: hyperledger
spec:
  serviceName: orderer-headless
  replicas: 3
  selector:
    matchLabels:
      app: orderer
  template:
    metadata:
      labels:
        app: orderer
    spec:
      containers:
        - name: orderer
          image: hyperledger/fabric-orderer:2.5
          env:
            - name: FABRIC_LOGGING_SPEC
              value: INFO
            - name: ORDERER_GENERAL_LISTENADDRESS
              value: 0.0.0.0
            - name: ORDERER_GENERAL_LISTENPORT
              value: "7050"
            - name: ORDERER_GENERAL_TLS_ENABLED
              value: "true"
            - name: ORDERER_GENERAL_BOOTSTRAPMETHOD
              value: none
            - name: ORDERER_CHANNELPARTICIPATION_ENABLED
              value: "true"
            - name: ORDERER_GENERAL_CONSENSUS_TYPE
              value: etcdraft
            - name: ORDERER_METRICS_PROVIDER
              value: prometheus
          ports:
            - containerPort: 7050
              name: grpc
            - containerPort: 8080
              name: metrics
          resources:
            requests:
              cpu: "1"
              memory: 1Gi
            limits:
              cpu: "4"
              memory: 4Gi
          volumeMounts:
            - name: orderer-data
              mountPath: /var/hyperledger/production/orderer
            - name: msp
              mountPath: /var/hyperledger/orderer/msp
            - name: tls
              mountPath: /var/hyperledger/orderer/tls
      volumes:
        - name: msp
          secret:
            secretName: orderer-msp
        - name: tls
          secret:
            secretName: orderer-tls
  volumeClaimTemplates:
    - metadata:
        name: orderer-data
      spec:
        storageClassName: fast-ssd
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 100Gi
EOF

  log "✓ Hyperledger Fabric network configured"
}

# ─── 3. Smart Contract Indexer (The Graph) ───────────────────────────────────

setup_graph_indexer() {
  log "=== Setting up The Graph Indexer ==="

  # Graph Node
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graph-node
  namespace: ethereum
spec:
  replicas: 2
  selector:
    matchLabels:
      app: graph-node
  template:
    metadata:
      labels:
        app: graph-node
    spec:
      containers:
        - name: graph-node
          image: graphprotocol/graph-node:v0.35.1
          env:
            - name: postgres_host
              value: postgresql:5432
            - name: postgres_user
              valueFrom:
                secretKeyRef:
                  name: graph-db
                  key: username
            - name: postgres_pass
              valueFrom:
                secretKeyRef:
                  name: graph-db
                  key: password
            - name: postgres_db
              value: graph_node
            - name: ipfs
              value: http://ipfs:5001
            - name: ethereum
              value: mainnet:http://geth-execution:8545
            - name: GRAPH_LOG
              value: info
            - name: EXPERIMENTAL_SUBGRAPH_VERSION_SWITCHING_MODE
              value: synced
            - name: GRAPH_STORE_CONNECTION_POOL_SIZE
              value: "25"
          ports:
            - containerPort: 8000
              name: http
            - containerPort: 8001
              name: ws
            - containerPort: 8020
              name: json-rpc
            - containerPort: 8030
              name: index-node
            - containerPort: 8040
              name: metrics
          resources:
            requests:
              cpu: "1"
              memory: 2Gi
            limits:
              cpu: "4"
              memory: 8Gi
EOF

  # Example Subgraph: ERC-20 Token Tracking
  cat <<'YAML' > /tmp/subgraph.yaml
specVersion: 0.0.5
schema:
  file: ./schema.graphql
dataSources:
  - kind: ethereum
    name: ERC20Token
    network: mainnet
    source:
      address: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"  # USDC
      abi: ERC20
      startBlock: 6082465
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Transfer
        - Account
        - Token
      abis:
        - name: ERC20
          file: ./abis/ERC20.json
      eventHandlers:
        - event: Transfer(indexed address,indexed address,uint256)
          handler: handleTransfer
      file: ./src/mapping.ts
YAML

  log "✓ Graph indexer configured"
}

# ─── 4. DeFi Protocol Infrastructure ─────────────────────────────────────────

setup_defi_infrastructure() {
  log "=== Setting up DeFi Protocol Infrastructure ==="

  # MEV Protection: Flashbots RPC Proxy
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flashbots-proxy
  namespace: ethereum
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flashbots-proxy
  template:
    metadata:
      labels:
        app: flashbots-proxy
    spec:
      containers:
        - name: proxy
          image: ghcr.io/flashbots/mev-relay-proxy:latest
          env:
            - name: RELAY_URL
              value: https://relay.flashbots.net
            - name: ETHEREUM_RPC
              value: http://geth-execution:8545
          ports:
            - containerPort: 8545
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
EOF

  # Transaction Monitoring (anti-money laundering)
  cat <<'PYTHON' > /tmp/tx_monitor.py
"""Blockchain transaction monitoring for compliance."""
import asyncio
import json
import httpx
from web3 import AsyncWeb3, AsyncHTTPProvider
from dataclasses import dataclass
from typing import Optional

# OFAC/sanctioned address list (simplified)
SANCTIONED_ADDRESSES = {
    "0x7F367cC41522cE07553e823bf3be79A889debe1B",
    "0xd882cfc20f52f2599d84b8e8d58c7fb62cfe344b",
}

@dataclass
class TxRiskScore:
    tx_hash: str
    risk_score: float  # 0-100
    flags: list[str]
    sanctioned_interaction: bool
    high_value: bool
    mixer_interaction: bool

class TransactionMonitor:
    def __init__(self, rpc_url: str):
        self.w3 = AsyncWeb3(AsyncHTTPProvider(rpc_url))

    async def analyze_transaction(self, tx_hash: str) -> TxRiskScore:
        tx = await self.w3.eth.get_transaction(tx_hash)
        receipt = await self.w3.eth.get_transaction_receipt(tx_hash)

        flags = []
        risk_score = 0.0

        # Check sanctioned addresses
        sanctioned = (
            tx['from'].lower() in {a.lower() for a in SANCTIONED_ADDRESSES} or
            (tx['to'] and tx['to'].lower() in {a.lower() for a in SANCTIONED_ADDRESSES})
        )
        if sanctioned:
            flags.append("SANCTIONED_ADDRESS")
            risk_score += 90.0

        # Check high value (>$100k equivalent)
        eth_value = self.w3.from_wei(tx['value'], 'ether')
        high_value = eth_value > 50  # ~$200k at $4000/ETH
        if high_value:
            flags.append("HIGH_VALUE")
            risk_score += 20.0

        # Check known mixer contracts
        known_mixers = {
            "0xba214cf10faae693a9f605e3f89c00af6a5ddf8e",  # Tornado Cash
        }
        mixer = tx.get('to', '').lower() in known_mixers
        if mixer:
            flags.append("MIXER_INTERACTION")
            risk_score += 70.0

        return TxRiskScore(
            tx_hash=tx_hash,
            risk_score=min(100.0, risk_score),
            flags=flags,
            sanctioned_interaction=sanctioned,
            high_value=high_value,
            mixer_interaction=mixer,
        )

    async def monitor_mempool(self):
        """Watch pending transactions for compliance issues."""
        async for tx_hash in self.w3.eth.subscribe("pendingTransactions"):
            try:
                score = await self.analyze_transaction(tx_hash.hex())
                if score.risk_score > 50:
                    print(f"HIGH RISK TX: {score}")
            except Exception:
                pass

if __name__ == "__main__":
    monitor = TransactionMonitor("http://geth-execution:8545")
    asyncio.run(monitor.monitor_mempool())
PYTHON

  log "✓ DeFi infrastructure configured"
}

main() {
  log "Starting Blockchain Infrastructure setup..."
  setup_ethereum_validator
  setup_hyperledger_fabric
  setup_graph_indexer
  setup_defi_infrastructure
  log "✓ Blockchain Infrastructure complete"
}
main "$@"
```

---

## ขั้นตอนที่ 586: Advanced Service Mesh — Ambient Mode และ L4/L7 Policies

### Istio Ambient Mode Architecture

```
Istio Ambient Mode (no sidecar):
                                                   
  Pod A ──► ztunnel (L4, node-level) ──► ztunnel ──► Pod B
                     │
                  Waypoint Proxy (L7, per-namespace)
                  ├── HTTP routing
                  ├── mTLS termination
                  ├── AuthorizationPolicy
                  └── VirtualService

Benefit: Reduced overhead vs sidecar model:
  - Sidecar: +200ms P99, +100Mi per pod
  - Ambient: +5ms P99, 0Mi per pod overhead
```

### `advanced-service-mesh.sh`

```bash
#!/usr/bin/env bash
# advanced-service-mesh.sh — Istio Ambient Mode + Advanced Policies
set -euo pipefail

LOG_FILE="/var/log/advanced-service-mesh.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Istio Ambient Mode Installation ──────────────────────────────────────

setup_istio_ambient() {
  log "=== Setting up Istio Ambient Mode ==="

  # Download Istio
  curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.22.1 sh -
  export PATH="$PWD/istio-1.22.1/bin:$PATH"

  # Install with ambient profile
  istioctl install --set profile=ambient \
    --set values.ztunnel.resources.requests.cpu=100m \
    --set values.ztunnel.resources.requests.memory=128Mi \
    --set values.pilot.resources.requests.cpu=500m \
    --set values.pilot.resources.requests.memory=2Gi \
    --set meshConfig.accessLogFile=/dev/stdout \
    --set meshConfig.accessLogEncoding=JSON \
    --set meshConfig.enablePrometheusMerge=true \
    -y

  # Enable ambient for namespaces
  kubectl label namespace production istio.io/dataplane-mode=ambient
  kubectl label namespace staging istio.io/dataplane-mode=ambient

  log "✓ Istio Ambient Mode installed"
}

# ─── 2. Waypoint Proxy (L7 policies) ─────────────────────────────────────────

setup_waypoint_proxy() {
  log "=== Setting up Waypoint Proxy ==="

  # Create waypoint for production namespace
  cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: waypoint
  namespace: production
  annotations:
    istio.io/waypoint-for: service
spec:
  gatewayClassName: istio-waypoint
  listeners:
    - name: mesh
      port: 15008
      protocol: HBONE
---
# L7 Authorization Policy (requires waypoint)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-policy
  namespace: production
spec:
  targetRef:
    group: ""
    kind: Service
    name: payment-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/checkout-service
              - cluster.local/ns/production/sa/order-service
      to:
        - operation:
            methods: [POST]
            paths: [/api/v1/payments/*, /api/v1/refunds/*]
      when:
        - key: request.headers[x-correlation-id]
          notValues: [""]
        - key: source.namespace
          values: [production]
    - from:
        - source:
            principals:
              - cluster.local/ns/monitoring/sa/prometheus
      to:
        - operation:
            methods: [GET]
            paths: [/metrics]
---
# Deny all other traffic to payment service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-deny-all
  namespace: production
spec:
  targetRef:
    group: ""
    kind: Service
    name: payment-service
  action: DENY
  rules:
    - from:
        - source:
            notPrincipals:
              - cluster.local/ns/production/sa/checkout-service
              - cluster.local/ns/production/sa/order-service
              - cluster.local/ns/monitoring/sa/prometheus
EOF

  # HTTP Traffic Management
  cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: checkout-service
  namespace: production
spec:
  hosts:
    - checkout-service
  http:
    - name: retry-policy
      route:
        - destination:
            host: checkout-service
            port:
              number: 8080
      retries:
        attempts: 3
        perTryTimeout: 5s
        retryOn: 5xx,connect-failure,reset,retriable-status-codes
        retryRemoteLocalities: true
      timeout: 15s
      fault:
        delay:
          percentage:
            value: 0.1  # 0.1% of traffic gets 2s delay (chaos)
          fixedDelay: 2s
    - name: header-routing
      match:
        - headers:
            x-version:
              exact: canary
      route:
        - destination:
            host: checkout-service-canary
            port:
              number: 8080
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: checkout-service
  namespace: production
spec:
  host: checkout-service
  trafficPolicy:
    connectionPool:
      http:
        h2UpgradePolicy: UPGRADE
        http2MaxRequests: 1000
        http1MaxPendingRequests: 200
      tcp:
        maxConnections: 100
        connectTimeout: 5s
        tcpKeepalive:
          time: 7200s
          interval: 75s
    loadBalancer:
      simple: LEAST_CONN
      warmupDurationSecs: 30
    outlierDetection:
      consecutive5xxErrors: 5
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50
      minHealthPercent: 30
    tls:
      mode: ISTIO_MUTUAL
EOF

  log "✓ Waypoint proxy and traffic policies configured"
}

# ─── 3. Service Mesh Observability ────────────────────────────────────────────

setup_mesh_observability() {
  log "=== Setting up Mesh Observability ==="

  # Kiali Service Mesh Console
  helm repo add kiali https://kiali.org/helm-charts
  helm upgrade --install kiali-operator kiali/kiali-operator \
    --namespace kiali-operator \
    --create-namespace

  cat <<'EOF' | kubectl apply -f -
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  name: kiali
  namespace: istio-system
spec:
  version: default
  auth:
    strategy: openid
    openid:
      client_id: kiali
      issuer_uri: https://keycloak.example.com/realms/platform
      username_claim: preferred_username
  external_services:
    prometheus:
      url: http://kube-prometheus-stack-prometheus:9090
    grafana:
      auth:
        type: bearer
        use_kiali_token: true
      enabled: true
      in_cluster_url: http://grafana:80
    tracing:
      enabled: true
      in_cluster_url: http://jaeger-query:16685
      use_grpc: true
    istio:
      config_map_name: istio
      istio_sidecar_annotation: sidecar.istio.io/status
  istio_namespace: istio-system
  deployment:
    replicas: 2
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
EOF

  # Service Mesh Metrics (L7 RED metrics)
  cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: service-mesh-alerts
  namespace: monitoring
spec:
  groups:
    - name: service-mesh
      interval: 30s
      rules:
        - alert: HighServiceLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(istio_request_duration_milliseconds_bucket{
                reporter="source",
                destination_service_namespace="production"
              }[5m])) by (le, destination_service_name)
            ) > 1000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Service {{ $labels.destination_service_name }} P99 latency > 1s"
            description: "P99 latency is {{ $value }}ms"

        - alert: HighServiceErrorRate
          expr: |
            sum(rate(istio_requests_total{
              reporter="source",
              response_code=~"5..",
              destination_service_namespace="production"
            }[5m])) by (destination_service_name)
            /
            sum(rate(istio_requests_total{
              reporter="source",
              destination_service_namespace="production"
            }[5m])) by (destination_service_name)
            > 0.01
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Service {{ $labels.destination_service_name }} error rate > 1%"

        - alert: mTLSNotEnforced
          expr: |
            sum(istio_requests_total{connection_security_policy="none"}) by (source_workload, destination_workload) > 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Unencrypted traffic detected between {{ $labels.source_workload }} and {{ $labels.destination_workload }}"
EOF

  log "✓ Mesh observability configured"
}

# ─── 4. Multi-Cluster Service Discovery ──────────────────────────────────────

setup_multicluster_discovery() {
  log "=== Setting up Multi-Cluster Service Discovery ==="

  # ServiceImport/ServiceExport for multi-cluster (MCS API)
  cat <<'EOF' | kubectl apply -f -
# Export payment-service from cluster-1 to be visible in cluster-2
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: payment-service
  namespace: production
---
# Import and use the exported service in cluster-2
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: payment-service
  namespace: production
spec:
  type: ClusterSetIP
  ports:
    - name: http
      protocol: TCP
      port: 8080
EOF

  # Cross-cluster DestinationRule
  cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-multicluster
  namespace: production
spec:
  host: payment-service.production.svc.clusterset.local
  trafficPolicy:
    loadBalancer:
      localityLbSetting:
        enabled: true
        distribute:
          - from: us-east-1/*
            to:
              us-east-1/*: 70
              us-west-2/*: 30
          - from: us-west-2/*
            to:
              us-west-2/*: 70
              us-east-1/*: 30
        failover:
          - from: us-east-1
            to: us-west-2
          - from: us-west-2
            to: us-east-1
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 30s
EOF

  log "✓ Multi-cluster service discovery configured"
}

main() {
  log "Starting Advanced Service Mesh setup..."
  setup_istio_ambient
  setup_waypoint_proxy
  setup_mesh_observability
  setup_multicluster_discovery
  log "✓ Advanced Service Mesh complete"
}
main "$@"
```

---

## ขั้นตอนที่ 587: Platform Security — Zero Trust Architecture

### Zero Trust Architecture

```
Zero Trust Security Model:
"Never Trust, Always Verify"

┌──────────────────────────────────────────────────────────┐
│                    Identity Layer                         │
│  SPIFFE/SPIRE (Workload Identity) │ OIDC/JWT (User)      │
└──────────────────────────────────────────────────────────┘
               │ Verify identity on EVERY request
               ▼
┌──────────────────────────────────────────────────────────┐
│                   Policy Engine                           │
│  OPA/Rego │ Kyverno │ Istio AuthPolicy                   │
└──────────────────────────────────────────────────────────┘
               │ Enforce least privilege
               ▼
┌──────────────────────────────────────────────────────────┐
│                  Network Layer                            │
│  mTLS everywhere │ NetworkPolicy │ eBPF firewall         │
└──────────────────────────────────────────────────────────┘
               │ Encrypt all traffic
               ▼
┌──────────────────────────────────────────────────────────┐
│                  Data Layer                               │
│  Vault secrets │ KMS encryption │ Sealed Secrets         │
└──────────────────────────────────────────────────────────┘
```

### `zero-trust-platform.sh`

```bash
#!/usr/bin/env bash
# zero-trust-platform.sh — Zero Trust Security Architecture
set -euo pipefail

LOG_FILE="/var/log/zero-trust.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. SPIFFE/SPIRE Workload Identity ────────────────────────────────────────

setup_spire() {
  log "=== Setting up SPIFFE/SPIRE Workload Identity ==="

  helm repo add spiffe https://spiffe.github.io/helm-charts-hardened/
  helm upgrade --install spire spiffe/spire \
    --namespace spire-system \
    --create-namespace \
    --set global.spire.trustDomain=platform.example.com \
    --set spire-server.replicaCount=3 \
    --set spire-server.dataStore.database.databaseType=postgres \
    --set spire-server.dataStore.database.connectionString="${SPIRE_DB_URL}" \
    --set spire-agent.serviceAccountName=spire-agent

  # Register workloads with SPIRE
  cat <<'EOF' | kubectl apply -f -
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: payment-service
spec:
  spiffeIDTemplate: "spiffe://platform.example.com/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spire-workload: payment-service
  workloadSelectorTemplates:
    - "k8s:ns:production"
    - "k8s:sa:payment-service"
  dnsNameTemplates:
    - "{{ .PodMeta.Name }}.{{ .PodMeta.Namespace }}.svc.cluster.local"
  ttl: 1h
EOF

  log "✓ SPIRE workload identity configured"
}

# ─── 2. HashiCorp Vault with Dynamic Secrets ─────────────────────────────────

setup_vault_dynamic_secrets() {
  log "=== Setting up Vault Dynamic Secrets ==="

  helm repo add hashicorp https://helm.releases.hashicorp.com
  helm upgrade --install vault hashicorp/vault \
    --namespace vault \
    --create-namespace \
    --set server.ha.enabled=true \
    --set server.ha.replicas=3 \
    --set server.ha.raft.enabled=true \
    --set server.ha.raft.config="
      ui = true
      listener \"tcp\" {
        tls_disable = 0
        address = \"[::]:8200\"
        tls_cert_file = \"/vault/userconfig/tls/tls.crt\"
        tls_key_file  = \"/vault/userconfig/tls/tls.key\"
      }
      storage \"raft\" {
        path = \"/vault/data\"
        node_id = \"\${POD_NAME}\"
        retry_join {
          leader_tls_servername = \"vault-0.vault-internal\"
          leader_api_addr = \"https://vault-0.vault-internal:8200\"
        }
        retry_join {
          leader_api_addr = \"https://vault-1.vault-internal:8200\"
        }
        retry_join {
          leader_api_addr = \"https://vault-2.vault-internal:8200\"
        }
      }
      seal \"awskms\" {
        region = \"us-east-1\"
        kms_key_id = \"${KMS_KEY_ID}\"
      }
      service_registration \"kubernetes\" {}
    " \
    --set injector.enabled=true \
    --set csi.enabled=true

  # Configure Vault after init
  cat <<'VAULTSCRIPT' > /tmp/vault-configure.sh
#!/bin/bash
export VAULT_ADDR=https://vault:8200
export VAULT_TOKEN="${ROOT_TOKEN}"

# Enable Kubernetes auth
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token \
  issuer="https://kubernetes.default.svc.cluster.local"

# Enable database secrets engine
vault secrets enable -path=database database

# PostgreSQL dynamic credentials
vault write database/config/postgres \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@postgresql:5432/appdb?sslmode=require" \
  allowed_roles="app-readonly,app-readwrite" \
  username="${PG_ADMIN_USER}" \
  password="${PG_ADMIN_PASSWORD}"

vault write database/roles/app-readwrite \
  db_name=postgres \
  creation_statements="
    CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";
    GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO \"{{name}}\";
  " \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Kubernetes service account → Vault role mapping
vault write auth/kubernetes/role/payment-service \
  bound_service_account_names=payment-service \
  bound_service_account_namespaces=production \
  policies=payment-service-policy \
  ttl=1h

# Policy
vault policy write payment-service-policy - <<EOF
path "database/creds/app-readwrite" {
  capabilities = ["read"]
}
path "secret/data/payment/*" {
  capabilities = ["read"]
}
path "pki/issue/payment-service" {
  capabilities = ["create", "update"]
}
EOF

# PKI (TLS certificates)
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki
vault write pki/root/generate/internal \
  common_name="platform.example.com" \
  ttl=87600h \
  key_type=ec \
  key_bits=384
vault write pki/roles/payment-service \
  allowed_domains=production.svc.cluster.local \
  allow_subdomains=true \
  max_ttl=72h \
  key_type=ec \
  key_bits=256
VAULTSCRIPT

  chmod +x /tmp/vault-configure.sh

  # Vault Secrets Operator for K8s native integration
  helm upgrade --install vault-secrets-operator hashicorp/vault-secrets-operator \
    --namespace vault \
    --set defaultVaultConnection.enabled=true \
    --set defaultVaultConnection.address=https://vault:8200

  # Example: Sync Vault secret to K8s Secret
  cat <<'EOF' | kubectl apply -f -
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: payment-service-auth
  namespace: production
spec:
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: payment-service
    serviceAccount: payment-service
    audiences:
      - vault
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultDynamicSecret
metadata:
  name: payment-db-credentials
  namespace: production
spec:
  vaultAuthRef: payment-service-auth
  mount: database
  path: creds/app-readwrite
  destination:
    name: payment-db-creds
    create: true
    type: kubernetes.io/basic-auth
  rolloutRestartTargets:
    - kind: Deployment
      name: payment-service
  renewalPercent: 67
EOF

  log "✓ Vault dynamic secrets configured"
}

# ─── 3. OPA (Open Policy Agent) ─────────────────────────────────────────────

setup_opa_gatekeeper() {
  log "=== Setting up OPA Gatekeeper ==="

  helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
  helm upgrade --install gatekeeper gatekeeper/gatekeeper \
    --namespace gatekeeper-system \
    --create-namespace \
    --set replicas=3 \
    --set auditInterval=60 \
    --set constraintViolationsLimit=100 \
    --set logLevel=WARNING \
    --set metricsBackend=prometheus \
    --set logMutations=true

  # ConstraintTemplate: Require resource limits
  cat <<'EOF' | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requireresourcelimits
spec:
  crd:
    spec:
      names:
        kind: RequireResourceLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            containers:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requireresourcelimits

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' must have CPU limits set", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' must have memory limits set", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          cpu := container.resources.limits.cpu
          memory := container.resources.limits.memory
          # Check if limits are suspiciously high (> 32 CPUs or > 128Gi)
          to_number(replace(cpu, "m", "")) > 32000
          msg := sprintf("Container '%v' CPU limit %v is too high (max 32 CPUs)", [container.name, cpu])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireResourceLimits
metadata:
  name: require-resource-limits
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
      - apiGroups: ["batch"]
        kinds: ["Job", "CronJob"]
    namespaces:
      - production
      - staging
    excludedNamespaces:
      - kube-system
      - monitoring
EOF

  # ConstraintTemplate: No privileged containers
  cat <<'EOF' | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: noprivilegedcontainers
spec:
  crd:
    spec:
      names:
        kind: NoPrivilegedContainers
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package noprivilegedcontainers

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Container '%v' must not run as privileged", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.allowPrivilegeEscalation == true
          msg := sprintf("Container '%v' must not allow privilege escalation", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          capability := container.securityContext.capabilities.add[_]
          not_safe_capabilities := {"SYS_ADMIN", "NET_ADMIN", "SYS_PTRACE", "DAC_OVERRIDE"}
          not_safe_capabilities[capability]
          msg := sprintf("Container '%v' adds dangerous capability: %v", [container.name, capability])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: NoPrivilegedContainers
metadata:
  name: no-privileged-containers
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["", "apps", "batch"]
        kinds: ["Pod", "Deployment", "StatefulSet", "DaemonSet", "Job"]
    excludedNamespaces:
      - kube-system
      - spire-system
      - kepler-system
EOF

  log "✓ OPA Gatekeeper configured"
}

# ─── 4. Security Scanning Pipeline ────────────────────────────────────────────

setup_security_scanning() {
  log "=== Setting up Security Scanning Pipeline ==="

  # SBOM generation and signing pipeline
  cat <<'PYTHON' > /tmp/security_pipeline.py
"""Automated security scanning pipeline."""
import subprocess
import json
import os
from dataclasses import dataclass

@dataclass
class SecurityReport:
    image: str
    critical_vulns: int
    high_vulns: int
    sbom_signed: bool
    policy_compliant: bool
    risk_score: float

class SecurityPipeline:
    def __init__(self, registry: str = "ghcr.io/example"):
        self.registry = registry

    def scan_image(self, image: str) -> SecurityReport:
        full_image = f"{self.registry}/{image}"

        # 1. Generate SBOM with syft
        sbom_result = subprocess.run(
            ["syft", full_image, "-o", "spdx-json=/tmp/sbom.json"],
            capture_output=True, text=True
        )

        # 2. Scan vulnerabilities with grype
        grype_result = subprocess.run(
            ["grype", f"sbom:/tmp/sbom.json", "-o", "json"],
            capture_output=True, text=True
        )
        grype_data = json.loads(grype_result.stdout) if grype_result.returncode == 0 else {}

        matches = grype_data.get("matches", [])
        critical = sum(1 for m in matches if m.get("vulnerability", {}).get("severity") == "Critical")
        high = sum(1 for m in matches if m.get("vulnerability", {}).get("severity") == "High")

        # 3. Sign SBOM with cosign
        sign_result = subprocess.run(
            ["cosign", "attest", "--predicate", "/tmp/sbom.json",
             "--type", "spdxjson", full_image],
            capture_output=True, text=True,
            env={**os.environ, "COSIGN_EXPERIMENTAL": "1"}
        )
        sbom_signed = sign_result.returncode == 0

        # 4. Check against policy (no critical vulns)
        policy_compliant = critical == 0

        # 5. Risk score calculation
        risk_score = min(100.0, critical * 20 + high * 5)

        return SecurityReport(
            image=full_image,
            critical_vulns=critical,
            high_vulns=high,
            sbom_signed=sbom_signed,
            policy_compliant=policy_compliant,
            risk_score=risk_score,
        )

    def gate_deployment(self, image: str) -> bool:
        """Return True if image passes security gate."""
        report = self.scan_image(image)
        print(f"Security Report for {image}:")
        print(f"  Critical: {report.critical_vulns}")
        print(f"  High:     {report.high_vulns}")
        print(f"  SBOM:     {'Signed' if report.sbom_signed else 'NOT signed'}")
        print(f"  Policy:   {'PASS' if report.policy_compliant else 'FAIL'}")
        print(f"  Risk:     {report.risk_score:.0f}/100")

        if not report.policy_compliant:
            print(f"GATE BLOCKED: {report.critical_vulns} critical vulnerabilities found")
            return False

        if report.risk_score > 50:
            print(f"WARNING: High risk score {report.risk_score:.0f}")

        return True

if __name__ == "__main__":
    import sys
    pipeline = SecurityPipeline()
    image = sys.argv[1] if len(sys.argv) > 1 else "payment-service:latest"
    passed = pipeline.gate_deployment(image)
    sys.exit(0 if passed else 1)
PYTHON

  log "✓ Security scanning pipeline configured"
}

main() {
  log "Starting Zero Trust Platform setup..."
  setup_spire
  setup_vault_dynamic_secrets
  setup_opa_gatekeeper
  setup_security_scanning
  log ""
  log "╔═══════════════════════════════════════════════════════╗"
  log "║         Zero Trust Platform Summary                    ║"
  log "╠═══════════════════════════════════════════════════════╣"
  log "║  Identity:   SPIFFE/SPIRE (SVID X.509 certs)          ║"
  log "║  Secrets:    Vault HA + KMS auto-unseal                ║"
  log "║  Dynamic DB: Vault PostgreSQL credentials (1h TTL)     ║"
  log "║  Policies:   OPA Gatekeeper (deny privileged)          ║"
  log "║  SBOM:       syft + cosign attestation                 ║"
  log "║  Network:    Istio Ambient mTLS everywhere             ║"
  log "╚═══════════════════════════════════════════════════════╝"
}
main "$@"
```

---

## ขั้นตอนที่ 588: Advanced Kubernetes Networking

### Advanced Networking Features

```bash
#!/usr/bin/env bash
# advanced-k8s-networking-2.sh — eBPF, BGP, IPv6 Dual-Stack
set -euo pipefail

LOG_FILE="/var/log/advanced-networking.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Dual-Stack IPv4/IPv6 ─────────────────────────────────────────────────

setup_dual_stack() {
  log "=== Setting up Dual-Stack IPv4/IPv6 ==="

  # EKS Dual-Stack cluster config
  cat <<'EOF' > /tmp/eks-dual-stack.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: dual-stack-cluster
  region: us-east-1

kubernetesNetworkConfig:
  ipFamily: IPv6

addons:
  - name: vpc-cni
    version: latest
    configurationValues: |
      env:
        ENABLE_IPv6: "true"
        ENABLE_PREFIX_DELEGATION: "true"
        ENABLE_POD_ENI: "true"
EOF

  # Dual-stack Service
  cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: dual-stack-service
  namespace: production
spec:
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 8080
  ipFamilies:
    - IPv4
    - IPv6
  ipFamilyPolicy: RequireDualStack
  type: LoadBalancer
EOF

  log "✓ Dual-Stack IPv4/IPv6 configured"
}

# ─── 2. BGP Load Balancing with MetalLB ──────────────────────────────────────

setup_metallb_bgp() {
  log "=== Setting up MetalLB BGP ==="

  helm repo add metallb https://metallb.github.io/metallb
  helm upgrade --install metallb metallb/metallb \
    --namespace metallb-system \
    --create-namespace \
    --set controller.tolerations[0].key=node-role.kubernetes.io/control-plane \
    --set controller.tolerations[0].operator=Exists

  cat <<'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: datacenter-router
  namespace: metallb-system
spec:
  myASN: 64512
  peerASN: 64513
  peerAddress: 192.168.1.1
  password: bgp-secret
  holdTime: 90s
  keepaliveTime: 30s
  gracefulRestart:
    enabled: true
    restartTime: 120s
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: production-services
  namespace: metallb-system
spec:
  ipAddressPools:
    - production-pool
  communities:
    - 64512:100  # no-export community
  aggregationLength: 32
  aggregationLengthV6: 128
  localPref: 100
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.100.0/24
  autoAssign: true
  avoidBuggyIPs: true
EOF

  log "✓ MetalLB BGP configured"
}

# ─── 3. Network Policies with Cilium ─────────────────────────────────────────

setup_cilium_advanced() {
  log "=== Setting up Advanced Cilium Policies ==="

  # DNS-based Network Policy
  cat <<'EOF' | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: dns-based-egress
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-service
  egress:
    # Allow DNS
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s:k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
          rules:
            dns:
              - matchPattern: "*"
    # Allow external APIs by FQDN
    - toFQDNs:
        - matchName: api.stripe.com
        - matchName: api.braintree.com
        - matchPattern: "*.paypal.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    # Allow internal services
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: production
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
---
# L7 HTTP Policy with path-based rules
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: http-l7-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api-gateway
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: /api/v1/products.*
              - method: POST
                path: /api/v1/orders
                headers:
                  - "Content-Type: application/json"
              - method: GET
                path: /health
              - method: GET
                path: /metrics
    # Block all other HTTP methods/paths
    - fromEndpoints:
        - matchLabels: {}  # any
      toPorts:
        - ports:
            - port: "8080"
          rules:
            http:
              - method: GET
                path: /health  # Only allow health check from anywhere
EOF

  log "✓ Advanced Cilium policies configured"
}

main() {
  log "Starting Advanced Kubernetes Networking..."
  setup_dual_stack
  setup_metallb_bgp
  setup_cilium_advanced
  log "✓ Advanced Networking complete"
}
main "$@"
```

---

## สรุป Part 59

| ขั้นตอน | หัวข้อ | เทคโนโลยีหลัก |
|---------|--------|---------------|
| 585 | Blockchain Infrastructure | Ethereum Geth+Lighthouse, Hyperledger Fabric, The Graph |
| 586 | Advanced Service Mesh | Istio Ambient Mode, Waypoint Proxy, Kiali, MCS |
| 587 | Zero Trust Platform | SPIFFE/SPIRE, Vault HA+KMS, OPA Gatekeeper, SBOM |
| 588 | Advanced K8s Networking | Dual-stack IPv4/IPv6, MetalLB BGP, Cilium L7 HTTP policies |

### แนวคิดสำคัญ

1. **Istio Ambient Mode** — ลด overhead โดยใช้ node-level ztunnel แทน per-pod sidecar, L7 routing ทำผ่าน Waypoint
2. **SPIFFE/SPIRE** — มาตรฐาน X.509 SVID สำหรับ workload identity ที่ rotate อัตโนมัติทุก 1 ชั่วโมง
3. **Vault Dynamic Secrets** — Database credentials ที่สร้างและ revoke อัตโนมัติ ไม่เก็บ static credentials
4. **OPA Gatekeeper** — Admission webhook ที่บังคับ policy ก่อน object เข้า cluster
5. **Cilium L7 HTTP** — Network policy ระดับ HTTP path/method/header โดยไม่ต้อง service mesh

---

ขั้นตอนต่อไป: **Part 60** — Machine Learning Platform, MLOps Pipeline และ Feature Store
