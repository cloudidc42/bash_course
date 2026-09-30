# Part 34: GitOps และ Platform Engineering

## Module 3: Advanced Level - การจัดการ GitOps และ Platform Engineering

---

## ขั้นตอนที่ 493: GitOps Fundamentals และ Flux CD

GitOps คือแนวปฏิบัติในการใช้ Git เป็น Single Source of Truth สำหรับ Infrastructure และ Application Deployment

### Flux CD Manager

```bash
#!/bin/bash
# flux-manager.sh - Flux CD GitOps Manager

set -euo pipefail

FLUX_NAMESPACE="${FLUX_NAMESPACE:-flux-system}"
GIT_REPO_URL="${GIT_REPO_URL:-}"
GIT_BRANCH="${GIT_BRANCH:-main}"
KUBECONFIG="${KUBECONFIG:-$HOME/.kube/config}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Install Flux CLI
install_flux_cli() {
    log "Installing Flux CLI..."
    
    if command -v flux &>/dev/null; then
        log "Flux CLI already installed: $(flux version --client 2>/dev/null | head -1)"
        return 0
    fi
    
    curl -s https://fluxcd.io/install.sh | bash
    
    if command -v flux &>/dev/null; then
        success "Flux CLI installed successfully"
    else
        error "Failed to install Flux CLI"
        return 1
    fi
}

# Bootstrap Flux on cluster
bootstrap_flux() {
    local git_token="${1:-$GITHUB_TOKEN}"
    local git_owner="${2:-}"
    local git_repo="${3:-}"
    
    if [[ -z "$git_token" || -z "$git_owner" || -z "$git_repo" ]]; then
        error "Usage: bootstrap_flux <token> <owner> <repo>"
        return 1
    fi
    
    log "Bootstrapping Flux on cluster..."
    
    flux bootstrap github \
        --owner="$git_owner" \
        --repository="$git_repo" \
        --branch="$GIT_BRANCH" \
        --path=clusters/my-cluster \
        --personal \
        --token-auth
    
    success "Flux bootstrapped successfully"
}

# Create GitRepository source
create_git_source() {
    local name="${1:-app-source}"
    local url="${2:-$GIT_REPO_URL}"
    local branch="${3:-main}"
    local interval="${4:-1m}"
    
    log "Creating GitRepository source: $name"
    
    flux create source git "$name" \
        --url="$url" \
        --branch="$branch" \
        --interval="$interval" \
        --export > "gitrepository-${name}.yaml"
    
    kubectl apply -f "gitrepository-${name}.yaml"
    success "GitRepository source created: $name"
}

# Create Kustomization
create_kustomization() {
    local name="${1:-app-kustomization}"
    local source="${2:-app-source}"
    local path="${3:-./deploy}"
    local interval="${4:-5m}"
    local namespace="${5:-default}"
    
    log "Creating Kustomization: $name"
    
    flux create kustomization "$name" \
        --source="GitRepository/$source" \
        --path="$path" \
        --prune=true \
        --interval="$interval" \
        --target-namespace="$namespace" \
        --export > "kustomization-${name}.yaml"
    
    kubectl apply -f "kustomization-${name}.yaml"
    success "Kustomization created: $name"
}

# Create HelmRelease
create_helm_release() {
    local name="${1:-my-app}"
    local chart="${2:-}"
    local chart_version="${3:-*}"
    local values_file="${4:-}"
    local namespace="${5:-default}"
    
    if [[ -z "$chart" ]]; then
        error "Usage: create_helm_release <name> <chart> [version] [values-file] [namespace]"
        return 1
    fi
    
    log "Creating HelmRelease: $name"
    
    local chart_name
    chart_name=$(echo "$chart" | cut -d/ -f2)
    local repo_name
    repo_name=$(echo "$chart" | cut -d/ -f1)
    
    # Create HelmRepository if needed
    cat <<EOF | kubectl apply -f -
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: ${repo_name}
  namespace: ${FLUX_NAMESPACE}
spec:
  interval: 1h
  url: https://charts.${repo_name}.io
EOF

    # Create HelmRelease manifest
    local values_section=""
    if [[ -n "$values_file" && -f "$values_file" ]]; then
        values_section="
  values:
$(cat "$values_file" | sed 's/^/    /')"
    fi
    
    cat <<EOF | kubectl apply -f -
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: ${name}
  namespace: ${namespace}
spec:
  interval: 5m
  chart:
    spec:
      chart: ${chart_name}
      version: '${chart_version}'
      sourceRef:
        kind: HelmRepository
        name: ${repo_name}
        namespace: ${FLUX_NAMESPACE}
  install:
    createNamespace: true
    remediation:
      retries: 3
  upgrade:
    remediation:
      remediateLastFailure: true
      retries: 3
      strategy: rollback${values_section}
EOF
    
    success "HelmRelease created: $name"
}

# Monitor Flux reconciliation
monitor_reconciliation() {
    local resource_type="${1:-all}"
    local watch="${2:-false}"
    
    log "Monitoring Flux reconciliation..."
    
    if [[ "$watch" == "true" ]]; then
        watch -n 10 flux get "$resource_type" --all-namespaces
    else
        flux get "$resource_type" --all-namespaces
    fi
}

# Get Flux logs
get_flux_logs() {
    local controller="${1:-all}"
    local since="${2:-1h}"
    
    if [[ "$controller" == "all" ]]; then
        for ctrl in source-controller kustomize-controller helm-controller notification-controller; do
            echo "=== $ctrl ==="
            kubectl logs -n "$FLUX_NAMESPACE" "deployment/$ctrl" --since="$since" 2>/dev/null || true
            echo ""
        done
    else
        kubectl logs -n "$FLUX_NAMESPACE" "deployment/${controller}-controller" --since="$since"
    fi
}

# Force reconcile
force_reconcile() {
    local type="${1:-kustomization}"
    local name="${2:-}"
    local namespace="${3:-default}"
    
    if [[ -z "$name" ]]; then
        error "Usage: force_reconcile <type> <name> [namespace]"
        return 1
    fi
    
    log "Force reconciling $type/$name in $namespace..."
    
    flux reconcile "$type" "$name" -n "$namespace"
    success "Reconciliation triggered for $type/$name"
}

# Suspend/Resume workloads
suspend_workload() {
    local type="${1:-kustomization}"
    local name="${2:-}"
    local namespace="${3:-default}"
    
    if [[ -z "$name" ]]; then
        error "Usage: suspend_workload <type> <name> [namespace]"
        return 1
    fi
    
    log "Suspending $type/$name..."
    flux suspend "$type" "$name" -n "$namespace"
    success "Suspended $type/$name"
}

resume_workload() {
    local type="${1:-kustomization}"
    local name="${2:-}"
    local namespace="${3:-default}"
    
    if [[ -z "$name" ]]; then
        error "Usage: resume_workload <type> <name> [namespace]"
        return 1
    fi
    
    log "Resuming $type/$name..."
    flux resume "$type" "$name" -n "$namespace"
    success "Resumed $type/$name"
}

# Create notification provider
create_notification() {
    local provider_name="${1:-slack}"
    local provider_type="${2:-slack}"
    local webhook_url="${3:-}"
    local alert_name="${4:-my-alert}"
    
    if [[ -z "$webhook_url" ]]; then
        error "Webhook URL required"
        return 1
    fi
    
    log "Creating notification provider: $provider_name"
    
    # Create secret for webhook
    kubectl create secret generic "${provider_name}-webhook" \
        --from-literal=address="$webhook_url" \
        -n "$FLUX_NAMESPACE" \
        --dry-run=client -o yaml | kubectl apply -f -
    
    # Create Provider
    cat <<EOF | kubectl apply -f -
apiVersion: notification.toolkit.fluxcd.io/v1beta2
kind: Provider
metadata:
  name: ${provider_name}
  namespace: ${FLUX_NAMESPACE}
spec:
  type: ${provider_type}
  secretRef:
    name: ${provider_name}-webhook
EOF

    # Create Alert
    cat <<EOF | kubectl apply -f -
apiVersion: notification.toolkit.fluxcd.io/v1beta2
kind: Alert
metadata:
  name: ${alert_name}
  namespace: ${FLUX_NAMESPACE}
spec:
  providerRef:
    name: ${provider_name}
  eventSeverity: info
  eventSources:
    - kind: GitRepository
      name: '*'
    - kind: Kustomization
      name: '*'
    - kind: HelmRelease
      name: '*'
  suspend: false
EOF
    
    success "Notification setup complete: $provider_name"
}

# Main
case "${1:-help}" in
    install) install_flux_cli ;;
    bootstrap) bootstrap_flux "$2" "$3" "$4" ;;
    source) create_git_source "${2:-}" "${3:-}" "${4:-main}" "${5:-1m}" ;;
    kustomize) create_kustomization "${2:-}" "${3:-}" "${4:-./deploy}" "${5:-5m}" ;;
    helm) create_helm_release "${2:-}" "${3:-}" "${4:-*}" "${5:-}" "${6:-default}" ;;
    monitor) monitor_reconciliation "${2:-all}" "${3:-false}" ;;
    logs) get_flux_logs "${2:-all}" "${3:-1h}" ;;
    reconcile) force_reconcile "${2:-kustomization}" "${3:-}" "${4:-default}" ;;
    suspend) suspend_workload "${2:-kustomization}" "${3:-}" "${4:-default}" ;;
    resume) resume_workload "${2:-kustomization}" "${3:-}" "${4:-default}" ;;
    notify) create_notification "${2:-slack}" "${3:-slack}" "${4:-}" "${5:-my-alert}" ;;
    *)
        echo "Usage: $0 {install|bootstrap|source|kustomize|helm|monitor|logs|reconcile|suspend|resume|notify}"
        ;;
esac
```

