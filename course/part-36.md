# Part 36: Advanced Kubernetes Patterns และ Operators

## Module 3: Advanced Level - Kubernetes Operators และ Custom Controllers

---

## ขั้นตอนที่ 503: Kubernetes Operators Framework

Kubernetes Operator เป็น Pattern สำหรับการ Automate การจัดการ Stateful Applications

```bash
#!/bin/bash
# operator-manager.sh - Kubernetes Operator Development Manager

set -euo pipefail

OPERATOR_DIR="${OPERATOR_DIR:-./my-operator}"
OPERATOR_NAME="${OPERATOR_NAME:-my-operator}"
OPERATOR_VERSION="${OPERATOR_VERSION:-v0.1.0}"
OPERATOR_IMAGE="${OPERATOR_IMAGE:-my-registry/my-operator}"
OPERATOR_NAMESPACE="${OPERATOR_NAMESPACE:-operator-system}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Initialize Operator SDK project
init_operator_project() {
    local domain="${1:-example.com}"
    local project_name="${2:-$OPERATOR_NAME}"
    local language="${3:-go}"
    
    log "Initializing Operator SDK project: $project_name"
    
    if ! command -v operator-sdk &>/dev/null; then
        log "Installing Operator SDK..."
        local os_arch
        os_arch=$(uname -m | sed 's/x86_64/amd64/g')
        curl -LO "https://github.com/operator-framework/operator-sdk/releases/latest/download/operator-sdk_linux_${os_arch}"
        chmod +x "operator-sdk_linux_${os_arch}"
        mv "operator-sdk_linux_${os_arch}" /usr/local/bin/operator-sdk
    fi
    
    mkdir -p "$OPERATOR_DIR"
    cd "$OPERATOR_DIR"
    
    operator-sdk init \
        --domain "$domain" \
        --repo "github.com/myorg/$project_name" \
        --project-name "$project_name"
    
    success "Operator project initialized: $project_name"
    cd - > /dev/null
}

# Create new API and Controller
create_api_controller() {
    local group="${1:-myapp}"
    local version="${2:-v1alpha1}"
    local kind="${3:-MyApp}"
    
    log "Creating API and Controller: $group/$version/$kind"
    
    cd "$OPERATOR_DIR"
    
    operator-sdk create api \
        --group "$group" \
        --version "$version" \
        --kind "$kind" \
        --resource \
        --controller
    
    success "API and Controller created: $kind"
    cd - > /dev/null
}

# Generate operator CRD and manifests
generate_operator_manifests() {
    log "Generating CRD and RBAC manifests..."
    
    cd "$OPERATOR_DIR"
    
    make generate
    make manifests
    
    success "Manifests generated"
    cd - > /dev/null
}

# Build and push operator image
build_push_operator() {
    local image="${1:-$OPERATOR_IMAGE}"
    local tag="${2:-$OPERATOR_VERSION}"
    
    log "Building and pushing operator image: $image:$tag"
    
    cd "$OPERATOR_DIR"
    
    make docker-build docker-push IMG="${image}:${tag}"
    
    success "Operator image built and pushed: $image:$tag"
    cd - > /dev/null
}

# Deploy operator to cluster
deploy_operator() {
    local image="${1:-$OPERATOR_IMAGE}"
    local tag="${2:-$OPERATOR_VERSION}"
    local namespace="${3:-$OPERATOR_NAMESPACE}"
    
    log "Deploying operator to cluster..."
    
    cd "$OPERATOR_DIR"
    
    make deploy IMG="${image}:${tag}" NAMESPACE="$namespace"
    
    # Wait for operator to be ready
    kubectl wait deployment "${OPERATOR_NAME}-controller-manager" \
        -n "$namespace" \
        --for=condition=Available \
        --timeout=120s
    
    success "Operator deployed"
    cd - > /dev/null
}

# Create sample CRD YAML for a database operator
create_database_operator_crd() {
    local output_dir="${1:-./operator-crds}"
    mkdir -p "$output_dir"
    
    log "Creating Database Operator CRD..."
    
    # Custom Resource Definition
    cat <<'EOF' > "${output_dir}/database-crd.yaml"
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.db.example.com
spec:
  group: db.example.com
  names:
    kind: Database
    listKind: DatabaseList
    plural: databases
    singular: database
    shortNames:
    - db
  scope: Namespaced
  versions:
  - name: v1alpha1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - engine
            - version
            - storage
            properties:
              engine:
                type: string
                enum:
                - postgresql
                - mysql
                - mongodb
              version:
                type: string
              storage:
                type: object
                required:
                - size
                properties:
                  size:
                    type: string
                    pattern: '^[0-9]+(Gi|Mi)$'
                  storageClass:
                    type: string
                    default: standard
              replicas:
                type: integer
                minimum: 1
                maximum: 5
                default: 1
              resources:
                type: object
                properties:
                  requests:
                    type: object
                    properties:
                      cpu:
                        type: string
                      memory:
                        type: string
                  limits:
                    type: object
                    properties:
                      cpu:
                        type: string
                      memory:
                        type: string
              backup:
                type: object
                properties:
                  enabled:
                    type: boolean
                    default: false
                  schedule:
                    type: string
                  retention:
                    type: integer
                    default: 7
          status:
            type: object
            properties:
              phase:
                type: string
                enum:
                - Pending
                - Provisioning
                - Running
                - Updating
                - Failed
                - Deleting
              readyReplicas:
                type: integer
              connectionString:
                type: string
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type:
                      type: string
                    status:
                      type: string
                    lastTransitionTime:
                      type: string
                    reason:
                      type: string
                    message:
                      type: string
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: Engine
      type: string
      jsonPath: .spec.engine
    - name: Version
      type: string
      jsonPath: .spec.version
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Ready
      type: integer
      jsonPath: .status.readyReplicas
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
EOF

    # Sample Database CR
    cat <<'EOF' > "${output_dir}/sample-database.yaml"
apiVersion: db.example.com/v1alpha1
kind: Database
metadata:
  name: my-postgresql
  namespace: default
spec:
  engine: postgresql
  version: "14.9"
  storage:
    size: 10Gi
    storageClass: gp2
  replicas: 3
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: "2"
      memory: 4Gi
  backup:
    enabled: true
    schedule: "0 2 * * *"
    retention: 7
EOF

    success "Database Operator CRD created in: $output_dir"
}

# Create Bash-based Controller logic
create_bash_controller() {
    local output_dir="${1:-./bash-operator}"
    mkdir -p "$output_dir"
    
    log "Creating Bash-based operator controller..."
    
    cat <<'CONTROLLER' > "${output_dir}/controller.sh"
#!/bin/bash
# bash-operator/controller.sh - Bash Kubernetes Controller

set -euo pipefail

RESOURCE_GROUP="${RESOURCE_GROUP:-db.example.com}"
RESOURCE_VERSION="${RESOURCE_VERSION:-v1alpha1}"
RESOURCE_KIND="${RESOURCE_KIND:-Database}"
NAMESPACE="${WATCH_NAMESPACE:-}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [CONTROLLER] $*"; }
error() { echo "[ERROR] $*" >&2; }

# Reconcile a Database resource
reconcile_database() {
    local name="${1:-}"
    local namespace="${2:-default}"
    
    log "Reconciling Database: $namespace/$name"
    
    # Get current resource state
    local resource
    resource=$(kubectl get database "$name" -n "$namespace" -o json 2>/dev/null) || {
        log "Resource not found: $namespace/$name - may have been deleted"
        return 0
    }
    
    local engine
    engine=$(echo "$resource" | jq -r '.spec.engine')
    local version
    version=$(echo "$resource" | jq -r '.spec.version')
    local storage
    storage=$(echo "$resource" | jq -r '.spec.storage.size')
    local replicas
    replicas=$(echo "$resource" | jq -r '.spec.replicas // 1')
    local phase
    phase=$(echo "$resource" | jq -r '.status.phase // "Pending"')
    
    log "Database: engine=$engine version=$version storage=$storage replicas=$replicas phase=$phase"
    
    # Update status to Provisioning
    if [[ "$phase" == "Pending" ]]; then
        update_status "$name" "$namespace" "Provisioning" 0 ""
    fi
    
    # Create StatefulSet for database
    ensure_statefulset "$name" "$namespace" "$engine" "$version" "$storage" "$replicas"
    
    # Create Service
    ensure_service "$name" "$namespace" "$engine"
    
    # Check if ready
    local ready_replicas
    ready_replicas=$(kubectl get statefulset "${name}" -n "$namespace" \
        -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo "0")
    
    if [[ "$ready_replicas" == "$replicas" && "$ready_replicas" -gt 0 ]]; then
        local connection_string
        connection_string="${engine}://${name}.${namespace}.svc.cluster.local:5432/defaultdb"
        update_status "$name" "$namespace" "Running" "$ready_replicas" "$connection_string"
    fi
}

# Ensure StatefulSet exists and is up-to-date
ensure_statefulset() {
    local name="${1:-}"
    local namespace="${2:-}"
    local engine="${3:-postgresql}"
    local version="${4:-14}"
    local storage="${5:-10Gi}"
    local replicas="${6:-1}"
    
    local image
    case "$engine" in
        postgresql) image="postgres:${version}-alpine" ;;
        mysql) image="mysql:${version}" ;;
        mongodb) image="mongo:${version}" ;;
        *) image="${engine}:${version}" ;;
    esac
    
    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ${name}
  namespace: ${namespace}
  labels:
    app: ${name}
    managed-by: db-operator
    db-engine: ${engine}
spec:
  serviceName: ${name}
  replicas: ${replicas}
  selector:
    matchLabels:
      app: ${name}
  template:
    metadata:
      labels:
        app: ${name}
        db-engine: ${engine}
    spec:
      containers:
      - name: ${engine}
        image: ${image}
        ports:
        - containerPort: 5432
          name: db
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: ${name}-credentials
              key: password
              optional: true
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: ${storage}
EOF
    
    log "StatefulSet ensured: $name"
}

# Ensure Service exists
ensure_service() {
    local name="${1:-}"
    local namespace="${2:-}"
    local engine="${3:-postgresql}"
    
    local port
    case "$engine" in
        postgresql) port=5432 ;;
        mysql) port=3306 ;;
        mongodb) port=27017 ;;
        *) port=5432 ;;
    esac
    
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${name}
  namespace: ${namespace}
  labels:
    app: ${name}
    managed-by: db-operator
spec:
  selector:
    app: ${name}
  ports:
  - port: ${port}
    targetPort: db
    name: db
  clusterIP: None
EOF
    
    log "Service ensured: $name"
}

# Update resource status
update_status() {
    local name="${1:-}"
    local namespace="${2:-}"
    local phase="${3:-}"
    local ready="${4:-0}"
    local connection="${5:-}"
    
    kubectl patch database "$name" -n "$namespace" \
        --type merge \
        --subresource=status \
        -p "{\"status\":{\"phase\":\"$phase\",\"readyReplicas\":$ready,\"connectionString\":\"$connection\"}}" \
        2>/dev/null || true
}

# Watch and reconcile
watch_resources() {
    log "Starting watch loop for $RESOURCE_KIND resources..."
    
    local watch_args=("get" "$RESOURCE_KIND" "--watch" "-o" "json")
    if [[ -n "$NAMESPACE" ]]; then
        watch_args+=("-n" "$NAMESPACE")
    else
        watch_args+=("--all-namespaces")
    fi
    
    kubectl "${watch_args[@]}" 2>/dev/null | while IFS= read -r line; do
        # Parse event
        local name namespace event_type
        name=$(echo "$line" | jq -r '.metadata.name // empty' 2>/dev/null) || continue
        namespace=$(echo "$line" | jq -r '.metadata.namespace // "default"' 2>/dev/null) || continue
        event_type=$(echo "$line" | jq -r '.type // "MODIFIED"' 2>/dev/null) || continue
        
        if [[ -z "$name" ]]; then
            continue
        fi
        
        log "Event: $event_type - $namespace/$name"
        
        case "$event_type" in
            ADDED|MODIFIED)
                reconcile_database "$name" "$namespace" &
                ;;
            DELETED)
                log "Database deleted: $namespace/$name - cleaning up"
                kubectl delete statefulset "$name" -n "$namespace" --ignore-not-found=true
                kubectl delete service "$name" -n "$namespace" --ignore-not-found=true
                ;;
        esac
    done
}

# Main
case "${1:-watch}" in
    watch) watch_resources ;;
    reconcile) reconcile_database "${2:-}" "${3:-default}" ;;
    *)
        echo "Usage: $0 {watch|reconcile <name> <namespace>}"
        ;;
esac
CONTROLLER
    
    chmod +x "${output_dir}/controller.sh"
    
    # Create Dockerfile for operator
    cat <<'DOCKERFILE' > "${output_dir}/Dockerfile"
FROM debian:bullseye-slim

# Install dependencies
RUN apt-get update && apt-get install -y \
    curl \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Install kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    chmod +x kubectl && \
    mv kubectl /usr/local/bin/

COPY controller.sh /usr/local/bin/controller
RUN chmod +x /usr/local/bin/controller

USER 65532:65532

ENTRYPOINT ["/usr/local/bin/controller", "watch"]
DOCKERFILE

    # Create Kubernetes deployment for operator
    cat <<EOF > "${output_dir}/operator-deployment.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-operator-controller
  namespace: operator-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-operator
  template:
    metadata:
      labels:
        app: db-operator
    spec:
      serviceAccountName: db-operator
      containers:
      - name: controller
        image: my-registry/db-operator:latest
        imagePullPolicy: Always
        env:
        - name: WATCH_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: db-operator
  namespace: operator-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: db-operator
rules:
- apiGroups: ["db.example.com"]
  resources: ["databases", "databases/status"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["services", "secrets", "persistentvolumeclaims"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: db-operator
subjects:
- kind: ServiceAccount
  name: db-operator
  namespace: operator-system
roleRef:
  kind: ClusterRole
  name: db-operator
  apiGroup: rbac.authorization.k8s.io
EOF
    
    success "Bash operator created in: $output_dir"
}

# Operator Lifecycle Manager (OLM) integration
setup_olm_bundle() {
    local operator_name="${1:-$OPERATOR_NAME}"
    local version="${2:-$OPERATOR_VERSION}"
    local output_dir="${3:-./bundle}"
    
    mkdir -p "${output_dir}/manifests" "${output_dir}/metadata"
    
    log "Creating OLM bundle for: $operator_name v$version"
    
    # Create ClusterServiceVersion
    cat <<EOF > "${output_dir}/manifests/${operator_name}.clusterserviceversion.yaml"
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: ${operator_name}.${version}
  namespace: operators
spec:
  displayName: "${operator_name}"
  description: |
    ${operator_name} automates database management in Kubernetes
  version: "${version#v}"
  maturity: alpha
  maintainers:
  - name: Platform Team
    email: platform@example.com
  provider:
    name: Example Corp
  keywords:
  - database
  - operator
  installModes:
  - type: OwnNamespace
    supported: true
  - type: SingleNamespace
    supported: true
  - type: MultiNamespace
    supported: false
  - type: AllNamespaces
    supported: true
  install:
    strategy: deployment
    spec:
      deployments:
      - name: ${operator_name}-controller-manager
        spec:
          replicas: 1
          selector:
            matchLabels:
              control-plane: controller-manager
          template:
            metadata:
              labels:
                control-plane: controller-manager
            spec:
              containers:
              - name: manager
                image: ${OPERATOR_IMAGE}:${version}
                resources:
                  limits:
                    cpu: 200m
                    memory: 256Mi
                  requests:
                    cpu: 100m
                    memory: 128Mi
  customresourcedefinitions:
    owned:
    - name: databases.db.example.com
      version: v1alpha1
      kind: Database
      displayName: Database
      description: Represents a database instance
EOF

    # Create annotations.yaml
    cat <<EOF > "${output_dir}/metadata/annotations.yaml"
annotations:
  # Core bundle annotations.
  operators.operatorframework.io.bundle.package.v1: ${operator_name}
  operators.operatorframework.io.bundle.channels.v1: alpha
  operators.operatorframework.io.bundle.channel.default.v1: alpha
  operators.operatorframework.io.bundle.mediatype.v1: registry+v1
EOF
    
    success "OLM bundle created in: $output_dir"
}

# Main
case "${1:-help}" in
    init) init_operator_project "${2:-example.com}" "${3:-$OPERATOR_NAME}" ;;
    create-api) create_api_controller "${2:-myapp}" "${3:-v1alpha1}" "${4:-MyApp}" ;;
    generate) generate_operator_manifests ;;
    build) build_push_operator "${2:-$OPERATOR_IMAGE}" "${3:-$OPERATOR_VERSION}" ;;
    deploy) deploy_operator "${2:-$OPERATOR_IMAGE}" "${3:-$OPERATOR_VERSION}" "${4:-$OPERATOR_NAMESPACE}" ;;
    create-crd) create_database_operator_crd "${2:-./operator-crds}" ;;
    bash-controller) create_bash_controller "${2:-./bash-operator}" ;;
    olm-bundle) setup_olm_bundle "${2:-$OPERATOR_NAME}" "${3:-$OPERATOR_VERSION}" "${4:-./bundle}" ;;
    *)
        echo "Usage: $0 {init|create-api|generate|build|deploy|create-crd|bash-controller|olm-bundle}"
        ;;
esac
```

