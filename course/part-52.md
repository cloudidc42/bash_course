# Part 52: Advanced GitOps and Progressive Delivery (ขั้นตอนที่ 557-560)

## GitOps ระดับ Enterprise และ Progressive Delivery

---

## ขั้นตอนที่ 557: Advanced ArgoCD — Multi-Cluster GitOps

```bash
#!/bin/bash
# argocd-enterprise.sh
# Advanced ArgoCD: Multi-cluster, ApplicationSets, Notifications, Image Updater

set -euo pipefail

ARGOCD_NAMESPACE="${ARGOCD_NAMESPACE:-argocd}"
ARGOCD_VERSION="${ARGOCD_VERSION:-v2.9.0}"
GITHUB_ORG="${GITHUB_ORG:-myorg}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== ArgoCD HA Installation ====================
install_argocd_ha() {
    log "Installing ArgoCD in HA mode..."
    
    kubectl create namespace "${ARGOCD_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    # HA installation
    kubectl apply -n "${ARGOCD_NAMESPACE}" \
        -f "https://raw.githubusercontent.com/argoproj/argo-cd/${ARGOCD_VERSION}/manifests/ha/install.yaml"
    
    # Wait for deployment
    kubectl wait deployment argocd-server \
        --namespace "${ARGOCD_NAMESPACE}" \
        --for condition=Available \
        --timeout 300s
    
    # ArgoCD ConfigMap with custom settings
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.company.com
  application.instanceLabelKey: argocd.argoproj.io/app-name
  
  # Enable Helm
  helm.enabled: "true"
  helm.valuesFileSchemes: "secrets,https"
  
  # Repositories
  repositories: |
    - url: https://github.com/myorg/gitops-config
      type: git
      name: gitops-config
    - url: https://charts.company.com
      type: helm
      name: company-charts
  
  # Resource customizations
  resource.customizations: |
    argoproj.io/Application:
      health.lua: |
        hs = {}
        hs.status = "Progressing"
        hs.message = ""
        if obj.status ~= nil then
          if obj.status.health ~= nil then
            hs.status = obj.status.health.status
            hs.message = obj.status.health.message
          end
        end
        return hs
    networking.k8s.io/Ingress:
      health.lua: |
        hs = {}
        hs.status = "Healthy"
        return hs
  
  # OIDC config
  oidc.config: |
    name: Company SSO
    issuer: https://auth.company.com
    clientID: argocd
    clientSecret: $oidc.clientSecret
    requestedScopes:
    - openid
    - profile
    - email
    - groups
  
  # Dex config (backup auth)
  dex.config: |
    connectors:
    - type: github
      id: github
      name: GitHub
      config:
        clientID: $dex.github.clientID
        clientSecret: $dex.github.clientSecret
        orgs:
        - name: myorg
EOF

    # RBAC policy
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Platform team - admin everywhere
    g, myorg:platform-team, role:admin
    
    # Dev team - deploy to dev/staging
    p, role:developer, applications, sync, dev/*, allow
    p, role:developer, applications, sync, staging/*, allow
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, create, dev/*, allow
    p, role:developer, logs, get, */*, allow
    p, role:developer, exec, create, dev/*, allow
    g, myorg:developers, role:developer
    
    # SRE - deploy to production
    p, role:sre, applications, *, */*, allow
    p, role:sre, clusters, get, *, allow
    p, role:sre, repositories, *, *, allow
    g, myorg:sre-team, role:sre
EOF

    log "ArgoCD HA installed and configured"
}

# ==================== Multi-Cluster Registration ====================
register_remote_clusters() {
    log "Registering remote clusters with ArgoCD..."
    
    local CLUSTERS=("staging" "production" "dr")
    
    for CLUSTER in "${CLUSTERS[@]}"; do
        log "Registering cluster: ${CLUSTER}..."
        
        # Get cluster credentials
        CLUSTER_SERVER=$(kubectl config view -o jsonpath="{.clusters[?(@.name==\"${CLUSTER}\")].cluster.server}" 2>/dev/null)
        
        if [[ -n "${CLUSTER_SERVER}" ]]; then
            argocd cluster add "${CLUSTER}" \
                --name "${CLUSTER}" \
                --system-namespace argocd \
                --upsert \
                2>/dev/null || log "WARN" "Could not add cluster ${CLUSTER}"
        fi
    done
    
    # Label clusters for ApplicationSets
    for CLUSTER_NAME in staging production dr; do
        kubectl label secret \
            -n "${ARGOCD_NAMESPACE}" \
            -l "argocd.argoproj.io/secret-type=cluster,name=${CLUSTER_NAME}" \
            "environment=${CLUSTER_NAME}" \
            --overwrite 2>/dev/null || true
    done
    
    log "Remote clusters registered"
}

# ==================== ApplicationSet ====================
create_applicationsets() {
    log "Creating ApplicationSets for multi-cluster deployment..."
    
    # App of Apps pattern with ApplicationSet
    cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices-all-clusters
  namespace: argocd
spec:
  generators:
  # List generator: explicit cluster-environment combinations
  - matrix:
      generators:
      # Cluster generator from labels
      - clusters:
          selector:
            matchLabels:
              environment: production
        values:
          namespace: production
          env: production
          replicaCount: "3"
      # Git directory generator: one app per directory
      - git:
          repoURL: https://github.com/myorg/gitops-config
          revision: HEAD
          directories:
          - path: apps/*
          - path: apps/*/chart
            exclude: true
  
  syncPolicy:
    applicationsSync: create-update
    preserveResourcesOnDeletion: true
  
  template:
    metadata:
      name: "{{path.basename}}-{{values.env}}"
      labels:
        app: "{{path.basename}}"
        env: "{{values.env}}"
      annotations:
        notifications.argoproj.io/subscribe.on-sync-failed.slack: platform-alerts
        notifications.argoproj.io/subscribe.on-deployed.slack: deployments
    spec:
      project: "{{values.env}}"
      source:
        repoURL: https://github.com/myorg/gitops-config
        targetRevision: HEAD
        path: "{{path}}"
        helm:
          releaseName: "{{path.basename}}"
          valueFiles:
          - values.yaml
          - "values-{{values.env}}.yaml"
          parameters:
          - name: replicaCount
            value: "{{values.replicaCount}}"
          - name: image.tag
            value: "{{metadata.annotations.image-tag}}"
      destination:
        server: "{{server}}"
        namespace: "{{values.namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
          allowEmpty: false
        syncOptions:
        - CreateNamespace=true
        - ServerSideApply=true
        - PrunePropagationPolicy=foreground
        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
      revisionHistoryLimit: 10
EOF

    # Pull Request generator for preview environments
    cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: pr-preview-environments
  namespace: argocd
spec:
  generators:
  - pullRequest:
      github:
        owner: myorg
        repo: application
        appSecretName: github-token
        labels:
        - preview
      requeueAfterSeconds: 60
  template:
    metadata:
      name: "preview-pr-{{number}}"
      labels:
        environment: preview
        pr: "{{number}}"
      annotations:
        notifications.argoproj.io/subscribe.on-sync-succeeded.github: ""
    spec:
      project: preview
      source:
        repoURL: https://github.com/myorg/application
        targetRevision: "{{head_sha}}"
        path: deploy/preview
        helm:
          parameters:
          - name: image.tag
            value: "{{head_short_sha}}"
          - name: ingress.host
            value: "pr-{{number}}.preview.company.com"
      destination:
        server: https://kubernetes.default.svc
        namespace: "preview-pr-{{number}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
EOF

    log "ApplicationSets created"
}

# ==================== ArgoCD Notifications ====================
setup_argocd_notifications() {
    log "Setting up ArgoCD Notifications..."
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Slack notification
  service.slack: |
    token: $slack-token
    apiURL: https://slack.com/api/
    username: ArgoCD
    icon: ":argo:"
  
  # GitHub notification
  service.github: |
    appID: $github-app-id
    installationID: $github-installation-id
    privateKey: $github-private-key
  
  # Webhook for custom integrations
  service.webhook.pagerduty: |
    url: https://events.pagerduty.com/v2/enqueue
    headers:
    - name: Content-Type
      value: application/json
  
  # Templates
  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} deployed successfully to {{.app.spec.destination.server}}
      Commit: {{.app.status.sync.revision}}
    slack:
      attachments: |
        [{
          "title": "✅ Deployment Successful",
          "color": "#36a64f",
          "fields": [
            {"title": "App", "value": "{{.app.metadata.name}}", "short": true},
            {"title": "Environment", "value": "{{.app.spec.destination.namespace}}", "short": true},
            {"title": "Revision", "value": "{{.app.status.sync.revision | substr 0 7}}", "short": true},
            {"title": "Sync Status", "value": "{{.app.status.sync.status}}", "short": true}
          ],
          "actions": [
            {"type": "button", "text": "View App", "url": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"}
          ]
        }]
    github:
      repoURLPath: "{{.app.spec.source.repoURL}}"
      revisionPath: "{{.app.status.sync.revision}}"
      status:
        state: success
        label: "ArgoCD"
        targetURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
  
  template.app-sync-failed: |
    message: |
      Application {{.app.metadata.name}} sync failed!
    slack:
      attachments: |
        [{
          "title": "❌ Sync Failed",
          "color": "#FF0000",
          "fields": [
            {"title": "App", "value": "{{.app.metadata.name}}", "short": true},
            {"title": "Error", "value": "{{.app.status.operationState.message | truncate 200}}", "short": false}
          ]
        }]
    github:
      repoURLPath: "{{.app.spec.source.repoURL}}"
      revisionPath: "{{.app.status.sync.revision}}"
      status:
        state: failure
        label: "ArgoCD"
        targetURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
  
  # Triggers
  trigger.on-deployed: |
    - description: Application deployed
      send:
      - app-deployed
      when: app.status.operationState.phase in ['Succeeded']
  
  trigger.on-sync-failed: |
    - description: Application sync failed
      send:
      - app-sync-failed
      when: app.status.operationState.phase in ['Error', 'Failed']
  
  defaultTriggers: |
    - on-deployed
    - on-sync-failed
EOF

    log "ArgoCD Notifications configured"
}

# ==================== ArgoCD Image Updater ====================
setup_image_updater() {
    log "Setting up ArgoCD Image Updater..."
    
    helm repo add argo https://argoproj.github.io/argo-helm
    helm upgrade --install argocd-image-updater argo/argocd-image-updater \
        --namespace "${ARGOCD_NAMESPACE}" \
        --set config.registries[0].name=ECR \
        --set "config.registries[0].prefix=123456789012.dkr.ecr.us-east-1.amazonaws.com" \
        --set config.registries[0].api_url=https://123456789012.dkr.ecr.us-east-1.amazonaws.com \
        --set config.registries[0].credentials="ext:/scripts/ecr-login.sh" \
        --wait
    
    # Annotate app to use image updater
    kubectl annotate application api-gateway -n "${ARGOCD_NAMESPACE}" \
        "argocd-image-updater.argoproj.io/image-list=api=123456789012.dkr.ecr.us-east-1.amazonaws.com/api-gateway" \
        "argocd-image-updater.argoproj.io/api.update-strategy=semver" \
        "argocd-image-updater.argoproj.io/api.allow-tags=regexp:^v[0-9]+\.[0-9]+\.[0-9]+$" \
        "argocd-image-updater.argoproj.io/write-back-method=git" \
        "argocd-image-updater.argoproj.io/git-branch=main" \
        2>/dev/null || true
    
    log "Image Updater configured"
}

main() {
    case "${1:-all}" in
        install)     install_argocd_ha ;;
        clusters)    register_remote_clusters ;;
        appsets)     create_applicationsets ;;
        notify)      setup_argocd_notifications ;;
        updater)     setup_image_updater ;;
        all)
            install_argocd_ha
            register_remote_clusters
            create_applicationsets
            setup_argocd_notifications
            setup_image_updater
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 558: Progressive Delivery — Argo Rollouts

```bash
#!/bin/bash
# argo-rollouts-progressive.sh
# Progressive Delivery: Canary, Blue-Green, Traffic Management with Argo Rollouts

