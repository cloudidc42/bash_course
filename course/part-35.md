# Part 35: Multi-cluster Kubernetes Management

## Module 3: Advanced Level - การจัดการ Kubernetes หลาย Cluster

---

## ขั้นตอนที่ 498: Multi-cluster Architecture และ Federation

การจัดการ Kubernetes หลาย Cluster เป็นความท้าทายที่สำคัญสำหรับองค์กรขนาดใหญ่

```bash
#!/bin/bash
# multi-cluster-manager.sh - Multi-cluster Kubernetes Manager

set -euo pipefail

FEDERATION_NAMESPACE="${FEDERATION_NAMESPACE:-kube-federation-system}"
CLUSTERS_CONFIG="${CLUSTERS_CONFIG:-$HOME/.kube/clusters.json}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Initialize clusters config
init_clusters_config() {
    if [[ ! -f "$CLUSTERS_CONFIG" ]]; then
        cat <<'EOF' > "$CLUSTERS_CONFIG"
{
  "clusters": [
    {
      "name": "prod-us-east",
      "region": "us-east-1",
      "provider": "aws",
      "role": "primary",
      "kubeconfig": "~/.kube/prod-us-east",
      "labels": {"env": "production", "region": "us-east-1"}
    },
    {
      "name": "prod-us-west",
      "region": "us-west-2",
      "provider": "aws",
      "role": "secondary",
      "kubeconfig": "~/.kube/prod-us-west",
      "labels": {"env": "production", "region": "us-west-2"}
    },
    {
      "name": "staging-eu",
      "region": "eu-west-1",
      "provider": "aws",
      "role": "staging",
      "kubeconfig": "~/.kube/staging-eu",
      "labels": {"env": "staging", "region": "eu-west-1"}
    }
  ]
}
EOF
        log "Created clusters config: $CLUSTERS_CONFIG"
    fi
}

# Execute command on all clusters
exec_all_clusters() {
    local cmd="${1:-}"
    local filter="${2:-all}"  # all, production, staging
    
    if [[ -z "$cmd" ]]; then
        error "Command required"
        return 1
    fi
    
    init_clusters_config
    
    local clusters
    if [[ "$filter" == "all" ]]; then
        clusters=$(jq -r '.clusters[].name' "$CLUSTERS_CONFIG")
    else
        clusters=$(jq -r ".clusters[] | select(.labels.env==\"$filter\") | .name" "$CLUSTERS_CONFIG")
    fi
    
    while IFS= read -r cluster; do
        local kubeconfig
        kubeconfig=$(jq -r ".clusters[] | select(.name==\"$cluster\") | .kubeconfig" "$CLUSTERS_CONFIG")
        kubeconfig="${kubeconfig/#\~/$HOME}"
        
        if [[ -f "$kubeconfig" ]]; then
            echo "=== Cluster: $cluster ==="
            KUBECONFIG="$kubeconfig" eval "$cmd" 2>/dev/null || echo "ERROR executing on $cluster"
            echo ""
        else
            log "Skipping $cluster: kubeconfig not found at $kubeconfig"
        fi
    done <<< "$clusters"
}

# Multi-cluster resource status
multi_cluster_status() {
    log "Getting status across all clusters..."
    
    exec_all_clusters "kubectl get nodes --no-headers | wc -l | xargs -I{} echo 'Nodes: {}'"
    
    echo ""
    log "Pod counts by namespace:"
    exec_all_clusters "kubectl get pods --all-namespaces --no-headers 2>/dev/null | awk '{print \$1}' | sort | uniq -c | sort -rn | head -10"
}

# Multi-cluster deployment
deploy_to_clusters() {
    local manifest="${1:-}"
    local clusters="${2:-all}"
    local namespace="${3:-default}"
    
    if [[ -z "$manifest" || ! -f "$manifest" ]]; then
        error "Valid manifest file required"
        return 1
    fi
    
    log "Deploying $manifest to $clusters clusters..."
    
    exec_all_clusters "kubectl apply -f $manifest -n $namespace" "$clusters"
    success "Deployment complete across $clusters clusters"
}

# Multi-cluster rollout status
check_rollout_status() {
    local deployment="${1:-}"
    local namespace="${2:-default}"
    
    if [[ -z "$deployment" ]]; then
        error "Deployment name required"
        return 1
    fi
    
    exec_all_clusters "kubectl rollout status deployment/$deployment -n $namespace --timeout=60s"
}

# Multi-cluster resource summary
resource_summary() {
    log "Resource usage across clusters..."
    
    exec_all_clusters "kubectl top nodes 2>/dev/null || echo 'Metrics server not available'"
}

# Federated service discovery
federated_service_info() {
    local service="${1:-}"
    local namespace="${2:-default}"
    
    if [[ -z "$service" ]]; then
        error "Service name required"
        return 1
    fi
    
    log "Service info across clusters: $service/$namespace"
    
    exec_all_clusters "kubectl get svc $service -n $namespace -o jsonpath='{.metadata.name}: {.status.loadBalancer.ingress[0].hostname}{.status.loadBalancer.ingress[0].ip}' 2>/dev/null && echo"
}

# Cluster health check
cluster_health_check() {
    log "Health check across all clusters..."
    
    init_clusters_config
    
    local clusters
    clusters=$(jq -r '.clusters[].name' "$CLUSTERS_CONFIG")
    
    echo "CLUSTER                  | STATUS  | NODES | READY | ISSUES"
    echo "-------------------------|---------|-------|-------|-------"
    
    while IFS= read -r cluster; do
        local kubeconfig
        kubeconfig=$(jq -r ".clusters[] | select(.name==\"$cluster\") | .kubeconfig" "$CLUSTERS_CONFIG")
        kubeconfig="${kubeconfig/#\~/$HOME}"
        
        if [[ ! -f "$kubeconfig" ]]; then
            printf "%-25s | %-7s | %-5s | %-5s | %s\n" \
                "$cluster" "NO_CFG" "-" "-" "kubeconfig missing"
            continue
        fi
        
        local total_nodes ready_nodes status
        total_nodes=$(KUBECONFIG="$kubeconfig" kubectl get nodes --no-headers 2>/dev/null | wc -l)
        ready_nodes=$(KUBECONFIG="$kubeconfig" kubectl get nodes --no-headers 2>/dev/null | grep -c " Ready " || echo 0)
        
        if [[ "$total_nodes" -eq "$ready_nodes" && "$total_nodes" -gt 0 ]]; then
            status="HEALTHY"
        elif [[ "$total_nodes" -gt 0 ]]; then
            status="DEGRADED"
        else
            status="UNKNOWN"
        fi
        
        local issues
        issues=$(KUBECONFIG="$kubeconfig" kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded \
            --no-headers 2>/dev/null | wc -l)
        
        printf "%-25s | %-7s | %-5s | %-5s | %s\n" \
            "$cluster" "$status" "$total_nodes" "$ready_nodes" "${issues} pods not running"
    done <<< "$clusters"
}

# Main
case "${1:-help}" in
    init) init_clusters_config ;;
    exec) exec_all_clusters "${2:-}" "${3:-all}" ;;
    status) multi_cluster_status ;;
    deploy) deploy_to_clusters "${2:-}" "${3:-all}" "${4:-default}" ;;
    rollout) check_rollout_status "${2:-}" "${3:-default}" ;;
    resources) resource_summary ;;
    service) federated_service_info "${2:-}" "${3:-default}" ;;
    health) cluster_health_check ;;
    *)
        echo "Usage: $0 {init|exec|status|deploy|rollout|resources|service|health}"
        ;;
esac
```

