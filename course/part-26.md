# Part 26: Kubernetes Integration

## Module 3: Advanced Level
### ขั้นตอนที่ 456-467: การทำงานกับ Kubernetes

---

## ขั้นตอนที่ 456: kubectl Fundamentals

```bash
#!/usr/bin/env bash
# kubectl_basics.sh

# ===== Kubernetes CLI Operations =====
set -euo pipefail

# kubectl wrapper ที่ดีขึ้น
k() {
    kubectl "$@"
}

# Check kubectl available and configured
check_kubectl() {
    if ! command -v kubectl &>/dev/null; then
        echo "kubectl not found" >&2
        echo "Install: https://kubernetes.io/docs/tasks/tools/"
        return 1
    fi
    
    if ! kubectl cluster-info &>/dev/null; then
        echo "Cannot connect to Kubernetes cluster" >&2
        return 1
    fi
    
    local context
    context=$(kubectl config current-context 2>/dev/null)
    echo "Connected to cluster (context: $context)"
}

# Namespace management
create_namespace() {
    local ns="$1"
    local labels="${2:-}"
    
    if kubectl get namespace "$ns" &>/dev/null; then
        echo "Namespace exists: $ns"
        return 0
    fi
    
    kubectl create namespace "$ns"
    
    if [[ -n "$labels" ]]; then
        kubectl label namespace "$ns" $labels
    fi
    
    echo "Created namespace: $ns"
}

# Resource operations
get_resources() {
    local type="$1"
    local ns="${2:--A}"  # All namespaces by default
    local selector="${3:-}"
    
    local ns_flag
    [[ "$ns" == "-A" ]] && ns_flag="-A" || ns_flag="-n $ns"
    
    local selector_flag=""
    [[ -n "$selector" ]] && selector_flag="-l $selector"
    
    kubectl get "$type" $ns_flag $selector_flag \
        -o wide 2>/dev/null
}

# Wait for resource
wait_for_ready() {
    local type="$1"
    local name="$2"
    local ns="${3:-default}"
    local timeout="${4:-300}"
    
    echo "Waiting for $type/$name in $ns (timeout: ${timeout}s)..."
    kubectl wait \
        --namespace="$ns" \
        --for=condition=ready \
        --timeout="${timeout}s" \
        "$type/$name"
}

# Scale deployment
scale_deployment() {
    local name="$1"
    local replicas="$2"
    local ns="${3:-default}"
    
    kubectl scale deployment "$name" \
        --replicas="$replicas" \
        --namespace="$ns"
    
    echo "Scaled $name to $replicas replicas"
    wait_for_ready "deployment" "$name" "$ns"
}

# Rollout management
rollout_status() {
    local deployment="$1"
    local ns="${2:-default}"
    
    kubectl rollout status deployment/"$deployment" -n "$ns"
}

rollout_restart() {
    local deployment="$1"
    local ns="${2:-default}"
    
    echo "Restarting deployment: $deployment"
    kubectl rollout restart deployment/"$deployment" -n "$ns"
    rollout_status "$deployment" "$ns"
}

rollout_undo() {
    local deployment="$1"
    local ns="${2:-default}"
    
    echo "Rolling back: $deployment"
    kubectl rollout undo deployment/"$deployment" -n "$ns"
}

# Port forwarding (background)
port_forward() {
    local type="$1"
    local name="$2"
    local local_port="$3"
    local remote_port="$4"
    local ns="${5:-default}"
    
    echo "Port forwarding $type/$name: $local_port -> $remote_port"
    kubectl port-forward \
        -n "$ns" \
        "$type/$name" \
        "${local_port}:${remote_port}" &
    
    local pf_pid=$!
    echo "Port forward PID: $pf_pid"
    
    # Return PID for cleanup
    echo "$pf_pid"
}

# Execute command in pod
pod_exec() {
    local pod="$1"
    local ns="${2:-default}"
    shift 2
    
    kubectl exec -n "$ns" -it "$pod" -- "$@"
}

# Get pod logs
get_logs() {
    local pod="$1"
    local ns="${2:-default}"
    local container="${3:-}"
    local lines="${4:-100}"
    local follow="${5:-false}"
    
    local args=(-n "$ns" --tail="$lines")
    [[ -n "$container" ]] && args+=(-c "$container")
    [[ "$follow" == "true" ]] && args+=(-f)
    
    kubectl logs "${args[@]}" "$pod"
}

# Resource usage
top_pods() {
    local ns="${1:--A}"
    
    if [[ "$ns" == "-A" ]]; then
        kubectl top pods --all-namespaces 2>/dev/null || echo "metrics-server not available"
    else
        kubectl top pods -n "$ns" 2>/dev/null || echo "metrics-server not available"
    fi
}

# Demo (requires kubectl and cluster)
if command -v kubectl &>/dev/null; then
    echo "=== Kubernetes Demo ==="
    kubectl version --client --short 2>/dev/null || kubectl version --client 2>/dev/null | head -1
    
    echo "Contexts available:"
    kubectl config get-contexts 2>/dev/null | head -5
else
    echo "kubectl not available"
    echo ""
    echo "Example usage:"
    echo "  create_namespace myapp"
    echo "  scale_deployment myapp 3 myapp-ns"
    echo "  rollout_restart myapp myapp-ns"
    echo "  get_logs myapp-pod-xxx myapp-ns app 200"
fi
```