set -euo pipefail

NAMESPACE="${NAMESPACE:-production}"
ROLLOUTS_VERSION="${ROLLOUTS_VERSION:-v1.6.0}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Argo Rollouts Installation ====================
install_argo_rollouts() {
    log "Installing Argo Rollouts..."
    
    kubectl create namespace argo-rollouts --dry-run=client -o yaml | kubectl apply -f -
    
    kubectl apply -n argo-rollouts \
        -f "https://github.com/argoproj/argo-rollouts/releases/download/${ROLLOUTS_VERSION}/install.yaml"
    
    # Install kubectl plugin
    curl -LO "https://github.com/argoproj/argo-rollouts/releases/download/${ROLLOUTS_VERSION}/kubectl-argo-rollouts-linux-amd64"
    chmod +x kubectl-argo-rollouts-linux-amd64
    sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
    
    kubectl wait deployment argo-rollouts \
        --namespace argo-rollouts \
        --for condition=Available \
        --timeout 300s
    
    log "Argo Rollouts installed"
}

# ==================== Canary Deployment with Analysis ====================
create_canary_rollout() {
    log "Creating Canary Rollout with automated analysis..."
    
    # AnalysisTemplate for automated verification
    cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-analysis
  namespace: production
spec:
  args:
  - name: service-name
  - name: namespace
    value: production
  - name: prometheus-url
    value: http://prometheus-operated.monitoring:9090
  
  metrics:
  # Success rate must be > 99%
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.99
    failureCondition: result[0] < 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: "{{args.prometheus-url}}"
        query: |
          sum(rate(http_requests_total{
            service="{{args.service-name}}",
            code!~"5..",
            namespace="{{args.namespace}}"
          }[2m])) /
          sum(rate(http_requests_total{
            service="{{args.service-name}}",
            namespace="{{args.namespace}}"
          }[2m]))
  
  # P99 latency must be < 500ms
  - name: latency-p99
    interval: 1m
    successCondition: result[0] <= 0.5
    failureCondition: result[0] > 1.0
    failureLimit: 2
    provider:
      prometheus:
        address: "{{args.prometheus-url}}"
        query: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket{
              service="{{args.service-name}}",
              namespace="{{args.namespace}}"
            }[2m])) by (le)
          )
  
  # Canary error rate must be similar to stable (< 2x)
  - name: error-rate-comparison
    interval: 2m
    successCondition: result[0] < 2.0
    failureCondition: result[0] >= 3.0
    provider:
      prometheus:
        address: "{{args.prometheus-url}}"
        query: |
          (
            sum(rate(http_requests_total{
              service="{{args.service-name}}-canary",
              code=~"5..",
              namespace="{{args.namespace}}"
            }[2m])) /
            sum(rate(http_requests_total{
              service="{{args.service-name}}-canary",
              namespace="{{args.namespace}}"
            }[2m]))
          ) / (
            sum(rate(http_requests_total{
              service="{{args.service-name}}",
              code=~"5..",
              namespace="{{args.namespace}}"
            }[2m])) /
            sum(rate(http_requests_total{
              service="{{args.service-name}}",
              namespace="{{args.namespace}}"
            }[2m]))
          )
  
  # Synthetic test via web job
  - name: synthetic-test
    interval: 5m
    failureLimit: 1
    provider:
      job:
        spec:
          template:
            spec:
              containers:
              - name: tester
                image: curlimages/curl:latest
                command:
                - sh
                - -c
                - |
                  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
                    -H "x-canary: always" \
                    https://api.company.com/health)
                  [ "$STATUS" = "200" ] && exit 0 || exit 1
              restartPolicy: Never
          backoffLimit: 0