---

## ขั้นตอนที่ 494: ArgoCD Advanced GitOps

ArgoCD เป็น Declarative GitOps Continuous Delivery tool สำหรับ Kubernetes

```bash
#!/bin/bash
# argocd-gitops.sh - Advanced ArgoCD GitOps Manager

set -euo pipefail

ARGOCD_SERVER="${ARGOCD_SERVER:-localhost:8080}"
ARGOCD_NAMESPACE="${ARGOCD_NAMESPACE:-argocd}"
ARGOCD_TOKEN="${ARGOCD_TOKEN:-}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# ArgoCD API call
argocd_api() {
    local method="${1:-GET}"
    local endpoint="${2:-}"
    local data="${3:-}"
    
    local headers=(-H "Content-Type: application/json")
    if [[ -n "$ARGOCD_TOKEN" ]]; then
        headers+=(-H "Authorization: Bearer $ARGOCD_TOKEN")
    fi
    
    if [[ -n "$data" ]]; then
        curl -sk -X "$method" \
            "${headers[@]}" \
            -d "$data" \
            "https://${ARGOCD_SERVER}/api/v1/${endpoint}"
    else
        curl -sk -X "$method" \
            "${headers[@]}" \
            "https://${ARGOCD_SERVER}/api/v1/${endpoint}"
    fi
}

# Login to ArgoCD
argocd_login() {
    local username="${1:-admin}"
    local password="${2:-}"
    
    if [[ -z "$password" ]]; then
        error "Password required"
        return 1
    fi
    
    log "Logging into ArgoCD..."
    
    local response
    response=$(argocd_api POST "session" \
        "{\"username\":\"$username\",\"password\":\"$password\"}")
    
    ARGOCD_TOKEN=$(echo "$response" | jq -r '.token // empty')
    
    if [[ -n "$ARGOCD_TOKEN" ]]; then
        success "Logged in as $username"
        export ARGOCD_TOKEN
    else
        error "Login failed: $response"
        return 1
    fi
}

# Create ArgoCD Application
create_application() {
    local app_name="${1:-my-app}"
    local git_repo="${2:-}"
    local git_path="${3:-./}"
    local git_revision="${4:-HEAD}"
    local dest_namespace="${5:-default}"
    local dest_cluster="${6:-https://kubernetes.default.svc}"
    local project="${7:-default}"
    local sync_policy="${8:-automated}"
    
    if [[ -z "$git_repo" ]]; then
        error "Usage: create_application <name> <git-repo> [path] [revision] [namespace] [cluster] [project]"
        return 1
    fi
    
    log "Creating ArgoCD application: $app_name"
    
    local auto_sync_spec=""
    if [[ "$sync_policy" == "automated" ]]; then
        auto_sync_spec='"automated":{"prune":true,"selfHeal":true},'
    fi
    
    local app_spec
    app_spec=$(cat <<EOF
{
  "metadata": {
    "name": "$app_name",
    "namespace": "$ARGOCD_NAMESPACE"
  },
  "spec": {
    "project": "$project",
    "source": {
      "repoURL": "$git_repo",
      "path": "$git_path",
      "targetRevision": "$git_revision"
    },
    "destination": {
      "server": "$dest_cluster",
      "namespace": "$dest_namespace"
    },
    "syncPolicy": {
      ${auto_sync_spec}
      "syncOptions": [
        "CreateNamespace=true",
        "PrunePropagationPolicy=foreground",
        "ApplyOutOfSyncOnly=true"
      ],
      "retry": {
        "limit": 5,
        "backoff": {
          "duration": "5s",
          "factor": 2,
          "maxDuration": "3m"
        }
      }
    }
  }
}
EOF
)
    
    argocd_api POST "applications" "$app_spec"
    success "Application created: $app_name"
}

# Create Application with Helm
create_helm_application() {
    local app_name="${1:-my-helm-app}"
    local chart_repo="${2:-}"
    local chart="${3:-}"
    local chart_version="${4:-*}"
    local values_file="${5:-}"
    local dest_namespace="${6:-default}"
    
    if [[ -z "$chart_repo" || -z "$chart" ]]; then
        error "Usage: create_helm_application <name> <chart-repo> <chart> [version] [values-file] [namespace]"
        return 1
    fi
    
    log "Creating Helm Application: $app_name"
    
    local helm_values=""
    if [[ -n "$values_file" && -f "$values_file" ]]; then
        helm_values='"valueFiles":["'$values_file'"],'
    fi
    
    local app_spec
    app_spec=$(cat <<EOF
{
  "metadata": {
    "name": "$app_name",
    "namespace": "$ARGOCD_NAMESPACE"
  },
  "spec": {
    "project": "default",
    "source": {
      "repoURL": "$chart_repo",
      "chart": "$chart",
      "targetRevision": "$chart_version",
      "helm": {
        ${helm_values}
        "releaseName": "$app_name"
      }
    },
    "destination": {
      "server": "https://kubernetes.default.svc",
      "namespace": "$dest_namespace"
    },
    "syncPolicy": {
      "automated": {
        "prune": true,
        "selfHeal": true
      },
      "syncOptions": ["CreateNamespace=true"]
    }
  }
}
EOF
)
    
    argocd_api POST "applications" "$app_spec"
    success "Helm Application created: $app_name"
}

# Create AppProject
create_project() {
    local project_name="${1:-my-project}"
    local description="${2:-My ArgoCD Project}"
    local repo_urls="${3:-*}"
    local dest_namespaces="${4:-*}"
    local allowed_cluster="${5:-*}"
    
    log "Creating ArgoCD Project: $project_name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ${project_name}
  namespace: ${ARGOCD_NAMESPACE}
spec:
  description: ${description}
  sourceRepos:
$(echo "$repo_urls" | tr ',' '\n' | while read -r repo; do
    echo "  - $repo"
done)
  destinations:
  - namespace: '${dest_namespaces}'
    server: '${allowed_cluster}'
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota
  - group: ''
    kind: LimitRange
  roles:
  - name: dev-role
    description: Developer role
    policies:
    - p, proj:${project_name}:dev-role, applications, get, ${project_name}/*, allow
    - p, proj:${project_name}:dev-role, applications, sync, ${project_name}/*, allow
    groups:
    - developers
EOF
    
    success "Project created: $project_name"
}

# Sync application
sync_application() {
    local app_name="${1:-}"
    local revision="${2:-HEAD}"
    local dry_run="${3:-false}"
    
    if [[ -z "$app_name" ]]; then
        error "Application name required"
        return 1
    fi
    
    log "Syncing application: $app_name (revision: $revision)"
    
    local sync_options="{}"
    if [[ "$dry_run" == "true" ]]; then
        sync_options='{"dryRun":true}'
    fi
    
    local sync_request
    sync_request=$(cat <<EOF
{
  "revision": "$revision",
  "syncOptions": {
    "items": [
      "CreateNamespace=true"
    ]
  },
  "dryRun": $([[ "$dry_run" == "true" ]] && echo "true" || echo "false")
}
EOF
)
    
    argocd_api POST "applications/${app_name}/sync" "$sync_request"
    success "Sync triggered for: $app_name"
}

# Get application status
get_app_status() {
    local app_name="${1:-}"
    
    if [[ -z "$app_name" ]]; then
        # List all applications
        argocd_api GET "applications" | jq -r '.items[] | "\(.metadata.name)\t\(.status.health.status)\t\(.status.sync.status)"' | \
            column -t -s $'\t' -N "APPLICATION,HEALTH,SYNC"
    else
        local status
        status=$(argocd_api GET "applications/$app_name")
        echo "$status" | jq '{
            name: .metadata.name,
            health: .status.health.status,
            sync: .status.sync.status,
            revision: .status.sync.revision,
            syncedAt: .status.operationState.finishedAt
        }'
    fi
}

# Rollback application
rollback_application() {
    local app_name="${1:-}"
    local history_id="${2:-}"
    
    if [[ -z "$app_name" ]]; then
        error "Application name required"
        return 1
    fi
    
    if [[ -z "$history_id" ]]; then
        # Show history and pick last successful
        log "Fetching deployment history for: $app_name"
        argocd_api GET "applications/${app_name}" | \
            jq -r '.status.history[] | "\(.id)\t\(.revision)\t\(.deployedAt)"' | \
            column -t -s $'\t' -N "ID,REVISION,DEPLOYED_AT"
        return 0
    fi
    
    log "Rolling back $app_name to history ID: $history_id"
    
    argocd_api POST "applications/${app_name}/rollback" \
        "{\"id\": $history_id}"
    
    success "Rollback triggered for $app_name"
}

# Health check with retry
health_check() {
    local app_name="${1:-}"
    local timeout="${2:-300}"
    local interval="${3:-10}"
    
    if [[ -z "$app_name" ]]; then
        error "Application name required"
        return 1
    fi
    
    log "Waiting for $app_name to be healthy (timeout: ${timeout}s)..."
    
    local elapsed=0
    while [[ $elapsed -lt $timeout ]]; do
        local health_status
        health_status=$(argocd_api GET "applications/$app_name" | \
            jq -r '.status.health.status // "Unknown"')
        
        local sync_status
        sync_status=$(argocd_api GET "applications/$app_name" | \
            jq -r '.status.sync.status // "Unknown"')
        
        log "Health: $health_status | Sync: $sync_status (${elapsed}s/${timeout}s)"
        
        if [[ "$health_status" == "Healthy" && "$sync_status" == "Synced" ]]; then
            success "Application $app_name is Healthy and Synced"
            return 0
        fi
        
        sleep "$interval"
        elapsed=$((elapsed + interval))
    done
    
    error "Timeout waiting for $app_name to become healthy"
    return 1
}

# ApplicationSet for multi-cluster deployment
create_application_set() {
    local name="${1:-my-appset}"
    local git_repo="${2:-}"
    local apps_path="${3:-apps}"
    
    if [[ -z "$git_repo" ]]; then
        error "Usage: create_application_set <name> <git-repo> [apps-path]"
        return 1
    fi
    
    log "Creating ApplicationSet: $name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: ${name}
  namespace: ${ARGOCD_NAMESPACE}
spec:
  generators:
  - git:
      repoURL: ${git_repo}
      revision: HEAD
      directories:
      - path: ${apps_path}/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: ${git_repo}
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
EOF
    
    success "ApplicationSet created: $name"
}

# Create multi-cluster ApplicationSet
create_multi_cluster_appset() {
    local name="${1:-multi-cluster-app}"
    local git_repo="${2:-}"
    local app_path="${3:-./deploy}"
    
    if [[ -z "$git_repo" ]]; then
        error "Usage: create_multi_cluster_appset <name> <git-repo> [app-path]"
        return 1
    fi
    
    log "Creating multi-cluster ApplicationSet: $name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: ${name}
  namespace: ${ARGOCD_NAMESPACE}
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production
  template:
    metadata:
      name: '${name}-{{name}}'
    spec:
      project: default
      source:
        repoURL: ${git_repo}
        targetRevision: HEAD
        path: ${app_path}
      destination:
        server: '{{server}}'
        namespace: '${name}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
EOF
    
    success "Multi-cluster ApplicationSet created: $name"
}

# Main
case "${1:-help}" in
    login) argocd_login "${2:-admin}" "${3:-}" ;;
    create-app) create_application "${2:-}" "${3:-}" "${4:-./}" "${5:-HEAD}" "${6:-default}" "${7:-}" "${8:-default}" ;;
    create-helm) create_helm_application "${2:-}" "${3:-}" "${4:-}" "${5:-*}" "${6:-}" "${7:-default}" ;;
    create-project) create_project "${2:-}" "${3:-}" "${4:-*}" "${5:-*}" "${6:-*}" ;;
    sync) sync_application "${2:-}" "${3:-HEAD}" "${4:-false}" ;;
    status) get_app_status "${2:-}" ;;
    rollback) rollback_application "${2:-}" "${3:-}" ;;
    health) health_check "${2:-}" "${3:-300}" "${4:-10}" ;;
    appset) create_application_set "${2:-}" "${3:-}" "${4:-apps}" ;;
    multi-cluster) create_multi_cluster_appset "${2:-}" "${3:-}" "${4:-./deploy}" ;;
    *)
        echo "Usage: $0 {login|create-app|create-helm|create-project|sync|status|rollback|health|appset|multi-cluster}"
        ;;
esac
```

