# Part 40: Module 3 Completion - Advanced Patterns Workshop

## Module 3: Advanced Level - Complete Workshop และ Module Summary

---

## ขั้นตอนที่ 516: Complete Advanced Platform Workshop

การสรุป Module 3 ด้วย Workshop ที่รวมทุกหัวข้อที่เรียนมา

```bash
#!/bin/bash
# advanced-platform-workshop.sh - Complete Advanced Platform Workshop

set -euo pipefail

WORKSHOP_NAME="Advanced Platform Engineering Workshop"
WORKSHOP_DIR="${WORKSHOP_DIR:-./advanced-workshop}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
separator() { echo "=========================================="; }
success() { echo "[SUCCESS] $*"; }
header() { echo ""; echo "### $1 ###"; echo ""; }

# Component: Full Stack Observability
setup_observability_stack() {
    header "Setting up Full Stack Observability"
    
    local namespace="${1:-observability}"
    
    # Create namespace
    kubectl create namespace "$namespace" --dry-run=client -o yaml | kubectl apply -f -
    
    # Install Kube Prometheus Stack (Prometheus + Grafana + Alertmanager)
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update
    
    cat <<EOF > /tmp/monitoring-values.yaml
prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          resources:
            requests:
              storage: 50Gi
    additionalScrapeConfigs: []
    
grafana:
  adminPassword: admin
  persistence:
    enabled: true
    size: 10Gi
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: 'default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default
  
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 5Gi

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true
EOF
    
    helm upgrade --install kube-prometheus-stack \
        prometheus-community/kube-prometheus-stack \
        -n "$namespace" \
        -f /tmp/monitoring-values.yaml \
        --wait --timeout 10m
    
    log "Monitoring stack deployed"
    
    # Install Loki for logs
    helm upgrade --install loki grafana/loki-stack \
        -n "$namespace" \
        --set grafana.enabled=false \
        --set loki.persistence.enabled=true \
        --set loki.persistence.size=20Gi \
        --wait || log "Loki install may take more time"
    
    log "Loki log aggregation deployed"
}

# Component: CI/CD Pipeline Setup
setup_cicd_pipeline() {
    header "Setting up CI/CD Pipeline"
    
    local git_repo="${1:-https://github.com/myorg/myapp}"
    local registry="${2:-registry.example.com}"
    
    mkdir -p "${WORKSHOP_DIR}/cicd"
    
    # GitHub Actions workflow
    mkdir -p "${WORKSHOP_DIR}/.github/workflows"
    
    cat <<EOF > "${WORKSHOP_DIR}/.github/workflows/main.yml"
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ${registry}
  IMAGE_NAME: myapp

jobs:
  lint-test:
    name: Lint and Test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run linting
      run: make lint
    
    - name: Run unit tests
      run: make test-unit
    
    - name: Run integration tests
      run: make test-integration
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run SAST
      uses: github/codeql-action/analyze@v2
    
    - name: Scan dependencies
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: \${{ secrets.SNYK_TOKEN }}

  build-push:
    name: Build and Push
    needs: [lint-test, security-scan]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    outputs:
      image-tag: \${{ steps.meta.outputs.tags }}
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to Registry
      uses: docker/login-action@v3
      with:
        registry: \${{ env.REGISTRY }}
        username: \${{ secrets.REGISTRY_USERNAME }}
        password: \${{ secrets.REGISTRY_PASSWORD }}
    
    - name: Docker metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: \${{ env.REGISTRY }}/\${{ env.IMAGE_NAME }}
        tags: |
          type=sha
          type=raw,value=latest
    
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: \${{ steps.meta.outputs.tags }}
        labels: \${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        sbom: true
        provenance: true
    
    - name: Scan built image
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: \${{ env.REGISTRY }}/\${{ env.IMAGE_NAME }}:latest
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  performance-test:
    name: Performance Test
    needs: build-push
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup k6
      uses: grafana/setup-k6-action@v1
    
    - name: Run performance tests
      run: k6 run tests/performance/load-test.js
    
    - name: Check regressions
      run: ./scripts/check-perf-regression.sh

  deploy-staging:
    name: Deploy to Staging
    needs: [build-push, performance-test]
    runs-on: ubuntu-latest
    environment: staging
    steps:
    - uses: actions/checkout@v4
    
    - name: Update staging manifest
      run: |
        sed -i "s|image:.*|image: \${{ needs.build-push.outputs.image-tag }}|g" \
          deploy/staging/deployment.yaml
        git config user.email "ci@example.com"
        git config user.name "CI Bot"
        git add deploy/staging/deployment.yaml
        git commit -m "deploy: update staging to \${{ github.sha }}"
        git push
    
    - name: Wait for ArgoCD sync
      run: |
        argocd app wait myapp-staging \
          --health \
          --sync \
          --timeout 300

  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
    - uses: actions/checkout@v4
    
    - name: Progressive rollout (Canary)
      run: |
        kubectl argo rollouts promote myapp-rollout -n production
    
    - name: Monitor canary metrics
      run: ./scripts/monitor-canary.sh --duration 10m --error-threshold 0.01
    
    - name: Full promotion
      run: |
        kubectl argo rollouts promote myapp-rollout -n production --full
EOF
    
    log "CI/CD pipeline created"
}

# Component: GitOps Configuration
setup_gitops_config() {
    header "Setting up GitOps Configuration"
    
    local git_repo="${1:-https://github.com/myorg/platform-config}"
    
    mkdir -p "${WORKSHOP_DIR}/gitops"/{clusters,apps,infra}
    
    # Flux system kustomization
    cat <<EOF > "${WORKSHOP_DIR}/gitops/clusters/production/kustomization.yaml"
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../base
- flux-system/
- platform/
- apps/
EOF

    # App deployment with progressive rollout
    cat <<EOF > "${WORKSHOP_DIR}/gitops/apps/myapp.yaml"
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: registry.example.com/myapp:latest
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
  strategy:
    canary:
      canaryService: myapp-canary
      stableService: myapp-stable
      trafficRouting:
        istio:
          virtualService:
            name: myapp-vsvc
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 25
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 75
      - pause: {duration: 5m}
      - setWeight: 100
      analysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: myapp-canary
EOF
    
    log "GitOps configuration created"
}

# Component: SRE runbook automation
setup_sre_automation() {
    header "Setting up SRE Automation"
    
    mkdir -p "${WORKSHOP_DIR}/sre"/{runbooks,automation,on-call}
    
    # Automated incident runbook
    cat <<'EOF' > "${WORKSHOP_DIR}/sre/automation/auto-remediate.sh"
#!/bin/bash
# auto-remediate.sh - Automated incident remediation

ALERT_NAME="${1:-}"
NAMESPACE="${2:-default}"
SERVICE="${3:-}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [REMEDIATION] $*" | tee -a /var/log/remediation.log; }

case "$ALERT_NAME" in
    HighErrorRate)
        log "High error rate detected for $SERVICE"
        
        # Check if rolling restart helps
        log "Attempting rolling restart..."
        kubectl rollout restart deployment/"$SERVICE" -n "$NAMESPACE"
        
        # Wait and check
        sleep 60
        ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=job:http_errors:rate5m{job=\"$SERVICE\"}" | \
            jq -r '.data.result[0].value[1] // "0"')
        
        if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
            log "Error rate still high after restart: $ERROR_RATE"
            log "Escalating to on-call engineer"
            # Page engineer
            curl -s -X POST "${PAGERDUTY_URL:-}" \
                -H "Content-Type: application/json" \
                -d "{\"event_action\":\"trigger\",\"payload\":{\"summary\":\"High error rate for $SERVICE after restart\",\"severity\":\"critical\"}}"
        else
            log "Error rate normalized after restart: $ERROR_RATE"
        fi
        ;;
    
    HighMemoryUsage)
        log "High memory usage detected"
        
        # Scale up if HPA allows
        CURRENT=$(kubectl get hpa -n "$NAMESPACE" "${SERVICE}-hpa" \
            -o jsonpath='{.status.currentReplicas}' 2>/dev/null || echo "0")
        MAX=$(kubectl get hpa -n "$NAMESPACE" "${SERVICE}-hpa" \
            -o jsonpath='{.spec.maxReplicas}' 2>/dev/null || echo "0")
        
        if [[ "$CURRENT" -lt "$MAX" ]]; then
            log "Scaling up $SERVICE from $CURRENT to $((CURRENT + 2)) replicas"
            kubectl scale deployment "$SERVICE" -n "$NAMESPACE" \
                --replicas=$((CURRENT + 2))
        else
            log "At max replicas ($MAX). Cannot scale further"
        fi
        ;;
    
    PodCrashLooping)
        log "Pod crash looping detected for $SERVICE"
        
        # Get logs before restart
        kubectl logs -n "$NAMESPACE" -l "app=$SERVICE" \
            --previous --tail=100 >> /var/log/crash-logs-"$SERVICE"-$(date +%Y%m%d%H%M%S).log 2>/dev/null
        
        # Delete crashing pods (will be recreated)
        CRASHING=$(kubectl get pods -n "$NAMESPACE" -l "app=$SERVICE" \
            --field-selector=status.containerStatuses[0].state.waiting.reason=CrashLoopBackOff \
            -o name 2>/dev/null)
        
        if [[ -n "$CRASHING" ]]; then
            log "Deleting crashing pods: $CRASHING"
            echo "$CRASHING" | xargs kubectl delete -n "$NAMESPACE"
        fi
        ;;
    
    DatabaseConnectionsExhausted)
        log "Database connections exhausted"
        
        # Kill idle connections
        PGPASSWORD="${PGPASSWORD:-}" psql \
            -h "${DB_HOST:-localhost}" \
            -U "${DB_USER:-postgres}" \
            -d "${DB_NAME:-myapp}" \
            -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND state_change < NOW() - INTERVAL '5 minutes';" \
            2>/dev/null || log "Could not kill idle DB connections"
        ;;
    
    *)
        log "Unknown alert: $ALERT_NAME - no automated remediation available"
        ;;
esac
EOF
    
    chmod +x "${WORKSHOP_DIR}/sre/automation/auto-remediate.sh"
    
    # Error budget tracking
    cat <<'EOF' > "${WORKSHOP_DIR}/sre/automation/error-budget-tracker.sh"
#!/bin/bash
# error-budget-tracker.sh - SLO Error Budget Tracker

PROMETHEUS_URL="${PROMETHEUS_URL:-http://prometheus:9090}"
SERVICE="${1:-my-service}"
SLO_TARGET="${2:-99.9}"
WINDOW_DAYS="${3:-30}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

# Calculate error budget
query_prometheus() {
    local query="${1:-}"
    curl -s "${PROMETHEUS_URL}/api/v1/query" \
        --data-urlencode "query=$query" | \
    jq -r '.data.result[0].value[1] // "0"'
}

# Total requests in window
TOTAL_REQUESTS=$(query_prometheus \
    "sum(increase(http_requests_total{job=\"$SERVICE\"}[${WINDOW_DAYS}d]))")

# Error requests in window
ERROR_REQUESTS=$(query_prometheus \
    "sum(increase(http_requests_total{job=\"$SERVICE\",status_code=~\"5..\"}[${WINDOW_DAYS}d]))")

# Calculate actual availability
if [[ -n "$TOTAL_REQUESTS" && "$TOTAL_REQUESTS" != "0" ]]; then
    ACTUAL_AVAILABILITY=$(echo "scale=4; (1 - ($ERROR_REQUESTS / $TOTAL_REQUESTS)) * 100" | bc)
    
    # Calculate error budget
    ALLOWED_ERROR_RATE=$(echo "scale=4; 100 - $SLO_TARGET" | bc)
    ACTUAL_ERROR_RATE=$(echo "scale=4; ($ERROR_REQUESTS / $TOTAL_REQUESTS) * 100" | bc)
    
    ERROR_BUDGET_REMAINING=$(echo "scale=2; (($ALLOWED_ERROR_RATE - $ACTUAL_ERROR_RATE) / $ALLOWED_ERROR_RATE) * 100" | bc)
    
    echo "=== Error Budget Status for: $SERVICE ==="
    echo "SLO Target:        ${SLO_TARGET}%"
    echo "Actual Availability: ${ACTUAL_AVAILABILITY}%"
    echo "Error Budget Remaining: ${ERROR_BUDGET_REMAINING}%"
    echo "Total Requests:    $TOTAL_REQUESTS"
    echo "Error Requests:    $ERROR_REQUESTS"
    echo ""
    
    if (( $(echo "$ERROR_BUDGET_REMAINING < 0" | bc -l) )); then
        echo "STATUS: ❌ ERROR BUDGET EXHAUSTED - Freeze deployments!"
    elif (( $(echo "$ERROR_BUDGET_REMAINING < 25" | bc -l) )); then
        echo "STATUS: ⚠️  ERROR BUDGET LOW - Slow down deployments"
    else
        echo "STATUS: ✅ Error budget healthy"
    fi
else
    log "No metrics available for: $SERVICE"
fi
EOF
    
    chmod +x "${WORKSHOP_DIR}/sre/automation/error-budget-tracker.sh"
    
    log "SRE automation setup complete"
}

# Run complete workshop
run_complete_workshop() {
    log "Starting: $WORKSHOP_NAME"
    separator
    
    mkdir -p "$WORKSHOP_DIR"
    
    echo "Workshop Agenda:"
    echo "  1. Observability Stack"
    echo "  2. CI/CD Pipeline"
    echo "  3. GitOps Configuration"
    echo "  4. SRE Automation"
    echo ""
    
    setup_cicd_pipeline "https://github.com/myorg/myapp" "registry.example.com"
    
    setup_gitops_config "https://github.com/myorg/platform-config"
    
    setup_sre_automation
    
    echo ""
    success "$WORKSHOP_NAME Complete!"
    separator
    
    echo ""
    echo "Workshop Output:"
    find "$WORKSHOP_DIR" -type f | sort
    
    echo ""
    echo "Next steps:"
    echo "  1. Push code to GitHub"
    echo "  2. Configure cluster secrets"
    echo "  3. Bootstrap Flux CD"
    echo "  4. Test CI/CD pipeline"
    echo "  5. Verify observability stack"
}

# Main
case "${1:-workshop}" in
    observability) setup_observability_stack "${2:-observability}" ;;
    cicd) setup_cicd_pipeline "${2:-}" "${3:-}" ;;
    gitops) setup_gitops_config "${2:-}" ;;
    sre) setup_sre_automation ;;
    workshop) run_complete_workshop ;;
    *)
        echo "Usage: $0 {observability|cicd|gitops|sre|workshop}"
        ;;
esac
```