EOF

    # Rollout with Istio traffic management
    cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-gateway
  namespace: production
spec:
  replicas: 10
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/api-gateway:v1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
  strategy:
    canary:
      # Istio traffic management
      trafficRouting:
        istio:
          virtualService:
            name: api-gateway-vsvc
            routes:
            - primary
          destinationRule:
            name: api-gateway-destrule
            canarySubsetName: canary
            stableSubsetName: stable
      
      # Canary steps
      steps:
      - setWeight: 5
      - pause: {duration: 2m}
      - analysis:
          templates:
          - templateName: canary-analysis
          args:
          - name: service-name
            value: api-gateway
      - setWeight: 20
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: canary-analysis
          args:
          - name: service-name
            value: api-gateway
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 80
      - pause: {duration: 2m}
      - setWeight: 100
      
      # Maximum number of pods above replicas
      maxSurge: 2
      maxUnavailable: 0
      
      # Abort automatically if analysis fails
      abortScaleDownDelaySeconds: 30
EOF

    log "Canary rollout created"
}

# ==================== Blue-Green Deployment ====================
create_bluegreen_rollout() {
    log "Creating Blue-Green Rollout..."
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: checkout-service
  namespace: production
spec:
  replicas: 5
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: checkout-service
  template:
    metadata:
      labels:
        app: checkout-service
    spec:
      containers:
      - name: checkout
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/checkout:v2.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
  strategy:
    blueGreen:
      # Service names
      activeService: checkout-active
      previewService: checkout-preview
      
      # Auto-promote after preview tests pass
      autoPromotionEnabled: false
      
      # Wait for pre-promotion analysis
      prePromotionAnalysis:
        templates:
        - templateName: canary-analysis
        args:
        - name: service-name
          value: checkout-service
      
      # Post-promotion analysis
      postPromotionAnalysis:
        templates:
        - templateName: canary-analysis
        args:
        - name: service-name
          value: checkout-service
      
      # Keep old (blue) pods for fast rollback
      scaleDownDelaySeconds: 600
      
      # Preview replicas
      previewReplicaCount: 2
      
      # Anti-affinity with active set
      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}