---

## ขั้นตอนที่ 457: Kubernetes Manifest Generation

```bash
#!/usr/bin/env bash
# k8s_manifest_generator.sh

# ===== Generate Kubernetes Manifests =====

# Generate Deployment manifest
generate_deployment() {
    local name="$1"
    local image="$2"
    local replicas="${3:-1}"
    local port="${4:-8080}"
    local ns="${5:-default}"
    local cpu_req="${6:-100m}"
    local mem_req="${7:-128Mi}"
    local cpu_lim="${8:-500m}"
    local mem_lim="${9:-512Mi}"
    
    cat << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $name
  namespace: $ns
  labels:
    app: $name
    managed-by: bash-scripts
spec:
  replicas: $replicas
  selector:
    matchLabels:
      app: $name
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: $name
    spec:
      containers:
      - name: $name
        image: $image
        ports:
        - containerPort: $port
        resources:
          requests:
            cpu: $cpu_req
            memory: $mem_req
          limits:
            cpu: $cpu_lim
            memory: $mem_lim
        livenessProbe:
          httpGet:
            path: /health
            port: $port
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: $port
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: APP_PORT
          value: "$port"
        - name: APP_ENV
          value: production
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      terminationGracePeriodSeconds: 60
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
EOF
}

# Generate Service manifest
generate_service() {
    local name="$1"
    local port="${2:-8080}"
    local type="${3:-ClusterIP}"  # ClusterIP, NodePort, LoadBalancer
    local ns="${4:-default}"
    
    cat << EOF
apiVersion: v1
kind: Service
metadata:
  name: $name
  namespace: $ns
  labels:
    app: $name
spec:
  type: $type
  selector:
    app: $name
  ports:
  - name: http
    port: 80
    targetPort: $port
    protocol: TCP
EOF
}

# Generate ConfigMap
generate_configmap() {
    local name="$1"
    local ns="${2:-default}"
    shift 2
    
    cat << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: $name
  namespace: $ns
data:
EOF
    
    while [[ $# -gt 0 ]]; do
        local kv="$1"
        local key="${kv%%=*}"
        local value="${kv#*=}"
        echo "  $key: \"$value\""
        shift
    done
}

# Generate Secret
generate_secret() {
    local name="$1"
    local ns="${2:-default}"
    shift 2
    
    cat << EOF
apiVersion: v1
kind: Secret
metadata:
  name: $name
  namespace: $ns
type: Opaque
data:
EOF
    
    while [[ $# -gt 0 ]]; do
        local kv="$1"
        local key="${kv%%=*}"
        local value="${kv#*=}"
        local encoded
        encoded=$(echo -n "$value" | base64)
        echo "  $key: $encoded"
        shift
    done
}

# Generate HorizontalPodAutoscaler
generate_hpa() {
    local name="$1"
    local min="${2:-2}"
    local max="${3:-10}"
    local cpu_target="${4:-70}"
    local ns="${5:-default}"
    
    cat << EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: $name
  namespace: $ns
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: $name
  minReplicas: $min
  maxReplicas: $max
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: $cpu_target
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
EOF
}

# Generate complete application stack
generate_app_stack() {
    local app_name="$1"
    local image="$2"
    local ns="${3:-default}"
    local output_dir="${4:-.}"
    
    mkdir -p "$output_dir"
    
    echo "Generating manifests for $app_name..."
    
    generate_deployment "$app_name" "$image" 2 8080 "$ns" \
        > "$output_dir/deployment.yaml"
    
    generate_service "$app_name" 8080 "ClusterIP" "$ns" \
        > "$output_dir/service.yaml"
    
    generate_configmap "${app_name}-config" "$ns" \
        "APP_ENV=production" \
        "LOG_LEVEL=info" \
        "MAX_CONNECTIONS=100" \
        > "$output_dir/configmap.yaml"
    
    generate_secret "${app_name}-secret" "$ns" \
        "DB_PASSWORD=secretpassword" \
        "API_KEY=myapikey" \
        > "$output_dir/secret.yaml"
    
    generate_hpa "$app_name" 2 10 70 "$ns" \
        > "$output_dir/hpa.yaml"
    
    echo "Manifests generated in: $output_dir"
    ls -la "$output_dir"/*.yaml
}

# Apply manifests
apply_manifests() {
    local dir="$1"
    local dry_run="${2:-false}"
    
    local args=()
    [[ "$dry_run" == "true" ]] && args+=(--dry-run=client)
    
    for yaml in "$dir"/*.yaml; do
        echo "Applying: $(basename "$yaml")"
        kubectl apply -f "$yaml" "${args[@]}"
    done
}

# Demo
echo "=== Manifest Generation Demo ==="
OUTPUT_DIR="/tmp/k8s_manifests_$$"
generate_app_stack "myapp" "myapp:latest" "production" "$OUTPUT_DIR"

echo ""
echo "=== Deployment Manifest ==="
cat "$OUTPUT_DIR/deployment.yaml"

rm -rf "$OUTPUT_DIR"
```