---

## ขั้นตอนที่ 499: Cluster API (CAPI) - Declarative Cluster Management

```bash
#!/bin/bash
# cluster-api-manager.sh - Cluster API Manager

set -euo pipefail

CAPI_NAMESPACE="${CAPI_NAMESPACE:-capi-system}"
MANAGEMENT_CLUSTER="${MANAGEMENT_CLUSTER:-management}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Initialize Cluster API with clusterctl
init_cluster_api() {
    local infrastructure="${1:-aws}"
    
    log "Initializing Cluster API with $infrastructure provider..."
    
    # Install clusterctl
    if ! command -v clusterctl &>/dev/null; then
        log "Installing clusterctl..."
        curl -L https://github.com/kubernetes-sigs/cluster-api/releases/latest/download/clusterctl-linux-amd64 \
            -o /usr/local/bin/clusterctl
        chmod +x /usr/local/bin/clusterctl
    fi
    
    case "$infrastructure" in
        aws)
            clusterctl init \
                --infrastructure aws \
                --control-plane kubeadm \
                --bootstrap kubeadm
            ;;
        azure)
            clusterctl init \
                --infrastructure azure \
                --control-plane kubeadm \
                --bootstrap kubeadm
            ;;
        gcp)
            clusterctl init \
                --infrastructure gcp \
                --control-plane kubeadm \
                --bootstrap kubeadm
            ;;
        vsphere)
            clusterctl init \
                --infrastructure vsphere \
                --control-plane kubeadm \
                --bootstrap kubeadm
            ;;
        *)
            error "Unsupported infrastructure: $infrastructure"
            return 1
            ;;
    esac
    
    success "Cluster API initialized with $infrastructure"
}

# Create workload cluster (AWS)
create_aws_cluster() {
    local cluster_name="${1:-my-workload-cluster}"
    local region="${2:-us-east-1}"
    local k8s_version="${3:-v1.28.0}"
    local control_plane_count="${4:-3}"
    local worker_count="${5:-3}"
    local control_plane_machine_type="${6:-t3.large}"
    local worker_machine_type="${7:-t3.large}"
    
    log "Creating AWS cluster: $cluster_name"
    
    # Generate cluster template
    clusterctl generate cluster "$cluster_name" \
        --infrastructure aws \
        --kubernetes-version "$k8s_version" \
        --control-plane-machine-count "$control_plane_count" \
        --worker-machine-count "$worker_count" \
        --flavor machinedeployment \
        > "${cluster_name}-cluster.yaml"
    
    # Apply cluster
    kubectl apply -f "${cluster_name}-cluster.yaml"
    
    success "Cluster creation initiated: $cluster_name"
    log "Monitor with: clusterctl describe cluster $cluster_name"
}

# Get cluster kubeconfig
get_cluster_kubeconfig() {
    local cluster_name="${1:-}"
    local namespace="${2:-default}"
    local output_file="${3:-}"
    
    if [[ -z "$cluster_name" ]]; then
        error "Cluster name required"
        return 1
    fi
    
    output_file="${output_file:-$HOME/.kube/${cluster_name}.yaml}"
    
    log "Getting kubeconfig for cluster: $cluster_name"
    
    clusterctl get kubeconfig "$cluster_name" \
        -n "$namespace" > "$output_file"
    
    success "Kubeconfig saved to: $output_file"
}

# Scale cluster workers
scale_cluster() {
    local cluster_name="${1:-}"
    local replica_count="${2:-3}"
    local machine_deployment="${3:-}"
    local namespace="${4:-default}"
    
    if [[ -z "$cluster_name" ]]; then
        error "Cluster name required"
        return 1
    fi
    
    if [[ -z "$machine_deployment" ]]; then
        machine_deployment="${cluster_name}-md-0"
    fi
    
    log "Scaling cluster $cluster_name to $replica_count workers..."
    
    kubectl patch machinedeployment "$machine_deployment" \
        -n "$namespace" \
        --type merge \
        -p "{\"spec\":{\"replicas\":$replica_count}}"
    
    success "Cluster $cluster_name scaled to $replica_count workers"
}

# Upgrade cluster
upgrade_cluster() {
    local cluster_name="${1:-}"
    local new_version="${2:-}"
    local namespace="${3:-default}"
    
    if [[ -z "$cluster_name" || -z "$new_version" ]]; then
        error "Usage: upgrade_cluster <cluster-name> <k8s-version> [namespace]"
        return 1
    fi
    
    log "Upgrading cluster $cluster_name to $new_version..."
    
    # Update control plane version
    kubectl patch kubeadmcontrolplane "${cluster_name}-control-plane" \
        -n "$namespace" \
        --type merge \
        -p "{\"spec\":{\"version\":\"$new_version\"}}"
    
    # Wait for control plane to be ready
    log "Waiting for control plane upgrade..."
    kubectl wait kubeadmcontrolplane "${cluster_name}-control-plane" \
        -n "$namespace" \
        --for=condition=Ready \
        --timeout=600s
    
    # Update worker nodes version
    local machine_deployments
    machine_deployments=$(kubectl get machinedeployments -n "$namespace" \
        -l "cluster.x-k8s.io/cluster-name=$cluster_name" \
        -o jsonpath='{.items[*].metadata.name}')
    
    for md in $machine_deployments; do
        log "Upgrading MachineDeployment: $md"
        kubectl patch machinedeployment "$md" \
            -n "$namespace" \
            --type merge \
            -p "{\"spec\":{\"template\":{\"spec\":{\"version\":\"$new_version\"}}}}"
    done
    
    success "Cluster upgrade initiated: $cluster_name to $new_version"
}

# Delete cluster
delete_cluster() {
    local cluster_name="${1:-}"
    local namespace="${2:-default}"
    local confirm="${3:-false}"
    
    if [[ -z "$cluster_name" ]]; then
        error "Cluster name required"
        return 1
    fi
    
    if [[ "$confirm" != "true" ]]; then
        log "To delete cluster $cluster_name, run with confirm=true"
        log "This will delete all workloads and data!"
        return 1
    fi
    
    log "Deleting cluster: $cluster_name"
    
    kubectl delete cluster "$cluster_name" -n "$namespace"
    
    success "Cluster deletion initiated: $cluster_name"
}

# List all CAPI clusters
list_clusters() {
    log "Listing all CAPI-managed clusters..."
    
    echo "=== Clusters ==="
    kubectl get clusters --all-namespaces
    
    echo ""
    echo "=== Machine Deployments ==="
    kubectl get machinedeployments --all-namespaces
    
    echo ""
    echo "=== Machine Sets ==="
    kubectl get machinesets --all-namespaces
    
    echo ""
    echo "=== Machines ==="
    kubectl get machines --all-namespaces | head -20
}

# Monitor cluster provisioning
monitor_provisioning() {
    local cluster_name="${1:-}"
    local namespace="${2:-default}"
    local timeout="${3:-600}"
    
    if [[ -z "$cluster_name" ]]; then
        error "Cluster name required"
        return 1
    fi
    
    log "Monitoring provisioning of: $cluster_name"
    
    local elapsed=0
    local interval=15
    
    while [[ $elapsed -lt $timeout ]]; do
        local phase
        phase=$(kubectl get cluster "$cluster_name" -n "$namespace" \
            -o jsonpath='{.status.phase}' 2>/dev/null || echo "Unknown")
        
        local ready
        ready=$(kubectl get cluster "$cluster_name" -n "$namespace" \
            -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null || echo "Unknown")
        
        log "Cluster $cluster_name: Phase=$phase, Ready=$ready (${elapsed}s/${timeout}s)"
        
        if [[ "$phase" == "Provisioned" && "$ready" == "True" ]]; then
            success "Cluster $cluster_name is Ready!"
            return 0
        fi
        
        if [[ "$phase" == "Failed" ]]; then
            error "Cluster provisioning failed!"
            kubectl describe cluster "$cluster_name" -n "$namespace"
            return 1
        fi
        
        sleep "$interval"
        elapsed=$((elapsed + interval))
    done
    
    error "Timeout waiting for cluster to be ready"
    return 1
}

# Main
case "${1:-help}" in
    init) init_cluster_api "${2:-aws}" ;;
    create-aws) create_aws_cluster "${2:-}" "${3:-us-east-1}" "${4:-v1.28.0}" "${5:-3}" "${6:-3}" ;;
    get-kubeconfig) get_cluster_kubeconfig "${2:-}" "${3:-default}" "${4:-}" ;;
    scale) scale_cluster "${2:-}" "${3:-3}" "${4:-}" "${5:-default}" ;;
    upgrade) upgrade_cluster "${2:-}" "${3:-}" "${4:-default}" ;;
    delete) delete_cluster "${2:-}" "${3:-default}" "${4:-false}" ;;
    list) list_clusters ;;
    monitor) monitor_provisioning "${2:-}" "${3:-default}" "${4:-600}" ;;
    *)
        echo "Usage: $0 {init|create-aws|get-kubeconfig|scale|upgrade|delete|list|monitor}"
        ;;
esac
```