EOF

    # Create preview ingress for testing
    cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: checkout-preview
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "x-preview"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  rules:
  - host: checkout.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: checkout-preview
            port:
              number: 80
EOF

    log "Blue-Green rollout created"
}

# ==================== Rollout Promotion Script ====================
promote_rollout() {
    local APP="${1}"
    local NAMESPACE="${2:-production}"
    
    log "Promoting rollout: ${APP}..."
    
    # Check current status
    local STATUS=$(kubectl argo rollouts status "${APP}" -n "${NAMESPACE}" 2>/dev/null || echo "unknown")
    log "Current status: ${STATUS}"
    
    # Get analysis run results
    local ANALYSIS_RUNS=$(kubectl get analysisruns -n "${NAMESPACE}" \
        -l "rollouts-pod-template-hash=$(kubectl argo rollouts get rollout "${APP}" -n "${NAMESPACE}" -o json | python3 -c "import json,sys; print(json.load(sys.stdin).get('status',{}).get('canary',{}).get('currentStepHash',''))" 2>/dev/null)" \
        --no-headers 2>/dev/null | head -5)
    
    echo "Analysis runs:"
    echo "${ANALYSIS_RUNS}"
    
    # Promote
    kubectl argo rollouts promote "${APP}" -n "${NAMESPACE}"
    
    # Watch progress
    kubectl argo rollouts status "${APP}" -n "${NAMESPACE}" --watch --timeout 600s
    
    log "Rollout ${APP} promoted"
}