---

## ขั้นตอนที่ 495: Platform Engineering - Internal Developer Platform (IDP)

Platform Engineering คือการสร้าง Internal Developer Platform เพื่อเพิ่มประสิทธิภาพของ Developer Experience

```bash
#!/bin/bash
# idp-manager.sh - Internal Developer Platform Manager

set -euo pipefail

IDP_NAMESPACE="${IDP_NAMESPACE:-platform}"
BACKSTAGE_URL="${BACKSTAGE_URL:-http://localhost:7007}"
CROSSPLANE_NAMESPACE="${CROSSPLANE_NAMESPACE:-crossplane-system}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Setup Crossplane for Infrastructure Abstraction
setup_crossplane() {
    log "Setting up Crossplane..."
    
    # Install Crossplane
    helm repo add crossplane-stable https://charts.crossplane.io/stable
    helm repo update
    
    helm upgrade --install crossplane \
        crossplane-stable/crossplane \
        --namespace "$CROSSPLANE_NAMESPACE" \
        --create-namespace \
        --set args='{--enable-environment-configs}' \
        --wait
    
    success "Crossplane installed"
}

# Install Crossplane AWS Provider
install_aws_provider() {
    local version="${1:-v0.41.0}"
    
    log "Installing AWS Provider for Crossplane..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: "xpkg.upbound.io/upbound/provider-aws:${version}"
EOF

    log "Waiting for provider to be installed..."
    kubectl wait provider provider-aws \
        --for=condition=Installed \
        --timeout=300s
    
    success "AWS Provider installed"
}

# Create Composite Resource Definition (XRD) for Database
create_database_xrd() {
    log "Creating Database Composite Resource Definition..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: XDatabase
    plural: xdatabases
  claimNames:
    kind: Database
    plural: databases
  connectionSecretKeys:
    - username
    - password
    - endpoint
    - port
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
            properties:
              parameters:
                type: object
                properties:
                  storageGB:
                    type: integer
                    default: 20
                    minimum: 10
                    maximum: 100
                  dbName:
                    type: string
                  engineVersion:
                    type: string
                    default: "14.9"
                  dbClass:
                    type: string
                    default: db.t3.micro
                    enum:
                    - db.t3.micro
                    - db.t3.small
                    - db.t3.medium
                    - db.m5.large
                required:
                - storageGB
                - dbName
EOF
    
    success "Database XRD created"
}

# Create Composition for Database
create_database_composition() {
    local region="${1:-us-east-1}"
    local subnet_group="${2:-my-subnet-group}"
    local vpc_security_group="${3:-sg-xxxxxxxx}"
    
    log "Creating Database Composition..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xdatabases.aws.platform.example.com
  labels:
    provider: aws
    db: postgresql
spec:
  compositeTypeRef:
    apiVersion: platform.example.com/v1alpha1
    kind: XDatabase
  writeConnectionSecretsToNamespace: ${CROSSPLANE_NAMESPACE}
  resources:
  - name: rdsinstance
    base:
      apiVersion: rds.aws.upbound.io/v1beta1
      kind: Instance
      spec:
        forProvider:
          region: ${region}
          instanceClass: db.t3.micro
          engine: postgres
          engineVersion: "14.9"
          username: masteruser
          autoGeneratePassword: true
          masterUserPasswordSecretRef:
            namespace: ${CROSSPLANE_NAMESPACE}
            name: master-db-password
            key: password
          skipFinalSnapshot: true
          dbSubnetGroupName: ${subnet_group}
          vpcSecurityGroupIds:
          - ${vpc_security_group}
          publiclyAccessible: false
          backupRetentionPeriod: 7
          storageEncrypted: true
          deletionProtection: false
        writeConnectionSecretToRef:
          namespace: ${CROSSPLANE_NAMESPACE}
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.storageGB
      toFieldPath: spec.forProvider.allocatedStorage
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.dbName
      toFieldPath: spec.forProvider.dbName
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.engineVersion
      toFieldPath: spec.forProvider.engineVersion
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.dbClass
      toFieldPath: spec.forProvider.instanceClass
    - type: FromCompositeFieldPath
      fromFieldPath: metadata.uid
      toFieldPath: spec.writeConnectionSecretToRef.name
      transforms:
      - type: string
        string:
          fmt: '%s-postgresql'
    connectionDetails:
    - fromConnectionSecretKey: username
    - fromConnectionSecretKey: password
    - fromConnectionSecretKey: endpoint
    - fromConnectionSecretKey: port
EOF
    
    success "Database Composition created"
}

# Developer self-service: Request a database
request_database() {
    local name="${1:-my-db}"
    local namespace="${2:-default}"
    local db_name="${3:-myapp}"
    local storage="${4:-20}"
    local db_class="${5:-db.t3.micro}"
    
    log "Requesting database: $name in namespace $namespace"
    
    cat <<EOF | kubectl apply -f -
apiVersion: platform.example.com/v1alpha1
kind: Database
metadata:
  name: ${name}
  namespace: ${namespace}
spec:
  parameters:
    storageGB: ${storage}
    dbName: ${db_name}
    dbClass: ${db_class}
  writeConnectionSecretToRef:
    name: ${name}-credentials
EOF
    
    success "Database claim created: $name"
    log "Connection details will be in secret: ${name}-credentials"
}

# Setup Backstage Software Catalog
setup_backstage_catalog() {
    local component_name="${1:-my-service}"
    local owner="${2:-team-platform}"
    local description="${3:-My Service}"
    local component_type="${4:-service}"
    local lifecycle="${5:-production}"
    local git_repo="${6:-}"
    
    log "Creating Backstage catalog entry: $component_name"
    
    cat <<EOF > "catalog-info-${component_name}.yaml"
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${component_name}
  description: ${description}
  annotations:
    backstage.io/techdocs-ref: dir:.
    github.com/project-slug: ${git_repo}
    grafana/dashboard-selector: '{"app":"${component_name}"}'
    prometheus.io/scrape: 'true'
  tags:
    - ${component_type}
    - ${lifecycle}
  links:
    - url: https://grafana.example.com/d/${component_name}
      title: Grafana Dashboard
      icon: dashboard
    - url: https://argocd.example.com/applications/${component_name}
      title: ArgoCD
      icon: web
spec:
  type: ${component_type}
  lifecycle: ${lifecycle}
  owner: ${owner}
  dependsOn:
    - resource:default/database
  providesApis:
    - ${component_name}-api
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: ${component_name}-api
  description: ${component_name} API
spec:
  type: openapi
  lifecycle: ${lifecycle}
  owner: ${owner}
  definition: |
    openapi: "3.0.0"
    info:
      title: ${component_name} API
      version: "1.0.0"
    paths: {}
EOF
    
    success "Catalog file created: catalog-info-${component_name}.yaml"
    
    if command -v curl &>/dev/null && [[ -n "$BACKSTAGE_URL" ]]; then
        log "Refreshing Backstage catalog..."
        curl -s -X POST \
            "${BACKSTAGE_URL}/api/catalog/locations" \
            -H "Content-Type: application/json" \
            -d "{\"type\":\"url\",\"target\":\"https://github.com/${git_repo}/blob/main/catalog-info-${component_name}.yaml\"}" \
            || log "Manual refresh required in Backstage UI"
    fi
}

# Create Service Template for Scaffolding
create_service_template() {
    local template_name="${1:-my-service-template}"
    local output_dir="${2:-./.backstage/templates}"
    
    mkdir -p "$output_dir"
    
    log "Creating Backstage service template: $template_name"
    
    cat <<'TEMPLATE' > "${output_dir}/${template_name}.yaml"
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: service-template
  title: Create New Microservice
  description: Template to create a new microservice with all platform integrations
  tags:
    - recommended
    - nodejs
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
          description: Unique name for the service
          pattern: '^[a-z0-9-]+$'
        description:
          title: Description
          type: string
        owner:
          title: Owner
          type: string
          ui:field: OwnerPicker
          ui:options:
            allowedKinds:
              - Group
        system:
          title: System
          type: string
          ui:field: EntityPicker
          ui:options:
            allowedKinds:
              - System
    - title: Infrastructure
      properties:
        needsDatabase:
          title: Database Required
          type: boolean
          default: false
        dbType:
          title: Database Type
          type: string
          enum:
            - postgres
            - mysql
            - mongodb
          default: postgres
        environment:
          title: Environment
          type: string
          enum:
            - development
            - staging
            - production
          default: development
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
    - id: create-database
      if: ${{ parameters.needsDatabase }}
      name: Provision Database
      action: kubernetes:apply
      input:
        manifest: |
          apiVersion: platform.example.com/v1alpha1
          kind: Database
          metadata:
            name: ${{ parameters.name }}-db
          spec:
            parameters:
              dbName: ${{ parameters.name }}
              storageGB: 20
            writeConnectionSecretToRef:
              name: ${{ parameters.name }}-db-credentials
    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        allowedHosts:
          - github.com
        description: ${{ parameters.description }}
        repoUrl: github.com?owner=myorg&repo=${{ parameters.name }}
    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
  output:
    links:
      - title: Repository
        url: ${{ steps['publish'].output.remoteUrl }}
      - title: Open in Catalog
        icon: catalog
        entityRef: ${{ steps['register'].output.entityRef }}
TEMPLATE
    
    success "Template created: ${output_dir}/${template_name}.yaml"
}

# Platform Golden Path - Complete service setup
platform_golden_path() {
    local service_name="${1:-}"
    local team="${2:-platform-team}"
    local git_org="${3:-myorg}"
    local environment="${4:-staging}"
    
    if [[ -z "$service_name" ]]; then
        error "Service name required"
        return 1
    fi
    
    log "Setting up golden path for service: $service_name"
    
    # 1. Create directory structure
    mkdir -p "${service_name}"/{src,tests,deploy,docs}
    
    # 2. Create Dockerfile
    cat <<EOF > "${service_name}/Dockerfile"
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY src ./src
USER node
EXPOSE 3000
CMD ["node", "src/index.js"]
EOF

    # 3. Create Kubernetes manifests
    mkdir -p "${service_name}/deploy"
    cat <<EOF > "${service_name}/deploy/deployment.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${service_name}
  labels:
    app: ${service_name}
    team: ${team}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${service_name}
  template:
    metadata:
      labels:
        app: ${service_name}
        team: ${team}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
    spec:
      containers:
      - name: ${service_name}
        image: ${git_org}/${service_name}:latest
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: NODE_ENV
          value: "${environment}"
        - name: SERVICE_NAME
          value: "${service_name}"
EOF

    cat <<EOF > "${service_name}/deploy/service.yaml"
apiVersion: v1
kind: Service
metadata:
  name: ${service_name}
  labels:
    app: ${service_name}
spec:
  selector:
    app: ${service_name}
  ports:
  - port: 80
    targetPort: 3000
    name: http
EOF

    cat <<EOF > "${service_name}/deploy/hpa.yaml"
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ${service_name}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ${service_name}
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
EOF

    # 4. Create GitHub Actions CI/CD
    mkdir -p "${service_name}/.github/workflows"
    cat <<EOF > "${service_name}/.github/workflows/ci-cd.yaml"
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    - run: npm ci
    - run: npm test
    - run: npm run lint

  build-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build and Push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${git_org}/${service_name}:latest,${git_org}/${service_name}:\${{ github.sha }}
    
    - name: Update manifest
      run: |
        sed -i 's|${git_org}/${service_name}:.*|${git_org}/${service_name}:\${{ github.sha }}|' deploy/deployment.yaml
        git config user.email "ci@example.com"
        git config user.name "CI Bot"
        git add deploy/deployment.yaml
        git commit -m "Update image to \${{ github.sha }}"
        git push
EOF

    # 5. Create catalog-info.yaml
    cat <<EOF > "${service_name}/catalog-info.yaml"
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${service_name}
  description: ${service_name} microservice
  annotations:
    github.com/project-slug: ${git_org}/${service_name}
    backstage.io/techdocs-ref: dir:.
  tags:
    - nodejs
    - microservice
spec:
  type: service
  lifecycle: experimental
  owner: ${team}
EOF

    success "Golden path setup complete for: $service_name"
    log "Files created in ./${service_name}/"
    find "${service_name}" -type f | sort
}

# Platform metrics dashboard
platform_metrics() {
    log "Platform Metrics Overview"
    echo "================================"
    
    # Crossplane metrics
    echo "=== Crossplane Managed Resources ==="
    kubectl get managed --all-namespaces 2>/dev/null | head -20 || echo "Crossplane not available"
    
    # ArgoCD application health
    echo ""
    echo "=== ArgoCD Applications ==="
    kubectl get applications -n argocd 2>/dev/null | head -20 || echo "ArgoCD not available"
    
    # Flux status
    echo ""
    echo "=== Flux GitOps ==="
    flux get all --all-namespaces 2>/dev/null | head -20 || echo "Flux not available"
    
    # Platform namespace usage
    echo ""
    echo "=== Namespace Resource Usage ==="
    kubectl top namespaces 2>/dev/null || echo "Metrics server not available"
}

# Main
case "${1:-help}" in
    setup-crossplane) setup_crossplane ;;
    aws-provider) install_aws_provider "${2:-v0.41.0}" ;;
    db-xrd) create_database_xrd ;;
    db-composition) create_database_composition "${2:-us-east-1}" "${3:-}" "${4:-}" ;;
    request-db) request_database "${2:-}" "${3:-default}" "${4:-myapp}" "${5:-20}" "${6:-db.t3.micro}" ;;
    catalog) setup_backstage_catalog "${2:-}" "${3:-}" "${4:-}" "${5:-service}" "${6:-production}" "${7:-}" ;;
    template) create_service_template "${2:-service-template}" "${3:-./.backstage/templates}" ;;
    golden-path) platform_golden_path "${2:-}" "${3:-platform-team}" "${4:-myorg}" "${5:-staging}" ;;
    metrics) platform_metrics ;;
    *)
        echo "Usage: $0 {setup-crossplane|aws-provider|db-xrd|db-composition|request-db|catalog|template|golden-path|metrics}"
        ;;
esac
```

