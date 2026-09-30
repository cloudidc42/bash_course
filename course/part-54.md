# Part 54: Module 4 Completion — API Gateway Enterprise และ Service Mesh Mastery (ขั้นตอนที่ 565-568)

---

## ขั้นตอนที่ 565: Enterprise API Gateway — Kong และ AWS API Gateway

```bash
#!/bin/bash
# enterprise-api-gateway.sh
# Enterprise API Gateway: Kong, AWS API Gateway, Rate Limiting, Auth, Plugins

set -euo pipefail

KONG_NAMESPACE="${KONG_NAMESPACE:-kong}"
KONG_VERSION="${KONG_VERSION:-3.5}"
AWS_REGION="${AWS_REGION:-us-east-1}"
DOMAIN="${DOMAIN:-api.company.com}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Kong Gateway Installation ====================
install_kong_enterprise() {
    log "Installing Kong Gateway..."
    
    helm repo add kong https://charts.konghq.com
    helm repo update
    
    cat <<EOF > /tmp/kong-values.yaml
image:
  repository: kong/kong-gateway
  tag: "${KONG_VERSION}"

env:
  database: postgres
  pg_host: postgresql.databases.svc.cluster.local
  pg_port: 5432
  pg_database: kong
  pg_user: kong
  pg_password:
    valueFrom:
      secretKeyRef:
        name: kong-postgres-password
        key: password
  
  # Plugins
  plugins: bundled
  
  # Rate limiting storage
  nginx_worker_processes: "4"
  
  # Logs
  log_level: notice
  
  # Admin API
  admin_listen: "0.0.0.0:8001, 0.0.0.0:8444 ssl"
  proxy_listen: "0.0.0.0:8000, 0.0.0.0:8443 ssl"

proxy:
  enabled: true
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  http:
    enabled: true
    servicePort: 80
    containerPort: 8000
  tls:
    enabled: true
    servicePort: 443
    containerPort: 8443

admin:
  enabled: true
  type: ClusterIP
  http:
    enabled: true
    servicePort: 8001
    containerPort: 8001

ingressController:
  enabled: true
  installCRDs: true

manager:
  enabled: true

replicaCount: 3

resources:
  requests:
    cpu: 1000m
    memory: 2Gi
  limits:
    cpu: 4000m
    memory: 8Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 2
EOF

    helm upgrade --install kong kong/kong \
        --namespace "${KONG_NAMESPACE}" \
        --create-namespace \
        -f /tmp/kong-values.yaml \
        --wait
    
    log "Kong installed"
}

# ==================== Kong Configuration via KongIngress ====================
configure_kong_routes() {
    log "Configuring Kong routes and plugins..."
    
    # KongPlugin: JWT Authentication
    cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: jwt-auth
  namespace: production
plugin: jwt
config:
  claims_to_verify:
  - exp
  - nbf
  header_names:
  - Authorization
  uri_param_names:
  - jwt
  secret_is_base64: false
  run_on_preflight: true
---
# Rate limiting plugin
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limit-basic
  namespace: production
plugin: rate-limiting
config:
  minute: 1000
  hour: 50000
  day: 1000000
  policy: redis
  redis_host: redis-master.databases.svc.cluster.local
  redis_port: 6379
  redis_password: "${REDIS_PASSWORD}"
  fault_tolerant: true
  hide_client_headers: false
  error_code: 429
  error_message: "Too many requests, please slow down"
---
# Response transformer
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: add-security-headers
  namespace: production
plugin: response-transformer
config:
  add:
    headers:
    - "X-Content-Type-Options:nosniff"
    - "X-Frame-Options:DENY"
    - "X-XSS-Protection:1; mode=block"
    - "Strict-Transport-Security:max-age=31536000; includeSubDomains"
    - "Content-Security-Policy:default-src 'self'"
---
# CORS plugin
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: cors-policy
  namespace: production
plugin: cors
config:
  origins:
  - https://app.company.com
  - https://admin.company.com
  methods:
  - GET
  - POST
  - PUT
  - DELETE
  - OPTIONS
  headers:
  - Authorization
  - Content-Type
  - X-Request-ID
  exposed_headers:
  - X-Auth-Token
  - X-RateLimit-Remaining-Minute
  credentials: true
  max_age: 3600
EOF

    # KongIngress for API versioning
    cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-v1
  namespace: production
  annotations:
    kubernetes.io/ingress.class: kong
    konghq.com/plugins: jwt-auth,rate-limit-basic,add-security-headers,cors-policy
    konghq.com/strip-path: "true"
    konghq.com/preserve-host: "true"
spec:
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-gateway-v1
            port:
              number: 8080
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-gateway-v2
            port:
              number: 8080
EOF

    log "Kong routes configured"
}

# ==================== AWS API Gateway Configuration ====================
setup_aws_api_gateway() {
    log "Setting up AWS API Gateway (HTTP API)..."
    
    # Create HTTP API
    local API_ID=$(aws apigatewayv2 create-api \
        --name "company-api" \
        --protocol-type HTTP \
        --cors-configuration '{
            "AllowHeaders": ["*"],
            "AllowMethods": ["*"],
            "AllowOrigins": ["https://app.company.com"],
            "ExposeHeaders": ["X-RateLimit-Remaining"],
            "MaxAge": 86400
        }' \
        --query "ApiId" --output text)
    
    log "API created: ${API_ID}"
    
    # Create JWT authorizer
    local AUTH_ID=$(aws apigatewayv2 create-authorizer \
        --api-id "${API_ID}" \
        --authorizer-type JWT \
        --identity-source '$request.header.Authorization' \
        --name "CognitoAuth" \
        --jwt-configuration '{
            "Audience": ["company-api"],
            "Issuer": "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_xxxxx"
        }' \
        --query "AuthorizerId" --output text)
    
    # Create VPC Link for private EKS
    local SECURITY_GROUP=$(aws ec2 describe-security-groups \
        --filters "Name=tag:Name,Values=eks-api-gateway-sg" \
        --query "SecurityGroups[0].GroupId" --output text)
    
    local SUBNET_IDS=$(aws ec2 describe-subnets \
        --filters "Name=tag:kubernetes.io/role/internal-elb,Values=1" \
        --query "Subnets[*].SubnetId" --output json | python3 -c "import json,sys; print(','.join(json.load(sys.stdin)))")
    
    local VPC_LINK_ID=$(aws apigatewayv2 create-vpc-link \
        --name "eks-vpc-link" \
        --subnet-ids $(echo ${SUBNET_IDS} | tr ',' ' ') \
        --security-group-ids "${SECURITY_GROUP}" \
        --query "VpcLinkId" --output text)
    
    # Wait for VPC link
    aws apigatewayv2 wait vpc-link-available \
        --vpc-link-id "${VPC_LINK_ID}" \
        --region "${AWS_REGION}" 2>/dev/null || sleep 60
    
    # Create integration (ALB)
    local INTEGRATION_ID=$(aws apigatewayv2 create-integration \
        --api-id "${API_ID}" \
        --integration-type HTTP_PROXY \
        --integration-method ANY \
        --integration-uri "http://k8s-production-internalelb.${AWS_REGION}.elb.amazonaws.com/{proxy}" \
        --connection-type VPC_LINK \
        --connection-id "${VPC_LINK_ID}" \
        --payload-format-version "1.0" \
        --query "IntegrationId" --output text)
    
    # Create routes
    aws apigatewayv2 create-route \
        --api-id "${API_ID}" \
        --route-key "ANY /api/{proxy+}" \
        --authorization-type JWT \
        --authorizer-id "${AUTH_ID}" \
        --target "integrations/${INTEGRATION_ID}"
    
    # Create stage
    aws apigatewayv2 create-stage \
        --api-id "${API_ID}" \
        --stage-name production \
        --auto-deploy \
        --default-route-settings '{
            "ThrottlingBurstLimit": 5000,
            "ThrottlingRateLimit": 10000
        }' \
        --stage-variables '{
            "env": "production"
        }'
    
    # Custom domain
    aws apigatewayv2 create-domain-name \
        --domain-name "${DOMAIN}" \
        --domain-name-configurations \
            "CertificateArn=arn:aws:acm:${AWS_REGION}:$(aws sts get-caller-identity --query Account --output text):certificate/xxxx,EndpointType=REGIONAL" \
        2>/dev/null || true
    
    log "AWS API Gateway configured: ${API_ID}"
}

# ==================== Kong Deck (Declarative Config) ====================
manage_kong_declarative() {
    log "Managing Kong with deck (declarative config)..."
    
    # Install deck
    curl -sL https://github.com/kong/deck/releases/download/v1.28.0/deck_1.28.0_linux_amd64.tar.gz | \
        tar -xz -C /usr/local/bin deck 2>/dev/null || true
    
    cat <<'EOF' > /tmp/kong-deck-config.yaml
_format_version: "3.0"

services:
- name: api-gateway
  url: http://api-gateway.production.svc.cluster.local:8080
  tags:
  - production
  routes:
  - name: api-v1-routes
    paths:
    - /v1
    methods:
    - GET
    - POST
    - PUT
    - DELETE
    strip_path: true
    preserve_host: true
  plugins:
  - name: rate-limiting
    config:
      minute: 1000
      policy: redis
      redis_host: redis-master.databases.svc.cluster.local

- name: admin-api
  url: http://admin-service.production.svc.cluster.local:8080
  tags:
  - production
  - admin
  routes:
  - name: admin-routes
    paths:
    - /admin
    methods:
    - GET
    - POST
    - PUT
    - DELETE
  plugins:
  - name: jwt
    config:
      claims_to_verify:
      - exp
  - name: ip-restriction
    config:
      allow:
      - 10.0.0.0/8
      - 172.16.0.0/12
      deny: []

plugins:
- name: prometheus
  config:
    per_consumer: true
    bandwidth_metrics: true
    latency_metrics: true
    status_code_metrics: true
    upstream_health_metrics: true

consumers:
- username: mobile-app
  custom_id: mobile-v1
  plugins:
  - name: rate-limiting
    config:
      minute: 100
      hour: 10000

upstreams:
- name: api-gateway.production.svc.cluster.local
  algorithm: least-connections
  healthchecks:
    active:
      healthy:
        interval: 10
        successes: 2
      unhealthy:
        interval: 5
        http_failures: 3
    passive:
      healthy:
        successes: 5
      unhealthy:
        http_failures: 5
        timeouts: 5
  targets:
  - target: api-gateway-0.api-gateway.production.svc.cluster.local:8080
    weight: 100
  - target: api-gateway-1.api-gateway.production.svc.cluster.local:8080
    weight: 100
  - target: api-gateway-2.api-gateway.production.svc.cluster.local:8080
    weight: 100
EOF

    # Apply config
    deck sync -s /tmp/kong-deck-config.yaml \
        --kong-addr "http://kong-admin.${KONG_NAMESPACE}:8001" \
        2>/dev/null || echo "deck sync requires Kong to be running"
    
    log "Kong declarative config applied"
}

main() {
    case "${1:-all}" in
        install)    install_kong_enterprise ;;
        routes)     configure_kong_routes ;;
        aws)        setup_aws_api_gateway ;;
        deck)       manage_kong_declarative ;;
        all)
            install_kong_enterprise
            configure_kong_routes
            manage_kong_declarative
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 566: Advanced Container Security — Runtime Protection

```bash
#!/bin/bash
# container-runtime-security.sh
# Advanced Container Security: Falco, Trivy Operator, NeuVector, Image Signing