# ==================== Rollout Dashboard ====================
setup_rollouts_dashboard() {
    log "Setting up Argo Rollouts dashboard..."
    
    # Expose dashboard
    cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argo-rollouts-dashboard
  namespace: argo-rollouts
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/auth-url: "https://oauth.company.com/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://oauth.company.com/oauth2/start"
spec:
  tls:
  - hosts:
    - rollouts.company.com
    secretName: rollouts-tls
  rules:
  - host: rollouts.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argo-rollouts-dashboard
            port:
              number: 3100
EOF

    log "Rollouts dashboard exposed at https://rollouts.company.com"
}

main() {
    case "${1:-all}" in
        install)   install_argo_rollouts ;;
        canary)    create_canary_rollout ;;
        bluegreen) create_bluegreen_rollout ;;
        promote)   promote_rollout "${2}" "${3:-production}" ;;
        dashboard) setup_rollouts_dashboard ;;
        all)
            install_argo_rollouts
            create_canary_rollout
            create_bluegreen_rollout
            setup_rollouts_dashboard
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 559: Flux CD — GitOps Alternative

```bash
#!/bin/bash
# fluxcd-enterprise.sh
# Flux CD: Automated GitOps with Helm Controller, Image Automation

set -euo pipefail

FLUX_VERSION="${FLUX_VERSION:-v2.2.0}"
GITHUB_USER="${GITHUB_USER:-myorg}"
GITHUB_REPO="${GITHUB_REPO:-gitops-fleet}"
GITHUB_BRANCH="${GITHUB_BRANCH:-main}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Flux Bootstrap ====================
bootstrap_flux() {
    log "Bootstrapping Flux CD..."
    
    # Install Flux CLI
    curl -s "https://fluxcd.io/install.sh" | sudo bash 2>/dev/null || \
        { flux version --client 2>/dev/null && log "Flux CLI already installed"; }
    
    # Prerequisite check
    flux check --pre
    
    # Bootstrap with GitHub
    flux bootstrap github \
        --token-auth \
        --owner "${GITHUB_USER}" \
        --repository "${GITHUB_REPO}" \
        --branch "${GITHUB_BRANCH}" \
        --path clusters/production \
        --personal false \
        --private false \
        --reconcile \
        --components-extra=image-reflector-controller,image-automation-controller
    
    log "Flux bootstrapped"
}

# ==================== Flux Sources and Kustomizations ====================
setup_flux_sources() {
    log "Setting up Flux Sources..."
    
    # Git repository source
    cat <<EOF | kubectl apply -f -
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: application-manifests
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/${GITHUB_USER}/${GITHUB_REPO}
  ref:
    branch: ${GITHUB_BRANCH}
  secretRef:
    name: flux-system
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: company-charts
  namespace: flux-system
spec:
  interval: 5m
  url: https://charts.company.com
  type: oci
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: production-configs
  namespace: flux-system
spec:
  interval: 5m
  url: oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/gitops-configs
  ref:
    tag: latest
  provider: aws
EOF

    # Kustomization for production
    cat <<'EOF' | kubectl apply -f -
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: production-infrastructure
  namespace: flux-system
spec:
  interval: 5m
  timeout: 10m
  sourceRef:
    kind: GitRepository
    name: application-manifests
  path: ./infrastructure/production
  prune: true
  wait: true
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
    namespace: production
  - apiVersion: apps/v1
    kind: StatefulSet
    name: postgresql
    namespace: databases
  postBuild:
    substituteFrom:
    - kind: ConfigMap
      name: cluster-vars
    - kind: Secret
      name: cluster-secrets
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: production-applications
  namespace: flux-system
spec:
  interval: 2m
  dependsOn:
  - name: production-infrastructure
  sourceRef:
    kind: GitRepository
    name: application-manifests
  path: ./apps/production
  prune: true
  targetNamespace: production
  patches:
  - patch: |
      - op: replace
        path: /spec/replicas
        value: 3
    target:
      kind: Deployment
      name: "*"
      namespace: production
      labelSelector: "tier=frontend"
EOF

    log "Flux sources configured"
}

# ==================== Flux HelmRelease ====================
create_helm_releases() {
    log "Creating Flux HelmReleases..."
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: production-monitoring
  namespace: flux-system
spec:
  interval: 1h
  chart:
    spec:
      chart: kube-prometheus-stack
      version: ">=50.0.0 <60.0.0"
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
        namespace: flux-system
      interval: 12h
  install:
    remediation:
      retries: 3
  upgrade:
    cleanupOnFail: true
    remediation:
      retries: 3
      strategy: rollback
  rollback:
    cleanupOnFail: true
  uninstall:
    keepHistory: false
  targetNamespace: monitoring
  valuesFrom:
  - kind: ConfigMap
    name: monitoring-values
  - kind: Secret
    name: monitoring-secrets
    optional: true
  values:
    prometheus:
      prometheusSpec:
        retention: 30d
        storageSpec:
          volumeClaimTemplate:
            spec:
              storageClassName: gp3
              accessModes: [ReadWriteOnce]
              resources:
                requests:
                  storage: 200Gi
    grafana:
      adminPassword: "${GRAFANA_ADMIN_PASSWORD}"
    alertmanager:
      config:
        receivers:
        - name: pagerduty
          pagerduty_configs:
          - routing_key: "${PAGERDUTY_KEY}"
EOF

    log "HelmReleases created"
}

# ==================== Flux Image Automation ====================
setup_image_automation() {
    log "Setting up Flux Image Automation..."
    
    # ImageRepository - track ECR
    cat <<EOF | kubectl apply -f -
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: api-gateway
  namespace: flux-system
spec:
  image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/api-gateway
  interval: 5m
  provider: aws
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: api-gateway-semver
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: api-gateway
  policy:
    semver:
      range: ">=1.0.0 <2.0.0"
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: production-image-updates
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: application-manifests
  git:
    push:
      branch: ${GITHUB_BRANCH}
    commit:
      author:
        name: Flux Image Automation
        email: flux@company.com
      messageTemplate: |
        chore: update images in production

        {{ range .Updated.Images -}}
        - {{ .Identifier }} -> {{ .NewTag }}
        {{ end -}}
  update:
    strategy: Setters
    path: ./apps/production
EOF

    # Annotate deployment for image updates
    kubectl annotate deployment api-gateway -n production \
        "image.toolkit.fluxcd.io/update-policy=api-gateway-semver" \
        2>/dev/null || true
    
    log "Image automation configured"
}

# ==================== Multi-Tenancy with Flux ====================
setup_multitenancy() {
    log "Setting up Flux multi-tenancy..."
    
    # Team namespaces with RBAC
    for TEAM in backend frontend data; do
        # ServiceAccount for team
        kubectl create serviceaccount "flux-${TEAM}" -n "team-${TEAM}" --dry-run=client -o yaml | kubectl apply -f -
        
        # Tenant Kustomization
        cat <<EOF | kubectl apply -f -
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team-${TEAM}
  namespace: flux-system
spec:
  serviceAccountName: flux-${TEAM}
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: team-${TEAM}-gitops
  path: ./
  prune: true
  targetNamespace: team-${TEAM}
EOF
    done
    
    log "Multi-tenancy configured"
}

main() {
    case "${1:-all}" in
        bootstrap) bootstrap_flux ;;
        sources)   setup_flux_sources ;;
        releases)  create_helm_releases ;;
        images)    setup_image_automation ;;
        tenancy)   setup_multitenancy ;;
        all)
            bootstrap_flux
            setup_flux_sources
            create_helm_releases
            setup_image_automation
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 560: GitOps Repository Structure and Standards

```bash
#!/bin/bash
# gitops-repo-structure.sh
# GitOps Repository Structure: Monorepo, Environment Promotion, Git Flow

