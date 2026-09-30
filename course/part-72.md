# Part 72: Zero-Trust Security Architecture, mTLS Mesh และ Policy as Code

## ขั้นตอนที่ 635: Zero-Trust Network Architecture

### `zero-trust-architecture.sh`

```bash
#!/bin/bash
# Zero-Trust: SPIFFE/SPIRE identity, mTLS everywhere, micro-segmentation

# Install SPIRE (SPIFFE Runtime Environment)
helm repo add spiffe https://spiffe.github.io/helm-charts/
helm repo update

cat > spire-values.yaml << 'EOF'
spire-server:
  replicaCount: 3
  jwtIssuer: "https://spire.payment.internal"
  trustDomain: "payment.internal"
  controllerManager:
    enabled: true
  persistence:
    enabled: true
    storageClass: gp3-io2
    size: 10Gi
  federation:
    enabled: false
  service:
    type: ClusterIP
  nodeAttestor:
    k8sPsat:
      enabled: true
      serviceAccountAllowList:
        - "spire-system:spire-agent"
  keyManager:
    disk:
      enabled: false
    awsKms:
      enabled: true
      keyArn: arn:aws:kms:ap-southeast-1:123456789:key/mrk-abc123
      region: ap-southeast-1
  ca:
    ttl: 24h
    keyType: rsa-4096
  upstreamAuthority:
    awsPca:
      region: ap-southeast-1
      certificateAuthorityArn: arn:aws:acm-pca:ap-southeast-1:123456789:certificate-authority/abc

spire-agent:
  trustDomain: "payment.internal"
  server:
    address: spire-server
    port: 443
  workloadAttestors:
    k8s:
      enabled: true
      skipKubeletVerification: false
      nodeName: ""
  nodeAttestor:
    k8sPsat:
      enabled: true
      cluster: "payment-prod"
  sds:
    enabled: true
    defaultSvidName: "default"
    defaultBundleName: "ROOTCA"

spiffe-csi-driver:
  enabled: true

spiffe-oidc-discovery-provider:
  enabled: true
  config:
    domains:
      - spire.payment.internal
EOF

helm install spire spiffe/spire \
  --namespace spire-system \
  --create-namespace \
  --values spire-values.yaml

# Register SPIFFE entries for services
cat > register-spiffe-entries.sh << 'BASH'
#!/bin/bash
SPIRE_SERVER="spire-system/spire-server-0"

# Payment service
kubectl exec -n spire-system $SPIRE_SERVER -- \
  /opt/spire/bin/spire-server entry create \
  -spiffeID spiffe://payment.internal/ns/payment-prod/sa/payment-service \
  -parentID spiffe://payment.internal/k8s-psat/payment-prod/node \
  -selector k8s:ns:payment-prod \
  -selector k8s:sa:payment-service \
  -ttl 3600

# API Gateway
kubectl exec -n spire-system $SPIRE_SERVER -- \
  /opt/spire/bin/spire-server entry create \
  -spiffeID spiffe://payment.internal/ns/payment-prod/sa/api-gateway \
  -parentID spiffe://payment.internal/k8s-psat/payment-prod/node \
  -selector k8s:ns:payment-prod \
  -selector k8s:sa:api-gateway \
  -ttl 3600

# Fraud detection
kubectl exec -n spire-system $SPIRE_SERVER -- \
  /opt/spire/bin/spire-server entry create \
  -spiffeID spiffe://payment.internal/ns/payment-prod/sa/fraud-service \
  -parentID spiffe://payment.internal/k8s-psat/payment-prod/node \
  -selector k8s:ns:payment-prod \
  -selector k8s:sa:fraud-service \
  -ttl 3600

echo "SPIFFE entries registered"
BASH
chmod +x register-spiffe-entries.sh

# Istio mTLS STRICT mode with SPIRE integration
cat > istio-mtls-strict.yaml << 'EOF'
# Enforce STRICT mTLS globally
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: payment-prod
spec:
  mtls:
    mode: STRICT
---
# Payment service: only accept connections from API gateway + order service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-authz
  namespace: payment-prod
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/payment-prod/sa/api-gateway"
              - "cluster.local/ns/payment-prod/sa/order-service"
            namespaces:
              - payment-prod
      to:
        - operation:
            methods: ["POST", "GET"]
            paths: ["/api/v2/payments*", "/api/v2/refunds*"]
      when:
        - key: request.headers[x-request-id]
          notValues: [""]
---
# Deny all egress except allowlisted
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-egress-deny-all
  namespace: payment-prod
spec:
  action: DENY
  rules:
    - from:
        - source:
            namespaces: ["payment-prod"]
      to:
        - operation:
            ports: ["*"]
    # Exception: allow DNS (port 53 handled by Cilium, not Istio)
---
# Allow payment-service → PostgreSQL, Kafka, Stripe
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-egress-allow
  namespace: payment-prod
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
    - to:
        - operation:
            ports: ["5432"]  # PostgreSQL
    - to:
        - operation:
            ports: ["9092"]  # Kafka
    - to:
        - operation:
            hosts: ["api.stripe.com"]
            ports: ["443"]
---
# JWT authentication for external requests
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: payment-prod
spec:
  selector:
    matchLabels:
      app: api-gateway
  jwtRules:
    - issuer: "https://auth.payment.internal"
      jwksUri: "https://auth.payment.internal/.well-known/jwks.json"
      audiences:
        - "payment-api"
      forwardOriginalToken: true
      outputClaimToHeaders:
        - header: x-jwt-sub
          claim: sub
        - header: x-jwt-tier
          claim: tier
EOF
kubectl apply -f istio-mtls-strict.yaml

echo "Zero-trust architecture configured"
```