---

## ขั้นตอนที่ 496: Developer Portal และ Service Catalog

### Backstage Configuration Manager

```bash
#!/bin/bash
# backstage-manager.sh - Backstage Developer Portal Manager

set -euo pipefail

BACKSTAGE_DIR="${BACKSTAGE_DIR:-./backstage}"
BACKSTAGE_PORT="${BACKSTAGE_PORT:-7007}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Install Backstage
create_backstage_app() {
    local app_name="${1:-developer-portal}"
    
    log "Creating Backstage app: $app_name"
    
    npx @backstage/create-app@latest --path "$app_name"
    
    success "Backstage app created: $app_name"
}

# Configure app-config.yaml
configure_backstage() {
    local github_org="${1:-myorg}"
    local github_token="${2:-$GITHUB_TOKEN}"
    local catalog_locations="${3:-}"
    local postgres_host="${4:-localhost}"
    local postgres_db="${5:-backstage}"
    
    log "Configuring Backstage..."
    
    cat <<EOF > "${BACKSTAGE_DIR}/app-config.production.yaml"
app:
  title: Developer Portal
  baseUrl: https://backstage.example.com

backend:
  baseUrl: https://backstage.example.com
  listen:
    port: ${BACKSTAGE_PORT}
  cors:
    origin: https://backstage.example.com
  database:
    client: pg
    connection:
      host: ${postgres_host}
      port: 5432
      user: backstage
      password: \${POSTGRES_PASSWORD}
      database: ${postgres_db}
  cache:
    store: redis
    connection: redis://redis:6379

integrations:
  github:
    - host: github.com
      token: \${GITHUB_TOKEN}

auth:
  environment: production
  providers:
    github:
      production:
        clientId: \${GITHUB_CLIENT_ID}
        clientSecret: \${GITHUB_CLIENT_SECRET}

catalog:
  import:
    entityFilename: catalog-info.yaml
    pullRequestBranchName: backstage-integration
  rules:
    - allow: [Component, System, API, Resource, Location, Template, User, Group]
  locations:
    - type: url
      target: https://github.com/${github_org}/.backstage-catalog/blob/main/all.yaml
    - type: url
      target: https://github.com/${github_org}/.backstage-catalog/blob/main/templates/all-templates.yaml

techdocs:
  builder: 'external'
  generator:
    runIn: 'local'
  publisher:
    type: 'awsS3'
    awsS3:
      bucketName: \${TECHDOCS_S3_BUCKET}
      region: us-east-1

kubernetes:
  serviceLocatorMethod:
    type: 'multiTenant'
  clusterLocatorMethods:
    - type: 'config'
      clusters:
        - name: production
          url: https://kubernetes.default.svc
          authProvider: 'serviceAccount'
          serviceAccountToken: \${K8S_SERVICE_ACCOUNT_TOKEN}
          skipTLSVerify: false
          skipMetricsLookup: false

permission:
  enabled: true
EOF
    
    success "Backstage configured"
}

# Create catalog location index
create_catalog_index() {
    local org="${1:-myorg}"
    local output_dir="${2:-./.backstage-catalog}"
    
    mkdir -p "$output_dir/templates"
    
    log "Creating catalog index for org: $org"
    
    cat <<EOF > "${output_dir}/all.yaml"
apiVersion: backstage.io/v1alpha1
kind: Location
metadata:
  name: ${org}-catalog
  description: ${org} Component Catalog
spec:
  targets:
    - ./components/*.yaml
    - ./apis/*.yaml
    - ./resources/*.yaml
    - ./systems/*.yaml
    - ./domains/*.yaml
EOF

    cat <<EOF > "${output_dir}/templates/all-templates.yaml"
apiVersion: backstage.io/v1alpha1
kind: Location
metadata:
  name: ${org}-templates
  description: ${org} Service Templates
spec:
  targets:
    - ./service-templates/*.yaml
    - ./infrastructure-templates/*.yaml
EOF

    # Create sample system
    mkdir -p "${output_dir}/systems"
    cat <<EOF > "${output_dir}/systems/platform-system.yaml"
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: platform-system
  description: Core Platform Infrastructure
  tags:
    - platform
spec:
  owner: platform-team
  domain: infrastructure
EOF

    # Create Platform team
    mkdir -p "${output_dir}/teams"
    cat <<EOF > "${output_dir}/teams/platform-team.yaml"
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: platform-team
  description: Platform Engineering Team
spec:
  type: team
  profile:
    displayName: Platform Team
    email: platform@example.com
  parent: engineering
  children: []
  members:
    - user:default/john.doe
    - user:default/jane.smith
EOF

    success "Catalog index created in: $output_dir"
}

# Deploy Backstage to Kubernetes
deploy_backstage_k8s() {
    local namespace="${1:-backstage}"
    local image="${2:-backstage/backstage:latest}"
    local replicas="${3:-2}"
    
    log "Deploying Backstage to Kubernetes..."
    
    kubectl create namespace "$namespace" --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backstage
  namespace: ${namespace}
spec:
  replicas: ${replicas}
  selector:
    matchLabels:
      app: backstage
  template:
    metadata:
      labels:
        app: backstage
    spec:
      serviceAccountName: backstage
      containers:
      - name: backstage
        image: ${image}
        imagePullPolicy: Always
        ports:
        - containerPort: ${BACKSTAGE_PORT}
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: backstage-secrets
              key: postgres-password
        - name: GITHUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: backstage-secrets
              key: github-token
        - name: GITHUB_CLIENT_ID
          valueFrom:
            secretKeyRef:
              name: backstage-secrets
              key: github-client-id
        - name: GITHUB_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: backstage-secrets
              key: github-client-secret
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1
            memory: 2Gi
        readinessProbe:
          httpGet:
            path: /healthcheck
            port: ${BACKSTAGE_PORT}
          initialDelaySeconds: 60
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: ${BACKSTAGE_PORT}
          initialDelaySeconds: 90
          periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: backstage
  namespace: ${namespace}
spec:
  selector:
    app: backstage
  ports:
  - port: 80
    targetPort: ${BACKSTAGE_PORT}
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backstage
  namespace: ${namespace}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backstage-read-only
rules:
- apiGroups: ['']
  resources: ['pods', 'services', 'namespaces', 'nodes']
  verbs: ['get', 'list', 'watch']
- apiGroups: ['apps']
  resources: ['deployments', 'replicasets', 'statefulsets', 'daemonsets']
  verbs: ['get', 'list', 'watch']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: backstage-read-only
subjects:
- kind: ServiceAccount
  name: backstage
  namespace: ${namespace}
roleRef:
  kind: ClusterRole
  name: backstage-read-only
  apiGroup: rbac.authorization.k8s.io
EOF
    
    success "Backstage deployed to namespace: $namespace"
}

# Main
case "${1:-help}" in
    create) create_backstage_app "${2:-developer-portal}" ;;
    configure) configure_backstage "${2:-myorg}" "${3:-}" "${4:-}" "${5:-localhost}" "${6:-backstage}" ;;
    catalog-index) create_catalog_index "${2:-myorg}" "${3:-./.backstage-catalog}" ;;
    deploy) deploy_backstage_k8s "${2:-backstage}" "${3:-backstage/backstage:latest}" "${4:-2}" ;;
    *)
        echo "Usage: $0 {create|configure|catalog-index|deploy}"
        ;;
esac
```