set -euo pipefail

REPO_ROOT="${1:-/tmp/gitops-repo}"
ORG="${ORG:-myorg}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== GitOps Repository Structure ====================
create_gitops_structure() {
    log "Creating GitOps repository structure..."
    
    mkdir -p "${REPO_ROOT}"/{clusters,infrastructure,apps,tenants,policies}
    
    # Cluster structure
    for ENV in dev staging production dr; do
        mkdir -p "${REPO_ROOT}/clusters/${ENV}"/{flux-system,infrastructure,apps}
        
        # Cluster kustomization
        cat <<EOF > "${REPO_ROOT}/clusters/${ENV}/kustomization.yaml"
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- flux-system/gotk-components.yaml
- flux-system/gotk-sync.yaml
- infrastructure/kustomization.yaml
- apps/kustomization.yaml
EOF
    done
    
    # Infrastructure layers
    mkdir -p "${REPO_ROOT}/infrastructure"/{base,overlays/{dev,staging,production}}
    mkdir -p "${REPO_ROOT}/infrastructure/base"/{cert-manager,ingress-nginx,monitoring,databases,security}
    
    # App structure
    mkdir -p "${REPO_ROOT}/apps"/{base,overlays/{dev,staging,production}}
    
    # Create base app template
    cat <<'EOF' > "${REPO_ROOT}/apps/base/kustomization.yaml"
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- api-gateway/
- backend-service/
- checkout-service/
- worker-service/
EOF

    # Create app manifests structure
    for APP in api-gateway backend-service checkout-service; do
        mkdir -p "${REPO_ROOT}/apps/base/${APP}"
        
        cat <<EOF > "${REPO_ROOT}/apps/base/${APP}/deployment.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${APP}
  template:
    metadata:
      labels:
        app: ${APP}
    spec:
      containers:
      - name: ${APP}
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/${APP}:latest # {"$imagepolicy": "flux-system:${APP}"}
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
EOF
        
        cat <<EOF > "${REPO_ROOT}/apps/base/${APP}/kustomization.yaml"
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
- hpa.yaml
EOF
    done
    
    # Environment overlays
    for ENV in dev staging production; do
        cat <<EOF > "${REPO_ROOT}/apps/overlays/${ENV}/kustomization.yaml"
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base
patches:
- path: replica-count.yaml
- path: resource-limits.yaml
images:
- name: 123456789012.dkr.ecr.us-east-1.amazonaws.com/api-gateway
  newTag: ${ENV}-latest
EOF
    done
    
    log "GitOps repo structure created at ${REPO_ROOT}"
}

