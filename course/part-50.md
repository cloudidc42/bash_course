# Part 50: Module 4 Completion — Advanced Networking and Service Mesh (ขั้นตอนที่ 549-552)

## สรุป Module 4 Professional Level และ Advanced Networking

---

## ขั้นตอนที่ 549: Advanced Kubernetes Networking

```bash
#!/bin/bash
# advanced-k8s-networking.sh
# Advanced Kubernetes Networking: CNI, eBPF, Network Policies, Multi-cluster

set -euo pipefail

CLUSTER_NAME="${CLUSTER_NAME:-production}"
NAMESPACE="${NAMESPACE:-networking}"
CILIUM_VERSION="${CILIUM_VERSION:-1.14.0}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Cilium CNI Installation ====================
install_cilium_cni() {
    log "Installing Cilium CNI with eBPF..."
    
    helm repo add cilium https://helm.cilium.io/
    helm repo update
    
    # Install Cilium with advanced features
    helm upgrade --install cilium cilium/cilium \
        --version "${CILIUM_VERSION}" \
        --namespace kube-system \
        --set kubeProxyReplacement=strict \
        --set k8sServiceHost="${API_SERVER_HOST:-auto}" \
        --set k8sServicePort="${API_SERVER_PORT:-6443}" \
        --set hubble.relay.enabled=true \
        --set hubble.ui.enabled=true \
        --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}" \
        --set bandwidthManager.enabled=true \
        --set loadBalancer.algorithm=maglev \
        --set maglev.tableSize=16381 \
        --set localRedirectPolicy=true \
        --set endpointRoutes.enabled=true \
        --set ipv6.enabled=false \
        --set prometheus.enabled=true \
        --set operator.prometheus.enabled=true \
        --set clustermesh.useAPIServer=true \
        --wait
    
    log "Cilium CNI installed successfully"
}

# ==================== Network Policy Advanced ====================
create_advanced_network_policies() {
    log "Creating advanced network policies..."
    
    # Default deny all ingress/egress
    cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

    # Allow DNS egress
    cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF

    # Cilium Network Policy with L7 inspection
    cat <<'EOF' | kubectl apply -f -
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: api-gateway-l7-policy
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
        - method: "GET"
          path: "/api/v1/.*"
        - method: "POST"
          path: "/api/v1/.*"
          headers:
          - "Content-Type: application/json"
  egress:
  - toEndpoints:
    - matchLabels:
        app: backend-service
    toPorts:
    - ports:
      - port: "9090"
        protocol: TCP
  - toFQDNs:
    - matchPattern: "*.internal.company.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
EOF

    log "Advanced network policies created"
}

# ==================== Cilium Cluster Mesh ====================
setup_cluster_mesh() {
    log "Setting up Cilium Cluster Mesh for multi-cluster connectivity..."
    
    local PRIMARY_CLUSTER="${1:-cluster1}"
    local SECONDARY_CLUSTER="${2:-cluster2}"
    
    # Enable cluster mesh on primary
    cilium clustermesh enable \
        --context "kind-${PRIMARY_CLUSTER}" \
        --service-type LoadBalancer
    
    # Enable cluster mesh on secondary
    cilium clustermesh enable \
        --context "kind-${SECONDARY_CLUSTER}" \
        --service-type LoadBalancer
    
    # Connect clusters
    cilium clustermesh connect \
        --context "kind-${PRIMARY_CLUSTER}" \
        --destination-context "kind-${SECONDARY_CLUSTER}"
    
    # Verify mesh status
    cilium clustermesh status --context "kind-${PRIMARY_CLUSTER}" --wait
    
    # Create global service for cross-cluster load balancing
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: global-backend
  namespace: production
  annotations:
    service.cilium.io/global: "true"
    service.cilium.io/shared: "true"
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
EOF

    log "Cluster Mesh configured"
}

# ==================== eBPF Traffic Monitoring ====================
setup_hubble_monitoring() {
    log "Setting up Hubble eBPF traffic monitoring..."
    
    # Install Hubble CLI
    export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
    curl -L --fail --remote-name-all \
        "https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-amd64.tar.gz"
    tar xzvf hubble-linux-amd64.tar.gz
    sudo mv hubble /usr/local/bin/
    
    # Port forward Hubble relay
    kubectl port-forward -n kube-system svc/hubble-relay 4245:80 &
    
    # Observe flows
    hubble observe --namespace production --last 100
    
    # Observe specific pod flows
    hubble observe \
        --namespace production \
        --pod api-gateway \
        --type trace:to-endpoint \
        --last 50
    
    # Create Hubble flow filter for security monitoring
    cat <<'EOF' > /tmp/hubble-security-filter.yaml
# Monitor dropped packets
filters:
  - verdict: DROPPED
    namespace: production
  - verdict: ERROR
    namespace: production
EOF

    log "Hubble monitoring configured"
}

# ==================== BGP Configuration ====================
setup_bgp_load_balancing() {
    log "Setting up BGP for Load Balancer IP advertisement..."
    
    # CiliumBGPPeeringPolicy
    cat <<'EOF' | kubectl apply -f -
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPPeeringPolicy
metadata:
  name: rack0-bgp-peering
spec:
  nodeSelector:
    matchLabels:
      rack: rack0
  virtualRouters:
  - localASN: 64512
    exportPodCIDR: true
    serviceSelector:
      matchExpressions:
      - key: somekey
        operator: NotIn
        values:
        - never-used-value
    neighbors:
    - peerAddress: 192.168.1.1/32
      peerASN: 64500
      eBGPMultihopTTL: 10
      connectRetryTimeSeconds: 120
      holdTimeSeconds: 90
      keepAliveTimeSeconds: 30
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 120
EOF

    # LoadBalancer IP Pool
    cat <<'EOF' | kubectl apply -f -
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: production-pool
spec:
  cidrs:
  - cidr: 10.0.10.0/24
EOF

    log "BGP load balancing configured"
}

# ==================== Service Mesh with Istio Advanced ====================
setup_istio_advanced() {
    log "Setting up advanced Istio service mesh features..."
    
    # Install Istio with advanced config
    cat <<'EOF' > /tmp/istio-config.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: production-istio
  namespace: istio-system
spec:
  profile: production
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        zipkin:
          address: jaeger-collector.observability:9411
        sampling: 100.0
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
    enableEnvoyAccessLogService: true
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
    localityLbSetting:
      enabled: true
      distribute:
      - from: us-east1/us-east1-a/*
        to:
          us-east1/us-east1-a/*: 70
          us-east1/us-east1-b/*: 20
          us-east1/us-east1-c/*: 10
    trustDomain: cluster.local
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 500m
            memory: 2048Mi
          limits:
            cpu: 2000m
            memory: 4096Mi
        hpaSpec:
          minReplicas: 2
          maxReplicas: 5
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        serviceAnnotations:
          service.beta.kubernetes.io/aws-load-balancer-type: nlb
          service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
        hpaSpec:
          minReplicas: 3
          maxReplicas: 10
    egressGateways:
    - name: istio-egressgateway
      enabled: true
  values:
    global:
      mtls:
        enabled: true
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 2000m
            memory: 1024Mi
      tracer:
        zipkin:
          address: jaeger-collector.observability:9411
EOF

    istioctl install -f /tmp/istio-config.yaml --verify
    
    # Istio AuthorizationPolicy
    cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: production-authz
  namespace: production
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/production/sa/api-gateway"
        - "cluster.local/ns/ingress/sa/ingress-controller"
    to:
    - operation:
        methods: ["GET", "POST", "PUT", "DELETE"]
        paths: ["/api/*"]
    when:
    - key: request.auth.claims[iss]
      values: ["https://auth.company.com"]
EOF

    log "Advanced Istio configured"
}

# ==================== Network Observability ====================
setup_network_observability() {
    log "Setting up network observability stack..."
    
    # Install Network Observability Operator (OpenShift) or similar
    cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: network-observer
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: network-observer
  template:
    metadata:
      labels:
        app: network-observer
    spec:
      serviceAccountName: network-observer
      containers:
      - name: observer
        image: ghcr.io/netobserv/network-observability-operator:latest
        env:
        - name: LOG_LEVEL
          value: info
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
EOF

    # Netflow/IPFIX collector
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: netflow-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      netflow:
        endpoint: 0.0.0.0:2055
    exporters:
      prometheus:
        endpoint: 0.0.0.0:8889
      loki:
        endpoint: http://loki:3100/loki/api/v1/push
        labels:
          job: netflow
    service:
      pipelines:
        logs:
          receivers: [netflow]
          exporters: [prometheus, loki]
EOF

    log "Network observability configured"
}

# ==================== Main Execution ====================
main() {
    log "Starting Advanced Kubernetes Networking setup..."
    
    case "${1:-all}" in
        cilium)    install_cilium_cni ;;
        policies)  create_advanced_network_policies ;;
        mesh)      setup_cluster_mesh ;;
        hubble)    setup_hubble_monitoring ;;
        bgp)       setup_bgp_load_balancing ;;
        istio)     setup_istio_advanced ;;
        observe)   setup_network_observability ;;
        all)
            install_cilium_cni
            create_advanced_network_policies
            setup_hubble_monitoring
            setup_bgp_load_balancing
            setup_istio_advanced
            setup_network_observability
            ;;
    esac
    
    log "Advanced networking setup complete!"
}

main "$@"
```