---

## ขั้นตอนที่ 500: Service Mesh Federation และ Multi-cluster Traffic Management

ยินดีด้วย! ขั้นตอนที่ 500 เป็นจุดสำคัญในการเรียนรู้ Bash Script!

```bash
#!/bin/bash
# service-mesh-federation.sh - Multi-cluster Service Mesh Federation

set -euo pipefail

ISTIO_NAMESPACE="${ISTIO_NAMESPACE:-istio-system}"
LINKERD_NAMESPACE="${LINKERD_NAMESPACE:-linkerd}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Setup Istio multi-cluster (Primary-Remote)
setup_istio_multicluster() {
    local primary_cluster="${1:-cluster1}"
    local remote_cluster="${2:-cluster2}"
    local primary_kubeconfig="${3:-$HOME/.kube/cluster1.yaml}"
    local remote_kubeconfig="${4:-$HOME/.kube/cluster2.yaml}"
    local network="${5:-network1}"
    
    log "Setting up Istio multi-cluster: $primary_cluster (primary) + $remote_cluster (remote)"
    
    # Install Istio on primary cluster
    log "Installing Istio on primary cluster: $primary_cluster"
    KUBECONFIG="$primary_kubeconfig" istioctl install -y \
        --set profile=default \
        --set values.pilot.env.EXTERNAL_ISTIOD=true \
        --set values.global.meshID=mesh1 \
        --set values.global.multiCluster.clusterName="$primary_cluster" \
        --set values.global.network="$network"
    
    # Create remote secrets
    log "Creating remote cluster secrets..."
    KUBECONFIG="$primary_kubeconfig" istioctl create-remote-secret \
        --kubeconfig="$remote_kubeconfig" \
        --name="$remote_cluster" | \
        KUBECONFIG="$primary_kubeconfig" kubectl apply -f -
    
    # Install Istio on remote cluster
    log "Installing Istio on remote cluster: $remote_cluster"
    
    # Get primary cluster address
    local primary_address
    primary_address=$(KUBECONFIG="$primary_kubeconfig" kubectl get service istio-pilot \
        -n "$ISTIO_NAMESPACE" \
        -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || \
        KUBECONFIG="$primary_kubeconfig" kubectl get service istio-pilot \
        -n "$ISTIO_NAMESPACE" \
        -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    
    KUBECONFIG="$remote_kubeconfig" istioctl install -y \
        --set profile=remote \
        --set values.istiodRemote.injectionURL="https://${primary_address}:15017/inject" \
        --set values.base.validationURL="https://${primary_address}:15017/validate" \
        --set values.global.meshID=mesh1 \
        --set values.global.multiCluster.clusterName="$remote_cluster" \
        --set values.global.network="$network" \
        --set values.global.remotePilotAddress="$primary_address"
    
    success "Istio multi-cluster setup complete"
}

# Create service entry for cross-cluster communication
create_cross_cluster_service() {
    local service_name="${1:-my-service}"
    local service_namespace="${2:-default}"
    local remote_cluster_name="${3:-cluster2}"
    local service_port="${4:-80}"
    
    log "Creating cross-cluster service entry: $service_name"
    
    # ServiceEntry for cross-cluster service discovery
    cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: ${service_name}-${remote_cluster_name}
  namespace: ${service_namespace}
spec:
  hosts:
  - ${service_name}.${service_namespace}.svc.cluster.local
  location: MESH_INTERNAL
  ports:
  - name: http
    number: ${service_port}
    protocol: HTTP
  resolution: STATIC
  workloadSelector:
    labels:
      app: ${service_name}
      cluster: ${remote_cluster_name}
  exportTo:
  - "*"
EOF

    # Create Destination Rule for mTLS
    cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ${service_name}-mtls
  namespace: ${service_namespace}
spec:
  host: ${service_name}.${service_namespace}.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: cluster1
    labels:
      cluster: cluster1
  - name: cluster2
    labels:
      cluster: cluster2
EOF
    
    success "Cross-cluster service entry created: $service_name"
}

# Multi-cluster load balancing
setup_multi_cluster_lb() {
    local service_name="${1:-my-service}"
    local namespace="${2:-default}"
    local cluster1_weight="${3:-50}"
    local cluster2_weight="${4:-50}"
    
    log "Setting up multi-cluster load balancing for: $service_name"
    
    local cluster2_weight_calc=$((100 - cluster1_weight))
    
    cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ${service_name}-multi-cluster
  namespace: ${namespace}
spec:
  hosts:
  - ${service_name}
  http:
  - route:
    - destination:
        host: ${service_name}
        subset: cluster1
      weight: ${cluster1_weight}
    - destination:
        host: ${service_name}
        subset: cluster2
      weight: ${cluster2_weight_calc}
    retries:
      attempts: 3
      perTryTimeout: 2s
    timeout: 10s
EOF
    
    success "Multi-cluster LB configured: cluster1=${cluster1_weight}%, cluster2=${cluster2_weight_calc}%"
}

# Failover configuration
setup_cluster_failover() {
    local service_name="${1:-my-service}"
    local namespace="${2:-default}"
    local primary_cluster="${3:-cluster1}"
    local failover_cluster="${4:-cluster2}"
    
    log "Setting up cluster failover for: $service_name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ${service_name}-failover
  namespace: ${namespace}
spec:
  host: ${service_name}.${namespace}.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    loadBalancer:
      simple: ROUND_ROBIN
      localityLbSetting:
        enabled: true
        failover:
        - from: ${primary_cluster}
          to: ${failover_cluster}
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: ${primary_cluster}
    labels:
      cluster: ${primary_cluster}
  - name: ${failover_cluster}
    labels:
      cluster: ${failover_cluster}
EOF
    
    success "Cluster failover configured: $primary_cluster -> $failover_cluster"
}

# Setup Linkerd multi-cluster
setup_linkerd_multicluster() {
    local cluster1_kubeconfig="${1:-$HOME/.kube/cluster1.yaml}"
    local cluster2_kubeconfig="${2:-$HOME/.kube/cluster2.yaml}"
    
    log "Setting up Linkerd multi-cluster..."
    
    # Install Linkerd multi-cluster on both clusters
    for kubeconfig in "$cluster1_kubeconfig" "$cluster2_kubeconfig"; do
        if [[ -f "$kubeconfig" ]]; then
            log "Installing Linkerd multi-cluster on cluster with kubeconfig: $kubeconfig"
            
            KUBECONFIG="$kubeconfig" linkerd multicluster install | \
                KUBECONFIG="$kubeconfig" kubectl apply -f -
            
            KUBECONFIG="$kubeconfig" linkerd multicluster check || true
        fi
    done
    
    # Link clusters
    if [[ -f "$cluster2_kubeconfig" ]]; then
        log "Linking cluster2 to cluster1..."
        
        KUBECONFIG="$cluster2_kubeconfig" linkerd multicluster link \
            --cluster-name cluster2 | \
            KUBECONFIG="$cluster1_kubeconfig" kubectl apply -f -
    fi
    
    success "Linkerd multi-cluster setup complete"
}

# Export service via Linkerd
export_linkerd_service() {
    local service_name="${1:-my-service}"
    local namespace="${2:-default}"
    local mirror_namespace="${3:-default}"
    
    log "Exporting service via Linkerd: $service_name"
    
    # Add mirror label to service
    kubectl label service "$service_name" \
        -n "$namespace" \
        mirror.linkerd.io/exported=true \
        --overwrite
    
    success "Service exported via Linkerd: $service_name"
    log "Service will be mirrored as: ${service_name}-cluster2.$mirror_namespace"
}

# Multi-cluster network topology
show_network_topology() {
    log "Multi-cluster network topology..."
    
    echo "=== Clusters ==="
    kubectl get clusters --all-namespaces 2>/dev/null || echo "No CAPI clusters"
    
    echo ""
    echo "=== Istio Service Entries ==="
    kubectl get serviceentries --all-namespaces 2>/dev/null | head -20
    
    echo ""
    echo "=== Linkerd Service Mirrors ==="
    kubectl get services --all-namespaces -l mirror.linkerd.io/cluster-name 2>/dev/null | head -20
    
    echo ""
    echo "=== Virtual Services ==="
    kubectl get virtualservices --all-namespaces 2>/dev/null | head -20
    
    echo ""
    echo "=== Destination Rules ==="
    kubectl get destinationrules --all-namespaces 2>/dev/null | head -20
}

# Traffic analysis across clusters
analyze_cross_cluster_traffic() {
    local namespace="${1:-default}"
    local duration="${2:-5m}"
    
    log "Analyzing cross-cluster traffic (last $duration)..."
    
    # Query Prometheus for cross-cluster metrics
    local prom_url="${PROMETHEUS_URL:-http://prometheus:9090}"
    
    local query="sum(rate(istio_requests_total{destination_cluster!=\"\"}[${duration}])) by (source_cluster, destination_cluster, response_code)"
    
    curl -s "${prom_url}/api/v1/query" \
        --data-urlencode "query=$query" | \
        jq -r '.data.result[] | "\(.metric.source_cluster) -> \(.metric.destination_cluster): \(.metric.response_code) (\(.value[1]) rps)"' \
        2>/dev/null || echo "Prometheus not available"
}

# Main
case "${1:-help}" in
    istio-setup) setup_istio_multicluster "${2:-cluster1}" "${3:-cluster2}" "${4:-}" "${5:-}" ;;
    cross-cluster-svc) create_cross_cluster_service "${2:-}" "${3:-default}" "${4:-cluster2}" "${5:-80}" ;;
    multi-lb) setup_multi_cluster_lb "${2:-}" "${3:-default}" "${4:-50}" "${5:-50}" ;;
    failover) setup_cluster_failover "${2:-}" "${3:-default}" "${4:-cluster1}" "${5:-cluster2}" ;;
    linkerd-setup) setup_linkerd_multicluster "${2:-}" "${3:-}" ;;
    linkerd-export) export_linkerd_service "${2:-}" "${3:-default}" ;;
    topology) show_network_topology ;;
    traffic) analyze_cross_cluster_traffic "${2:-default}" "${3:-5m}" ;;
    *)
        echo "Usage: $0 {istio-setup|cross-cluster-svc|multi-lb|failover|linkerd-setup|linkerd-export|topology|traffic}"
        ;;
esac
```