---

## ขั้นตอนที่ 636: Open Policy Agent (OPA) — Policy as Code

### `policy-as-code.sh`

```bash
#!/bin/bash
# OPA: Rego policies for K8s admission, API authorization, data governance

# Install OPA Gatekeeper
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update

helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace \
  --set replicas=3 \
  --set controllerManager.dnsPolicy=ClusterFirstWithHostNet \
  --set audit.logLevel=INFO \
  --set validatingWebhookTimeoutSeconds=15 \
  --set mutatingWebhookTimeoutSeconds=2 \
  --set disableCertRotation=false \
  --set enableExternalData=true

# ConstraintTemplate: Require security context
cat > opa/constraint-templates/require-security-context.yaml << 'EOF'
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requiresecuritycontext
  annotations:
    description: "Requires pods to run as non-root with read-only root filesystem"
spec:
  crd:
    spec:
      names:
        kind: RequireSecurityContext
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedRunAsUser:
              type: integer
              description: "Minimum allowed UID"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requiresecuritycontext

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container %v must have runAsNonRoot: true", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.readOnlyRootFilesystem
          msg := sprintf("Container %v must have readOnlyRootFilesystem: true", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.allowPrivilegeEscalation == true
          msg := sprintf("Container %v must not allow privilege escalation", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.capabilities.add
          not is_allowed_capability(container.securityContext.capabilities.add[_])
          msg := sprintf("Container %v adds disallowed Linux capabilities", [container.name])
        }

        is_allowed_capability(cap) {
          allowed := {"NET_BIND_SERVICE"}
          cap == allowed[_]
        }
EOF

cat > opa/constraint-templates/require-resource-limits.yaml << 'EOF'
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requireresourcelimits
spec:
  crd:
    spec:
      names:
        kind: RequireResourceLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            maxCPU:
              type: string
            maxMemory:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requireresourcelimits
        import future.keywords.in

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container %v missing CPU limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container %v missing memory limit", [container.name])
        }
EOF

cat > opa/constraint-templates/allowed-registries.yaml << 'EOF'
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: allowedregistries
spec:
  crd:
    spec:
      names:
        kind: AllowedRegistries
      validation:
        openAPIV3Schema:
          type: object
          properties:
            registries:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package allowedregistries

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          image := container.image
          not starts_with_allowed(image)
          msg := sprintf("Container %v uses disallowed registry: %v", [container.name, image])
        }

        starts_with_allowed(image) {
          registry := input.parameters.registries[_]
          startswith(image, registry)
        }
EOF

# Apply constraints
cat > opa/constraints/payment-prod-constraints.yaml << 'EOF'
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireSecurityContext
metadata:
  name: require-security-context-payment-prod
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["payment-prod"]
  parameters:
    allowedRunAsUser: 1000
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireResourceLimits
metadata:
  name: require-resource-limits-payment-prod
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["payment-prod"]
  parameters:
    maxCPU: "8"
    maxMemory: "16Gi"
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRegistries
metadata:
  name: allowed-registries-payment-prod
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["payment-prod"]
  parameters:
    registries:
      - "gcr.io/myproject/"
      - "ghcr.io/myorg/"
      - "registry.k8s.io/"
EOF
kubectl apply -f opa/constraint-templates/
kubectl apply -f opa/constraints/

# OPA for API authorization (standalone, not K8s)
cat > opa/policies/api-authz.rego << 'REGO'
package api.authz

import future.keywords.if
import future.keywords.in

default allow := false

# Allow if user has required role for the action
allow if {
    has_permission(input.user.roles, input.resource, input.action)
}

# Deny if user is suspended
allow := false if {
    input.user.suspended == true
}

# Permission matrix
has_permission(roles, resource, action) if {
    role := roles[_]
    permissions[role][resource][_] == action
}

permissions := {
    "admin": {
        "payments": {"read", "write", "delete", "refund"},
        "users": {"read", "write", "delete"},
        "reports": {"read", "export"},
    },
    "operator": {
        "payments": {"read", "refund"},
        "users": {"read"},
        "reports": {"read"},
    },
    "viewer": {
        "payments": {"read"},
        "reports": {"read"},
    },
}

# Rate limit check
allow := false if {
    input.rate_limit_exceeded == true
    not is_admin(input.user.roles)
}

is_admin(roles) if {
    roles[_] == "admin"
}

# Geographic restrictions (OFAC compliance)
allow := false if {
    blocked_countries[input.user.country]
}

blocked_countries := {
    "KP",  # North Korea
    "IR",  # Iran
    "CU",  # Cuba
    "SY",  # Syria
}

# Time-based access (maintenance windows)
allow := false if {
    is_maintenance_window
    input.resource != "status"
}

is_maintenance_window if {
    now := time.clock([time.now_ns(), "UTC"])
    now[0] == 3  # 03:00-04:00 UTC maintenance window
    now[1] < 60
}
REGO

# Conftest for CI policy testing
cat > opa/policies/k8s-security.rego << 'REGO'
package main

import future.keywords.if
import future.keywords.every

deny[msg] if {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    not container.resources.limits.memory
    msg := sprintf("Deployment %s: container %s missing memory limit", [input.metadata.name, container.name])
}

deny[msg] if {
    input.kind == "Deployment"
    not input.spec.template.spec.securityContext.runAsNonRoot
    msg := sprintf("Deployment %s must run as non-root", [input.metadata.name])
}

deny[msg] if {
    input.kind == "Deployment"
    not input.spec.template.spec.securityContext.seccompProfile
    msg := sprintf("Deployment %s must set seccompProfile", [input.metadata.name])
}

warn[msg] if {
    input.kind == "Deployment"
    not input.metadata.labels.version
    msg := sprintf("Deployment %s is missing version label", [input.metadata.name])
}
REGO

# Test policies with conftest
conftest test opa/policies/

echo "OPA Policy as Code setup complete"
```