---

## ขั้นตอนที่ 550: Multi-Region Architecture และ Disaster Recovery

```bash
#!/bin/bash
# multi-region-dr.sh
# Multi-Region Architecture and Disaster Recovery Automation

set -euo pipefail

PRIMARY_REGION="${PRIMARY_REGION:-us-east-1}"
DR_REGION="${DR_REGION:-us-west-2}"
APP_NAME="${APP_NAME:-production-app}"
RTO_HOURS="${RTO_HOURS:-1}"
RPO_MINUTES="${RPO_MINUTES:-15}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] [$1] $2"; }

# ==================== Route53 Health-based Failover ====================
setup_route53_failover() {
    log "INFO" "Setting up Route53 multi-region failover..."
    
    local HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name \
        --dns-name "company.com" \
        --query "HostedZones[0].Id" --output text | cut -d'/' -f3)
    
    # Create health checks for each region
    PRIMARY_HEALTH_ID=$(aws route53 create-health-check \
        --caller-reference "primary-$(date +%s)" \
        --health-check-config '{
            "Type": "HTTPS",
            "FullyQualifiedDomainName": "api-us-east.company.com",
            "Port": 443,
            "ResourcePath": "/health",
            "RequestInterval": 10,
            "FailureThreshold": 2,
            "EnableSNI": true
        }' \
        --query "HealthCheck.Id" --output text)
    
    DR_HEALTH_ID=$(aws route53 create-health-check \
        --caller-reference "dr-$(date +%s)" \
        --health-check-config '{
            "Type": "HTTPS",
            "FullyQualifiedDomainName": "api-us-west.company.com",
            "Port": 443,
            "ResourcePath": "/health",
            "RequestInterval": 10,
            "FailureThreshold": 2,
            "EnableSNI": true
        }' \
        --query "HealthCheck.Id" --output text)
    
    # Add CloudWatch alarm to health check
    aws route53 update-health-check \
        --health-check-id "${PRIMARY_HEALTH_ID}" \
        --alarm-identifier '{
            "Region": "us-east-1",
            "Name": "api-primary-health"
        }' \
        --insufficient-data-health-status Healthy
    
    # Create failover DNS records
    aws route53 change-resource-record-sets \
        --hosted-zone-id "${HOSTED_ZONE_ID}" \
        --change-batch "{
            \"Changes\": [
                {
                    \"Action\": \"UPSERT\",
                    \"ResourceRecordSet\": {
                        \"Name\": \"api.company.com\",
                        \"Type\": \"A\",
                        \"SetIdentifier\": \"primary\",
                        \"Failover\": \"PRIMARY\",
                        \"AliasTarget\": {
                            \"HostedZoneId\": \"Z35SXDOTRQ7X7K\",
                            \"DNSName\": \"primary-lb.us-east-1.elb.amazonaws.com\",
                            \"EvaluateTargetHealth\": true
                        },
                        \"HealthCheckId\": \"${PRIMARY_HEALTH_ID}\"
                    }
                },
                {
                    \"Action\": \"UPSERT\",
                    \"ResourceRecordSet\": {
                        \"Name\": \"api.company.com\",
                        \"Type\": \"A\",
                        \"SetIdentifier\": \"secondary\",
                        \"Failover\": \"SECONDARY\",
                        \"AliasTarget\": {
                            \"HostedZoneId\": \"Z1H1FL5HABSF5\",
                            \"DNSName\": \"dr-lb.us-west-2.elb.amazonaws.com\",
                            \"EvaluateTargetHealth\": true
                        },
                        \"HealthCheckId\": \"${DR_HEALTH_ID}\"
                    }
                }
            ]
        }"
    
    log "INFO" "Route53 failover configured. Primary: ${PRIMARY_HEALTH_ID}, DR: ${DR_HEALTH_ID}"
}

# ==================== RDS Multi-Region Replication ====================
setup_rds_replication() {
    log "INFO" "Setting up RDS cross-region replication..."
    
    local PRIMARY_DB_ID="${APP_NAME}-primary"
    local DR_DB_ID="${APP_NAME}-dr-replica"
    
    # Create read replica in DR region
    aws rds create-db-instance-read-replica \
        --db-instance-identifier "${DR_DB_ID}" \
        --source-db-instance-identifier "arn:aws:rds:${PRIMARY_REGION}:$(aws sts get-caller-identity --query Account --output text):db:${PRIMARY_DB_ID}" \
        --db-instance-class db.r6g.xlarge \
        --availability-zone "${DR_REGION}a" \
        --publicly-accessible false \
        --multi-az true \
        --auto-minor-version-upgrade true \
        --deletion-protection true \
        --tags Key=Environment,Value=dr Key=App,Value="${APP_NAME}" \
        --region "${DR_REGION}"
    
    # Wait for replica
    aws rds wait db-instance-available \
        --db-instance-identifier "${DR_DB_ID}" \
        --region "${DR_REGION}"
    
    # Enable automated backups
    aws rds modify-db-instance \
        --db-instance-identifier "${DR_DB_ID}" \
        --backup-retention-period 7 \
        --preferred-backup-window "03:00-04:00" \
        --region "${DR_REGION}" \
        --apply-immediately
    
    log "INFO" "RDS replication configured: ${DR_DB_ID} in ${DR_REGION}"
}

# ==================== S3 Cross-Region Replication ====================
setup_s3_replication() {
    log "INFO" "Setting up S3 cross-region replication..."
    
    local PRIMARY_BUCKET="${APP_NAME}-primary-${PRIMARY_REGION}"
    local DR_BUCKET="${APP_NAME}-dr-${DR_REGION}"
    local REPLICATION_ROLE_ARN="arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/S3ReplicationRole"
    
    # Enable versioning on both buckets
    aws s3api put-bucket-versioning \
        --bucket "${PRIMARY_BUCKET}" \
        --versioning-configuration Status=Enabled
    
    aws s3api put-bucket-versioning \
        --bucket "${DR_BUCKET}" \
        --versioning-configuration Status=Enabled \
        --region "${DR_REGION}"
    
    # Configure replication
    aws s3api put-bucket-replication \
        --bucket "${PRIMARY_BUCKET}" \
        --replication-configuration "{
            \"Role\": \"${REPLICATION_ROLE_ARN}\",
            \"Rules\": [
                {
                    \"ID\": \"full-replication\",
                    \"Status\": \"Enabled\",
                    \"Filter\": {\"Prefix\": \"\"},
                    \"Destination\": {
                        \"Bucket\": \"arn:aws:s3:::${DR_BUCKET}\",
                        \"StorageClass\": \"STANDARD_IA\",
                        \"ReplicationTime\": {
                            \"Status\": \"Enabled\",
                            \"Time\": {\"Minutes\": 15}
                        },
                        \"Metrics\": {
                            \"Status\": \"Enabled\",
                            \"EventThreshold\": {\"Minutes\": 15}
                        }
                    },
                    \"DeleteMarkerReplication\": {\"Status\": \"Enabled\"}
                }
            ]
        }"
    
    log "INFO" "S3 replication configured: ${PRIMARY_BUCKET} -> ${DR_BUCKET}"
}

# ==================== EKS Multi-Region Setup ====================
setup_eks_dr() {
    log "INFO" "Setting up EKS DR cluster..."
    
    # Create DR cluster using eksctl
    cat <<EOF > /tmp/dr-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${APP_NAME}-dr
  region: ${DR_REGION}
  version: "1.28"
  tags:
    Environment: dr
    App: ${APP_NAME}
vpc:
  cidr: 172.20.0.0/16
  enableDnsHostnames: true
  enableDnsSupport: true
managedNodeGroups:
- name: dr-nodes
  instanceType: m6i.2xlarge
  minSize: 2
  maxSize: 10
  desiredCapacity: 3
  availabilityZones:
  - ${DR_REGION}a
  - ${DR_REGION}b
  - ${DR_REGION}c
  iam:
    attachPolicyARNs:
    - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
    - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
    - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
  tags:
    k8s.io/cluster-autoscaler/enabled: "true"
    k8s.io/cluster-autoscaler/${APP_NAME}-dr: "owned"
addons:
- name: vpc-cni
- name: coredns
- name: kube-proxy
- name: aws-ebs-csi-driver
cloudWatch:
  clusterLogging:
    enableTypes:
    - api
    - audit
    - authenticator
EOF

    eksctl create cluster -f /tmp/dr-cluster.yaml
    
    log "INFO" "EKS DR cluster created in ${DR_REGION}"
}

# ==================== Velero Backup and Restore ====================
setup_velero_dr() {
    log "INFO" "Setting up Velero for K8s backup/restore..."
    
    local BACKUP_BUCKET="${APP_NAME}-velero-backup"
    
    # Install Velero with AWS plugin
    velero install \
        --provider aws \
        --plugins velero/velero-plugin-for-aws:v1.7.0 \
        --bucket "${BACKUP_BUCKET}" \
        --backup-location-config region="${PRIMARY_REGION}" \
        --snapshot-location-config region="${PRIMARY_REGION}" \
        --secret-file /tmp/velero-credentials \
        --use-node-agent \
        --wait
    
    # Create backup schedule
    cat <<'EOF' | kubectl apply -f -
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: production-hourly-backup
  namespace: velero
spec:
  schedule: "0 * * * *"
  template:
    includedNamespaces:
    - production
    - monitoring
    - databases
    snapshotVolumes: true
    storageLocation: default
    volumeSnapshotLocations:
    - default
    ttl: 720h0m0s
    labelSelector:
      matchLabels:
        backup: "true"
EOF

    # Create cross-region backup location
    cat <<EOF | kubectl apply -f -
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: dr-region
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: ${APP_NAME}-velero-dr
  config:
    region: ${DR_REGION}
EOF

    log "INFO" "Velero DR configured"
}

# ==================== DR Runbook Automation ====================
dr_failover_runbook() {
    log "INFO" "Executing DR Failover Runbook..."
    
    local FAILOVER_TYPE="${1:-full}"
    local INCIDENT_ID="${2:-$(date +%Y%m%d%H%M%S)}"
    
    echo "=== DISASTER RECOVERY FAILOVER ==="
    echo "Incident ID: ${INCIDENT_ID}"
    echo "Failover Type: ${FAILOVER_TYPE}"
    echo "RTO Target: ${RTO_HOURS}h | RPO Target: ${RPO_MINUTES}min"
    echo "Started at: $(date)"
    echo ""
    
    # Step 1: Verify DR cluster health
    log "STEP 1" "Verifying DR cluster health..."
    aws eks update-kubeconfig --name "${APP_NAME}-dr" --region "${DR_REGION}"
    kubectl get nodes --context "arn:aws:eks:${DR_REGION}:$(aws sts get-caller-identity --query Account --output text):cluster/${APP_NAME}-dr"
    
    # Step 2: Check replication lag
    log "STEP 2" "Checking replication lag..."
    local LAG=$(aws rds describe-db-instances \
        --db-instance-identifier "${APP_NAME}-dr-replica" \
        --query "DBInstances[0].ReplicaLag" \
        --output text \
        --region "${DR_REGION}" 2>/dev/null || echo "N/A")
    log "INFO" "Replication lag: ${LAG} seconds"
    
    # Step 3: Promote DR database
    if [[ "${FAILOVER_TYPE}" == "full" ]]; then
        log "STEP 3" "Promoting DR replica to primary..."
        aws rds promote-read-replica \
            --db-instance-identifier "${APP_NAME}-dr-replica" \
            --region "${DR_REGION}"
        
        aws rds wait db-instance-available \
            --db-instance-identifier "${APP_NAME}-dr-replica" \
            --region "${DR_REGION}"
        log "INFO" "DR replica promoted to primary"
    fi
    
    # Step 4: Restore K8s workloads from Velero
    log "STEP 4" "Restoring K8s workloads from latest backup..."
    LATEST_BACKUP=$(velero backup get --context dr --output json 2>/dev/null | \
        python3 -c "import json,sys; backups=json.load(sys.stdin)['items']; \
        backups.sort(key=lambda x: x['metadata']['creationTimestamp']); \
        print(backups[-1]['metadata']['name'])" 2>/dev/null || echo "manual-restore-needed")
    
    if [[ "${LATEST_BACKUP}" != "manual-restore-needed" ]]; then
        velero restore create --from-backup "${LATEST_BACKUP}" \
            --include-namespaces production \
            --wait
    fi
    
    # Step 5: Update DNS
    log "STEP 5" "Updating Route53 to DR region..."
    # DNS will auto-failover via health checks, but force update if needed
    
    # Step 6: Verify services
    log "STEP 6" "Verifying DR services..."
    kubectl get deployments -n production
    kubectl get services -n production
    
    # Step 7: Send notification
    log "STEP 7" "Sending failover notification..."
    aws sns publish \
        --topic-arn "arn:aws:sns:${PRIMARY_REGION}:$(aws sts get-caller-identity --query Account --output text):dr-notifications" \
        --message "DR Failover completed. Incident: ${INCIDENT_ID}. Services now running in ${DR_REGION}"
    
    log "INFO" "DR Failover complete! Incident: ${INCIDENT_ID}"
}

# ==================== DR Test ====================
test_dr_readiness() {
    log "INFO" "Testing DR readiness..."
    
    local SCORE=0
    local TOTAL=10
    
    # Test 1: Cluster accessibility
    if aws eks describe-cluster --name "${APP_NAME}-dr" --region "${DR_REGION}" &>/dev/null; then
        log "PASS" "DR cluster accessible"
        ((SCORE++))
    else
        log "FAIL" "DR cluster not accessible"
    fi
    
    # Test 2: Database replica status
    local DB_STATUS=$(aws rds describe-db-instances \
        --db-instance-identifier "${APP_NAME}-dr-replica" \
        --query "DBInstances[0].DBInstanceStatus" \
        --output text --region "${DR_REGION}" 2>/dev/null)
    if [[ "${DB_STATUS}" == "available" ]]; then
        log "PASS" "DR database replica available"
        ((SCORE++))
    else
        log "FAIL" "DR database status: ${DB_STATUS}"
    fi
    
    # Test 3: S3 replication status
    local REPL_STATUS=$(aws s3api get-bucket-replication \
        --bucket "${APP_NAME}-primary-${PRIMARY_REGION}" \
        --query "ReplicationConfiguration.Rules[0].Status" \
        --output text 2>/dev/null)
    if [[ "${REPL_STATUS}" == "Enabled" ]]; then
        log "PASS" "S3 replication enabled"
        ((SCORE++))
    else
        log "FAIL" "S3 replication not enabled"
    fi
    
    # Test 4: Velero backup freshness (< 2 hours)
    local BACKUP_AGE=$(velero backup get --output json 2>/dev/null | \
        python3 -c "
import json, sys
from datetime import datetime, timezone
data = json.load(sys.stdin)
if data['items']:
    last = sorted(data['items'], key=lambda x: x['metadata']['creationTimestamp'])[-1]
    created = datetime.fromisoformat(last['metadata']['creationTimestamp'].replace('Z', '+00:00'))
    age = (datetime.now(timezone.utc) - created).total_seconds() / 3600
    print(f'{age:.1f}')
else:
    print('99')
" 2>/dev/null || echo "99")
    
    if (( $(echo "${BACKUP_AGE} < 2" | bc -l) )); then
        log "PASS" "Velero backup fresh: ${BACKUP_AGE}h"
        ((SCORE++))
    else
        log "FAIL" "Velero backup stale: ${BACKUP_AGE}h"
    fi
    
    echo ""
    echo "=== DR READINESS SCORE: ${SCORE}/${TOTAL} ==="
    
    if (( SCORE >= 8 )); then
        echo "STATUS: READY (GREEN)"
    elif (( SCORE >= 5 )); then
        echo "STATUS: PARTIALLY READY (YELLOW) - Review failed checks"
    else
        echo "STATUS: NOT READY (RED) - Immediate action required"
    fi
}

# ==================== Main ====================
main() {
    case "${1:-status}" in
        route53)   setup_route53_failover ;;
        rds)       setup_rds_replication ;;
        s3)        setup_s3_replication ;;
        eks)       setup_eks_dr ;;
        velero)    setup_velero_dr ;;
        failover)  dr_failover_runbook "${2:-full}" "${3:-$(date +%Y%m%d%H%M%S)}" ;;
        test)      test_dr_readiness ;;
        all)
            setup_route53_failover
            setup_rds_replication
            setup_s3_replication
            setup_velero_dr
            test_dr_readiness
            ;;
        *)
            echo "Usage: $0 {route53|rds|s3|eks|velero|failover|test|all}"
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 551: Platform Engineering — Backstage Developer Portal

```bash
#!/bin/bash
# backstage-platform.sh
# Backstage Developer Portal for Internal Developer Platform (IDP)