---

## ขั้นตอนที่ 501: Advanced Node Management และ Autoscaling

```bash
#!/bin/bash
# advanced-node-manager.sh - Advanced Node and Cluster Autoscaling

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Setup Cluster Autoscaler (AWS)
setup_cluster_autoscaler_aws() {
    local cluster_name="${1:-my-cluster}"
    local region="${2:-us-east-1}"
    local min_nodes="${3:-1}"
    local max_nodes="${4:-10}"
    local namespace="${5:-kube-system}"
    
    log "Setting up Cluster Autoscaler for: $cluster_name"
    
    # Create IAM policy for autoscaler
    cat <<'EOF' > /tmp/cluster-autoscaler-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*"
        }
    ]
}
EOF
    
    # Deploy Cluster Autoscaler
    helm repo add autoscaler https://kubernetes.github.io/autoscaler
    helm repo update
    
    helm upgrade --install cluster-autoscaler \
        autoscaler/cluster-autoscaler \
        -n "$namespace" \
        --set autoDiscovery.clusterName="$cluster_name" \
        --set awsRegion="$region" \
        --set rbac.serviceAccount.annotations."eks\\.amazonaws\\.com/role-arn"="arn:aws:iam::ACCOUNT_ID:role/ClusterAutoScaler" \
        --set extraArgs.balance-similar-node-groups=true \
        --set extraArgs.skip-nodes-with-system-pods=false \
        --set extraArgs.scale-down-utilization-threshold=0.5 \
        --set extraArgs.scale-down-delay-after-add=2m \
        --set extraArgs.scale-down-unneeded-time=2m \
        --wait
    
    success "Cluster Autoscaler deployed"
}

# Setup KEDA (Kubernetes Event-Driven Autoscaling)
setup_keda() {
    local namespace="${1:-keda}"
    
    log "Installing KEDA..."
    
    helm repo add kedacore https://kedacore.github.io/charts
    helm repo update
    
    helm upgrade --install keda kedacore/keda \
        -n "$namespace" \
        --create-namespace \
        --wait
    
    success "KEDA installed"
}

# Create KEDA ScaledObject for queue-based scaling
create_keda_scaledobject() {
    local name="${1:-my-scaler}"
    local deployment="${2:-my-deployment}"
    local namespace="${3:-default}"
    local trigger_type="${4:-rabbitmq}"
    local queue_name="${5:-my-queue}"
    local queue_length="${6:-10}"
    local min_replicas="${7:-1}"
    local max_replicas="${8:-20}"
    
    log "Creating KEDA ScaledObject: $name"
    
    local trigger_spec
    case "$trigger_type" in
        rabbitmq)
            trigger_spec=$(cat <<EOF
  - type: rabbitmq
    metadata:
      protocol: amqp
      queueName: ${queue_name}
      mode: QueueLength
      value: "${queue_length}"
    authenticationRef:
      name: rabbitmq-auth
EOF
)
            ;;
        kafka)
            trigger_spec=$(cat <<EOF
  - type: kafka
    metadata:
      bootstrapServers: kafka-broker:9092
      consumerGroup: my-group
      topic: ${queue_name}
      lagThreshold: "${queue_length}"
EOF
)
            ;;
        prometheus)
            trigger_spec=$(cat <<EOF
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: http_requests_per_second
      threshold: "${queue_length}"
      query: sum(rate(http_requests_total[2m]))
EOF
)
            ;;
        *)
            trigger_spec=$(cat <<EOF
  - type: cpu
    metadata:
      type: AverageValue
      value: "${queue_length}"
EOF
)
            ;;
    esac
    
    cat <<EOF | kubectl apply -f -
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ${name}
  namespace: ${namespace}
spec:
  scaleTargetRef:
    name: ${deployment}
  pollingInterval: 15
  cooldownPeriod: 30
  idleReplicaCount: 0
  minReplicaCount: ${min_replicas}
  maxReplicaCount: ${max_replicas}
  triggers:
${trigger_spec}
EOF
    
    success "KEDA ScaledObject created: $name"
}

# Node affinity and anti-affinity management
configure_node_affinity() {
    local deployment="${1:-}"
    local namespace="${2:-default}"
    local node_label_key="${3:-node-type}"
    local node_label_value="${4:-high-memory}"
    local affinity_type="${5:-required}"  # required or preferred
    
    if [[ -z "$deployment" ]]; then
        error "Deployment name required"
        return 1
    fi
    
    log "Configuring node affinity for: $deployment"
    
    if [[ "$affinity_type" == "required" ]]; then
        local affinity_spec
        affinity_spec=$(cat <<EOF
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      - key: ${node_label_key}
        operator: In
        values:
        - ${node_label_value}
EOF
)
    else
        local affinity_spec
        affinity_spec=$(cat <<EOF
nodeAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    preference:
      matchExpressions:
      - key: ${node_label_key}
        operator: In
        values:
        - ${node_label_value}
EOF
)
    fi
    
    kubectl patch deployment "$deployment" -n "$namespace" \
        --type merge \
        -p "{\"spec\":{\"template\":{\"spec\":{\"affinity\":{${affinity_spec}}}}}}" 2>/dev/null || \
    log "Affinity patch requires manual YAML formatting - see output above"
    
    success "Node affinity configured for: $deployment"
}

# Taint and toleration management
manage_taints() {
    local action="${1:-add}"
    local node="${2:-}"
    local taint_key="${3:-}"
    local taint_value="${4:-}"
    local taint_effect="${5:-NoSchedule}"
    
    if [[ -z "$node" || -z "$taint_key" ]]; then
        error "Usage: manage_taints <add|remove> <node> <taint-key> [taint-value] [effect]"
        return 1
    fi
    
    case "$action" in
        add)
            if [[ -n "$taint_value" ]]; then
                kubectl taint node "$node" "${taint_key}=${taint_value}:${taint_effect}"
            else
                kubectl taint node "$node" "${taint_key}:${taint_effect}"
            fi
            success "Taint added to node: $node"
            ;;
        remove)
            kubectl taint node "$node" "${taint_key}:${taint_effect}-"
            success "Taint removed from node: $node"
            ;;
        list)
            kubectl describe node "$node" | grep Taints
            ;;
    esac
}

# Node drain and maintenance
drain_node() {
    local node="${1:-}"
    local force="${2:-false}"
    local timeout="${3:-300}"
    
    if [[ -z "$node" ]]; then
        error "Node name required"
        return 1
    fi
    
    log "Draining node: $node"
    
    # Cordon node first
    kubectl cordon "$node"
    
    # Drain node
    local drain_opts="--ignore-daemonsets --delete-emptydir-data --timeout=${timeout}s"
    
    if [[ "$force" == "true" ]]; then
        drain_opts="$drain_opts --force"
    fi
    
    kubectl drain "$node" $drain_opts
    
    success "Node drained: $node"
}

# Uncordon node after maintenance
uncordon_node() {
    local node="${1:-}"
    
    if [[ -z "$node" ]]; then
        error "Node name required"
        return 1
    fi
    
    kubectl uncordon "$node"
    success "Node uncordoned: $node"
}

# Node resource optimization
analyze_node_utilization() {
    log "Analyzing node utilization..."
    
    echo "=== Node Resource Requests vs Capacity ==="
    kubectl describe nodes | awk '/Allocated resources:/,/Events:/' | \
        grep -E "(Resource|cpu|memory|Namespace|Name:)" | head -40
    
    echo ""
    echo "=== Pods per Node ==="
    kubectl get pods --all-namespaces -o wide --no-headers | \
        awk '{print $8}' | sort | uniq -c | sort -rn
    
    echo ""
    echo "=== Top Nodes by CPU ==="
    kubectl top nodes 2>/dev/null | sort -k3 -rn | head -10 || echo "Metrics server not available"
    
    echo ""
    echo "=== Top Nodes by Memory ==="
    kubectl top nodes 2>/dev/null | sort -k5 -rn | head -10 || echo "Metrics server not available"
}

# VPA (Vertical Pod Autoscaler) recommendations
get_vpa_recommendations() {
    local namespace="${1:-default}"
    
    log "Getting VPA recommendations for namespace: $namespace"
    
    kubectl get vpa -n "$namespace" -o json 2>/dev/null | \
        jq -r '.items[] | "\(.metadata.name): CPU Min=\(.status.recommendation.containerRecommendations[0].lowerBound.cpu) CPU Target=\(.status.recommendation.containerRecommendations[0].target.cpu) Mem Min=\(.status.recommendation.containerRecommendations[0].lowerBound.memory) Mem Target=\(.status.recommendation.containerRecommendations[0].target.memory)"' \
        || echo "No VPA resources found"
}

# Main
case "${1:-help}" in
    autoscaler-aws) setup_cluster_autoscaler_aws "${2:-}" "${3:-us-east-1}" "${4:-1}" "${5:-10}" ;;
    keda) setup_keda "${2:-keda}" ;;
    keda-scale) create_keda_scaledobject "${2:-}" "${3:-}" "${4:-default}" "${5:-rabbitmq}" "${6:-my-queue}" "${7:-10}" "${8:-1}" "${9:-20}" ;;
    affinity) configure_node_affinity "${2:-}" "${3:-default}" "${4:-node-type}" "${5:-high-memory}" "${6:-required}" ;;
    taint) manage_taints "${2:-add}" "${3:-}" "${4:-}" "${5:-}" "${6:-NoSchedule}" ;;
    drain) drain_node "${2:-}" "${3:-false}" "${4:-300}" ;;
    uncordon) uncordon_node "${2:-}" ;;
    analyze) analyze_node_utilization ;;
    vpa) get_vpa_recommendations "${2:-default}" ;;
    *)
        echo "Usage: $0 {autoscaler-aws|keda|keda-scale|affinity|taint|drain|uncordon|analyze|vpa}"
        ;;
esac
```