---

## ขั้นตอนที่ 504: Advanced Helm Development

```bash
#!/bin/bash
# advanced-helm-manager.sh - Advanced Helm Chart Development Manager

set -euo pipefail

HELM_CHARTS_DIR="${HELM_CHARTS_DIR:-./charts}"
HELM_REGISTRY="${HELM_REGISTRY:-oci://registry.example.com/charts}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Create Helm chart from template
create_chart() {
    local chart_name="${1:-my-app}"
    local chart_version="${2:-0.1.0}"
    local app_version="${3:-1.0.0}"
    local description="${4:-A Helm chart for my app}"
    
    log "Creating Helm chart: $chart_name"
    
    mkdir -p "${HELM_CHARTS_DIR}/${chart_name}"/{templates,charts}
    
    # Chart.yaml
    cat <<EOF > "${HELM_CHARTS_DIR}/${chart_name}/Chart.yaml"
apiVersion: v2
name: ${chart_name}
description: ${description}
type: application
version: ${chart_version}
appVersion: "${app_version}"
keywords:
  - microservice
  - kubernetes
maintainers:
  - name: Platform Team
    email: platform@example.com
sources:
  - https://github.com/myorg/${chart_name}
annotations:
  category: Application
EOF

    # values.yaml
    cat <<EOF > "${HELM_CHARTS_DIR}/${chart_name}/values.yaml"
# Default values for ${chart_name}
replicaCount: 2

image:
  repository: myregistry/${chart_name}
  pullPolicy: IfNotPresent
  tag: ""  # Defaults to appVersion

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

env: {}
envFrom: []

configMap:
  enabled: false
  data: {}

secrets:
  enabled: false
  data: {}

persistence:
  enabled: false
  storageClass: ""
  accessMode: ReadWriteOnce
  size: 1Gi

metrics:
  enabled: true
  port: 9090
  path: /metrics

healthCheck:
  enabled: true
  livenessPath: /health
  readinessPath: /ready
  initialDelaySeconds: 30

podDisruptionBudget:
  enabled: true
  minAvailable: 1
EOF

    # Deployment template
    cat <<'EOF' > "${HELM_CHARTS_DIR}/${chart_name}/templates/deployment.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "CHART_NAME.fullname" . }}
  labels:
    {{- include "CHART_NAME.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "CHART_NAME.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "CHART_NAME.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "CHART_NAME.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
            {{- if .Values.metrics.enabled }}
            - name: metrics
              containerPort: {{ .Values.metrics.port }}
              protocol: TCP
            {{- end }}
          {{- if .Values.healthCheck.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.healthCheck.livenessPath }}
              port: http
            initialDelaySeconds: {{ .Values.healthCheck.initialDelaySeconds }}
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: {{ .Values.healthCheck.readinessPath }}
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          {{- end }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          {{- with .Values.envFrom }}
          envFrom:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
EOF
    
    # Fix CHART_NAME placeholder
    sed -i "s/CHART_NAME/${chart_name}/g" \
        "${HELM_CHARTS_DIR}/${chart_name}/templates/deployment.yaml"

    # _helpers.tpl
    cat <<EOF > "${HELM_CHARTS_DIR}/${chart_name}/templates/_helpers.tpl"
{{/*
Expand the name of the chart.
*/}}
{{- define "${chart_name}.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "${chart_name}.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- \$name := default .Chart.Name .Values.nameOverride }}
{{- if contains \$name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name \$name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart label
*/}}
{{- define "${chart_name}.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "${chart_name}.labels" -}}
helm.sh/chart: {{ include "${chart_name}.chart" . }}
{{ include "${chart_name}.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "${chart_name}.selectorLabels" -}}
app.kubernetes.io/name: {{ include "${chart_name}.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
ServiceAccount name
*/}}
{{- define "${chart_name}.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "${chart_name}.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
EOF

    success "Helm chart created: ${HELM_CHARTS_DIR}/${chart_name}"
}

# Helm chart testing
test_chart() {
    local chart_path="${1:-}"
    local values_file="${2:-}"
    local release_name="${3:-test-release}"
    local namespace="${4:-helm-test}"
    
    if [[ -z "$chart_path" ]]; then
        error "Chart path required"
        return 1
    fi
    
    log "Testing Helm chart: $chart_path"
    
    # Lint chart
    log "Running helm lint..."
    helm lint "$chart_path" \
        ${values_file:+--values "$values_file"} \
        --strict
    
    # Template rendering test
    log "Testing template rendering..."
    helm template "$release_name" "$chart_path" \
        ${values_file:+--values "$values_file"} \
        --namespace "$namespace" \
        --debug > /dev/null
    
    # Dry run
    log "Running dry run..."
    kubectl create namespace "$namespace" --dry-run=client -o yaml | kubectl apply -f - 2>/dev/null || true
    
    helm install "$release_name" "$chart_path" \
        ${values_file:+--values "$values_file"} \
        --namespace "$namespace" \
        --dry-run
    
    success "Chart tests passed: $chart_path"
}

# Package and push chart to registry
package_push_chart() {
    local chart_path="${1:-}"
    local registry="${2:-$HELM_REGISTRY}"
    
    if [[ -z "$chart_path" ]]; then
        error "Chart path required"
        return 1
    fi
    
    log "Packaging and pushing chart: $chart_path"
    
    # Package chart
    helm package "$chart_path" --destination /tmp/helm-packages/
    
    local chart_name
    chart_name=$(helm show chart "$chart_path" | grep "^name:" | awk '{print $2}')
    local chart_version
    chart_version=$(helm show chart "$chart_path" | grep "^version:" | awk '{print $2}')
    
    local package_file="/tmp/helm-packages/${chart_name}-${chart_version}.tgz"
    
    # Push to OCI registry
    helm push "$package_file" "$registry"
    
    success "Chart pushed to registry: ${registry}/${chart_name}:${chart_version}"
}

# Helm diff plugin
show_diff() {
    local release_name="${1:-}"
    local chart_path="${2:-}"
    local values_file="${3:-}"
    local namespace="${4:-default}"
    
    if [[ -z "$release_name" || -z "$chart_path" ]]; then
        error "Usage: show_diff <release-name> <chart-path> [values-file] [namespace]"
        return 1
    fi
    
    log "Showing diff for: $release_name"
    
    if helm plugin list | grep -q diff; then
        helm diff upgrade "$release_name" "$chart_path" \
            ${values_file:+--values "$values_file"} \
            --namespace "$namespace"
    else
        log "Installing helm-diff plugin..."
        helm plugin install https://github.com/databus23/helm-diff
        
        helm diff upgrade "$release_name" "$chart_path" \
            ${values_file:+--values "$values_file"} \
            --namespace "$namespace"
    fi
}

# Main
case "${1:-help}" in
    create) create_chart "${2:-my-app}" "${3:-0.1.0}" "${4:-1.0.0}" "${5:-A Helm chart}" ;;
    test) test_chart "${2:-}" "${3:-}" "${4:-test-release}" "${5:-helm-test}" ;;
    package) package_push_chart "${2:-}" "${3:-$HELM_REGISTRY}" ;;
    diff) show_diff "${2:-}" "${3:-}" "${4:-}" "${5:-default}" ;;
    *)
        echo "Usage: $0 {create|test|package|diff}"
        ;;
esac
```