---

## ขั้นตอนที่ 517: Module 3 Review และ Best Practices

```bash
#!/bin/bash
# module3-review.sh - Module 3 Review และ Best Practices

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
separator() { echo "=========================================="; }

# Run comprehensive checklist
run_production_readiness_check() {
    log "Production Readiness Assessment"
    separator
    
    local passed=0
    local failed=0
    local warnings=0
    
    check_item() {
        local description="${1:-}"
        local cmd="${2:-true}"
        local severity="${3:-required}"  # required, recommended, optional
        
        if eval "$cmd" &>/dev/null 2>&1; then
            echo "  ✅ $description"
            passed=$((passed + 1))
        else
            case "$severity" in
                required)
                    echo "  ❌ $description [REQUIRED]"
                    failed=$((failed + 1))
                    ;;
                recommended)
                    echo "  ⚠️  $description [RECOMMENDED]"
                    warnings=$((warnings + 1))
                    ;;
                optional)
                    echo "  ℹ️  $description [OPTIONAL]"
                    ;;
            esac
        fi
    }
    
    echo ""
    echo "=== Kubernetes Cluster Health ==="
    check_item "Cluster reachable" "kubectl cluster-info" required
    check_item "All nodes ready" "kubectl get nodes | grep -v ' Ready ' | grep -v NAME | wc -l | xargs -I{} test {} -eq 0" required
    check_item "Metrics server running" "kubectl get deployment metrics-server -n kube-system" recommended
    check_item "Cert-manager installed" "kubectl get deployment cert-manager -n cert-manager" recommended
    
    echo ""
    echo "=== GitOps ==="
    check_item "Flux installed" "flux check --pre" required
    check_item "ArgoCD running" "kubectl get deployment argocd-server -n argocd" recommended
    check_item "All Flux reconciliations healthy" "flux get all --all-namespaces | grep -v 'True'" optional
    
    echo ""
    echo "=== Security ==="
    check_item "PSS policies enforced" "kubectl get namespaces -o json | jq '.items[] | .metadata.labels | has(\"pod-security.kubernetes.io/enforce\")' | grep -c true | xargs -I{} test {} -gt 0" required
    check_item "Network policies configured" "kubectl get networkpolicies --all-namespaces | wc -l | xargs -I{} test {} -gt 0" required
    check_item "RBAC configured" "kubectl get clusterrolebindings | wc -l | xargs -I{} test {} -gt 5" required
    check_item "OPA Gatekeeper installed" "kubectl get deployment gatekeeper-controller-manager -n gatekeeper-system" recommended
    check_item "Falco running" "kubectl get deployment falco -n falco" recommended
    
    echo ""
    echo "=== Observability ==="
    check_item "Prometheus running" "kubectl get deployment prometheus -n monitoring 2>/dev/null || kubectl get deployment kube-prometheus-stack-prometheus -n monitoring" required
    check_item "Grafana running" "kubectl get deployment grafana -n monitoring 2>/dev/null || kubectl get deployment kube-prometheus-stack-grafana -n monitoring" required
    check_item "Alertmanager configured" "kubectl get deployment alertmanager -n monitoring 2>/dev/null || kubectl get alertmanager -n monitoring" required
    check_item "Loki for logs" "kubectl get deployment loki -n monitoring 2>/dev/null || kubectl get statefulset loki -n monitoring" recommended
    check_item "Jaeger/Tempo for tracing" "kubectl get deployment jaeger-query -n observability 2>/dev/null || kubectl get deployment tempo-query -n observability" optional
    
    echo ""
    echo "=== Performance ==="
    check_item "Cluster Autoscaler running" "kubectl get deployment cluster-autoscaler -n kube-system" recommended
    check_item "HPA configured for deployments" "kubectl get hpa --all-namespaces | wc -l | xargs -I{} test {} -gt 0" required
    check_item "Resource limits set" "kubectl get pods --all-namespaces -o json | jq '[.items[] | .spec.containers[] | select(.resources.limits == null)] | length' | xargs -I{} test {} -eq 0" required
    check_item "PodDisruptionBudgets configured" "kubectl get pdb --all-namespaces | wc -l | xargs -I{} test {} -gt 0" recommended
    
    echo ""
    echo "=== Storage and Backup ==="
    check_item "StorageClass configured" "kubectl get storageclass | grep default" required
    check_item "Velero backup configured" "kubectl get deployment velero -n velero" recommended
    check_item "Volume snapshots available" "kubectl get volumesnapshotclasses" optional
    
    echo ""
    separator
    echo ""
    echo "Results:"
    echo "  Passed:   $passed"
    echo "  Failed:   $failed (REQUIRED)"
    echo "  Warnings: $warnings (RECOMMENDED)"
    echo ""
    
    if [[ $failed -gt 0 ]]; then
        echo "❌ NOT production ready: $failed required checks failed"
        return 1
    elif [[ $warnings -gt 5 ]]; then
        echo "⚠️  Production ready with gaps: $warnings recommended items missing"
    else
        echo "✅ Production ready!"
    fi
}

# Module 3 Summary
show_module3_summary() {
    log "Module 3: Advanced Level Summary"
    separator
    
    echo ""
    echo "Module 3 ครอบคลุมขั้นตอนที่ 456-517"
    echo ""
    
    echo "Parts และหัวข้อ:"
    cat <<'SUMMARY'
┌──────────┬────────────────────────────────────────────┬──────────────┐
│ Part     │ หัวข้อ                                      │ Steps        │
├──────────┼────────────────────────────────────────────┼──────────────┤
│ Part 26  │ Kubernetes Integration                      │ 456-460      │
│ Part 27  │ Cloud Provider Integration                  │ 461-465      │
│ Part 28  │ CI/CD Pipeline Automation                   │ 466-472      │
│ Part 29  │ Infrastructure as Code                      │ 473-477      │
│ Part 30  │ Advanced Networking & Service Mesh          │ 478-482      │
│ Part 31  │ Observability                               │ 483-486      │
│ Part 32  │ Advanced Security and Compliance            │ 487-489      │
│ Part 33  │ SRE Practices                               │ 490-492      │
│ Part 34  │ GitOps และ Platform Engineering             │ 493-497      │
│ Part 35  │ Multi-cluster Kubernetes Management         │ 498-502      │
│ Part 36  │ Advanced Kubernetes Patterns & Operators    │ 503-505      │
│ Part 37  │ Advanced Container Patterns & Optimization  │ 506-509      │
│ Part 38  │ Advanced Monitoring & Observability         │ 510-512      │
│ Part 39  │ Performance Engineering & Load Testing      │ 513-515      │
│ Part 40  │ Module 3 Completion Workshop               │ 516-517      │
└──────────┴────────────────────────────────────────────┴──────────────┘

เทคโนโลยีหลักที่เรียนใน Module 3:
  Container Orchestration: Kubernetes, Helm, Kustomize
  Cloud Providers: AWS, GCP, Azure
  CI/CD: GitHub Actions, GitLab CI, Jenkins, Argo Workflows
  IaC: Terraform, Pulumi, Ansible, Crossplane
  GitOps: Flux CD, ArgoCD, ApplicationSets
  Service Mesh: Istio, Linkerd
  Networking: Cilium, Multus, Multi-cluster
  Observability: Prometheus, Grafana, Jaeger, Tempo, Loki
  Security: Falco, OPA, AppArmor, Seccomp, Zero Trust
  SRE: SLI/SLO/SLA, Error Budget, Chaos Engineering
  Platform Engineering: Backstage, Internal Developer Platform
  Performance: k6, pprof, py-spy, JFR
  Storage: Rook-Ceph, Velero, PVC Snapshots
SUMMARY
    
    echo ""
    echo "ถัดไป: Module 4 - Professional Level (Parts 41-75)"
    echo ""
    echo "Module 4 จะครอบคลุม:"
    echo "  - Enterprise Kubernetes Patterns"
    echo "  - Multi-tenant Platform Architecture"
    echo "  - Advanced Security Architecture"
    echo "  - Global Load Balancing & CDN"
    echo "  - Database Patterns at Scale"
    echo "  - Event-Driven Architecture"
    echo "  - Machine Learning Infrastructure"
    echo "  - FinOps & Cost Optimization"
}

# Main
case "${1:-summary}" in
    check) run_production_readiness_check ;;
    summary) show_module3_summary ;;
    *)
        echo "Usage: $0 {check|summary}"
        ;;
esac
```