---

## ขั้นตอนที่ 502: Workshop - Multi-cluster Production Setup

```bash
#!/bin/bash
# multi-cluster-production-workshop.sh - Production Multi-cluster Workshop

set -euo pipefail

PRIMARY_CLUSTER="${PRIMARY_CLUSTER:-prod-us-east}"
DR_CLUSTER="${DR_CLUSTER:-prod-us-west}"
DOMAIN="${DOMAIN:-example.com}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
separator() { echo "=========================================="; }
success() { echo "[SUCCESS] $*"; }

# Production multi-cluster checklist
production_checklist() {
    log "Production Multi-cluster Readiness Checklist"
    separator
    
    local checks=(
        "Cluster Autoscaler configured"
        "HPA configured for all deployments"
        "VPA recommendations applied"
        "PodDisruptionBudgets set"
        "Network policies enforced"
        "RBAC properly configured"
        "Secrets encrypted at rest"
        "Resource quotas set per namespace"
        "LimitRanges configured"
        "Monitoring and alerting active"
        "Log aggregation configured"
        "Backup and DR tested"
        "GitOps reconciliation healthy"
        "mTLS enabled between services"
        "Cross-cluster failover tested"
    )
    
    for check in "${checks[@]}"; do
        echo "  □ $check"
    done
    
    echo ""
    echo "Verify each item before production deployment"
}

# Setup production namespaces
setup_production_namespaces() {
    log "Setting up production namespaces..."
    separator
    
    local namespaces=("production" "staging" "monitoring" "logging" "security" "platform")
    
    for ns in "${namespaces[@]}"; do
        kubectl create namespace "$ns" --dry-run=client -o yaml | kubectl apply -f -
        
        # Label namespace
        kubectl label namespace "$ns" \
            environment="$ns" \
            managed-by=platform \
            --overwrite
        
        log "Namespace ready: $ns"
    done
    
    success "Production namespaces configured"
}

# Setup global ingress with cert-manager
setup_global_ingress() {
    local domain="${1:-$DOMAIN}"
    local email="${2:-admin@$DOMAIN}"
    
    log "Setting up global ingress for: $domain"
    separator
    
    # Install cert-manager
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml 2>/dev/null || true
    
    log "Waiting for cert-manager..."
    kubectl wait deployment cert-manager \
        -n cert-manager \
        --for=condition=Available \
        --timeout=120s 2>/dev/null || log "cert-manager may need more time"
    
    # Create ClusterIssuer for Let's Encrypt
    cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ${email}
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: ${email}
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
    
    # Install nginx ingress controller
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    
    helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
        -n ingress-nginx \
        --create-namespace \
        --set controller.service.type=LoadBalancer \
        --set controller.metrics.enabled=true \
        --set controller.podAnnotations."prometheus\.io/scrape"=true \
        --set controller.podAnnotations."prometheus\.io/port"=10254 \
        --wait
    
    success "Global ingress setup complete for: $domain"
}

# Setup external-dns
setup_external_dns() {
    local provider="${1:-aws}"
    local domain="${2:-$DOMAIN}"
    
    log "Setting up ExternalDNS for: $provider"
    
    helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
    helm repo update
    
    case "$provider" in
        aws)
            helm upgrade --install external-dns external-dns/external-dns \
                -n external-dns \
                --create-namespace \
                --set provider=aws \
                --set aws.region=us-east-1 \
                --set "domainFilters[0]=$domain" \
                --set policy=upsert-only \
                --set registry=txt \
                --set txtOwnerId=my-cluster \
                --wait
            ;;
        gcp)
            helm upgrade --install external-dns external-dns/external-dns \
                -n external-dns \
                --create-namespace \
                --set provider=google \
                --set "domainFilters[0]=$domain" \
                --wait
            ;;
    esac
    
    success "ExternalDNS configured for: $provider"
}

# Disaster Recovery test
test_disaster_recovery() {
    log "Running Disaster Recovery Test"
    separator
    
    echo "Step 1: Identify critical services"
    kubectl get deployments --all-namespaces \
        -l tier=critical 2>/dev/null | head -10 || \
    kubectl get deployments -n production 2>/dev/null | head -10
    
    echo ""
    echo "Step 2: Check backup status"
    kubectl get volumesnapshots --all-namespaces 2>/dev/null | head -10 || \
    echo "Volume snapshots: Velero or similar tool required"
    
    echo ""
    echo "Step 3: Verify DR cluster health"
    echo "Primary cluster ($PRIMARY_CLUSTER): checking..."
    kubectl cluster-info 2>/dev/null | head -3
    
    echo ""
    echo "Step 4: Estimate RTO/RPO"
    echo "  RTO (Recovery Time Objective): ~5-15 minutes"
    echo "  RPO (Recovery Point Objective): ~1-5 minutes with async replication"
    
    echo ""
    echo "Step 5: DR runbook checklist"
    echo "  □ Alert on-call team"
    echo "  □ Assess impact scope"
    echo "  □ Activate DR cluster"
    echo "  □ Update DNS records"
    echo "  □ Verify service health in DR"
    echo "  □ Notify stakeholders"
    echo "  □ Document incident timeline"
    
    success "DR test simulation complete"
}

# Run workshop
run_workshop() {
    log "Starting Multi-cluster Production Workshop"
    separator
    
    production_checklist
    echo ""
    
    setup_production_namespaces
    echo ""
    
    setup_global_ingress
    echo ""
    
    test_disaster_recovery
    echo ""
    
    success "Multi-cluster Production Workshop Complete!"
    separator
    echo ""
    echo "Production Architecture:"
    echo "  • Primary: $PRIMARY_CLUSTER (us-east-1)"
    echo "  • DR: $DR_CLUSTER (us-west-2)"
    echo "  • Domain: $DOMAIN"
    echo "  • GitOps: Flux CD / ArgoCD"
    echo "  • Ingress: Nginx + cert-manager"
    echo "  • DNS: ExternalDNS"
    echo "  • Monitoring: Prometheus + Grafana"
}

# Main
case "${1:-workshop}" in
    checklist) production_checklist ;;
    namespaces) setup_production_namespaces ;;
    ingress) setup_global_ingress "${2:-$DOMAIN}" "${3:-}" ;;
    dns) setup_external_dns "${2:-aws}" "${3:-$DOMAIN}" ;;
    dr-test) test_disaster_recovery ;;
    workshop) run_workshop ;;
    *)
        echo "Usage: $0 {checklist|namespaces|ingress|dns|dr-test|workshop}"
        ;;
esac
```

---

## สรุป Part 35

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | เครื่องมือ | ขั้นตอน |
|--------|-----------|---------|
| Multi-cluster Architecture | kubectl, kubeconfig management | 498 |
| Cluster API (CAPI) | clusterctl, MachineDeployment | 499 |
| **ขั้นตอนที่ 500!** Service Mesh Federation | Istio multi-cluster, Linkerd | 500 |
| Advanced Node Management | KEDA, Cluster Autoscaler, VPA | 501 |
| Production Multi-cluster Workshop | Complete production setup | 502 |

**เทคโนโลยีที่ใช้:**
- Cluster API: Declarative cluster lifecycle
- Istio/Linkerd: Service mesh federation
- KEDA: Event-driven autoscaling
- cert-manager: TLS certificate automation
- ExternalDNS: DNS automation
- OPA Gatekeeper: Policy enforcement

🎉 **ยินดีด้วย! ขั้นตอนที่ 500 สำเร็จแล้ว!**

ขั้นตอนต่อไป: Part 36 - Advanced Kubernetes Patterns และ Operators