---

## ขั้นตอนที่ 505: Kubernetes Security Hardening

```bash
#!/bin/bash
# k8s-security-hardener.sh - Kubernetes Security Hardening

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Apply Pod Security Standards
apply_pod_security_standards() {
    local namespace="${1:-default}"
    local level="${2:-restricted}"  # privileged, baseline, restricted
    
    log "Applying Pod Security Standards: $level for namespace $namespace"
    
    kubectl label namespace "$namespace" \
        "pod-security.kubernetes.io/enforce=$level" \
        "pod-security.kubernetes.io/audit=$level" \
        "pod-security.kubernetes.io/warn=$level" \
        --overwrite
    
    success "Pod Security Standards applied: $level to $namespace"
}

# Create secure pod template
create_secure_pod_template() {
    local name="${1:-secure-app}"
    local namespace="${2:-default}"
    local image="${3:-nginx:alpine}"
    
    log "Creating secure pod: $name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ${name}
  namespace: ${namespace}
  labels:
    app: ${name}
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: ${image}
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
  automountServiceAccountToken: false
EOF
    
    success "Secure pod created: $name"
}

# Network policy - default deny
apply_network_policies() {
    local namespace="${1:-default}"
    
    log "Applying network policies for namespace: $namespace"
    
    # Default deny ingress and egress
    cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: ${namespace}
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: ${namespace}
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal
  namespace: ${namespace}
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ${namespace}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ${namespace}
EOF
    
    success "Network policies applied for: $namespace"
}

# Secret management with External Secrets Operator
setup_external_secrets() {
    local namespace="${1:-default}"
    local provider="${2:-aws}"  # aws, gcp, azure, vault
    
    log "Setting up External Secrets Operator with $provider..."
    
    helm repo add external-secrets https://charts.external-secrets.io
    helm repo update
    
    helm upgrade --install external-secrets \
        external-secrets/external-secrets \
        -n external-secrets-operator \
        --create-namespace \
        --wait
    
    # Create ClusterSecretStore
    case "$provider" in
        aws)
            cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets-operator
EOF
            ;;
        vault)
            cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets-operator
EOF
            ;;
    esac
    
    success "External Secrets Operator configured with: $provider"
}

# Create ExternalSecret resource
create_external_secret() {
    local name="${1:-my-secret}"
    local namespace="${2:-default}"
    local provider_key="${3:-my-app/credentials}"
    local secret_store="${4:-aws-secrets-manager}"
    
    log "Creating ExternalSecret: $name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ${name}
  namespace: ${namespace}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: ${secret_store}
    kind: ClusterSecretStore
  target:
    name: ${name}
    creationPolicy: Owner
    template:
      type: Opaque
  dataFrom:
  - extract:
      key: ${provider_key}
EOF
    
    success "ExternalSecret created: $name"
}

# Audit policy for API server
create_audit_policy() {
    local output_file="${1:-/etc/kubernetes/audit-policy.yaml}"
    
    log "Creating audit policy..."
    
    cat <<EOF > "$output_file"
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Don't log read-only requests to certain non-sensitive resources
- level: None
  verbs: ["get", "watch", "list"]
  resources:
  - group: "" # core
    resources: ["events", "endpoints", "nodes", "pods", "services"]
  - group: "apps"
    resources: ["daemonsets", "deployments", "replicasets", "statefulsets"]

# Don't log requests to /api/v1/nodes (high volume)
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
  resources:
  - group: "" # core
    resources: ["endpoints", "services", "services/status"]

# Log secrets, configmaps at Metadata level
- level: Metadata
  resources:
  - group: "" # core
    resources: ["secrets", "configmaps"]

# Log auth at RequestResponse level
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]

# Default: log at Request level
- level: Request
  verbs: ["create", "update", "patch", "delete"]
EOF
    
    success "Audit policy created: $output_file"
}

# Security scan with Trivy
scan_cluster_security() {
    log "Scanning cluster security with Trivy..."
    
    if ! command -v trivy &>/dev/null; then
        log "Installing Trivy..."
        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
    fi
    
    echo "=== Kubernetes Configuration Audit ==="
    trivy k8s --report=summary all 2>/dev/null || \
    echo "Trivy k8s scan requires cluster access"
    
    echo ""
    echo "=== Container Image Scanning Example ==="
    trivy image nginx:latest --severity HIGH,CRITICAL 2>/dev/null | head -30 || true
}

# Main
case "${1:-help}" in
    pss) apply_pod_security_standards "${2:-default}" "${3:-restricted}" ;;
    secure-pod) create_secure_pod_template "${2:-secure-app}" "${3:-default}" "${4:-nginx:alpine}" ;;
    network-policy) apply_network_policies "${2:-default}" ;;
    ext-secrets) setup_external_secrets "${2:-default}" "${3:-aws}" ;;
    ext-secret) create_external_secret "${2:-my-secret}" "${3:-default}" "${4:-my-app/credentials}" "${5:-aws-secrets-manager}" ;;
    audit) create_audit_policy "${2:-/tmp/audit-policy.yaml}" ;;
    scan) scan_cluster_security ;;
    *)
        echo "Usage: $0 {pss|secure-pod|network-policy|ext-secrets|ext-secret|audit|scan}"
        ;;
esac
```

---

## สรุป Part 36

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | เครื่องมือ | ขั้นตอน |
|--------|-----------|---------|
| Kubernetes Operators | Operator SDK, Bash Controller | 503 |
| Advanced Helm Development | Chart templates, OLM bundle | 504 |
| Kubernetes Security Hardening | PSS, Network Policy, External Secrets | 505 |

**เทคโนโลยีที่ใช้:**
- Operator SDK: Go/Bash operator framework
- Helm: Advanced chart development
- OLM: Operator Lifecycle Manager
- External Secrets Operator: Secret management
- Pod Security Standards: Built-in security policies
- Trivy: Container/K8s security scanning

ขั้นตอนต่อไป: Part 37 - Advanced Container Patterns และ Optimization
