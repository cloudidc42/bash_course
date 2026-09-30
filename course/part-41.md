# Part 41: Enterprise Kubernetes Architecture

## Module 4: Professional Level - Enterprise Platform Engineering

---

## ขั้นตอนที่ 518: Multi-tenant SaaS Platform Architecture

Multi-tenant Platform คือสถาปัตยกรรมที่ช่วยให้ลูกค้าหลายรายใช้งาน Infrastructure ร่วมกันได้อย่างปลอดภัย

```bash
#!/bin/bash
# multitenant-platform.sh - Multi-tenant SaaS Platform Manager

set -euo pipefail

PLATFORM_NAMESPACE="${PLATFORM_NAMESPACE:-platform-system}"
TENANT_OPERATOR_NAMESPACE="${TENANT_OPERATOR_NAMESPACE:-tenant-operator}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Create tenant CRD
create_tenant_crd() {
    log "Creating Tenant Custom Resource Definition..."
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: tenants.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: Tenant
    plural: tenants
    singular: tenant
    shortNames:
    - tn
  scope: Cluster
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - id
            - name
            - tier
            properties:
              id:
                type: string
                pattern: '^[a-z0-9-]+$'
              name:
                type: string
              tier:
                type: string
                enum:
                - free
                - starter
                - professional
                - enterprise
              quotas:
                type: object
                properties:
                  cpu:
                    type: string
                  memory:
                    type: string
                  storage:
                    type: string
                  pods:
                    type: integer
                  namespaces:
                    type: integer
              features:
                type: object
                properties:
                  monitoring:
                    type: boolean
                    default: false
                  advancedNetworking:
                    type: boolean
                    default: false
                  customDomain:
                    type: boolean
                    default: false
                  sso:
                    type: boolean
                    default: false
                  audit:
                    type: boolean
                    default: false
              admin:
                type: object
                properties:
                  email:
                    type: string
                  team:
                    type: string
          status:
            type: object
            properties:
              phase:
                type: string
                enum:
                - Provisioning
                - Active
                - Suspended
                - Terminated
              namespaces:
                type: array
                items:
                  type: string
              createdAt:
                type: string
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: ID
      type: string
      jsonPath: .spec.id
    - name: Tier
      type: string
      jsonPath: .spec.tier
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
EOF
    
    success "Tenant CRD created"
}

# Tier configuration
get_tier_quotas() {
    local tier="${1:-free}"
    
    case "$tier" in
        free)
            echo '{"cpu":"500m","memory":"512Mi","storage":"5Gi","pods":10,"namespaces":1}'
            ;;
        starter)
            echo '{"cpu":"2","memory":"4Gi","storage":"20Gi","pods":30,"namespaces":3}'
            ;;
        professional)
            echo '{"cpu":"8","memory":"16Gi","storage":"100Gi","pods":100,"namespaces":5}'
            ;;
        enterprise)
            echo '{"cpu":"32","memory":"64Gi","storage":"1Ti","pods":500,"namespaces":20}'
            ;;
        *)
            echo '{"cpu":"500m","memory":"512Mi","storage":"5Gi","pods":10,"namespaces":1}'
            ;;
    esac
}

# Provision tenant
provision_tenant() {
    local tenant_id="${1:-}"
    local tenant_name="${2:-}"
    local tier="${3:-free}"
    local admin_email="${4:-}"
    
    if [[ -z "$tenant_id" || -z "$tenant_name" || -z "$admin_email" ]]; then
        error "Usage: provision_tenant <id> <name> <tier> <admin-email>"
        return 1
    fi
    
    log "Provisioning tenant: $tenant_id ($tier)"
    
    local quotas
    quotas=$(get_tier_quotas "$tier")
    
    # Create Tenant CR
    cat <<EOF | kubectl apply -f -
apiVersion: platform.example.com/v1
kind: Tenant
metadata:
  name: ${tenant_id}
spec:
  id: ${tenant_id}
  name: "${tenant_name}"
  tier: ${tier}
  quotas:
    cpu: $(echo "$quotas" | jq -r '.cpu')
    memory: $(echo "$quotas" | jq -r '.memory')
    storage: $(echo "$quotas" | jq -r '.storage')
    pods: $(echo "$quotas" | jq -r '.pods')
    namespaces: $(echo "$quotas" | jq -r '.namespaces')
  features:
    monitoring: $([ "$tier" != "free" ] && echo "true" || echo "false")
    advancedNetworking: $([ "$tier" == "enterprise" ] && echo "true" || echo "false")
    customDomain: $([ "$tier" != "free" ] && echo "true" || echo "false")
    sso: $([ "$tier" == "enterprise" ] || [ "$tier" == "professional" ] && echo "true" || echo "false")
    audit: $([ "$tier" == "enterprise" ] && echo "true" || echo "false")
  admin:
    email: ${admin_email}
EOF
    
    # Create tenant namespace
    local ns="tenant-${tenant_id}"
    kubectl create namespace "$ns" --dry-run=client -o yaml | kubectl apply -f -
    
    # Label namespace
    kubectl label namespace "$ns" \
        tenant-id="$tenant_id" \
        tenant-tier="$tier" \
        managed-by=platform \
        --overwrite
    
    # Apply ResourceQuota
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: ${ns}
spec:
  hard:
    requests.cpu: $(echo "$quotas" | jq -r '.cpu')
    requests.memory: $(echo "$quotas" | jq -r '.memory')
    limits.cpu: $(echo "$quotas" | jq -r '.cpu | tonumber * 2 | tostring + "m"' 2>/dev/null || echo "$(echo "$quotas" | jq -r '.cpu')")
    limits.memory: $(echo "$quotas" | jq -r '.memory')
    pods: $(echo "$quotas" | jq -r '.pods')
    services: 20
    secrets: 50
    configmaps: 50
    persistentvolumeclaims: 10
EOF

    # Apply LimitRange
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limitrange
  namespace: ${ns}
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
      cpu: "4"
      memory: 8Gi
    min:
      cpu: 50m
      memory: 64Mi
  - type: PersistentVolumeClaim
    max:
      storage: $(echo "$quotas" | jq -r '.storage')
    min:
      storage: 1Gi
EOF

    # Create RBAC for tenant admin
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-admin
  namespace: ${ns}
  annotations:
    platform.example.com/tenant-id: ${tenant_id}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-admin-role
  namespace: ${ns}
rules:
- apiGroups: ["", "apps", "batch", "autoscaling"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses", "networkpolicies"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["resourcequotas", "limitranges"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-admin-binding
  namespace: ${ns}
subjects:
- kind: ServiceAccount
  name: tenant-admin
  namespace: ${ns}
roleRef:
  kind: Role
  name: tenant-admin-role
  apiGroup: rbac.authorization.k8s.io
EOF

    # Apply network isolation
    cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: ${ns}
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant-id: ${tenant_id}
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tenant-id: ${tenant_id}
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: platform-system
EOF

    # Generate tenant kubeconfig
    generate_tenant_kubeconfig "$tenant_id" "$ns"
    
    success "Tenant provisioned: $tenant_id (namespace: $ns)"
    
    # Update tenant status
    kubectl patch tenant "$tenant_id" \
        --type merge \
        --subresource=status \
        -p "{\"status\":{\"phase\":\"Active\",\"namespaces\":[\"$ns\"],\"createdAt\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}}" \
        2>/dev/null || true
}

# Generate tenant-specific kubeconfig
generate_tenant_kubeconfig() {
    local tenant_id="${1:-}"
    local namespace="${2:-}"
    
    if [[ -z "$tenant_id" || -z "$namespace" ]]; then
        return 1
    fi
    
    log "Generating kubeconfig for tenant: $tenant_id"
    
    # Create service account token
    kubectl create token tenant-admin \
        -n "$namespace" \
        --duration=8760h 2>/dev/null > "/tmp/tenant-${tenant_id}-token" || true
    
    local server
    server=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}' 2>/dev/null)
    
    local ca_data
    ca_data=$(kubectl config view --minify --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' 2>/dev/null)
    
    local token
    token=$(cat "/tmp/tenant-${tenant_id}-token" 2>/dev/null || echo "")
    
    cat <<EOF > "/tmp/kubeconfig-${tenant_id}.yaml"
apiVersion: v1
kind: Config
clusters:
- name: ${tenant_id}-cluster
  cluster:
    server: ${server}
    certificate-authority-data: ${ca_data}
contexts:
- name: ${tenant_id}
  context:
    cluster: ${tenant_id}-cluster
    user: tenant-admin-${tenant_id}
    namespace: tenant-${tenant_id}
current-context: ${tenant_id}
users:
- name: tenant-admin-${tenant_id}
  user:
    token: ${token}
EOF
    
    log "Tenant kubeconfig: /tmp/kubeconfig-${tenant_id}.yaml"
}

# Suspend tenant
suspend_tenant() {
    local tenant_id="${1:-}"
    
    if [[ -z "$tenant_id" ]]; then
        error "Tenant ID required"
        return 1
    fi
    
    log "Suspending tenant: $tenant_id"
    
    # Scale down all deployments
    kubectl scale deployments --all \
        -n "tenant-${tenant_id}" \
        --replicas=0 2>/dev/null || true
    
    # Update status
    kubectl patch tenant "$tenant_id" \
        --type merge \
        --subresource=status \
        -p '{"status":{"phase":"Suspended"}}' 2>/dev/null || true
    
    success "Tenant suspended: $tenant_id"
}

# Terminate tenant
terminate_tenant() {
    local tenant_id="${1:-}"
    local confirm="${2:-false}"
    
    if [[ "$confirm" != "true" ]]; then
        error "Must confirm tenant termination with confirm=true"
        return 1
    fi
    
    log "Terminating tenant: $tenant_id"
    
    # Delete namespace
    kubectl delete namespace "tenant-${tenant_id}" --wait=false 2>/dev/null || true
    
    # Delete tenant CR
    kubectl delete tenant "$tenant_id" 2>/dev/null || true
    
    success "Tenant termination initiated: $tenant_id"
}

# List all tenants
list_tenants() {
    log "Listing all tenants..."
    
    kubectl get tenants --all-namespaces 2>/dev/null || \
    kubectl get namespaces -l managed-by=platform -o \
        jsonpath='{range .items[*]}{.metadata.labels.tenant-id}{"\t"}{.metadata.labels.tenant-tier}{"\t"}{.metadata.name}{"\n"}{end}' | \
    column -t -N "TENANT_ID,TIER,NAMESPACE"
}

# Tenant usage metrics
tenant_metrics() {
    local tenant_id="${1:-}"
    
    if [[ -z "$tenant_id" ]]; then
        # All tenants
        for ns in $(kubectl get namespaces -l managed-by=platform -o name | sed 's/namespace\///'); do
            local tid
            tid=$(kubectl get namespace "$ns" -o jsonpath='{.metadata.labels.tenant-id}' 2>/dev/null)
            echo "=== Tenant: ${tid:-$ns} ==="
            kubectl top pods -n "$ns" 2>/dev/null || echo "No metrics"
            echo ""
        done
    else
        local ns="tenant-${tenant_id}"
        echo "=== Resource Usage for Tenant: $tenant_id ==="
        kubectl describe resourcequota tenant-quota -n "$ns" 2>/dev/null
        echo ""
        kubectl top pods -n "$ns" 2>/dev/null || echo "No metrics available"
    fi
}

# Main
case "${1:-help}" in
    create-crd) create_tenant_crd ;;
    provision) provision_tenant "${2:-}" "${3:-}" "${4:-free}" "${5:-}" ;;
    suspend) suspend_tenant "${2:-}" ;;
    terminate) terminate_tenant "${2:-}" "${3:-false}" ;;
    list) list_tenants ;;
    metrics) tenant_metrics "${2:-}" ;;
    kubeconfig) generate_tenant_kubeconfig "${2:-}" "tenant-${2:-}" ;;
    *)
        echo "Usage: $0 {create-crd|provision|suspend|terminate|list|metrics|kubeconfig}"
        ;;
esac
```