---

## ขั้นตอนที่ 458: Kubernetes Deployment Automation

```bash
#!/usr/bin/env bash
# k8s_deploy.sh

# ===== Kubernetes Deployment Script =====
set -euo pipefail

APP_NAME="${APP_NAME:-myapp}"
NAMESPACE="${NAMESPACE:-default}"
IMAGE_REPO="${IMAGE_REPO:-myrepo}"
IMAGE_TAG="${IMAGE_TAG:-latest}"
REPLICAS="${REPLICAS:-2}"
DEPLOY_TIMEOUT="${DEPLOY_TIMEOUT:-300}"

# Colors
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m'

log_info() { echo -e "${GREEN}[INFO]${NC} $*"; }
log_warn() { echo -e "${YELLOW}[WARN]${NC} $*"; }
log_error() { echo -e "${RED}[ERROR]${NC} $*" >&2; }

# Pre-deploy checks
pre_deploy_checks() {
    log_info "Running pre-deploy checks..."
    
    # Check kubectl
    if ! command -v kubectl &>/dev/null; then
        log_error "kubectl not found"
        return 1
    fi
    
    # Check namespace
    if ! kubectl get namespace "$NAMESPACE" &>/dev/null; then
        log_warn "Namespace $NAMESPACE not found, creating..."
        kubectl create namespace "$NAMESPACE"
    fi
    
    # Check if deployment exists
    if kubectl get deployment "$APP_NAME" -n "$NAMESPACE" &>/dev/null; then
        log_info "Existing deployment found: $APP_NAME"
        export DEPLOYMENT_EXISTS=true
    else
        log_info "New deployment: $APP_NAME"
        export DEPLOYMENT_EXISTS=false
    fi
    
    log_info "Pre-deploy checks passed"
}

# Image validation
validate_image() {
    local image="${IMAGE_REPO}/${APP_NAME}:${IMAGE_TAG}"
    log_info "Validating image: $image"
    
    if command -v docker &>/dev/null; then
        if docker manifest inspect "$image" &>/dev/null 2>&1; then
            log_info "Image found: $image"
            return 0
        else
            log_warn "Image not found in registry: $image"
            return 0  # Don't fail - might be pulled from cluster
        fi
    fi
    
    return 0
}

# Deploy application
deploy() {
    local image="${IMAGE_REPO}/${APP_NAME}:${IMAGE_TAG}"
    
    log_info "Deploying $APP_NAME with image $image..."
    
    if $DEPLOYMENT_EXISTS; then
        # Rolling update
        kubectl set image \
            deployment/"$APP_NAME" \
            "${APP_NAME}=${image}" \
            -n "$NAMESPACE"
        
        log_info "Image updated, monitoring rollout..."
    else
        # New deployment
        kubectl create deployment "$APP_NAME" \
            --image="$image" \
            --replicas="$REPLICAS" \
            -n "$NAMESPACE" || true
    fi
    
    # Monitor rollout
    if ! kubectl rollout status deployment/"$APP_NAME" \
        -n "$NAMESPACE" \
        --timeout="${DEPLOY_TIMEOUT}s"; then
        
        log_error "Deployment failed!"
        show_failure_info
        return 1
    fi
    
    log_info "Deployment successful!"
}

# Show failure info
show_failure_info() {
    log_warn "Deployment failure info:"
    
    echo "--- Events ---"
    kubectl get events -n "$NAMESPACE" \
        --sort-by='.lastTimestamp' \
        --field-selector "involvedObject.name=$APP_NAME" 2>/dev/null | tail -10
    
    echo "--- Pod Status ---"
    kubectl get pods -n "$NAMESPACE" \
        -l "app=$APP_NAME" 2>/dev/null
    
    echo "--- Recent Pod Logs ---"
    kubectl logs -n "$NAMESPACE" \
        -l "app=$APP_NAME" \
        --tail=50 \
        --previous 2>/dev/null | tail -20
}

# Rollback on failure
rollback() {
    log_warn "Rolling back deployment..."
    
    kubectl rollout undo deployment/"$APP_NAME" -n "$NAMESPACE"
    
    if kubectl rollout status deployment/"$APP_NAME" \
        -n "$NAMESPACE" \
        --timeout="120s"; then
        log_info "Rollback successful"
    else
        log_error "Rollback also failed!"
        return 1
    fi
}

# Smoke test
smoke_test() {
    local service_url="${1:-}"
    
    if [[ -z "$service_url" ]]; then
        log_warn "No service URL for smoke test"
        return 0
    fi
    
    log_info "Running smoke tests..."
    
    local max_attempts=5
    local attempt=0
    
    while [[ $attempt -lt $max_attempts ]]; do
        ((attempt++))
        
        if curl -sf "$service_url/health" &>/dev/null; then
            log_info "Smoke test passed (attempt $attempt)"
            return 0
        fi
        
        log_warn "Attempt $attempt failed, retrying..."
        sleep 10
    done
    
    log_error "Smoke tests failed after $max_attempts attempts"
    return 1
}

# Post-deploy tasks
post_deploy() {
    log_info "Running post-deploy tasks..."
    
    # Scale if needed
    local current_replicas
    current_replicas=$(kubectl get deployment "$APP_NAME" \
        -n "$NAMESPACE" \
        -o jsonpath='{.spec.replicas}' 2>/dev/null || echo "0")
    
    if [[ "$current_replicas" != "$REPLICAS" ]]; then
        log_info "Scaling to $REPLICAS replicas..."
        kubectl scale deployment "$APP_NAME" \
            --replicas="$REPLICAS" \
            -n "$NAMESPACE"
    fi
    
    log_info "Post-deploy complete"
}

# Main deployment flow
main() {
    local image_tag="${1:-$IMAGE_TAG}"
    IMAGE_TAG="$image_tag"
    
    log_info "=== Deployment: $APP_NAME ==="
    log_info "Image: ${IMAGE_REPO}/${APP_NAME}:${IMAGE_TAG}"
    log_info "Namespace: $NAMESPACE"
    log_info "Replicas: $REPLICAS"
    echo ""
    
    # Run with error handling
    if pre_deploy_checks && validate_image; then
        if deploy; then
            post_deploy
            log_info "Deployment complete: $APP_NAME"
        else
            log_error "Deployment failed, attempting rollback..."
            rollback || true
            exit 1
        fi
    else
        log_error "Pre-deploy checks failed"
        exit 1
    fi
}

# Demo (without real cluster)
if command -v kubectl &>/dev/null; then
    echo "kubectl available, running demo..."
    main "v1.2.3" 2>/dev/null || echo "Demo complete (cluster may not be available)"
else
    echo "kubectl not available"
    echo "This script provides functions for Kubernetes deployment automation"
    echo ""
    echo "Key functions:"
    echo "  pre_deploy_checks - validate environment"
    echo "  deploy - perform rolling update"
    echo "  rollback - undo deployment"
    echo "  smoke_test - verify deployment"
fi
```