set -euo pipefail

NAMESPACE="${NAMESPACE:-security}"
FALCO_VERSION="${FALCO_VERSION:-0.36.0}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== Falco Advanced Rules ====================
setup_falco_enterprise() {
    log "Deploying Falco with enterprise rules..."
    
    helm repo add falcosecurity https://falcosecurity.github.io/charts
    
    cat <<'EOF' > /tmp/falco-values.yaml
falco:
  jsonOutput: true
  jsonIncludeOutputProperty: true
  logLevel: info
  
  grpc:
    enabled: true
    bindAddress: "unix:///run/falco/falco.sock"
  
  grpcOutput:
    enabled: true
  
  webserver:
    enabled: true
    listenPort: 8765
  
  rulesFile:
  - /etc/falco/falco_rules.yaml
  - /etc/falco/falco_rules.local.yaml
  - /etc/falco/k8s_audit_rules.yaml
  - /etc/falco/custom_rules.yaml

customRules:
  custom_rules.yaml: |
    # Detect cryptocurrency mining
    - rule: Detected Cryptomining Activity
      desc: Detect cryptomining processes
      condition: >
        spawned_process and
        (proc.name in (xmrig, minerd, cpuminer, cgminer, bfgminer) or
         proc.cmdline contains "stratum+tcp" or
         proc.cmdline contains "pool.minergate.com")
      output: >
        Cryptomining detected (proc=%proc.name user=%user.name
        cmd=%proc.cmdline container=%container.name)
      priority: CRITICAL
      tags: [security, cryptomining]
    
    # Detect sensitive file access
    - rule: Sensitive File Access
      desc: Detect access to sensitive files
      condition: >
        open_read and
        fd.name pmatch (/etc/shadow, /etc/gshadow, /etc/sudoers,
                        /proc/*/mem, /root/.ssh/*, /home/*/.ssh/*)
        and not proc.name in (sshd, sudo, su, passwd)
        and container.id != host
      output: >
        Sensitive file read (file=%fd.name proc=%proc.name user=%user.name
        container=%container.name ns=%k8s.ns.name pod=%k8s.pod.name)
      priority: WARNING
      tags: [security, file_access]
    
    # Detect container escape
    - rule: Container Namespace Escape
      desc: Detect attempts to escape container namespace
      condition: >
        spawned_process and
        (proc.name in (nsenter, unshare) or
         (proc.name = docker and proc.args contains "--privileged") or
         proc.cmdline contains "/proc/1/ns")
        and container.id != host
      output: >
        Container escape attempt (proc=%proc.name args=%proc.args
        container=%container.name user=%user.name)
      priority: CRITICAL
      tags: [security, container_escape]
    
    # Detect lateral movement
    - rule: Kubectl Exec Suspicious Activity
      desc: Detect suspicious kubectl exec usage
      condition: >
        ka.verb = exec and
        ka.target.resource = pods and
        ka.uri.param[command] contains /bin/sh or
        ka.uri.param[command] contains /bin/bash
      output: >
        Kubectl exec into pod (user=%ka.user.name pod=%ka.target.name
        ns=%ka.target.namespace command=%ka.uri.param[command])
      priority: WARNING
      tags: [security, audit, lateral_movement]
    
    # Detect privilege escalation
    - rule: Privilege Escalation via setuid
      desc: Detect setuid execution
      condition: >
        spawned_process and
        proc.is_suid_exe = true and
        proc.name != sudo and
        container.id != host
      output: >
        Setuid program executed in container
        (proc=%proc.name parent=%proc.pname user=%user.name
        container=%container.name)
      priority: NOTICE
      tags: [security, privilege_escalation]
    
    # Data exfiltration
    - rule: Large Data Transfer
      desc: Detect large outbound data transfers
      condition: >
        (evt.type = sendto or evt.type = sendmsg) and
        fd.sip.name != "" and
        not fd.sip.name in (cluster_cidr) and
        evt.buflen > 10485760  # 10MB
      output: >
        Large data transfer detected (dest=%fd.sip.name size=%evt.buflen
        proc=%proc.name container=%container.name)
      priority: WARNING
      tags: [security, data_exfiltration]

falcoctl:
  artifact:
    follow:
      enabled: true

driver:
  kind: ebpf

syscalls:
  slow_syscalls:
    enabled: false

metrics:
  enabled: true
  interval: 15s
  output_rule: false
  resource_utilization_enabled: true
  state_counters_enabled: true
  kernel_event_counters_enabled: true
  libbpf_stats_enabled: true
EOF

    helm upgrade --install falco falcosecurity/falco \
        --namespace "${NAMESPACE}" \
        --create-namespace \
        -f /tmp/falco-values.yaml \
        --wait
    
    log "Falco enterprise deployed"
}

# ==================== Trivy Operator ====================
setup_trivy_operator() {
    log "Deploying Trivy Operator for continuous scanning..."
    
    helm repo add aqua https://aquasecurity.github.io/helm-charts/
    
    helm upgrade --install trivy-operator aqua/trivy-operator \
        --namespace "${NAMESPACE}" \
        --create-namespace \
        --set="trivy.ignoreUnfixed=true" \
        --set="trivy.severity=CRITICAL,HIGH" \
        --set="operator.scannerReportTTL=24h" \
        --set="compliance.failEntriesLimit=10" \
        --wait
    
    # Create ClusterComplianceReport for CIS benchmarks
    cat <<'EOF' | kubectl apply -f -
apiVersion: aquasecurity.github.io/v1alpha1
kind: ClusterComplianceReport
metadata:
  name: cis-k8s-benchmark
spec:
  compliance:
    id: cis-k8s
    title: CIS Kubernetes Benchmark
    description: CIS Kubernetes Benchmark v1.8
    relatedResources:
    - https://www.cisecurity.org/benchmark/kubernetes
    version: "1.0"
    controls:
    - id: "1.1.1"
      name: Ensure API server pod specification file permissions are set to 600 or more restrictive
      severity: HIGH
      defaultStatus: PASS
      checks:
      - id: AVD-KCV-0001
    - id: "5.1.1"
      name: Ensure that the cluster-admin role is only used where required
      severity: HIGH
      defaultStatus: FAIL
      checks:
      - id: AVD-KCV-0074
EOF

    log "Trivy Operator configured"
}

# ==================== Image Signing Verification ====================
setup_image_verification() {
    log "Setting up image signing verification pipeline..."
    
    # Cosign key generation
    cosign generate-key-pair k8s://"${NAMESPACE}"/cosign-keypair 2>/dev/null || \
        log "WARN" "cosign not installed, skipping key generation"
    
    # Kyverno ClusterPolicy for image verification
    cat <<'EOF' | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
  annotations:
    policies.kyverno.io/title: Verify Image Signatures
    policies.kyverno.io/category: Software Supply Chain Security
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: Enforce
  background: false
  webhookTimeoutSeconds: 30
  rules:
  - name: verify-production-images
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - production
    verifyImages:
    - imageReferences:
      - "123456789012.dkr.ecr.us-east-1.amazonaws.com/*"
      attestors:
      - count: 1
        entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
              -----END PUBLIC KEY-----
            signatureAlgorithm: sha256
            rekor:
              url: https://rekor.sigstore.dev
      attestations:
      - predicateType: https://slsa.dev/provenance/v0.2
        conditions:
        - all:
          - key: "{{ predicate.builder.id }}"
            operator: Equals
            value: "https://github.com/myorg/myapp/.github/workflows/ci.yml@refs/heads/main"
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-latest-image-tag
spec:
  validationFailureAction: Enforce
  rules:
  - name: no-latest-tag
    match:
      any:
      - resources:
          kinds: [Pod]
          namespaces: [production, staging]
    validate:
      message: "Image tag 'latest' is not allowed in production/staging"
      foreach:
      - list: "request.object.spec.containers"
        deny:
          conditions:
            any:
            - key: "{{element.image}}"
              operator: Equals
              value: "*:latest"
            - key: "{{element.image}}"
              operator: NotEquals
              value: "*:*"
EOF

    log "Image verification policies applied"
}

# ==================== Security Benchmarking ====================
run_security_benchmark() {
    log "Running CIS security benchmark..."
    
    # kube-bench
    if command -v kube-bench &>/dev/null; then
        kube-bench --json > /tmp/cis-benchmark.json
    else
        cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
  namespace: security
spec:
  template:
    spec:
      hostPID: true
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: kube-bench
        image: aquasec/kube-bench:latest
        command: [kube-bench, run, --json]
        volumeMounts:
        - name: var-lib-cni
          mountPath: /var/lib/cni
          readOnly: true
        - name: etc-systemd
          mountPath: /etc/systemd
          readOnly: true
        - name: etc-kubernetes
          mountPath: /etc/kubernetes
          readOnly: true
      restartPolicy: Never
      volumes:
      - name: var-lib-cni
        hostPath:
          path: /var/lib/cni
      - name: etc-systemd
        hostPath:
          path: /etc/systemd
      - name: etc-kubernetes
        hostPath:
          path: /etc/kubernetes
EOF
    fi
    
    log "Security benchmark complete"
}

main() {
    case "${1:-all}" in
        falco)     setup_falco_enterprise ;;
        trivy)     setup_trivy_operator ;;
        signing)   setup_image_verification ;;
        benchmark) run_security_benchmark ;;
        all)
            setup_falco_enterprise
            setup_trivy_operator
            setup_image_verification
            run_security_benchmark
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 567: Database Performance Tuning

```bash
#!/bin/bash
# database-performance-tuning.sh
# Database Performance: PostgreSQL tuning, Query optimization, Connection pooling

set -euo pipefail

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== PostgreSQL Performance Tuning ====================
tune_postgresql() {
    log "Tuning PostgreSQL performance..."
    
    # Performance tuning ConfigMap for CloudNativePG
    cat <<'EOF' | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: production-postgresql
  namespace: databases
spec:
  instances: 3
  
  postgresql:
    parameters:
      # Memory
      shared_buffers: "4GB"                   # 25% of RAM
      effective_cache_size: "12GB"            # 75% of RAM
      work_mem: "64MB"                        # Per operation
      maintenance_work_mem: "2GB"             # Vacuum, etc.
      max_wal_size: "4GB"
      
      # Connections
      max_connections: "200"
      
      # Checkpointing
      checkpoint_completion_target: "0.9"
      checkpoint_warning: "30s"
      
      # WAL
      wal_level: "replica"
      wal_compression: "on"
      wal_buffers: "64MB"
      
      # Query planner
      random_page_cost: "1.1"               # SSD
      effective_io_concurrency: "200"       # SSD
      
      # Parallelism
      max_parallel_workers_per_gather: "4"
      max_parallel_workers: "16"
      max_parallel_maintenance_workers: "4"
      
      # Autovacuum
      autovacuum_vacuum_scale_factor: "0.02"
      autovacuum_analyze_scale_factor: "0.01"
      autovacuum_max_workers: "6"
      autovacuum_vacuum_cost_delay: "2ms"
      
      # Logging
      log_min_duration_statement: "1000"    # Log queries > 1s
      log_checkpoints: "on"
      log_lock_waits: "on"
      log_temp_files: "0"
      
      # Statistics
      track_io_timing: "on"
      track_wal_io_timing: "on"
      
      # pg_stat_statements
      shared_preload_libraries: "pg_stat_statements,pg_prewarm,auto_explain"
      pg_stat_statements.track: "all"
      auto_explain.log_min_duration: "2000"
      auto_explain.log_analyze: "true"
      auto_explain.log_buffers: "true"
      auto_explain.log_nested_statements: "true"
EOF

    log "PostgreSQL tuning applied"
}

# ==================== Query Analysis ====================
analyze_slow_queries() {
    log "Analyzing slow queries..."
    
    python3 - <<'EOF'
"""PostgreSQL slow query analysis script"""
import subprocess
import json

QUERIES = {
    "top_slow_queries": """
        SELECT 
            query,
            calls,
            total_exec_time::numeric(10,2) as total_ms,
            mean_exec_time::numeric(10,2) as avg_ms,
            max_exec_time::numeric(10,2) as max_ms,
            stddev_exec_time::numeric(10,2) as stddev_ms,
            rows,
            shared_blks_hit,
            shared_blks_read,
            shared_blks_hit::float / NULLIF(shared_blks_hit + shared_blks_read, 0) * 100 as cache_hit_pct
        FROM pg_stat_statements
        WHERE query NOT LIKE '%pg_stat%'
        ORDER BY mean_exec_time DESC
        LIMIT 20;
    """,
    
    "missing_indexes": """
        SELECT 
            schemaname,
            tablename,
            seq_scan,
            seq_tup_read,
            idx_scan,
            seq_tup_read / NULLIF(seq_scan, 0) as avg_rows_per_seq_scan
        FROM pg_stat_user_tables
        WHERE seq_scan > idx_scan
          AND seq_scan > 100
          AND seq_tup_read / NULLIF(seq_scan, 0) > 1000
        ORDER BY seq_tup_read DESC;
    """,
    
    "table_bloat": """
        SELECT
            schemaname,
            tablename,
            pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
            pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
            (n_dead_tup::float / NULLIF(n_live_tup + n_dead_tup, 0) * 100)::numeric(5,2) AS dead_tuple_pct,
            last_vacuum,
            last_autovacuum,
            last_analyze
        FROM pg_stat_user_tables
        WHERE n_dead_tup > 10000
        ORDER BY dead_tuple_pct DESC;
    """,
    
    "lock_waits": """
        SELECT
            waiting.pid as waiting_pid,
            waiting.query as waiting_query,
            blocking.pid as blocking_pid,
            blocking.query as blocking_query,
            extract(epoch from now() - waiting.query_start) as wait_duration_seconds
        FROM pg_stat_activity waiting
        JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(waiting.pid))
        WHERE waiting.wait_event_type = 'Lock'
        ORDER BY wait_duration_seconds DESC;
    """,
    
    "index_usage": """
        SELECT
            schemaname,
            tablename,
            indexname,
            idx_scan as index_scans,
            idx_tup_read,
            idx_tup_fetch,
            pg_size_pretty(pg_relation_size(indexrelid)) as index_size
        FROM pg_stat_user_indexes
        WHERE idx_scan < 50 AND schemaname != 'pg_catalog'
        ORDER BY pg_relation_size(indexrelid) DESC;
    """
}

for query_name, sql in QUERIES.items():
    print(f"\n=== {query_name.upper()} ===")
    print("SQL Query:")
    print(sql[:200] + "..." if len(sql) > 200 else sql)
    print("(Run against your PostgreSQL instance to see results)")
EOF
}

# ==================== PgBouncer Advanced Config ====================
setup_pgbouncer_advanced() {
    log "Configuring PgBouncer for connection pooling..."
    
    cat <<'EOF' | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: production-pooler
  namespace: databases
spec:
  cluster:
    name: production-postgresql
  
  instances: 3
  type: rw
  
  pgbouncer:
    poolMode: transaction
    parameters:
      max_client_conn: "2000"
      default_pool_size: "50"
      min_pool_size: "10"
      reserve_pool_size: "5"
      reserve_pool_timeout: "5"
      max_db_connections: "200"
      max_user_connections: "100"
      server_lifetime: "3600"
      server_idle_timeout: "600"
      server_connect_timeout: "15"
      server_login_retry: "15"
      client_idle_timeout: "0"
      client_login_timeout: "60"
      autodb_idle_timeout: "3600"
      
      # Query cancel support
      cancel_wait_timeout: "10"
      
      # Logging
      log_connections: "0"
      log_disconnections: "0"
      log_pooler_errors: "1"
      
      # Auth
      auth_type: "scram-sha-256"
      
      # Admin
      admin_users: "postgres"
      stats_users: "monitoring"
      
      # DNS
      dns_max_ttl: "15"
      dns_nxdomain_ttl: "15"
  
  template:
    metadata:
      labels:
        app: pgbouncer
    spec:
      resources:
        requests:
          cpu: 500m
          memory: 512Mi
        limits:
          cpu: 2000m
          memory: 2Gi
EOF

    log "PgBouncer configured"
}

# ==================== Redis Performance ====================
tune_redis() {
    log "Tuning Redis performance..."
    
    # Redis config for production
    cat <<'EOF' > /tmp/redis-optimized.conf
# Memory
maxmemory 16gb
maxmemory-policy allkeys-lru
maxmemory-samples 10

# Persistence - optimize for performance
save ""
appendonly yes
appendfsync everysec
no-appendfsync-on-rewrite yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-rewrite-incremental-fsync yes

# Network
tcp-backlog 511
timeout 300
tcp-keepalive 300
bind-source-addr ""

# Lazy operations (non-blocking)
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
lazyfree-lazy-user-del yes
lazyfree-lazy-user-flush yes

# Threading
io-threads 4
io-threads-do-reads yes

# Keyspace notifications (for cache-aside patterns)
notify-keyspace-events "Ex"

# Slow log
slowlog-log-slower-than 10000
slowlog-max-len 1000

# Latency monitoring
latency-monitor-threshold 100

# JIT compilation (Redis 7+)
enable-debug-command local

# Cluster
cluster-node-timeout 15000
cluster-migration-barrier 1
cluster-require-full-coverage no
EOF

    kubectl create configmap redis-optimized-config \
        --from-file=redis.conf=/tmp/redis-optimized.conf \
        --namespace databases \
        --dry-run=client -o yaml | kubectl apply -f -
    
    log "Redis tuning config created"
}

main() {
    case "${1:-all}" in
        postgres)    tune_postgresql ;;
        slow-query)  analyze_slow_queries ;;
        pgbouncer)   setup_pgbouncer_advanced ;;
        redis)       tune_redis ;;
        all)
            tune_postgresql
            analyze_slow_queries
            setup_pgbouncer_advanced
            tune_redis
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 568: Module 4 Completion Review และ Best Practices

```bash
#!/bin/bash
# module4-completion.sh
# Module 4 Completion: Review, Validation, Documentation

set -euo pipefail

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

validate_module4_deployment() {
    log "Validating Module 4 deployment completeness..."
    
    local PASS=0
    local WARN=0
    local FAIL=0
    
    declare -A CHECKS=(
        ["K8s Cluster Health"]="kubectl get nodes --no-headers | grep -c Ready"
        ["ArgoCD Running"]="kubectl get deployment argocd-server -n argocd -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo 0"
        ["Prometheus Running"]="kubectl get statefulset prometheus-kube-prometheus-prometheus -n monitoring -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo 0"
        ["Loki Running"]="kubectl get deployment loki-write -n observability -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo 0"
        ["Cert Manager"]="kubectl get deployment cert-manager -n cert-manager -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo 0"
        ["Falco Security"]="kubectl get daemonset falco -n security -o jsonpath='{.status.numberReady}' 2>/dev/null || echo 0"
        ["Istio Service Mesh"]="kubectl get deployment istiod -n istio-system -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo 0"
        ["Kong API Gateway"]="kubectl get deployment kong-kong -n kong -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo 0"
    )
    
    echo ""
    echo "╔══════════════════════════════════════════════════════════════╗"
    echo "║           MODULE 4 VALIDATION REPORT                         ║"
    echo "╚══════════════════════════════════════════════════════════════╝"
    echo ""
    
    for CHECK_NAME in "${!CHECKS[@]}"; do
        local RESULT=$(eval "${CHECKS[$CHECK_NAME]}" 2>/dev/null || echo "0")
        
        if [[ "${RESULT}" -gt 0 ]] 2>/dev/null; then
            echo "  ✅ PASS: ${CHECK_NAME} (${RESULT} replicas)"
            ((PASS++))
        else
            echo "  ❌ FAIL: ${CHECK_NAME}"
            ((FAIL++))
        fi
    done
    
    echo ""
    echo "══════════════════════════════════════════════════════════════"
    echo "  RESULT: PASS=${PASS} / FAIL=${FAIL}"
    echo ""
    
    if [[ ${FAIL} -eq 0 ]]; then
        echo "  🎉 Module 4 deployment COMPLETE!"
    else
        echo "  ⚠️  ${FAIL} components need attention"
    fi
    echo "══════════════════════════════════════════════════════════════"
}

print_module4_summary() {
    cat <<'EOF'

╔══════════════════════════════════════════════════════════════════════╗
║                MODULE 4: PROFESSIONAL LEVEL COMPLETE                  ║
║                     Steps 518 - 568                                   ║
╚══════════════════════════════════════════════════════════════════════╝

📦 Topics Covered (51 Steps):

  Part 41: Enterprise K8s Architecture (518-520)
  ├── Multi-tenant platform management
  ├── Enterprise RBAC with namespaced Roles
  └── CIS K8s hardening: PSA, NetworkPolicy, audit

  Part 42: Data Platform Engineering (521-525)
  ├── Apache Kafka on Kubernetes (Strimzi)
  ├── Apache Flink (stream processing)
  ├── Delta Lake (Spark on K8s)
  ├── ClickHouse (analytics DB)
  └── Apache Airflow (orchestration)

  Part 43: FinOps & Cost Engineering (526-529)
  ├── AWS Cost Explorer automation
  ├── Kubecost for K8s cost visibility
  ├── Karpenter + Spot autoscaling
  └── Tagging governance & SCP

  Part 44: ML/AI Infrastructure (530-533)
  ├── MLflow (experiment tracking)
  ├── Kubeflow Pipelines (KFP)
  ├── GPU Operator + PyTorchJob
  └── NVIDIA Triton Inference Server

  Part 45: Event-Driven Architecture (534-537)
  ├── AWS EventBridge (rules, archives, Schema Registry)
  ├── RabbitMQ Enterprise (Cluster Operator)
  ├── NATS JetStream
  └── SAGA pattern with compensating transactions

  Part 46: Database Engineering at Scale (538-540)
  ├── CloudNativePG + Barman backup
  ├── MongoDB Community Operator
  └── Redis Cluster + advanced patterns

  Part 47: Global Load Balancing (541-543)
  ├── AWS Global Accelerator
  ├── CloudFront + WAF + Route53
  ├── Nginx Ingress (ModSecurity, canary)
  └── Istio advanced traffic management

  Part 48: Enterprise Security Architecture (544-545)
  ├── Zero Trust (Vault PKI, OPA Gatekeeper, Falco)
  └── SIEM (OpenSearch + K8s audit)

  Part 49: Advanced CI/CD & DevSecOps (546-548)
  ├── Jenkins Enterprise (JCasC, Kubernetes cloud)
  ├── GitHub Actions (reusable workflows, SBOM)
  └── SLSA Level 3 + Cosign + supply chain

  Part 50: Networking & Platform Engineering (549-552)
  ├── Cilium CNI (eBPF, BGP, ClusterMesh)
  ├── Multi-Region DR (Route53, RDS, Velero)
  ├── Backstage Developer Portal
  └── Crossplane (cloud infrastructure as K8s CRDs)

  Part 51: SRE & Incident Management (553-556)
  ├── SLO/Error Budget (Pyrra, multi-window alerts)
  ├── Incident lifecycle + automated runbooks
  ├── Chaos Engineering (Chaos Mesh, Game Day)
  └── Capacity planning + k6 load testing

  Part 52: GitOps & Progressive Delivery (557-560)
  ├── ArgoCD HA + ApplicationSets + Notifications
  ├── Argo Rollouts (Canary + AnalysisTemplate)
  ├── Flux CD + Image Automation
  └── GitOps repo structure + OPA validation

  Part 53: Advanced Observability & AIOps (561-564)
  ├── Thanos long-term metrics storage
  ├── Loki Enterprise distributed
  ├── AIOps (Isolation Forest, RCA engine, KEDA)
  └── OpenTelemetry (Operator, eBPF Beyla, tail-sampling)

  Part 54: API Gateway & Security (565-568)
  ├── Kong Enterprise + AWS API Gateway
  ├── Falco advanced rules + Trivy Operator
  ├── Image signing (Cosign + Kyverno)
  └── Database performance tuning

══════════════════════════════════════════════════════════════════════

🚀 NEXT: Module 5 — World-class Level (Steps 569+)
   Parts 55-100+: Platform Engineering, WebAssembly, FinTech,
   Quantum-ready Security, AI/LLM Infrastructure, and beyond

EOF
}

main() {
    case "${1:-all}" in
        validate) validate_module4_deployment ;;
        summary)  print_module4_summary ;;
        all)
            validate_module4_deployment
            print_module4_summary
            ;;
    esac
}

main "$@"
```

---

## สรุป Part 54

- ✅ **Step 565**: Enterprise API Gateway — Kong HA (deck declarative), AWS API Gateway HTTP API + JWT authorizer + VPC Link, KongPlugins (JWT, rate-limit, CORS, security headers)
- ✅ **Step 566**: Container Runtime Security — Falco enterprise rules (cryptomining/escape/exfiltration), Trivy Operator + CIS compliance, Cosign + Kyverno image signing
- ✅ **Step 567**: Database Performance — PostgreSQL tuning (24 parameters), slow query analysis, PgBouncer transaction pooling, Redis optimized config
- ✅ **Step 568**: Module 4 Completion — Validation script, comprehensive summary of all 51 steps covered

**🎉 Module 4 Professional Level COMPLETE (Steps 518-568)**
**ขั้นตอนต่อไป: Part 55 - Module 5 World-class Level เริ่มต้น**