---

## ขั้นตอนที่ 519: Enterprise RBAC และ Identity Management

```bash
#!/bin/bash
# enterprise-rbac.sh - Enterprise RBAC and Identity Management

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Create enterprise role hierarchy
create_enterprise_roles() {
    log "Creating enterprise RBAC role hierarchy..."
    
    # Platform Admin (full access)
    cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-admin
  labels:
    rbac.platform.io/role-level: admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
- nonResourceURLs: ["*"]
  verbs: ["*"]
---
# Platform Developer (can deploy, but not admin cluster)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-developer
  labels:
    rbac.platform.io/role-level: developer
rules:
- apiGroups: ["apps", "batch", "autoscaling"]
  resources: ["deployments", "statefulsets", "daemonsets", "jobs", "cronjobs", "horizontalpodautoscalers"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
---
# Platform Viewer (read-only access)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-viewer
  labels:
    rbac.platform.io/role-level: viewer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: []
---
# SRE Role (can do operations, view secrets, manage incidents)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sre-operator
  labels:
    rbac.platform.io/role-level: sre
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["update", "patch"]
  # Can rollout restart
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["delete"]  # Can delete pods for restart
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # Can view secrets for debugging
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "list", "watch"]
---
# Security Admin (can manage RBAC and security policies)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: security-admin
  labels:
    rbac.platform.io/role-level: security
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  verbs: ["*"]
- apiGroups: ["policy"]
  resources: ["podsecuritypolicies"]
  verbs: ["*"]
- apiGroups: ["networking.k8s.io"]
  resources: ["networkpolicies"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["serviceaccounts"]
  verbs: ["*"]
EOF
    
    success "Enterprise roles created"
}

# OIDC integration for SSO
setup_oidc_integration() {
    local issuer_url="${1:-https://accounts.google.com}"
    local client_id="${2:-my-client-id}"
    local admin_group="${3:-platform-admins}"
    local developer_group="${4:-developers}"
    
    log "Setting up OIDC integration with: $issuer_url"
    
    # Create ClusterRoleBindings for groups
    cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-admins-oidc
subjects:
- kind: Group
  name: ${admin_group}
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: platform-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developers-oidc
subjects:
- kind: Group
  name: ${developer_group}
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: platform-developer
  apiGroup: rbac.authorization.k8s.io
EOF

    # Generate kube-apiserver OIDC flags
    cat <<EOF

# Add these flags to kube-apiserver:
--oidc-issuer-url=${issuer_url}
--oidc-client-id=${client_id}
--oidc-username-claim=email
--oidc-groups-claim=groups
--oidc-username-prefix=oidc:
--oidc-groups-prefix=oidc:

# For EKS (in aws-auth ConfigMap or access entries):
# Map IAM roles to Kubernetes groups

# For GKE:
# Use Google Groups with Workload Identity Federation
EOF
    
    # Setup Dex as OIDC provider (if needed)
    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dex
  namespace: auth-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dex
  template:
    metadata:
      labels:
        app: dex
    spec:
      containers:
      - name: dex
        image: dexidp/dex:v2.38.0
        args:
        - dex
        - serve
        - /etc/dex/cfg/config.yaml
        ports:
        - containerPort: 5556
          name: https
        volumeMounts:
        - name: config
          mountPath: /etc/dex/cfg
      volumes:
      - name: config
        configMap:
          name: dex-config
EOF

    success "OIDC integration configured"
}

# Just-in-Time (JIT) access provisioning
setup_jit_access() {
    local namespace="${1:-default}"
    
    log "Setting up JIT access framework..."
    
    # CRD for access requests
    cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: accessrequests.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: AccessRequest
    plural: accessrequests
    singular: accessrequest
    shortNames:
    - ar
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - requester
            - role
            - justification
            properties:
              requester:
                type: string
              role:
                type: string
                enum:
                - sre-operator
                - security-admin
                - platform-admin
              namespace:
                type: string
              justification:
                type: string
                minLength: 20
              duration:
                type: string
                default: "4h"
              approvers:
                type: array
                items:
                  type: string
          status:
            type: object
            properties:
              phase:
                type: string
                enum:
                - Pending
                - Approved
                - Denied
                - Expired
                - Active
              approvedBy:
                type: string
              approvedAt:
                type: string
              expiresAt:
                type: string
    subresources:
      status: {}
EOF

    # JIT controller script
    cat <<'JITCONTROLLER' > /tmp/jit-controller.sh
#!/bin/bash
# JIT Access Controller

watch_access_requests() {
    kubectl get accessrequests --all-namespaces --watch -o json | \
    while IFS= read -r line; do
        local name namespace requester role phase
        name=$(echo "$line" | jq -r '.metadata.name // empty') || continue
        namespace=$(echo "$line" | jq -r '.metadata.namespace // "default"') || continue
        requester=$(echo "$line" | jq -r '.spec.requester // empty') || continue
        role=$(echo "$line" | jq -r '.spec.role // empty') || continue
        phase=$(echo "$line" | jq -r '.status.phase // "Pending"') || continue
        
        [[ -z "$name" || "$phase" != "Approved" ]] && continue
        
        echo "[JIT] Granting $role to $requester in $namespace"
        
        # Create temporary RoleBinding
        local duration
        duration=$(kubectl get accessrequest "$name" -n "$namespace" \
            -o jsonpath='{.spec.duration}' 2>/dev/null || echo "4h")
        
        kubectl create rolebinding "jit-${name}" \
            --clusterrole="$role" \
            --user="$requester" \
            -n "$namespace" \
            --dry-run=client -o yaml | kubectl apply -f -
        
        # Schedule cleanup
        local duration_seconds
        case "$duration" in
            *h) duration_seconds=$(( ${duration%h} * 3600 )) ;;
            *m) duration_seconds=$(( ${duration%m} * 60 )) ;;
            *) duration_seconds=14400 ;;  # 4h default
        esac
        
        (
            sleep "$duration_seconds"
            kubectl delete rolebinding "jit-${name}" -n "$namespace" 2>/dev/null
            kubectl patch accessrequest "$name" -n "$namespace" \
                --type merge --subresource=status \
                -p '{"status":{"phase":"Expired"}}' 2>/dev/null
            echo "[JIT] Access expired: $name"
        ) &
        
        # Update status
        kubectl patch accessrequest "$name" -n "$namespace" \
            --type merge --subresource=status \
            -p "{\"status\":{\"phase\":\"Active\",\"expiresAt\":\"$(date -d \"+${duration}\" +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date +%Y-%m-%dT%H:%M:%SZ)\"}}" 2>/dev/null
    done
}

watch_access_requests
JITCONTROLLER
    
    chmod +x /tmp/jit-controller.sh
    
    success "JIT access framework configured"
    log "Request access with: kubectl create -f access-request.yaml"
}

# Audit logging configuration
setup_audit_logging() {
    local log_backend="${1:-elasticsearch}"
    local es_url="${2:-http://elasticsearch:9200}"
    
    log "Setting up audit logging..."
    
    # Create audit policy
    cat <<'EOF' > /tmp/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
- RequestReceived
rules:
# Critical security events - log at RequestResponse level
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]
  - group: ""
    resources: ["secrets"]

# Authentication failures
- level: Metadata
  verbs: ["*"]
  namespaces: ["kube-system"]
  resources:
  - group: ""
    resources: ["secrets", "serviceaccounts/token"]

# Privileged pod creation
- level: RequestResponse
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods"]
  omitStages:
  - RequestReceived

# All write operations
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["*"]
  - group: "apps"
    resources: ["*"]

# Read operations on sensitive resources
- level: Metadata
  verbs: ["get", "list", "watch"]
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]

# Default: metadata only
- level: Metadata
  omitStages:
  - RequestReceived
EOF
    
    log "Audit policy created: /tmp/audit-policy.yaml"
    
    # Audit log forwarder
    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: audit-log-forwarder
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: audit-log-forwarder
  template:
    metadata:
      labels:
        app: audit-log-forwarder
    spec:
      tolerations:
      - operator: Exists
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.10.0
        volumeMounts:
        - name: audit-logs
          mountPath: /var/log/audit
        - name: filebeat-config
          mountPath: /usr/share/filebeat/filebeat.yml
          subPath: filebeat.yml
      volumes:
      - name: audit-logs
        hostPath:
          path: /var/log/kubernetes/audit
      - name: filebeat-config
        configMap:
          name: audit-filebeat-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: audit-filebeat-config
  namespace: kube-system
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: log
      paths:
      - /var/log/audit/*.log
      json.keys_under_root: true
      json.add_error_key: true
    
    output.elasticsearch:
      hosts: ["${es_url}"]
      index: "k8s-audit-%{+yyyy.MM.dd}"
EOF
    
    success "Audit logging configured"
}

# Main
case "${1:-help}" in
    create-roles) create_enterprise_roles ;;
    oidc) setup_oidc_integration "${2:-}" "${3:-}" "${4:-platform-admins}" "${5:-developers}" ;;
    jit) setup_jit_access "${2:-default}" ;;
    audit) setup_audit_logging "${2:-elasticsearch}" "${3:-http://elasticsearch:9200}" ;;
    *)
        echo "Usage: $0 {create-roles|oidc|jit|audit}"
        ;;
esac
```