---

## ขั้นตอนที่ 637: Secrets Management — Vault + External Secrets Operator

### `secrets-management.sh`

```bash
#!/bin/bash
# Secrets Management: HashiCorp Vault HA + External Secrets Operator

# Install Vault HA with Raft storage
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

cat > vault-values.yaml << 'EOF'
global:
  enabled: true
  tlsDisable: false

server:
  image:
    repository: hashicorp/vault
    tag: 1.16.2
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      config: |
        ui = true
        listener "tcp" {
          tls_disable = 0
          address     = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_cert_file = "/vault/userconfig/vault-tls/tls.crt"
          tls_key_file  = "/vault/userconfig/vault-tls/tls.key"
          tls_ca_cert_file = "/vault/userconfig/vault-tls/ca.crt"
          telemetry {
            unauthenticated_metrics_access = "false"
          }
        }
        storage "raft" {
          path    = "/vault/data"
          node_id = "$VAULT_RAFT_NODE_ID"
          retry_join {
            leader_tls_servername = "vault-active"
            leader_api_addr = "https://vault-0.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/vault-tls/ca.crt"
          }
          retry_join {
            leader_tls_servername = "vault-active"
            leader_api_addr = "https://vault-1.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/vault-tls/ca.crt"
          }
          retry_join {
            leader_tls_servername = "vault-active"
            leader_api_addr = "https://vault-2.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/vault-tls/ca.crt"
          }
          autopilot {
            cleanup_dead_servers = "true"
            last_contact_threshold = "200ms"
            min_quorum = 3
          }
        }
        service_registration "kubernetes" {}
        seal "awskms" {
          region     = "ap-southeast-1"
          kms_key_id = "mrk-vault-unseal-key-id"
        }
        telemetry {
          prometheus_retention_time = "30s"
          disable_hostname = true
        }
  extraEnvironmentVars:
    VAULT_RAFT_NODE_ID:
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
  dataStorage:
    enabled: true
    size: 50Gi
    storageClass: gp3-io2
  auditStorage:
    enabled: true
    size: 20Gi
    storageClass: gp3-io2
  resources:
    requests:
      cpu: "1"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "8Gi"
  extraVolumes:
    - type: secret
      name: vault-tls

ui:
  enabled: true
  serviceType: ClusterIP
EOF

helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --values vault-values.yaml

# Configure Vault after init
cat > setup-vault.sh << 'BASH'
#!/bin/bash
export VAULT_ADDR="https://vault-active.vault:8200"
export VAULT_TOKEN="${VAULT_INIT_TOKEN}"
export VAULT_CACERT="/vault/tls/ca.crt"

# Enable audit logging
vault audit enable file file_path=/vault/audit/audit.log

# Enable KV v2 secrets
vault secrets enable -path=secret kv-v2

# Enable database dynamic secrets for PostgreSQL
vault secrets enable -path=database database
vault write database/config/postgresql \
  plugin_name="postgresql-database-plugin" \
  allowed_roles="payment-service,readonly" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.payment-prod:5432/payments?sslmode=require" \
  username="vault_admin" \
  password="${POSTGRES_VAULT_ADMIN_PASSWORD}" \
  rotation_period="24h"

vault write database/roles/payment-service \
  db_name="postgresql" \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' IN ROLE payment_app_role;" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="4h"

vault write database/roles/readonly \
  db_name="postgresql" \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="2h" \
  max_ttl="8h"

# Enable PKI for internal TLS
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

vault write -field=certificate pki/root/generate/internal \
  common_name="payment.internal" \
  issuer_name="root-2024" \
  ttl=87600h > /tmp/root-cert.pem

vault write pki/config/cluster \
  path="https://vault-active.vault:8200/v1/pki"

vault write pki/roles/payment-internal \
  allowed_domains="payment.internal" \
  allow_subdomains=true \
  max_ttl="72h" \
  key_type="rsa" \
  key_bits=4096 \
  require_cn=false

# K8s auth backend
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Policy for payment service
vault policy write payment-service-policy - << 'POLICY'
path "secret/data/payment-prod/*" {
  capabilities = ["read"]
}
path "database/creds/payment-service" {
  capabilities = ["read"]
}
path "pki/issue/payment-internal" {
  capabilities = ["create", "update"]
}
POLICY

vault write auth/kubernetes/role/payment-service \
  bound_service_account_names=payment-service \
  bound_service_account_namespaces=payment-prod \
  policies=payment-service-policy \
  ttl=1h \
  max_ttl=4h

echo "Vault configuration complete"
BASH

# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true

# SecretStore pointing to Vault
cat > eso/vault-secretstore.yaml << 'EOF'
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault-active.vault:8200"
      path: "secret"
      version: "v2"
      caBundle: ""
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets-operator"
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
---
# ExternalSecret: sync Vault secret to K8s Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-service-secrets
  namespace: payment-prod
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: payment-service-secrets
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      engineVersion: v2
      metadata:
        annotations:
          reloader.stakater.com/match: "true"
      data:
        DATABASE_URL: "postgresql://{{ .db_user }}:{{ .db_password }}@postgres:5432/payments"
        STRIPE_SECRET_KEY: "{{ .stripe_secret }}"
        JWT_PRIVATE_KEY: "{{ .jwt_private_key }}"
  data:
    - secretKey: db_user
      remoteRef:
        key: payment-prod/database
        property: username
    - secretKey: db_password
      remoteRef:
        key: payment-prod/database
        property: password
    - secretKey: stripe_secret
      remoteRef:
        key: payment-prod/stripe
        property: secret_key
    - secretKey: jwt_private_key
      remoteRef:
        key: payment-prod/jwt
        property: private_key
EOF
kubectl apply -f eso/

echo "Vault + External Secrets Operator configured"
```

