# Part 48: Enterprise Security Architecture

## Module 4: Professional Level — Security Engineering

### ขั้นตอนที่ 544: Zero Trust Security Implementation

**Zero Trust** architecture สำหรับ enterprise cloud environments

```bash
#!/bin/bash
# zero-trust-architect.sh - Zero Trust Security Architecture

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

setup_vault_pki() {
    log "Setting up HashiCorp Vault PKI for Zero Trust..."
    
    local vault_addr="${VAULT_ADDR:-https://vault.company.com}"
    local vault_token="${VAULT_TOKEN:-}"
    
    export VAULT_ADDR="${vault_addr}"
    export VAULT_TOKEN="${vault_token}"
    
    vault secrets enable -path=pki pki 2>/dev/null || true
    vault secrets tune -max-lease-ttl=87600h pki
    
    vault write -field=certificate pki/root/generate/internal \
        common_name="company.com Root CA" \
        ttl=87600h \
        key_type=rsa \
        key_bits=4096 \
        ou="Security" \
        organization="Company Inc" \
        country="TH" \
        locality="Bangkok" > /tmp/root_ca.crt
    
    vault write pki/config/urls \
        issuing_certificates="${vault_addr}/v1/pki/ca" \
        crl_distribution_points="${vault_addr}/v1/pki/crl"
    
    vault secrets enable -path=pki_int pki 2>/dev/null || true
    vault secrets tune -max-lease-ttl=43800h pki_int
    
    vault write -format=json pki_int/intermediate/generate/internal \
        common_name="company.com Intermediate CA" \
        key_type=rsa \
        key_bits=4096 | \
        jq -r '.data.csr' > /tmp/intermediate_csr.pem
    
    vault write -format=json pki/root/sign-intermediate \
        csr=@/tmp/intermediate_csr.pem \
        format=pem_bundle \
        ttl="43800h" | \
        jq -r '.data.certificate' > /tmp/intermediate_cert.pem
    
    vault write pki_int/intermediate/set-signed \
        certificate=@/tmp/intermediate_cert.pem
    
    vault write pki_int/config/urls \
        issuing_certificates="${vault_addr}/v1/pki_int/ca" \
        crl_distribution_points="${vault_addr}/v1/pki_int/crl"
    
    vault write pki_int/roles/company-services \
        allowed_domains="svc.cluster.local,company.com" \
        allow_subdomains=true \
        allow_glob_domains=true \
        max_ttl="72h" \
        key_type=rsa \
        key_bits=2048 \
        require_cn=false \
        server_flag=true \
        client_flag=true \
        code_signing_flag=false \
        email_protection_flag=false
    
    log "Vault PKI configured for Zero Trust"
}

deploy_cert_manager_vault() {
    log "Configuring cert-manager with Vault backend..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-issuer
spec:
  vault:
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: cert-manager
        secretRef:
          name: cert-manager-vault-token
          key: token
    path: pki_int/sign/company-services
    server: https://vault.company.com
    caBundle: $(base64 -w0 /tmp/root_ca.crt 2>/dev/null || echo "REPLACE_WITH_BASE64_CA")
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-company-cert
  namespace: istio-system
spec:
  secretName: wildcard-tls
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  commonName: "*.company.com"
  dnsNames:
    - "*.company.com"
    - "company.com"
  duration: 24h
  renewBefore: 1h
  privateKey:
    algorithm: RSA
    size: 2048
    rotationPolicy: Always
EOF
    
    log "cert-manager Vault integration configured"
}

implement_opa_authorization() {
    log "Implementing OPA (Open Policy Agent) authorization..."
    
    helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
    helm repo update
    
    helm upgrade --install gatekeeper gatekeeper/gatekeeper \
        --namespace gatekeeper-system \
        --create-namespace \
        --set replicas=3 \
        --set auditInterval=60 \
        --set emitAdmissionEvents=true \
        --set emitAuditEvents=true \
        --wait
    
    cat <<EOF | kubectl apply -f -
# Security context constraint template
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsecuritycontext
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSecurityContext
      validation:
        openAPIV3Schema:
          type: object
          properties:
            runAsNonRoot:
              type: boolean
            allowPrivilegeEscalation:
              type: boolean
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredsecuritycontext
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container '%v' must set runAsNonRoot=true", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.allowPrivilegeEscalation
          msg := sprintf("Container '%v' must set allowPrivilegeEscalation=false", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.readOnlyRootFilesystem
          msg := sprintf("Container '%v' should have readOnlyRootFilesystem=true", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredSecurityContext
metadata:
  name: require-security-context
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    excludedNamespaces:
      - kube-system
      - monitoring
      - cert-manager
---
# No privileged containers
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spspprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspprivilegedcontainer
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged
          msg := sprintf("Privileged container not allowed: '%v'", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          container.securityContext.privileged
          msg := sprintf("Privileged init container not allowed: '%v'", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: no-privileged-containers
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["*"]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
EOF
    
    log "OPA Gatekeeper policies deployed"
}

setup_falco_runtime_security() {
    log "Setting up Falco runtime security..."
    
    helm repo add falcosecurity https://falcosecurity.github.io/charts
    helm repo update
    
    cat <<EOF > /tmp/falco-values.yaml
falco:
  grpc:
    enabled: true
  grpcOutput:
    enabled: true
  
  rules_file:
    - /etc/falco/falco_rules.yaml
    - /etc/falco/falco_rules.local.yaml
    - /etc/falco/k8s_audit_rules.yaml
    - /etc/falco/rules.d

customRules:
  custom-rules.yaml: |-
    - rule: Shell in Container
      desc: A shell has been spawned in a container
      condition: >
        spawned_process and container
        and not container.image.repository in (trusted_images)
        and proc.name in (shell_binaries)
        and not proc.pname in (runc, containerd, docker, shell_binaries, sh)
      output: >
        Shell spawned in container
        (user=%user.name user_loginuid=%user.loginuid container_id=%container.id
        container_name=%container.name shell=%proc.name parent=%proc.pname
        cmdline=%proc.cmdline terminal=%proc.tty)
      priority: WARNING
      tags: [container, shell, security]
    
    - rule: Cryptocurrency Mining Detection
      desc: Detect process associated with cryptocurrency mining
      condition: >
        spawned_process and
        proc.name in (known_miners) or
        proc.cmdline contains "--mine" or
        proc.cmdline contains "stratum+tcp"
      output: >
        Cryptocurrency miner detected
        (user=%user.name command=%proc.cmdline container_id=%container.id)
      priority: CRITICAL
      tags: [cryptomining, security]
    
    - rule: K8s Service Account Token Access
      desc: Detect access to service account token
      condition: >
        open_read and
        fd.name startswith "/var/run/secrets/kubernetes.io/serviceaccount/" and
        not proc.name in (trusted_k8s_processes)
      output: >
        K8s service account token accessed
        (user=%user.name command=%proc.cmdline fd.name=%fd.name container=%container.id)
      priority: WARNING
      tags: [k8s, security]
    
    - rule: Unexpected Network Connection
      desc: Outbound connections on unexpected ports
      condition: >
        outbound and
        not proc.name in (allowed_outbound_processes) and
        fd.sport not in (allowed_outbound_ports) and
        container
      output: >
        Unexpected outbound connection
        (user=%user.name command=%proc.cmdline connection=%fd.name container=%container.id)
      priority: NOTICE
      tags: [network, security]

falcosidekick:
  enabled: true
  config:
    slack:
      webhookurl: "https://hooks.slack.com/services/REPLACE/WITH/ACTUAL"
      minimumpriority: warning
    elasticsearch:
      hostport: "http://elasticsearch.monitoring:9200"
      index: falco
      type: event
      minimumpriority: notice
    pagerduty:
      routingkey: REPLACE_WITH_ACTUAL_KEY
      minimumpriority: critical
EOF
    
    helm upgrade --install falco falcosecurity/falco \
        --namespace falco \
        --create-namespace \
        -f /tmp/falco-values.yaml \
        --wait
    
    log "Falco runtime security deployed"
}

configure_network_policies() {
    local namespace="${1:-default}"
    
    log "Configuring comprehensive network policies for: ${namespace}"
    
    cat <<EOF | kubectl apply -f -
# Default deny all ingress and egress
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
# Allow DNS resolution
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
      to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
---
# Allow ingress from ingress controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: ${namespace}
spec:
  podSelector:
    matchLabels:
      expose: "true"
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - port: 8080
        - port: 8443
---
# Allow monitoring
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: ${namespace}
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: 9090
          protocol: TCP
        - port: 8080
          protocol: TCP
---
# API tier can reach database tier
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-to-database
  namespace: ${namespace}
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: api
      ports:
        - port: 5432
        - port: 27017
        - port: 6379
EOF
    
    log "Network policies configured for: ${namespace}"
}

audit_security_posture() {
    log "=== Security Posture Audit ==="
    
    echo "--- Privileged Pods ---"
    kubectl get pods -A \
        -o json | \
        jq -r '.items[] | select(.spec.containers[].securityContext.privileged == true) | "\(.metadata.namespace)/\(.metadata.name)"' 2>/dev/null | \
        head -20 || echo "None found"
    
    echo ""
    echo "--- Pods Running as Root ---"
    kubectl get pods -A \
        -o json | \
        jq -r '.items[] | select(.spec.securityContext.runAsUser == 0 or .spec.containers[].securityContext.runAsUser == 0) | "\(.metadata.namespace)/\(.metadata.name)"' 2>/dev/null | \
        head -20 || echo "None found"
    
    echo ""
    echo "--- Pods with hostPath Volumes ---"
    kubectl get pods -A \
        -o json | \
        jq -r '.items[] | select(.spec.volumes[]?.hostPath != null) | "\(.metadata.namespace)/\(.metadata.name): \(.spec.volumes[].hostPath.path // empty)"' 2>/dev/null | \
        head -20 || echo "None found"
    
    echo ""
    echo "--- ServiceAccounts with Cluster-Admin ---"
    kubectl get clusterrolebindings \
        -o json | \
        jq -r '.items[] | select(.roleRef.name == "cluster-admin") | .subjects[]? | "\(.kind)/\(.namespace)/\(.name)"' 2>/dev/null | \
        head -20
    
    echo ""
    echo "--- Network Policy Coverage ---"
    local total_namespaces=$(kubectl get ns --no-headers | wc -l)
    local namespaces_with_np=$(kubectl get networkpolicy -A --no-headers | awk '{print $1}' | sort -u | wc -l)
    echo "Namespaces with network policies: ${namespaces_with_np}/${total_namespaces}"
    
    echo ""
    echo "--- TLS Certificate Status ---"
    kubectl get certificate -A \
        -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,READY:.status.conditions[-1].status,EXPIRY:.status.notAfter" 2>/dev/null | head -20
}

case "${1:-help}" in
    "vault-pki") setup_vault_pki ;;
    "cert-manager") deploy_cert_manager_vault ;;
    "opa") implement_opa_authorization ;;
    "falco") setup_falco_runtime_security ;;
    "network-policies") configure_network_policies "${2:-default}" ;;
    "audit") audit_security_posture ;;
    *) echo "Usage: $0 {vault-pki|cert-manager|opa|falco|network-policies|audit}" ;;
esac
```