---

## ขั้นตอนที่ 520: Enterprise Kubernetes Hardening

```bash
#!/bin/bash
# enterprise-k8s-hardening.sh - Enterprise Kubernetes Hardening

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# CIS Kubernetes Benchmark hardening
apply_cis_hardening() {
    log "Applying CIS Kubernetes Benchmark hardening..."
    
    # 1.1 API Server configuration
    log "Checking API Server hardening..."
    
    # These require API server restart - check current config
    local api_server_config
    api_server_config=$(kubectl get pod kube-apiserver -n kube-system \
        -o jsonpath='{.spec.containers[0].command}' 2>/dev/null | \
        jq -r '.[]' 2>/dev/null || echo "Not accessible")
    
    echo "Current API Server flags (check for hardening):"
    echo "$api_server_config" | grep -E "(anonymous|auth|audit|tls|etcd)" | head -20
    
    # 2. etcd configuration - encrypt secrets
    cat <<'EOF' > /tmp/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: BASE64_ENCODED_32_BYTE_KEY_HERE
  - identity: {}
EOF
    
    log "Encryption config template: /tmp/encryption-config.yaml"
    
    # 3. RBAC hardening
    log "Checking for excessive permissions..."
    
    # Find cluster-admin bindings (should be minimal)
    echo ""
    echo "=== ClusterRoleBindings for cluster-admin ==="
    kubectl get clusterrolebindings -o json | \
        jq -r '.items[] | select(.roleRef.name == "cluster-admin") | "\(.metadata.name): \(.subjects[].name)"' \
        2>/dev/null | head -10
    
    # Find service accounts with cluster-admin
    echo ""
    echo "=== Service Accounts with cluster-admin (HIGH RISK) ==="
    kubectl get clusterrolebindings -o json | \
        jq -r '.items[] | select(.roleRef.name == "cluster-admin") | select(.subjects[] | .kind == "ServiceAccount") | "\(.metadata.name): \(.subjects[] | select(.kind == "ServiceAccount") | "\(.namespace)/\(.name)")"' \
        2>/dev/null
    
    # 4. Network policies
    log "Ensuring network policies in all namespaces..."
    
    local ns_without_policies
    ns_without_policies=$(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}' | \
        tr ' ' '\n' | while read -r ns; do
            local count
            count=$(kubectl get networkpolicies -n "$ns" --no-headers 2>/dev/null | wc -l)
            [[ $count -eq 0 ]] && echo "$ns"
        done)
    
    if [[ -n "$ns_without_policies" ]]; then
        echo "Namespaces without network policies:"
        echo "$ns_without_policies"
    fi
    
    success "CIS hardening check complete"
}

# Kyverno policy engine setup
setup_kyverno() {
    local namespace="${1:-kyverno}"
    
    log "Installing Kyverno policy engine..."
    
    helm repo add kyverno https://kyverno.github.io/kyverno/
    helm repo update
    
    helm upgrade --install kyverno kyverno/kyverno \
        -n "$namespace" \
        --create-namespace \
        --set replicaCount=3 \
        --set resources.requests.memory=128Mi \
        --wait
    
    # Install Kyverno policies
    kubectl apply -f https://github.com/kyverno/policies/raw/main/pod-security/baseline/disallow-privileged-containers/disallow-privileged-containers.yaml 2>/dev/null || true
    kubectl apply -f https://github.com/kyverno/policies/raw/main/pod-security/baseline/require-non-root-groups/require-non-root-groups.yaml 2>/dev/null || true
    kubectl apply -f https://github.com/kyverno/policies/raw/main/pod-security/restricted/require-run-as-nonroot/require-run-as-nonroot.yaml 2>/dev/null || true
    
    success "Kyverno installed"
}

# Create Kyverno custom policies
create_kyverno_policies() {
    log "Creating custom Kyverno policies..."
    
    # Policy: Require specific labels
    cat <<'EOF' | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-deployment-labels
  annotations:
    policies.kyverno.io/title: Require Labels
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/description: >-
      Deployments must have required labels for proper management.
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: require-team-label
    match:
      any:
      - resources:
          kinds:
          - Deployment
          namespaces:
          - production
          - staging
    validate:
      message: "The label 'app', 'version', and 'team' are required."
      pattern:
        metadata:
          labels:
            app: "?*"
            version: "?*"
            team: "?*"
---
# Policy: Disallow latest image tag
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations:
    policies.kyverno.io/title: Disallow Latest Tag
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: require-image-tag
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - production
    validate:
      message: "Using a mutable image tag e.g. 'latest' is not allowed in production."
      foreach:
      - list: "request.object.spec.containers"
        deny:
          conditions:
            any:
            - key: "{{element.image}}"
              operator: Equals
              value: "*:latest"
            - key: "{{element.image}}"
              operator: NotContains
              value: ":"
---
# Policy: Require resource limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
  annotations:
    policies.kyverno.io/title: Require Resource Limits
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: require-limits
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - production
    validate:
      message: "Resource limits are required for all containers."
      pattern:
        spec:
          containers:
          - name: "*"
            resources:
              limits:
                memory: "?*"
                cpu: "?*"
---
# Policy: Auto-add security context
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-security-context
  annotations:
    policies.kyverno.io/title: Add Security Context
    policies.kyverno.io/category: Security
spec:
  rules:
  - name: add-security-context
    match:
      any:
      - resources:
          kinds:
          - Pod
    mutate:
      patchStrategicMerge:
        spec:
          +(securityContext):
            +(runAsNonRoot): true
            +(runAsUser): 1000
            +(fsGroup): 3000
          containers:
          - (name): "*"
            +(securityContext):
              +(allowPrivilegeEscalation): false
              +(readOnlyRootFilesystem): true
              +(capabilities):
                +(drop):
                - ALL
EOF
    
    success "Kyverno policies created"
}

# Kubernetes audit report
generate_security_report() {
    local output_file="${1:-security-report-$(date +%Y%m%d).txt}"
    
    log "Generating Kubernetes security report..."
    
    {
        echo "Kubernetes Security Report"
        echo "Generated: $(date)"
        echo "================================"
        echo ""
        
        echo "=== Cluster Info ==="
        kubectl cluster-info 2>/dev/null | head -5
        echo ""
        
        echo "=== Node Security ==="
        kubectl get nodes -o wide
        echo ""
        
        echo "=== Namespace Count ==="
        kubectl get namespaces --no-headers | wc -l
        echo ""
        
        echo "=== Privileged Pods ==="
        kubectl get pods --all-namespaces -o json | \
            jq -r '.items[] | select(.spec.containers[] | .securityContext.privileged == true) | "\(.metadata.namespace)/\(.metadata.name)"' \
            2>/dev/null || echo "None"
        echo ""
        
        echo "=== Root Containers ==="
        kubectl get pods --all-namespaces -o json | \
            jq -r '.items[] | select(.spec.securityContext.runAsUser == 0) | "\(.metadata.namespace)/\(.metadata.name)"' \
            2>/dev/null || echo "None"
        echo ""
        
        echo "=== Service Accounts with Tokens Auto-mounted ==="
        kubectl get pods --all-namespaces -o json | \
            jq -r '.items[] | select(.spec.automountServiceAccountToken != false) | "\(.metadata.namespace)/\(.metadata.name)"' \
            2>/dev/null | wc -l
        echo "pods with automounted tokens"
        echo ""
        
        echo "=== ClusterRoleBindings ==="
        kubectl get clusterrolebindings --no-headers | wc -l
        echo "total ClusterRoleBindings"
        echo ""
        
        echo "=== Network Policies ==="
        kubectl get networkpolicies --all-namespaces --no-headers | wc -l
        echo "total NetworkPolicies"
        echo ""
        
        echo "=== PSS Enforcement ==="
        kubectl get namespaces -o json | \
            jq -r '.items[] | select(.metadata.labels | has("pod-security.kubernetes.io/enforce")) | "\(.metadata.name): \(.metadata.labels["pod-security.kubernetes.io/enforce"])"' \
            2>/dev/null || echo "No PSS labels found"
        echo ""
        
        echo "=== Kyverno Policy Results ==="
        kubectl get policyreports --all-namespaces 2>/dev/null | head -20 || echo "Kyverno not installed"
        
    } > "$output_file"
    
    success "Security report generated: $output_file"
}

# Main
case "${1:-help}" in
    cis-check) apply_cis_hardening ;;
    kyverno) setup_kyverno "${2:-kyverno}" ;;
    policies) create_kyverno_policies ;;
    report) generate_security_report "${2:-}" ;;
    *)
        echo "Usage: $0 {cis-check|kyverno|policies|report}"
        ;;
esac
```

---

## สรุป Part 41

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | เครื่องมือ | ขั้นตอน |
|--------|-----------|---------|
| Multi-tenant SaaS Architecture | Tenant CRD, Namespace Isolation, Quotas | 518 |
| Enterprise RBAC | Role Hierarchy, OIDC, JIT Access | 519 |
| Enterprise K8s Hardening | CIS Benchmark, Kyverno, Security Report | 520 |

**เทคโนโลยีที่ใช้:**
- Custom Tenant Operator: Automate tenant lifecycle
- Dex: OIDC identity provider
- Kyverno: Policy engine (validate, mutate, generate)
- JIT Access: Just-in-Time privilege escalation
- Audit Logging: K8s audit + Elasticsearch

ขั้นตอนต่อไป: Part 42 - Advanced Data Platform Engineering