set -euo pipefail

BACKSTAGE_NAMESPACE="${BACKSTAGE_NAMESPACE:-backstage}"
BACKSTAGE_VERSION="${BACKSTAGE_VERSION:-1.20.0}"
DOMAIN="${DOMAIN:-backstage.company.com}"
GITHUB_ORG="${GITHUB_ORG:-myorg}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Backstage App Configuration ====================
generate_backstage_config() {
    log "Generating Backstage app-config.yaml..."
    
    cat <<EOF > /tmp/app-config.production.yaml
app:
  title: ${GITHUB_ORG} Developer Portal
  baseUrl: https://${DOMAIN}
  
backend:
  baseUrl: https://${DOMAIN}
  listen:
    port: 7007
  csp:
    connect-src: ["'self'", 'http:', 'https:']
  cors:
    origin: https://${DOMAIN}
    methods: [GET, HEAD, PATCH, POST, PUT, DELETE]
    credentials: true
  database:
    client: pg
    connection:
      host: \${POSTGRES_HOST}
      port: 5432
      user: \${POSTGRES_USER}
      password: \${POSTGRES_PASSWORD}
      database: backstage
      ssl:
        require: true
        rejectUnauthorized: false
  cache:
    store: redis
    connection:
      host: \${REDIS_HOST}
      port: 6379
      password: \${REDIS_PASSWORD}

integrations:
  github:
  - host: github.com
    apps:
    - appId: \${GITHUB_APP_ID}
      webhookUrl: https://${DOMAIN}/api/catalog/github/webhook
      clientId: \${GITHUB_CLIENT_ID}
      clientSecret: \${GITHUB_CLIENT_SECRET}
      webhookSecret: \${GITHUB_WEBHOOK_SECRET}
      privateKey: |
        \${GITHUB_PRIVATE_KEY}

catalog:
  import:
    entityFilename: catalog-info.yaml
    pullRequestBranchName: backstage-integration
  rules:
  - allow:
    - Component
    - API
    - Resource
    - Location
    - System
    - Domain
    - Group
    - User
    - Template
  providers:
    github:
      providerId:
        organization: '${GITHUB_ORG}'
        catalogPath: '/catalog-info.yaml'
        filters:
          branch: 'main'
          repository: '.*'
        schedule:
          frequency: { minutes: 30 }
          timeout: { minutes: 3 }
    awsS3:
      yourOrganizationAws:
        bucketName: backstage-catalog
        prefix: catalog/
        region: us-east-1
        schedule:
          frequency: { minutes: 60 }
          timeout: { minutes: 5 }
    kubernetes:
      clusterLocatorMethods:
      - type: config
        clusters:
        - url: https://kubernetes.default.svc
          name: production
          authProvider: serviceAccount
          skipTLSVerify: false

kubernetes:
  serviceLocatorMethod:
    type: multiTenant
  clusterLocatorMethods:
  - type: config
    clusters:
    - url: \${K8S_URL}
      name: production
      authProvider: serviceAccount
      serviceAccountToken: \${K8S_SA_TOKEN}
      caData: \${K8S_CA_DATA}
  - type: config
    clusters:
    - url: \${K8S_DR_URL}
      name: dr-cluster
      authProvider: serviceAccount
      serviceAccountToken: \${K8S_DR_SA_TOKEN}

auth:
  session:
    secret: \${AUTH_SESSION_SECRET}
  providers:
    github:
      development:
        clientId: \${GITHUB_CLIENT_ID}
        clientSecret: \${GITHUB_CLIENT_SECRET}
    oidc:
      development:
        metadataUrl: https://auth.company.com/.well-known/openid-configuration
        clientId: \${OIDC_CLIENT_ID}
        clientSecret: \${OIDC_CLIENT_SECRET}
        prompt: auto
        signIn:
          resolvers:
          - resolver: emailLocalPartMatchingUserEntityName
          - resolver: emailMatchingUserEntityProfileEmail

techdocs:
  builder: external
  generator:
    runIn: docker
  publisher:
    type: awsS3
    awsS3:
      bucketName: backstage-techdocs
      region: us-east-1

permission:
  enabled: true
  
sonarqube:
  instances:
  - name: default
    baseUrl: https://sonarqube.company.com
    apiKey: \${SONAR_TOKEN}

costInsights:
  engineerCost: 200000
  products:
    computeEngine:
      name: Compute Engine
      icon: compute
    cloudStorage:
      name: Cloud Storage
      icon: storage
    bigQuery:
      name: BigQuery
      icon: data
  metrics:
    DAU:
      name: Daily Active Users
      default: true
    MSC:
      name: Monthly Subscribers

scaffolder:
  defaultAuthor:
    name: Backstage
    email: backstage@company.com
  defaultCommitMessage: "feat: scaffolded by Backstage"
EOF

    log "Backstage config generated"
}