---

## สรุป Module 3: Advanced Level

### ความสำเร็จของ Module 3

Module 3 ครอบคลุม **Steps 456-517** ใน **15 Parts (Part 26-40)** ซึ่งเป็นการเรียนรู้ระดับ Advanced ที่ครอบคลุมทุกด้านของ Modern Platform Engineering

### เทคโนโลยีที่เชี่ยวชาญแล้ว

| หมวดหมู่ | เทคโนโลยี |
|---------|-----------|
| **Container Orchestration** | Kubernetes, Helm, Kustomize, Operators |
| **Cloud Native** | AWS/GCP/Azure CLI, Crossplane |
| **CI/CD** | GitHub Actions, GitLab CI, Jenkins, Argo Rollouts |
| **GitOps** | Flux CD, ArgoCD, ApplicationSets |
| **IaC** | Terraform, Pulumi, Ansible, Packer |
| **Service Mesh** | Istio, Linkerd, Multi-cluster |
| **Networking** | Cilium, eBPF, Multus, CNI |
| **Observability** | Prometheus, Grafana, Jaeger, Tempo, Loki |
| **Security** | Falco, OPA, PSS, Zero Trust |
| **SRE** | SLI/SLO/SLA, Chaos Engineering |
| **Platform** | Backstage, Crossplane, IDP |
| **Performance** | k6, Load Testing, Profiling |
| **Storage** | Rook-Ceph, Velero, Snapshots |

---

## ยินดีด้วย! Module 3 เสร็จสมบูรณ์

ขั้นตอนต่อไป: **Module 4 - Professional Level** (Part 41-75, Steps 518+)

หัวข้อที่จะเรียนใน Module 4:
- Enterprise Architecture Patterns
- Multi-tenant SaaS Platform
- Advanced Security & Compliance
- Global Infrastructure Design
- Cost Optimization (FinOps)
- ML/AI Infrastructure
- Database Architecture at Scale
- Event-Driven Systems