---

## ขั้นตอนที่ 459: Kubernetes Monitoring

```bash
#!/usr/bin/env bash
# k8s_monitoring.sh

# ===== Kubernetes Monitoring Scripts =====

NAMESPACE="${NAMESPACE:--A}"

# Cluster health overview
cluster_health() {
    echo "=== Cluster Health ==="
    echo ""
    
    # Node status
    echo "--- Nodes ---"
    kubectl get nodes -o wide 2>/dev/null | \
        awk 'NR==1 || /NotReady/ {print "  " $0}'
    
    echo ""
    echo "Node count:"
    kubectl get nodes 2>/dev/null | grep -c "Ready" | \
        awk '{print "  Ready: " $1}'
    kubectl get nodes 2>/dev/null | grep -c "NotReady" | \
        awk '{print "  Not Ready: " $1}'
    
    # System pods
    echo ""
    echo "--- System Pods Status ---"
    kubectl get pods -n kube-system 2>/dev/null | \
        awk 'NR==1 || !/Running/ {print "  " $0}' | head -20
    
    # Resource usage
    echo ""
    echo "--- Resource Usage ---"
    kubectl top nodes 2>/dev/null || echo "  (metrics-server not available)"
}

# Find problematic pods
find_problem_pods() {
    local ns="${1:--A}"
    
    echo "=== Problem Pods ==="
    
    echo "CrashLoopBackOff:"
    kubectl get pods $([[ "$ns" != "-A" ]] && echo "-n $ns" || echo "-A") 2>/dev/null | \
        grep "CrashLoopBackOff" | awk '{print "  " $0}'
    
    echo ""
    echo "OOMKilled / Error:"
    kubectl get pods $([[ "$ns" != "-A" ]] && echo "-n $ns" || echo "-A") 2>/dev/null | \
        grep -E "Error|OOMKilled|Evicted" | awk '{print "  " $0}'
    
    echo ""
    echo "Pending pods:"
    kubectl get pods $([[ "$ns" != "-A" ]] && echo "-n $ns" || echo "-A") 2>/dev/null | \
        grep "Pending" | awk '{print "  " $0}'
    
    echo ""
    echo "Restart counts > 3:"
    kubectl get pods $([[ "$ns" != "-A" ]] && echo "-n $ns" || echo "-A") 2>/dev/null | \
        awk 'NR>1 {if ($4+0 > 3) print "  " $0}'
}

# Resource quota overview
resource_quotas() {
    echo "=== Resource Quotas ==="
    kubectl get resourcequota -A 2>/dev/null || echo "No quotas defined"
    
    echo ""
    echo "=== LimitRanges ==="
    kubectl get limitrange -A 2>/dev/null || echo "No limit ranges defined"
}

# Check PVC status
pvc_status() {
    echo "=== PersistentVolumeClaims ==="
    kubectl get pvc -A 2>/dev/null | awk 'NR==1 || !/Bound/ {print "  " $0}'
    
    echo ""
    echo "Available PVs:"
    kubectl get pv 2>/dev/null | awk 'NR==1 || /Available|Released/ {print "  " $0}'
}

# Service connectivity test
test_service() {
    local service="$1"
    local ns="${2:-default}"
    local port="${3:-80}"
    
    echo "Testing service: $service.$ns:$port"
    
    # Get service IP
    local cluster_ip
    cluster_ip=$(kubectl get service "$service" -n "$ns" \
        -o jsonpath='{.spec.clusterIP}' 2>/dev/null)
    
    if [[ -z "$cluster_ip" || "$cluster_ip" == "None" ]]; then
        echo "  Service has no ClusterIP"
        return 1
    fi
    
    echo "  ClusterIP: $cluster_ip"
    
    # Test from within cluster using a debug pod
    kubectl run test-pod-$$ \
        --image=busybox \
        --rm \
        --restart=Never \
        -it \
        -- wget -q -O- "http://${cluster_ip}:${port}/health" 2>/dev/null && \
        echo "  Service reachable" || \
        echo "  Service unreachable"
}

# Get pod logs with context
pod_log_analysis() {
    local label_selector="$1"
    local ns="${2:-default}"
    local time_window="${3:-1h}"
    
    echo "=== Log Analysis: $label_selector ==="
    
    # Get pod names
    local pods
    pods=$(kubectl get pods -n "$ns" -l "$label_selector" \
        -o jsonpath='{.items[*].metadata.name}' 2>/dev/null)
    
    for pod in $pods; do
        echo "--- Pod: $pod ---"
        kubectl logs -n "$ns" "$pod" \
            --since="$time_window" 2>/dev/null | \
            grep -E "ERROR|WARN|FATAL|Exception|panic" | \
            tail -10
    done
}

# Deployment status dashboard
deployment_dashboard() {
    local ns="${1:-default}"
    
    echo "=== Deployment Dashboard: $ns ==="
    printf "%-30s %8s %8s %10s\n" "DEPLOYMENT" "DESIRED" "READY" "STATUS"
    printf "%-30s %8s %8s %10s\n" "----------" "-------" "-----" "------"
    
    kubectl get deployments -n "$ns" 2>/dev/null | tail -n +2 | \
        while read -r name ready up_to_date available age; do
            local desired="${ready%%/*}"
            local actual="${ready##*/}"
            
            local status="OK"
            [[ "$desired" != "$actual" ]] && status="DEGRADED"
            
            printf "%-30s %8s %8s %10s\n" \
                "$name" "$desired" "$actual" "$status"
        done
}

# Demo
if command -v kubectl &>/dev/null && kubectl cluster-info &>/dev/null 2>&1; then
    echo "=== Kubernetes Monitoring Demo ==="
    cluster_health
    echo ""
    find_problem_pods
else
    echo "Kubernetes not available"
    echo "Functions available:"
    echo "  cluster_health - overview of cluster health"
    echo "  find_problem_pods - find pods with issues"
    echo "  deployment_dashboard [namespace] - show deployment status"
    echo "  pod_log_analysis [selector] [ns] - analyze pod logs"
fi
```