# ==================== Backstage Kubernetes Deployment ====================
deploy_backstage_k8s() {
    log "Deploying Backstage to Kubernetes..."
    
    kubectl create namespace "${BACKSTAGE_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    # Create secrets
    kubectl create secret generic backstage-secrets \
        --namespace "${BACKSTAGE_NAMESPACE}" \
        --from-literal=POSTGRES_PASSWORD="${POSTGRES_PASSWORD:-changeme}" \
        --from-literal=GITHUB_CLIENT_SECRET="${GITHUB_CLIENT_SECRET:-}" \
        --from-literal=OIDC_CLIENT_SECRET="${OIDC_CLIENT_SECRET:-}" \
        --from-literal=AUTH_SESSION_SECRET="$(openssl rand -hex 32)" \
        --dry-run=client -o yaml | kubectl apply -f -
    
    # Create ConfigMap from app-config
    kubectl create configmap backstage-config \
        --namespace "${BACKSTAGE_NAMESPACE}" \
        --from-file=app-config.production.yaml=/tmp/app-config.production.yaml \
        --dry-run=client -o yaml | kubectl apply -f -
    
    # Deploy Backstage
    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backstage
  namespace: ${BACKSTAGE_NAMESPACE}
  labels:
    app: backstage
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backstage
  template:
    metadata:
      labels:
        app: backstage
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "7007"
    spec:
      serviceAccountName: backstage
      securityContext:
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
      containers:
      - name: backstage
        image: ghcr.io/${GITHUB_ORG}/backstage:${BACKSTAGE_VERSION}
        imagePullPolicy: Always
        ports:
        - containerPort: 7007
        readinessProbe:
          httpGet:
            path: /healthcheck
            port: 7007
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: 7007
          initialDelaySeconds: 60
          periodSeconds: 30
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        env:
        - name: NODE_ENV
          value: production
        - name: POSTGRES_HOST
          value: postgresql.databases.svc.cluster.local
        - name: POSTGRES_USER
          value: backstage
        - name: REDIS_HOST
          value: redis-master.databases.svc.cluster.local
        envFrom:
        - secretRef:
            name: backstage-secrets
        volumeMounts:
        - name: config
          mountPath: /app/app-config.production.yaml
          subPath: app-config.production.yaml
      volumes:
      - name: config
        configMap:
          name: backstage-config
---
apiVersion: v1
kind: Service
metadata:
  name: backstage
  namespace: ${BACKSTAGE_NAMESPACE}
spec:
  selector:
    app: backstage
  ports:
  - port: 80
    targetPort: 7007
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backstage
  namespace: ${BACKSTAGE_NAMESPACE}
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: 50m
spec:
  tls:
  - hosts:
    - ${DOMAIN}
    secretName: backstage-tls
  rules:
  - host: ${DOMAIN}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backstage
            port:
              number: 80
EOF

    log "Backstage deployed"
}