---

## ขั้นตอนที่ 497: Workshop - Complete GitOps Platform

```bash
#!/bin/bash
# gitops-platform-workshop.sh - Complete GitOps Platform Workshop

set -euo pipefail

WORKSHOP_DIR="${WORKSHOP_DIR:-./gitops-platform}"
CLUSTER_NAME="${CLUSTER_NAME:-my-cluster}"
GIT_ORG="${GIT_ORG:-myorg}"
DOMAIN="${DOMAIN:-example.com}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
separator() { echo "=========================================="; }

# Step 1: Prerequisites check
check_prerequisites() {
    log "Checking prerequisites..."
    separator
    
    local missing=()
    
    for tool in kubectl helm flux argocd jq curl git; do
        if command -v "$tool" &>/dev/null; then
            echo "✓ $tool: $(command -v "$tool")"
        else
            echo "✗ $tool: NOT FOUND"
            missing+=("$tool")
        fi
    done
    
    if [[ ${#missing[@]} -gt 0 ]]; then
        echo ""
        echo "Missing tools: ${missing[*]}"
        return 1
    fi
    
    echo ""
    echo "✓ All prerequisites met"
}

# Step 2: Setup GitOps repository structure
setup_gitops_repo() {
    log "Setting up GitOps repository structure..."
    separator
    
    mkdir -p "$WORKSHOP_DIR"/{clusters,apps,infra,platform}
    mkdir -p "$WORKSHOP_DIR/clusters/${CLUSTER_NAME}"/{flux-system,platform,apps}
    mkdir -p "$WORKSHOP_DIR/apps"/{production,staging,development}
    mkdir -p "$WORKSHOP_DIR/infra"/{networking,storage,monitoring,security}
    mkdir -p "$WORKSHOP_DIR/platform"/{crossplane,backstage,policies}
    
    # Create cluster-wide GitRepository
    cat <<EOF > "$WORKSHOP_DIR/clusters/${CLUSTER_NAME}/flux-system/gotk-sync.yaml"
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m
  ref:
    branch: main
  url: https://github.com/${GIT_ORG}/platform-config
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m
  path: ./clusters/${CLUSTER_NAME}
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
EOF

    # Create apps kustomization
    cat <<EOF > "$WORKSHOP_DIR/clusters/${CLUSTER_NAME}/apps/kustomization.yaml"
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../apps/production
EOF

    # Create apps/production kustomization
    cat <<EOF > "$WORKSHOP_DIR/apps/production/kustomization.yaml"
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - frontend.yaml
  - backend.yaml
  - database.yaml
EOF

    # Create sample app manifests
    cat <<EOF > "$WORKSHOP_DIR/apps/production/frontend.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: production
  labels:
    app: frontend
    environment: production
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        environment: production
        version: "1.0.0"
    spec:
      containers:
      - name: frontend
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
EOF

    # Create infrastructure kustomizations
    cat <<EOF > "$WORKSHOP_DIR/infra/monitoring/kustomization.yaml"
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - prometheus.yaml
  - grafana.yaml
EOF

    cat <<EOF > "$WORKSHOP_DIR/infra/monitoring/prometheus.yaml"
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: prometheus-stack
  namespace: monitoring
spec:
  interval: 1h
  chart:
    spec:
      chart: kube-prometheus-stack
      version: '*'
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
        namespace: flux-system
  install:
    createNamespace: true
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  values:
    prometheus:
      prometheusSpec:
        retention: 30d
        storageSpec:
          volumeClaimTemplate:
            spec:
              storageClassName: gp2
              resources:
                requests:
                  storage: 50Gi
    grafana:
      adminPassword: admin
      ingress:
        enabled: true
        hosts:
          - grafana.${DOMAIN}
EOF

    log "GitOps repository structure created in: $WORKSHOP_DIR"
}

# Step 3: Deploy monitoring stack
deploy_monitoring() {
    log "Deploying monitoring stack..."
    separator
    
    kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
    
    # Add Prometheus helm repo via Flux
    cat <<EOF | kubectl apply -f -
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: prometheus-community
  namespace: flux-system
spec:
  interval: 1h
  url: https://prometheus-community.github.io/helm-charts
EOF

    log "Monitoring stack deployment initiated"
}

# Step 4: Setup policy enforcement
setup_policy_enforcement() {
    log "Setting up OPA Gatekeeper policies..."
    separator
    
    # Install OPA Gatekeeper
    kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml 2>/dev/null || true
    
    # Wait for gatekeeper to be ready
    kubectl wait deployment gatekeeper-controller-manager \
        -n gatekeeper-system \
        --for=condition=Available \
        --timeout=120s 2>/dev/null || log "Gatekeeper may take more time to install"
    
    # Create constraint template: Require labels
    cat <<'EOF' | kubectl apply -f - 2>/dev/null || true
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      
      violation[{"msg": msg, "details": {"missing_labels": missing}}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("you must provide labels: %v", [missing])
      }
EOF

    # Create constraint: Require app and environment labels
    cat <<'EOF' | kubectl apply -f - 2>/dev/null || true
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-labels
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
    namespaces:
    - production
    - staging
  parameters:
    labels:
    - "app"
    - "environment"
    - "version"
EOF

    # Create network policies
    cat <<EOF | kubectl apply -f - 2>/dev/null || true
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
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
  egress:
  - to:
    - podSelector: {}
EOF
    
    log "Policy enforcement setup complete"
}

# Step 5: Multi-tenancy setup
setup_multi_tenancy() {
    log "Setting up multi-tenancy..."
    separator
    
    for team in team-a team-b team-c; do
        # Create namespace
        cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: ${team}
  labels:
    team: ${team}
    managed-by: platform
EOF

        # Create ResourceQuota
        cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ${team}-quota
  namespace: ${team}
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
EOF

        # Create LimitRange
        cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: ${team}-limitrange
  namespace: ${team}
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: 50m
      memory: 64Mi
EOF

        # Create ServiceAccount
        cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ${team}-deployer
  namespace: ${team}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ${team}-deployer-role
  namespace: ${team}
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ${team}-deployer-binding
  namespace: ${team}
subjects:
- kind: ServiceAccount
  name: ${team}-deployer
  namespace: ${team}
roleRef:
  kind: Role
  name: ${team}-deployer-role
  apiGroup: rbac.authorization.k8s.io
EOF

        log "Multi-tenancy setup complete for: $team"
    done
}

# Step 6: GitOps pipeline health check
check_gitops_health() {
    log "Checking GitOps platform health..."
    separator
    
    echo "=== Flux Status ==="
    flux get all --all-namespaces 2>/dev/null || echo "Flux not available"
    
    echo ""
    echo "=== ArgoCD Applications ==="
    kubectl get applications -n argocd 2>/dev/null | head -10 || echo "ArgoCD not available"
    
    echo ""
    echo "=== Crossplane Managed Resources ==="
    kubectl get managed 2>/dev/null | head -10 || echo "Crossplane not available"
    
    echo ""
    echo "=== OPA Gatekeeper Constraints ==="
    kubectl get constraints --all-namespaces 2>/dev/null | head -10 || echo "Gatekeeper not available"
    
    echo ""
    echo "=== Platform Namespaces ==="
    kubectl get namespaces -l managed-by=platform 2>/dev/null
    
    echo ""
    echo "=== Resource Quotas ==="
    kubectl get resourcequotas --all-namespaces 2>/dev/null | head -15
}

# Step 7: Deployment pipeline simulation
simulate_deployment() {
    local app_name="${1:-demo-app}"
    local image_tag="${2:-v1.0.0}"
    
    log "Simulating deployment for: $app_name:$image_tag"
    separator
    
    echo "Step 1: Validate manifests"
    echo "  ✓ Kubernetes manifests validated"
    echo "  ✓ Policy constraints passed"
    echo "  ✓ Resource limits set"
    
    echo ""
    echo "Step 2: Update GitOps repository"
    echo "  ✓ Image tag updated: $image_tag"
    echo "  ✓ Changes committed to Git"
    echo "  ✓ PR opened for review"
    
    echo ""
    echo "Step 3: GitOps reconciliation"
    echo "  ✓ Flux/ArgoCD detected changes"
    echo "  ✓ Sync triggered automatically"
    echo "  ✓ Old pods terminating"
    echo "  ✓ New pods starting"
    
    echo ""
    echo "Step 4: Health verification"
    echo "  ✓ Health checks passing"
    echo "  ✓ Readiness probes healthy"
    echo "  ✓ Service endpoints verified"
    
    echo ""
    echo "Step 5: Observability verification"
    echo "  ✓ Metrics flowing to Prometheus"
    echo "  ✓ Traces visible in Jaeger"
    echo "  ✓ Logs indexed in Elasticsearch"
    echo "  ✓ Dashboard updated in Grafana"
    
    success "Deployment simulation complete: $app_name:$image_tag"
}

# Run complete workshop
run_workshop() {
    log "Starting GitOps Platform Workshop"
    separator
    
    check_prerequisites || { log "Prerequisites not met, continuing anyway..."; }
    echo ""
    
    setup_gitops_repo
    echo ""
    
    deploy_monitoring
    echo ""
    
    setup_policy_enforcement
    echo ""
    
    setup_multi_tenancy
    echo ""
    
    check_gitops_health
    echo ""
    
    simulate_deployment "demo-app" "v1.0.0"
    echo ""
    
    success "GitOps Platform Workshop Complete!"
    separator
    echo ""
    echo "Platform Stack Summary:"
    echo "  • GitOps: Flux CD / ArgoCD"
    echo "  • IDP: Backstage + Crossplane"
    echo "  • Policy: OPA Gatekeeper"
    echo "  • Monitoring: Prometheus + Grafana"
    echo "  • Tenancy: Namespace isolation + RBAC"
    echo ""
    echo "Repository: $WORKSHOP_DIR"
    echo "Cluster: $CLUSTER_NAME"
}

# Main
case "${1:-workshop}" in
    prereqs) check_prerequisites ;;
    setup-repo) setup_gitops_repo ;;
    monitoring) deploy_monitoring ;;
    policies) setup_policy_enforcement ;;
    tenancy) setup_multi_tenancy ;;
    health) check_gitops_health ;;
    simulate) simulate_deployment "${2:-demo-app}" "${3:-v1.0.0}" ;;
    workshop) run_workshop ;;
    *)
        echo "Usage: $0 {prereqs|setup-repo|monitoring|policies|tenancy|health|simulate|workshop}"
        ;;
esac
```

---

## สรุป Part 34

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | เครื่องมือ | ขั้นตอน |
|--------|-----------|---------|
| Flux CD GitOps | Flux CLI, HelmRelease, Kustomization | 493 |
| ArgoCD Advanced | ApplicationSet, Multi-cluster, Projects | 494 |
| Platform Engineering | Crossplane, Backstage, IDP | 495 |
| Developer Portal | Backstage Catalog, Templates | 496 |
| GitOps Workshop | Complete Platform Stack | 497 |

**เทคโนโลยีที่ใช้:**
- Flux CD: GitOps reconciliation engine
- ArgoCD: Declarative CD with UI
- Crossplane: Infrastructure as Code via K8s
- Backstage: Internal Developer Portal
- OPA Gatekeeper: Policy enforcement
- Multi-tenancy: Namespace isolation + RBAC

ขั้นตอนต่อไป: Part 35 - Multi-cluster Kubernetes Management