---

## ขั้นตอนที่ 460: Helm Integration

```bash
#!/usr/bin/env bash
# helm_integration.sh

# ===== Helm Chart Management =====

# Check Helm
check_helm() {
    if ! command -v helm &>/dev/null; then
        echo "Helm not found" >&2
        echo "Install: https://helm.sh/docs/intro/install/"
        return 1
    fi
    echo "Helm: $(helm version --short 2>/dev/null)"
}

# Add repository
helm_add_repo() {
    local name="$1"
    local url="$2"
    
    if helm repo list 2>/dev/null | grep -q "^$name"; then
        echo "Repo exists: $name"
        return 0
    fi
    
    helm repo add "$name" "$url"
    helm repo update
    echo "Added repo: $name"
}

# Install or upgrade chart
helm_deploy() {
    local release_name="$1"
    local chart="$2"
    local ns="${3:-default}"
    local values_file="${4:-}"
    shift 4
    
    local args=(
        upgrade --install
        "$release_name"
        "$chart"
        --namespace "$ns"
        --create-namespace
        --wait
        --timeout 5m
    )
    
    [[ -n "$values_file" && -f "$values_file" ]] && \
        args+=(-f "$values_file")
    
    # Additional --set arguments
    while [[ $# -gt 0 ]]; do
        args+=(--set "$1")
        shift
    done
    
    echo "Deploying: $release_name from $chart in $ns"
    helm "${args[@]}"
}

# Generate Helm chart
create_helm_chart() {
    local app_name="$1"
    local output_dir="${2:-./charts}"
    
    mkdir -p "$output_dir/$app_name"/{templates,charts}
    
    # Chart.yaml
    cat > "$output_dir/$app_name/Chart.yaml" << EOF
apiVersion: v2
name: $app_name
description: A Helm chart for $app_name
type: application
version: 0.1.0
appVersion: "1.0.0"
EOF
    
    # values.yaml
    cat > "$output_dir/$app_name/values.yaml" << EOF
replicaCount: 2

image:
  repository: $app_name
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

config:
  logLevel: info
  appEnv: production
EOF
    
    # templates/deployment.yaml
    cat > "$output_dir/$app_name/templates/deployment.yaml" << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Release.Name }}
    chart: {{ .Chart.Name }}-{{ .Chart.Version }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: {{ .Release.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        env:
        - name: LOG_LEVEL
          value: {{ .Values.config.logLevel | quote }}
        - name: APP_ENV
          value: {{ .Values.config.appEnv | quote }}
EOF
    
    # templates/service.yaml
    cat > "$output_dir/$app_name/templates/service.yaml" << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
  selector:
    app: {{ .Release.Name }}
EOF
    
    echo "Chart created: $output_dir/$app_name"
    ls -la "$output_dir/$app_name/"
}

# List and manage releases
helm_status() {
    local ns="${1:-default}"
    
    echo "=== Helm Releases in $ns ==="
    helm list -n "$ns" 2>/dev/null
}

helm_history() {
    local release="$1"
    local ns="${2:-default}"
    
    echo "=== Release History: $release ==="
    helm history "$release" -n "$ns" 2>/dev/null
}

helm_rollback() {
    local release="$1"
    local revision="${2:-}"
    local ns="${3:-default}"
    
    if [[ -n "$revision" ]]; then
        helm rollback "$release" "$revision" -n "$ns"
    else
        helm rollback "$release" -n "$ns"  # rollback to previous
    fi
    
    echo "Rolled back: $release"
}

# Demo
if command -v helm &>/dev/null; then
    check_helm
    
    echo ""
    echo "Creating sample chart..."
    create_helm_chart "myapp" "/tmp/charts_$$"
    rm -rf "/tmp/charts_$$"
else
    echo "Helm not available"
    echo ""
    echo "Example usage:"
    echo "  helm_add_repo stable https://charts.helm.sh/stable"
    echo "  helm_deploy myapp stable/nginx production values.yaml"
    echo "  create_helm_chart myapp ./charts"
fi
```

---

## สรุป Part 26

### สิ่งที่เรียนรู้ (Steps 456-460):

| Step | หัวข้อ | เนื้อหา |
|------|--------|---------|
| 456 | kubectl Basics | commands, namespace, rollout, port-forward |
| 457 | Manifest Generation | Deployment, Service, ConfigMap, Secret, HPA |
| 458 | Deployment Automation | rolling update, rollback, smoke test |
| 459 | Monitoring | cluster health, problem pods, log analysis |
| 460 | Helm Integration | chart management, deployment, rollback |

**ขั้นตอนต่อไป**: Part 27 - Cloud Provider Integration