# ==================== Software Templates ====================
create_service_template() {
    log "Creating Backstage Software Templates..."
    
    mkdir -p /tmp/templates/microservice
    
    # Template definition
    cat <<'EOF' > /tmp/templates/microservice/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: python-microservice
  title: Python Microservice
  description: Create a production-ready Python microservice
  tags:
  - python
  - microservice
  - kubernetes
spec:
  owner: platform-team
  type: service
  parameters:
  - title: Service Information
    required:
    - name
    - description
    - owner
    properties:
      name:
        title: Service Name
        type: string
        description: The name of the service (lowercase, hyphens only)
        pattern: '^[a-z][a-z0-9-]*$'
      description:
        title: Description
        type: string
      owner:
        title: Team
        type: string
        ui:field: OwnerPicker
        ui:options:
          catalogFilter:
            kind: Group
      system:
        title: System
        type: string
        ui:field: EntityPicker
        ui:options:
          catalogFilter:
            kind: System
  - title: Infrastructure Options
    properties:
      database:
        title: Database
        type: boolean
        default: false
        description: Include PostgreSQL database
      cache:
        title: Cache
        type: boolean
        default: false
        description: Include Redis cache
      replicas:
        title: Initial Replicas
        type: number
        default: 2
        enum: [1, 2, 3, 5]
      cpu:
        title: CPU Request
        type: string
        default: "250m"
      memory:
        title: Memory Request
        type: string
        default: "512Mi"
  steps:
  - id: fetch-base
    name: Fetch Base Template
    action: fetch:template
    input:
      url: ./skeleton
      values:
        name: ${{ parameters.name }}
        description: ${{ parameters.description }}
        owner: ${{ parameters.owner }}
        replicas: ${{ parameters.replicas }}
        cpu: ${{ parameters.cpu }}
        memory: ${{ parameters.memory }}
  - id: publish
    name: Publish to GitHub
    action: publish:github
    input:
      allowedHosts: ['github.com']
      description: "Service: ${{ parameters.description }}"
      repoUrl: github.com?owner=${{ parameters.owner }}&repo=${{ parameters.name }}
      defaultBranch: main
      repoVisibility: private
      topics:
      - microservice
      - python
      - ${{ parameters.system }}
  - id: register
    name: Register Component
    action: catalog:register
    input:
      repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
      catalogInfoPath: /catalog-info.yaml
  output:
    links:
    - title: Repository
      url: ${{ steps.publish.output.remoteUrl }}
    - title: Open in Catalog
      icon: catalog
      entityRef: ${{ steps.register.output.entityRef }}
EOF

    # Push template to catalog
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: backstage-templates
  namespace: ${BACKSTAGE_NAMESPACE}
  labels:
    backstage.io/kubernetes-id: backstage
data:
  python-microservice.yaml: |
$(cat /tmp/templates/microservice/template.yaml | sed 's/^/    /')
EOF

    log "Software templates created"
}