# ==================== Environment Promotion Script ====================
promote_to_environment() {
    local FROM_ENV="${1:-staging}"
    local TO_ENV="${2:-production}"
    local APP="${3:-api-gateway}"
    local IMAGE_TAG="${4}"
    
    log "Promoting ${APP} from ${FROM_ENV} to ${TO_ENV}: tag=${IMAGE_TAG}"
    
    cd "${REPO_ROOT}" || exit 1
    
    # Get current tag in source environment
    if [[ -z "${IMAGE_TAG}" ]]; then
        IMAGE_TAG=$(grep -r "newTag:" "apps/overlays/${FROM_ENV}/" | \
            grep "${APP}" | head -1 | awk '{print $2}')
        log "Using tag from ${FROM_ENV}: ${IMAGE_TAG}"
    fi
    
    # Update target environment
    local KUSTOMIZE_FILE="${REPO_ROOT}/apps/overlays/${TO_ENV}/kustomization.yaml"
    
    if grep -q "name:.*${APP}" "${KUSTOMIZE_FILE}" 2>/dev/null; then
        # Update existing tag
        sed -i "s|newTag:.*# ${APP}|newTag: ${IMAGE_TAG} # ${APP}|g" "${KUSTOMIZE_FILE}" 2>/dev/null || \
        python3 - <<PYEOF
import yaml
with open('${KUSTOMIZE_FILE}') as f:
    config = yaml.safe_load(f)

for img in config.get('images', []):
    if '${APP}' in img.get('name', ''):
        img['newTag'] = '${IMAGE_TAG}'

with open('${KUSTOMIZE_FILE}', 'w') as f:
    yaml.dump(config, f, default_flow_style=False)
print("Updated ${APP} to ${IMAGE_TAG} in ${TO_ENV}")
PYEOF
    fi
    
    # Commit and push
    git -C "${REPO_ROOT}" add "apps/overlays/${TO_ENV}/"
    git -C "${REPO_ROOT}" commit -m "feat: promote ${APP} ${IMAGE_TAG} to ${TO_ENV}

Promoted from: ${FROM_ENV}
Image: ${APP}:${IMAGE_TAG}
Promoted by: $(git config user.email 2>/dev/null || echo 'automation')
Promotion timestamp: $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
    
    git -C "${REPO_ROOT}" push origin "${GITHUB_BRANCH:-main}"
    
    log "Promotion complete: ${APP} ${IMAGE_TAG} → ${TO_ENV}"
}