---

## ขั้นตอนที่ 638: Security Compliance Automation — CIS Benchmarks & DISA STIG

### `compliance-automation.sh`

```bash
#!/bin/bash
# Security Compliance: CIS Benchmark scanning, DISA STIG, automated remediation

cat > compliance/compliance_checker.py << 'PYTHON'
import asyncio
import json
import logging
import subprocess
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
from datetime import datetime

logger = logging.getLogger(__name__)


class Severity(str, Enum):
    CRITICAL = "critical"
    HIGH     = "high"
    MEDIUM   = "medium"
    LOW      = "low"
    INFO     = "info"


class ComplianceStatus(str, Enum):
    PASS = "pass"
    FAIL = "fail"
    WARN = "warn"
    NOT_APPLICABLE = "not_applicable"


@dataclass
class ComplianceFinding:
    id: str
    title: str
    description: str
    severity: Severity
    status: ComplianceStatus
    remediation: str
    resource: str
    framework: str  # CIS, STIG, SOC2, PCI-DSS
    control_id: str
    auto_remediable: bool = False


@dataclass
class ComplianceReport:
    timestamp: datetime = field(default_factory=datetime.utcnow)
    findings: list[ComplianceFinding] = field(default_factory=list)
    score: float = 0.0
    passed: int = 0
    failed: int = 0
    total: int = 0

    def calculate_score(self):
        self.total = len(self.findings)
        self.passed = sum(1 for f in self.findings if f.status == ComplianceStatus.PASS)
        self.failed = sum(1 for f in self.findings if f.status == ComplianceStatus.FAIL)
        self.score = (self.passed / self.total * 100) if self.total > 0 else 0


class CISBenchmarkChecker:
    """
    Implements CIS Kubernetes Benchmark v1.8 checks.
    """

    async def run_all_checks(self, namespace: str) -> list[ComplianceFinding]:
        findings = []
        checks = [
            self._check_anonymous_auth,
            self._check_rbac_enabled,
            self._check_audit_logging,
            self._check_etcd_encryption,
            self._check_pod_security_standards,
            self._check_network_policies,
            self._check_service_account_tokens,
            self._check_privileged_containers,
            self._check_host_path_volumes,
            self._check_resource_quotas,
        ]
        for check in checks:
            try:
                finding = await check(namespace)
                if finding:
                    findings.append(finding)
            except Exception as e:
                logger.error(f"Check {check.__name__} failed: {e}")
        return findings

    async def _check_anonymous_auth(self, namespace: str) -> Optional[ComplianceFinding]:
        result = subprocess.run(
            ["kubectl", "get", "pod", "-n", "kube-system", "-l", "component=kube-apiserver",
             "-o", "jsonpath={.items[0].spec.containers[0].command}"],
            capture_output=True, text=True
        )
        command = result.stdout
        anonymous_disabled = "--anonymous-auth=false" in command

        return ComplianceFinding(
            id="CIS-1.2.1",
            title="Anonymous Authentication disabled on API server",
            description="The API server should not allow anonymous requests",
            severity=Severity.CRITICAL,
            status=ComplianceStatus.PASS if anonymous_disabled else ComplianceStatus.FAIL,
            remediation="Add --anonymous-auth=false to kube-apiserver flags",
            resource="kube-apiserver",
            framework="CIS",
            control_id="1.2.1",
            auto_remediable=False,
        )

    async def _check_rbac_enabled(self, namespace: str) -> Optional[ComplianceFinding]:
        result = subprocess.run(
            ["kubectl", "get", "pod", "-n", "kube-system", "-l", "component=kube-apiserver",
             "-o", "jsonpath={.items[0].spec.containers[0].command}"],
            capture_output=True, text=True
        )
        rbac_enabled = "RBAC" in result.stdout and "--authorization-mode" in result.stdout

        return ComplianceFinding(
            id="CIS-1.2.7",
            title="RBAC enabled as authorization mode",
            description="Role-based access control must be enabled",
            severity=Severity.CRITICAL,
            status=ComplianceStatus.PASS if rbac_enabled else ComplianceStatus.FAIL,
            remediation="Ensure --authorization-mode includes RBAC",
            resource="kube-apiserver",
            framework="CIS",
            control_id="1.2.7",
        )

    async def _check_audit_logging(self, namespace: str) -> Optional[ComplianceFinding]:
        result = subprocess.run(
            ["kubectl", "get", "pod", "-n", "kube-system", "-l", "component=kube-apiserver",
             "-o", "jsonpath={.items[0].spec.containers[0].command}"],
            capture_output=True, text=True
        )
        audit_enabled = "--audit-log-path" in result.stdout

        return ComplianceFinding(
            id="CIS-1.2.22",
            title="Audit logging enabled",
            description="Kubernetes API server audit logging captures all requests",
            severity=Severity.HIGH,
            status=ComplianceStatus.PASS if audit_enabled else ComplianceStatus.FAIL,
            remediation="Add --audit-log-path and --audit-policy-file flags to kube-apiserver",
            resource="kube-apiserver",
            framework="CIS",
            control_id="1.2.22",
        )

    async def _check_etcd_encryption(self, namespace: str) -> Optional[ComplianceFinding]:
        result = subprocess.run(
            ["kubectl", "get", "pod", "-n", "kube-system", "-l", "component=etcd",
             "-o", "jsonpath={.items[0].spec.containers[0].command}"],
            capture_output=True, text=True
        )
        encrypted = "--client-cert-auth=true" in result.stdout

        return ComplianceFinding(
            id="CIS-2.1",
            title="etcd client certificate authentication",
            description="etcd must require client certificate authentication",
            severity=Severity.CRITICAL,
            status=ComplianceStatus.PASS if encrypted else ComplianceStatus.FAIL,
            remediation="Set --client-cert-auth=true on etcd",
            resource="etcd",
            framework="CIS",
            control_id="2.1",
        )

    async def _check_pod_security_standards(self, namespace: str) -> Optional[ComplianceFinding]:
        result = subprocess.run(
            ["kubectl", "get", "namespace", namespace,
             "-o", "jsonpath={.metadata.labels}"],
            capture_output=True, text=True
        )
        labels = json.loads(result.stdout or "{}")
        pss_enforced = labels.get("pod-security.kubernetes.io/enforce") == "restricted"

        return ComplianceFinding(
            id="CIS-5.2.1",
            title="Pod Security Standards enforced",
            description="Namespace must enforce restricted Pod Security Standards",
            severity=Severity.HIGH,
            status=ComplianceStatus.PASS if pss_enforced else ComplianceStatus.FAIL,
            remediation=f"kubectl label namespace {namespace} pod-security.kubernetes.io/enforce=restricted",
            resource=f"namespace/{namespace}",
            framework="CIS",
            control_id="5.2.1",
            auto_remediable=True,
        )

    async def _check_network_policies(self, namespace: str) -> Optional[ComplianceFinding]:
        result = subprocess.run(
            ["kubectl", "get", "networkpolicies", "-n", namespace, "-o", "json"],
            capture_output=True, text=True
        )
        policies = json.loads(result.stdout or '{"items":[]}')
        has_default_deny = any(
            not p.get("spec", {}).get("podSelector", {}).get("matchLabels")
            and not p.get("spec", {}).get("ingress")
            for p in policies.get("items", [])
        )

        return ComplianceFinding(
            id="CIS-5.3.2",
            title="Default-deny NetworkPolicy exists",
            description="All namespaces should have a default-deny NetworkPolicy",
            severity=Severity.MEDIUM,
            status=ComplianceStatus.PASS if has_default_deny else ComplianceStatus.FAIL,
            remediation="Apply a default-deny NetworkPolicy to the namespace",
            resource=f"namespace/{namespace}",
            framework="CIS",
            control_id="5.3.2",
            auto_remediable=True,
        )

    async def _check_service_account_tokens(self, namespace: str) -> Optional[ComplianceFinding]:
        result = subprocess.run(
            ["kubectl", "get", "serviceaccounts", "-n", namespace, "-o", "json"],
            capture_output=True, text=True
        )
        accounts = json.loads(result.stdout or '{"items":[]}')
        all_disable_automount = all(
            sa.get("automountServiceAccountToken") == False or sa["metadata"]["name"] == "default"
            for sa in accounts.get("items", [])
        )

        return ComplianceFinding(
            id="CIS-5.1.5",
            title="ServiceAccount token automounting disabled",
            description="Service accounts should not auto-mount tokens unless required",
            severity=Severity.MEDIUM,
            status=ComplianceStatus.PASS if all_disable_automount else ComplianceStatus.WARN,
            remediation="Set automountServiceAccountToken: false on ServiceAccounts",
            resource=f"namespace/{namespace}",
            framework="CIS",
            control_id="5.1.5",
            auto_remediable=True,
        )

    async def _check_privileged_containers(self, namespace: str) -> Optional[ComplianceFinding]:
        result = subprocess.run(
            ["kubectl", "get", "pods", "-n", namespace, "-o",
             "jsonpath={range .items[*]}{.metadata.name}:{range .spec.containers[*]}{.name}:{.securityContext.privileged}{','}{end}{'|'}{end}"],
            capture_output=True, text=True
        )
        privileged_found = "true" in result.stdout.lower()

        return ComplianceFinding(
            id="CIS-5.2.1",
            title="No privileged containers running",
            description="Containers must not run in privileged mode",
            severity=Severity.CRITICAL,
            status=ComplianceStatus.FAIL if privileged_found else ComplianceStatus.PASS,
            remediation="Remove securityContext.privileged: true from all containers",
            resource=f"namespace/{namespace}",
            framework="CIS",
            control_id="5.2.1",
        )

    async def _check_host_path_volumes(self, namespace: str) -> Optional[ComplianceFinding]:
        result = subprocess.run(
            ["kubectl", "get", "pods", "-n", namespace, "-o",
             "jsonpath={range .items[*]}{range .spec.volumes[*]}{.hostPath.path}{','}{end}{end}"],
            capture_output=True, text=True
        )
        dangerous_paths = {"/", "/etc", "/var", "/proc", "/sys", "/dev"}
        host_paths = set(result.stdout.split(",")) - {""}
        dangerous = host_paths & dangerous_paths

        return ComplianceFinding(
            id="CIS-5.2.7",
            title="No dangerous hostPath volumes",
            description="Pods must not mount dangerous host paths",
            severity=Severity.HIGH,
            status=ComplianceStatus.FAIL if dangerous else ComplianceStatus.PASS,
            remediation="Remove or restrict hostPath volume mounts",
            resource=f"namespace/{namespace}",
            framework="CIS",
            control_id="5.2.7",
        )

    async def _check_resource_quotas(self, namespace: str) -> Optional[ComplianceFinding]:
        result = subprocess.run(
            ["kubectl", "get", "resourcequota", "-n", namespace, "-o", "json"],
            capture_output=True, text=True
        )
        quotas = json.loads(result.stdout or '{"items":[]}')
        has_quota = len(quotas.get("items", [])) > 0

        return ComplianceFinding(
            id="CIS-5.7.3",
            title="ResourceQuota defined for namespace",
            description="Namespaces should have ResourceQuota to prevent resource exhaustion",
            severity=Severity.LOW,
            status=ComplianceStatus.PASS if has_quota else ComplianceStatus.WARN,
            remediation="Create a ResourceQuota for the namespace",
            resource=f"namespace/{namespace}",
            framework="CIS",
            control_id="5.7.3",
            auto_remediable=True,
        )


class AutoRemediator:
    async def remediate(self, finding: ComplianceFinding) -> bool:
        if not finding.auto_remediable:
            return False
        if finding.control_id == "5.2.1" and "PSS" in finding.title:
            return await self._apply_pss_label(finding.resource.split("/")[1])
        elif finding.control_id == "5.3.2":
            return await self._apply_default_deny(finding.resource.split("/")[1])
        return False

    async def _apply_pss_label(self, namespace: str) -> bool:
        result = subprocess.run([
            "kubectl", "label", "namespace", namespace,
            "pod-security.kubernetes.io/enforce=restricted",
            "pod-security.kubernetes.io/warn=restricted",
            "--overwrite"
        ])
        return result.returncode == 0

    async def _apply_default_deny(self, namespace: str) -> bool:
        policy = {
            "apiVersion": "networking.k8s.io/v1",
            "kind": "NetworkPolicy",
            "metadata": {"name": "default-deny-all", "namespace": namespace},
            "spec": {"podSelector": {}, "policyTypes": ["Ingress", "Egress"]}
        }
        result = subprocess.run(
            ["kubectl", "apply", "-f", "-"],
            input=json.dumps(policy),
            capture_output=True, text=True
        )
        return result.returncode == 0


async def main():
    checker = CISBenchmarkChecker()
    remediator = AutoRemediator()
    report = ComplianceReport()

    for namespace in ["payment-prod", "default", "kube-system"]:
        findings = await checker.run_all_checks(namespace)
        report.findings.extend(findings)

        # Auto-remediate safe findings
        for finding in findings:
            if finding.status == ComplianceStatus.FAIL and finding.auto_remediable:
                success = await remediator.remediate(finding)
                if success:
                    finding.status = ComplianceStatus.PASS
                    logger.info(f"Auto-remediated: {finding.id} in {namespace}")

    report.calculate_score()
    print(f"Compliance Score: {report.score:.1f}% ({report.passed}/{report.total} passed)")
    print(f"Critical failures: {sum(1 for f in report.findings if f.status == ComplianceStatus.FAIL and f.severity == Severity.CRITICAL)}")

    with open("compliance-report.json", "w") as f:
        json.dump({
            "timestamp": report.timestamp.isoformat(),
            "score": report.score,
            "passed": report.passed,
            "failed": report.failed,
            "total": report.total,
            "findings": [
                {
                    "id": f.id, "title": f.title, "severity": f.severity,
                    "status": f.status, "framework": f.framework,
                    "remediation": f.remediation, "resource": f.resource,
                }
                for f in report.findings
            ]
        }, f, indent=2, default=str)


if __name__ == "__main__":
    asyncio.run(main())
PYTHON

echo "Compliance automation complete"
```