# ==================== TechDocs Setup ====================
setup_techdocs() {
    log "Setting up TechDocs..."
    
    # Create S3 bucket for TechDocs
    aws s3api create-bucket \
        --bucket backstage-techdocs \
        --region "${AWS_REGION:-us-east-1}" \
        --create-bucket-configuration LocationConstraint="${AWS_REGION:-us-east-1}" \
        2>/dev/null || true
    
    # CORS for TechDocs
    aws s3api put-bucket-cors \
        --bucket backstage-techdocs \
        --cors-configuration '{
            "CORSRules": [
                {
                    "AllowedHeaders": ["*"],
                    "AllowedMethods": ["GET", "HEAD"],
                    "AllowedOrigins": ["https://'"${DOMAIN}"'"],
                    "ExposeHeaders": []
                }
            ]
        }'
    
    # mkdocs.yml template
    cat <<'EOF' > /tmp/mkdocs-template.yml
site_name: "{{{ name }}}"
site_description: "{{{ description }}}"
repo_url: "{{{ repoUrl }}}"
nav:
  - Home: index.md
  - Architecture: architecture.md
  - API Reference: api.md
  - Runbooks: runbooks.md
  - Changelog: changelog.md
plugins:
  - techdocs-core
  - search
markdown_extensions:
  - admonition
  - pymdownx.highlight
  - pymdownx.superfences
  - pymdownx.tabbed
  - tables
  - toc:
      permalink: true