# ==================== Git Tagging Strategy ====================
tag_release() {
    local APP="${1}"
    local VERSION="${2}"
    local ENV="${3:-production}"
    
    log "Tagging release: ${APP} ${VERSION} for ${ENV}..."
    
    local TAG="${ENV}/${APP}/v${VERSION}"
    local TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    
    git -C "${REPO_ROOT}" tag -a "${TAG}" \
        -m "Release ${APP} v${VERSION} to ${ENV}

Deployed at: ${TIMESTAMP}
Environment: ${ENV}
App: ${APP}
Version: v${VERSION}"
    
    git -C "${REPO_ROOT}" push origin "${TAG}"
    
    log "Tagged: ${TAG}"
}

# ==================== GitOps Policy Validation ====================
validate_gitops_policies() {
    log "Validating GitOps policies..."
    
    # Install conftest if not available
    if ! command -v conftest &>/dev/null; then
        curl -Lo conftest https://github.com/open-policy-agent/conftest/releases/download/v0.46.0/conftest_0.46.0_Linux_x86_64.tar.gz
        tar xzf conftest -C /usr/local/bin/ 2>/dev/null || true
    fi
    
    # OPA policies for GitOps
    mkdir -p "${REPO_ROOT}/policies"
    
    cat <<'EOF' > "${REPO_ROOT}/policies/kubernetes.rego"
package kubernetes.admission

# Deny images without digest or explicit tag in production
deny[msg] {
    input.kind == "Deployment"
    input.metadata.namespace == "production"
    container := input.spec.template.spec.containers[_]
    not contains(container.image, "@sha256:")
    not contains(container.image, ":")
    msg := sprintf("Container %v must use explicit image tag or digest", [container.name])
}

# Require resource limits
deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    not container.resources.limits.memory
    msg := sprintf("Container %v must have memory limits", [container.name])
}

# Require readiness probe
warn[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    not container.readinessProbe
    msg := sprintf("Container %v should have a readiness probe", [container.name])
}
EOF

    # Validate all manifests
    local PASS=0
    local FAIL=0
    
    find "${REPO_ROOT}" -name "*.yaml" -not -path "*/flux-system/*" | while read -r FILE; do
        if conftest test "${FILE}" \
            --policy "${REPO_ROOT}/policies/" \
            --output json 2>/dev/null | \
            python3 -c "import json,sys; d=json.load(sys.stdin); print('FAIL' if d[0]['failures'] else 'PASS')" 2>/dev/null; then
            echo "PASS: ${FILE}"
        else
            echo "FAIL: ${FILE}"
        fi
    done
    
    log "GitOps policy validation complete"
}

main() {
    case "${1:-structure}" in
        structure) create_gitops_structure ;;
        promote)   promote_to_environment "${2:-staging}" "${3:-production}" "${4:-api-gateway}" "${5:-}" ;;
        tag)       tag_release "${2}" "${3}" "${4:-production}" ;;
        validate)  validate_gitops_policies ;;
        all)
            create_gitops_structure
            validate_gitops_policies
            ;;
    esac
}

main "$@"
```

---

## สรุป Part 52

- ✅ **Step 557**: Advanced ArgoCD — HA install, Multi-cluster, ApplicationSets (PR Preview, Matrix generator), Notifications (Slack/GitHub/PagerDuty), Image Updater
- ✅ **Step 558**: Argo Rollouts — Canary with Prometheus AnalysisTemplate, Blue-Green with pre/post analysis, Dashboard
- ✅ **Step 559**: Flux CD — Bootstrap, GitRepository/HelmRelease/OCIRepository, Image Automation, Multi-tenancy
- ✅ **Step 560**: GitOps Repository Structure — Monorepo layout, Environment promotion, Git tagging, OPA policy validation

**ขั้นตอนต่อไป: Part 53 - Advanced Observability and AIOps**