---

## สรุป Part 72

Part 72 ครอบคลุม Zero-Trust Security และ Compliance ระดับ enterprise:

| Step | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|--------------|
| 635 | Zero-Trust Architecture | SPIFFE/SPIRE, Istio STRICT mTLS, RequestAuthentication JWT |
| 636 | Policy as Code (OPA) | Gatekeeper ConstraintTemplates, Rego authz policies, OFAC compliance |
| 637 | Secrets Management | Vault HA Raft, AWS KMS auto-unseal, External Secrets Operator |
| 638 | Compliance Automation | CIS Benchmark v1.8 checks, auto-remediation, JSON compliance report |

### Key Concepts ที่เรียนรู้:
- **SPIFFE/SPIRE**: workload identity (spiffe:// URIs), K8s PSAT attestation, OIDC discovery
- **Istio Authorization**: PeerAuthentication STRICT, AuthorizationPolicy principal matching
- **OPA Rego**: `violation[{"msg":msg}]` pattern, future.keywords.if, permission matrix
- **Gatekeeper**: ConstraintTemplate (Rego embedded), Constraint CR enforcement
- **Vault HA**: Raft consensus, AWS KMS seal, dynamic DB credentials, PKI engine
- **External Secrets**: ClusterSecretStore, ExternalSecret refresh, template transformation
- **CIS Benchmark**: kubectl-based checks, auto-remediation for safe controls

ขั้นตอนต่อไป: **Part 73** — Platform Engineering CLI, Developer Self-Service Portal และ Internal Developer Platform (IDP) Advanced