EOF

    log "TechDocs configured"
}

# ==================== Main ====================
main() {
    case "${1:-all}" in
        config)    generate_backstage_config ;;
        deploy)    deploy_backstage_k8s ;;
        templates) create_service_template ;;
        techdocs)  setup_techdocs ;;
        all)
            generate_backstage_config
            deploy_backstage_k8s
            create_service_template
            setup_techdocs
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 552: Crossplane — Infrastructure as Code on Kubernetes

```bash
#!/bin/bash
# crossplane-platform.sh
# Crossplane: Universal Cloud Infrastructure Management on Kubernetes

set -euo pipefail

CROSSPLANE_VERSION="${CROSSPLANE_VERSION:-1.14.0}"
NAMESPACE="${NAMESPACE:-crossplane-system}"
AWS_REGION="${AWS_REGION:-us-east-1}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Crossplane Installation ====================
install_crossplane() {
    log "Installing Crossplane..."
    
    helm repo add crossplane-stable https://charts.crossplane.io/stable
    helm repo update
    
    helm upgrade --install crossplane \
        crossplane-stable/crossplane \
        --namespace "${NAMESPACE}" \
        --create-namespace \
        --version "${CROSSPLANE_VERSION}" \
        --set args='{--debug}' \
        --set resourcesCrossplane.limits.cpu=500m \
        --set resourcesCrossplane.limits.memory=512Mi \
        --wait
    
    log "Crossplane installed"
}

# ==================== AWS Provider ====================
setup_aws_provider() {
    log "Setting up Crossplane AWS Provider..."
    
    # Install AWS Provider
    cat <<'EOF' | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws:v0.46.0
  controllerConfigRef:
    name: provider-aws-config
---
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: provider-aws-config
spec:
  podSecurityContext:
    fsGroup: 2000
  resources:
    limits:
      cpu: 2000m
      memory: 2Gi
    requests:
      cpu: 500m
      memory: 1Gi
EOF

    # Wait for provider
    kubectl wait provider.pkg.crossplane.io/provider-aws \
        --for condition=Healthy \
        --timeout 600s
    
    # Create ProviderConfig with IAM Role
    cat <<EOF | kubectl apply -f -
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: IRSA
  assumeRoleChain:
  - roleARN: arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/CrossplaneRole
EOF

    log "AWS Provider configured"
}

# ==================== Composite Resource Definition (XRD) ====================
create_application_xrd() {
    log "Creating Application Composite Resource Definition..."
    
    # XRD - defines the API
    cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xapplicationenvironments.platform.company.com
spec:
  group: platform.company.com
  names:
    kind: XApplicationEnvironment
    plural: xapplicationenvironments
  claimNames:
    kind: ApplicationEnvironment
    plural: applicationenvironments
  connectionSecretKeys:
  - kubeconfig
  - dbConnectionString
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
                - name
                - environment
                - region
                properties:
                  name:
                    type: string
                    description: Application name
                  environment:
                    type: string
                    enum: [dev, staging, production]
                  region:
                    type: string
                    default: us-east-1
                  nodeSize:
                    type: string
                    default: medium
                    enum: [small, medium, large]
                  nodeCount:
                    type: integer
                    default: 3
                    minimum: 1
                    maximum: 100
                  database:
                    type: object
                    properties:
                      enabled:
                        type: boolean
                        default: true
                      class:
                        type: string
                        default: db.t3.medium
                      storage:
                        type: integer
                        default: 100
                  cache:
                    type: object
                    properties:
                      enabled:
                        type: boolean
                        default: true
                      nodeType:
                        type: string
                        default: cache.t3.medium
          status:
            type: object
            properties:
              clusterName:
                type: string
              dbEndpoint:
                type: string
              cacheEndpoint:
                type: string
EOF

    log "XRD created"
}

# ==================== Composition ====================
create_composition() {
    log "Creating Crossplane Composition..."
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: applicationenvironment-aws
  labels:
    provider: aws
    environment: production
spec:
  compositeTypeRef:
    apiVersion: platform.company.com/v1alpha1
    kind: XApplicationEnvironment
  writeConnectionSecretsToNamespace: crossplane-system
  
  patchSets:
  - name: common-parameters
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.region
      toFieldPath: spec.forProvider.region
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.name
      toFieldPath: metadata.labels[app]
  
  resources:
  # EKS Cluster
  - name: eks-cluster
    base:
      apiVersion: eks.aws.upbound.io/v1beta1
      kind: Cluster
      spec:
        forProvider:
          version: "1.28"
          roleArnSelector:
            matchControllerRef: true
          vpcConfig:
          - subnetIdSelector:
              matchControllerRef: true
            endpointPrivateAccess: true
            endpointPublicAccess: true
        writeConnectionSecretToRef:
          namespace: crossplane-system
    patches:
    - type: PatchSet
      patchSetName: common-parameters
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.name
      toFieldPath: metadata.name
      transforms:
      - type: string
        string:
          fmt: "%s-cluster"
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.name
      toFieldPath: status.clusterName
  
  # RDS PostgreSQL
  - name: rds-instance
    base:
      apiVersion: rds.aws.upbound.io/v1beta1
      kind: Instance
      spec:
        forProvider:
          engine: postgres
          engineVersion: "15.4"
          autoMinorVersionUpgrade: true
          deletionProtection: true
          storageEncrypted: true
          multiAz: true
          backupRetentionPeriod: 7
          dbSubnetGroupNameSelector:
            matchControllerRef: true
          vpcSecurityGroupIdSelector:
            matchControllerRef: true
          skipFinalSnapshot: false
          username: appuser
          passwordSecretRef:
            namespace: crossplane-system
            name: rds-password
            key: password
        writeConnectionSecretToRef:
          namespace: crossplane-system
    patches:
    - type: PatchSet
      patchSetName: common-parameters
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.name
      toFieldPath: metadata.name
      transforms:
      - type: string
        string:
          fmt: "%s-db"
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.database.class
      toFieldPath: spec.forProvider.instanceClass
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.database.storage
      toFieldPath: spec.forProvider.allocatedStorage
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.address
      toFieldPath: status.dbEndpoint
    readinessChecks:
    - type: MatchString
      fieldPath: status.atProvider.dbInstanceStatus
      matchString: available
  
  # ElastiCache Redis
  - name: elasticache-cluster
    base:
      apiVersion: elasticache.aws.upbound.io/v1beta1
      kind: ReplicationGroup
      spec:
        forProvider:
          engine: redis
          engineVersion: "7.0"
          automaticFailoverEnabled: true
          multiAzEnabled: true
          numCacheClusters: 2
          atRestEncryptionEnabled: true
          transitEncryptionEnabled: true
          subnetGroupNameSelector:
            matchControllerRef: true
          securityGroupIdSelector:
            matchControllerRef: true
    patches:
    - type: PatchSet
      patchSetName: common-parameters
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.name
      toFieldPath: metadata.name
      transforms:
      - type: string
        string:
          fmt: "%s-cache"
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.cache.nodeType
      toFieldPath: spec.forProvider.nodeType
    - type: ToCompositeFieldPath
      fromFieldPath: status.atProvider.primaryEndpointAddress
      toFieldPath: status.cacheEndpoint
EOF

    log "Composition created"
}

# ==================== Claim Example ====================
create_environment_claim() {
    log "Creating ApplicationEnvironment claim..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: platform.company.com/v1alpha1
kind: ApplicationEnvironment
metadata:
  name: my-app-production
  namespace: production
spec:
  parameters:
    name: my-app
    environment: production
    region: ${AWS_REGION}
    nodeSize: large
    nodeCount: 5
    database:
      enabled: true
      class: db.r6g.xlarge
      storage: 500
    cache:
      enabled: true
      nodeType: cache.r6g.large
  writeConnectionSecretToRef:
    name: my-app-connection-secret
EOF

    # Watch status
    log "Watching claim status..."
    kubectl get applicationenvironment my-app-production -n production -w &
    
    sleep 10
    kill %1 2>/dev/null || true
    
    log "Environment claim created"
}

# ==================== Crossplane Functions ====================
setup_composition_functions() {
    log "Setting up Crossplane Composition Functions..."
    
    # Install Function provider
    cat <<'EOF' | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-auto-ready:v0.2.1
---
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.4.0
---
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-go-templating
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-go-templating:v0.4.0
EOF

    log "Composition Functions installed"
}

# ==================== GitOps with Crossplane ====================
setup_crossplane_gitops() {
    log "Setting up GitOps workflow for Crossplane..."
    
    # ArgoCD Application for Crossplane compositions
    cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-compositions
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/${GITHUB_ORG}/platform-compositions
    targetRevision: HEAD
    path: compositions
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-claims
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/${GITHUB_ORG}/platform-environments
    targetRevision: HEAD
    path: claims
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

    log "GitOps for Crossplane configured"
}

# ==================== Main ====================
main() {
    case "${1:-all}" in
        install)    install_crossplane ;;
        provider)   setup_aws_provider ;;
        xrd)        create_application_xrd ;;
        compose)    create_composition ;;
        claim)      create_environment_claim ;;
        functions)  setup_composition_functions ;;
        gitops)     setup_crossplane_gitops ;;
        all)
            install_crossplane
            setup_aws_provider
            setup_composition_functions
            create_application_xrd
            create_composition
            setup_crossplane_gitops
            ;;
    esac
}

main "$@"
```

---

## สรุป Module 4 Progress

| Part | หัวข้อ | Steps | Status |
|------|--------|-------|--------|
| 41 | Enterprise K8s Architecture | 518-520 | ✅ |
| 42 | Data Platform Engineering | 521-525 | ✅ |
| 43 | FinOps & Cost Engineering | 526-529 | ✅ |
| 44 | ML/AI Infrastructure | 530-533 | ✅ |
| 45 | Event-Driven Architecture | 534-537 | ✅ |
| 46 | Database Engineering at Scale | 538-540 | ✅ |
| 47 | Global Load Balancing | 541-543 | ✅ |
| 48 | Enterprise Security | 544-545 | ✅ |
| 49 | Advanced CI/CD & DevSecOps | 546-548 | ✅ |
| **50** | **Advanced Networking & Platform Eng** | **549-552** | **✅** |

**ขั้นตอนต่อไป: Part 51 - SRE Advanced Practices & Incident Management**