### ขั้นตอนที่ 545: SIEM and Security Monitoring

**SIEM** (Security Information and Event Management) สำหรับ enterprise

```bash
#!/bin/bash
# siem-security-monitoring.sh - SIEM and Security Monitoring

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

deploy_opensearch_siem() {
    log "Deploying OpenSearch as SIEM platform..."
    
    helm repo add opensearch https://opensearch-project.github.io/helm-charts/
    helm repo update
    
    kubectl create namespace siem --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF > /tmp/opensearch-siem-values.yaml
clusterName: siem-cluster
nodeGroup: master

opensearchJavaOpts: "-Xmx8g -Xms8g"

resources:
  requests:
    cpu: "2"
    memory: "8Gi"
  limits:
    cpu: "4"
    memory: "16Gi"

replicas: 3

persistence:
  enabled: true
  size: 500Gi
  storageClass: fast-ssd

extraEnvs:
  - name: DISABLE_INSTALL_DEMO_CONFIG
    value: "true"
  - name: DISABLE_SECURITY_DASHBOARDS_PLUGIN
    value: "false"

config:
  opensearch.yml: |
    cluster.name: siem-cluster
    network.host: 0.0.0.0
    
    plugins.security.ssl.transport.pemcert_filepath: esnode.pem
    plugins.security.ssl.transport.pemkey_filepath: esnode-key.pem
    plugins.security.ssl.transport.pemtrustedcas_filepath: root-ca.pem
    plugins.security.ssl.transport.enforce_hostname_verification: false
    plugins.security.ssl.http.enabled: true
    plugins.security.ssl.http.pemcert_filepath: esnode.pem
    plugins.security.ssl.http.pemkey_filepath: esnode-key.pem
    plugins.security.ssl.http.pemtrustedcas_filepath: root-ca.pem
    plugins.security.allow_default_init_securityindex: true
    plugins.security.authcz.admin_dn:
      - CN=admin,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA
    
    opendistro_security.audit.type: internal_opensearch
    opendistro_security.enable_snapshot_restore_privilege: true
    opendistro_security.check_snapshot_restore_write_privileges: true
    
    indices.query.bool.max_clause_count: 10000
    cluster.max_shards_per_node: 10000
EOF
    
    helm upgrade --install opensearch opensearch/opensearch \
        --namespace siem \
        -f /tmp/opensearch-siem-values.yaml \
        --wait
    
    log "OpenSearch SIEM deployed"
}

create_security_index_templates() {
    log "Creating security index templates in OpenSearch..."
    
    local os_url="${1:-https://opensearch.siem:9200}"
    
    cat <<JSON > /tmp/falco-index-template.json
{
    "index_patterns": ["falco-*"],
    "template": {
        "settings": {
            "number_of_shards": 3,
            "number_of_replicas": 1,
            "index.lifecycle.name": "security-ilm",
            "index.lifecycle.rollover_alias": "falco"
        },
        "mappings": {
            "properties": {
                "@timestamp": {"type": "date"},
                "rule": {"type": "keyword"},
                "priority": {"type": "keyword"},
                "output": {"type": "text", "analyzer": "standard"},
                "container_id": {"type": "keyword"},
                "container_name": {"type": "keyword"},
                "namespace": {"type": "keyword"},
                "pod_name": {"type": "keyword"},
                "user_name": {"type": "keyword"},
                "process_name": {"type": "keyword"},
                "process_cmdline": {"type": "text"},
                "tags": {"type": "keyword"},
                "source_ip": {"type": "ip"},
                "destination_ip": {"type": "ip"},
                "geo_location": {"type": "geo_point"}
            }
        }
    },
    "priority": 200,
    "composed_of": []
}
JSON
    
    curl -s -X PUT "${os_url}/_index_template/falco-logs" \
        -H "Content-Type: application/json" \
        -d @/tmp/falco-index-template.json || true
    
    cat <<JSON > /tmp/k8s-audit-index-template.json
{
    "index_patterns": ["k8s-audit-*"],
    "template": {
        "settings": {
            "number_of_shards": 3,
            "number_of_replicas": 1
        },
        "mappings": {
            "properties": {
                "@timestamp": {"type": "date"},
                "verb": {"type": "keyword"},
                "apiVersion": {"type": "keyword"},
                "resource": {"type": "keyword"},
                "namespace": {"type": "keyword"},
                "name": {"type": "keyword"},
                "user_username": {"type": "keyword"},
                "user_groups": {"type": "keyword"},
                "sourceIPs": {"type": "ip"},
                "responseStatus_code": {"type": "integer"},
                "objectRef_resource": {"type": "keyword"},
                "objectRef_name": {"type": "keyword"}
            }
        }
    }
}
JSON
    
    curl -s -X PUT "${os_url}/_index_template/k8s-audit-logs" \
        -H "Content-Type: application/json" \
        -d @/tmp/k8s-audit-index-template.json || true
    
    log "Security index templates created"
}

create_security_alerts() {
    local os_url="${1:-https://opensearch.siem:9200}"
    
    log "Creating security alert rules in OpenSearch..."
    
    cat <<JSON > /tmp/security-monitor.json
{
    "name": "Privilege Escalation Detection",
    "type": "monitor",
    "enabled": true,
    "schedule": {
        "period": {"interval": 1, "unit": "MINUTES"}
    },
    "inputs": [
        {
            "search": {
                "indices": ["falco-*"],
                "query": {
                    "query": {
                        "bool": {
                            "must": [
                                {"term": {"priority": "CRITICAL"}},
                                {"range": {"@timestamp": {"gte": "now-5m"}}}
                            ]
                        }
                    }
                }
            }
        }
    ],
    "triggers": [
        {
            "name": "Critical Security Event",
            "severity": "1",
            "condition": {
                "script": {
                    "source": "ctx.results[0].hits.total.value > 0",
                    "lang": "painless"
                }
            },
            "actions": [
                {
                    "name": "slack-alert",
                    "destination_id": "slack-destination",
                    "subject_template": {
                        "source": "CRITICAL Security Alert: {{ctx.results[0].hits.total.value}} events"
                    },
                    "message_template": {
                        "source": "Critical security events detected in last 5 minutes. Review Falco dashboard immediately."
                    }
                }
            ]
        }
    ]
}
JSON
    
    curl -s -X POST "${os_url}/_plugins/_alerting/monitors" \
        -H "Content-Type: application/json" \
        -d @/tmp/security-monitor.json || true
    
    log "Security alerts configured"
}

setup_kubernetes_audit_logging() {
    log "Configuring Kubernetes audit logging..."
    
    cat <<EOF > /tmp/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived
rules:
  # Log all requests to secrets, configmaps, tokens
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets", "configmaps", "serviceaccounts/token"]
  
  # Log all requests to pods/exec, pods/portforward, pods/proxy
  - level: Request
    resources:
    - group: ""
      resources: ["pods/exec", "pods/portforward", "pods/proxy"]
  
  # Log RBAC changes
  - level: RequestResponse
    resources:
    - group: rbac.authorization.k8s.io
      resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]
  
  # Log auth/authn
  - level: Metadata
    nonResourceURLs:
      - /api*
      - /version
    verbs: ["get"]
  
  # Log all modifications
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete", "deletecollection"]
  
  # Log node privileged actions
  - level: Request
    users: ["system:node:*"]
    resources:
    - group: ""
      resources: ["nodes/status", "nodes"]
  
  # Minimal logging for read-only requests
  - level: None
    users:
      - system:kube-scheduler
      - system:kube-proxy
    resources:
    - group: ""
      resources: ["endpoints", "services", "pods"]
    verbs: ["get", "list", "watch"]
  
  # Default - log metadata only
  - level: Metadata
    omitStages:
    - RequestReceived
EOF
    
    log "Kubernetes audit policy created: /tmp/audit-policy.yaml"
    log "Apply to API server with: --audit-policy-file=/etc/kubernetes/audit-policy.yaml --audit-log-path=/var/log/kubernetes/audit.log"
}

generate_security_report() {
    local output="${1:-security-report-$(date '+%Y%m%d').html}"
    
    log "Generating security posture report..."
    
    local total_pods=$(kubectl get pods -A --no-headers 2>/dev/null | wc -l)
    local privileged_pods=$(kubectl get pods -A -o json 2>/dev/null | jq '[.items[] | select(.spec.containers[].securityContext.privileged == true)] | length')
    local policies_count=$(kubectl get networkpolicy -A --no-headers 2>/dev/null | wc -l)
    local certs_count=$(kubectl get certificate -A --no-headers 2>/dev/null | wc -l)
    
    cat <<HTML > "${output}"
<!DOCTYPE html>
<html>
<head><title>Security Posture Report - $(date '+%Y-%m-%d')</title>
<style>
body { font-family: Arial, sans-serif; margin: 40px; }
.metric { display: inline-block; background: #f0f0f0; border-radius: 8px; padding: 20px; margin: 10px; min-width: 150px; text-align: center; }
.good { border-left: 4px solid #4CAF50; }
.warn { border-left: 4px solid #FF9800; }
.bad { border-left: 4px solid #F44336; }
.metric h2 { margin: 0; font-size: 2em; }
table { border-collapse: collapse; width: 100%; margin: 20px 0; }
th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
th { background-color: #4a90e2; color: white; }
</style>
</head>
<body>
<h1>Kubernetes Security Posture Report</h1>
<p>Generated: $(date '+%Y-%m-%d %H:%M:%S')</p>

<h2>Security Metrics</h2>
<div class="metric good">
  <p>Total Pods</p>
  <h2>${total_pods}</h2>
</div>
<div class="metric $([ "${privileged_pods:-0}" -gt 0 ] && echo bad || echo good)">
  <p>Privileged Pods</p>
  <h2>${privileged_pods:-0}</h2>
</div>
<div class="metric $([ "${policies_count:-0}" -gt 10 ] && echo good || echo warn)">
  <p>Network Policies</p>
  <h2>${policies_count:-0}</h2>
</div>
<div class="metric good">
  <p>TLS Certificates</p>
  <h2>${certs_count:-0}</h2>
</div>

<h2>Security Findings</h2>
<table>
<tr><th>Category</th><th>Status</th><th>Finding</th><th>Recommendation</th></tr>
<tr><td>Container Security</td><td>$([ "${privileged_pods:-0}" -eq 0 ] && echo "✅ PASS" || echo "❌ FAIL")</td><td>Privileged containers: ${privileged_pods:-0}</td><td>Remove privileged flag from all containers</td></tr>
<tr><td>Network Security</td><td>$([ "${policies_count:-0}" -gt 0 ] && echo "✅ PASS" || echo "❌ FAIL")</td><td>Network policies: ${policies_count:-0}</td><td>Implement network policies for all namespaces</td></tr>
<tr><td>TLS Encryption</td><td>$([ "${certs_count:-0}" -gt 0 ] && echo "✅ PASS" || echo "⚠️ WARN")</td><td>Managed certificates: ${certs_count:-0}</td><td>Ensure all services use TLS</td></tr>
</table>
</body>
</html>
HTML
    
    log "Security report generated: ${output}"
}

case "${1:-help}" in
    "deploy-siem") deploy_opensearch_siem ;;
    "index-templates") create_security_index_templates "${2:-}" ;;
    "create-alerts") create_security_alerts "${2:-}" ;;
    "audit-logging") setup_kubernetes_audit_logging ;;
    "report") generate_security_report "${2:-}" ;;
    *) echo "Usage: $0 {deploy-siem|index-templates|create-alerts|audit-logging|report}" ;;
esac
```

---

## สรุป Part 48

ในส่วนนี้เราได้เรียนรู้:

| ขั้นตอน | หัวข้อ | เครื่องมือหลัก |
|---------|--------|----------------|
| 544 | Zero Trust Security | Vault PKI, cert-manager, OPA Gatekeeper, Falco, Network Policies |
| 545 | SIEM Security Monitoring | OpenSearch SIEM, Security Alerts, K8s Audit Logging, Security Reports |

### ขั้นตอนต่อไป: Part 49 - Advanced CI/CD and DevSecOps Pipeline
