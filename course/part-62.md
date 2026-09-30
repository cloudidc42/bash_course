# Part 62: Advanced Kubernetes Security, Supply Chain Security และ Runtime Protection

## ภาพรวม
Part นี้ครอบคลุม Security ระดับ World-Class สำหรับ Kubernetes workloads ตั้งแต่ Supply Chain ถึง Runtime

---

## Step 597: Advanced Kubernetes RBAC Hardening และ Audit Logging

```bash
cat > advanced-rbac-hardening.sh << 'SCRIPT'
#!/bin/bash
# Advanced RBAC Hardening and Audit Logging

set -euo pipefail

echo "=== Advanced Kubernetes RBAC Hardening ==="

# ─── 1. Audit Policy ───────────────────────────────────────────────────────
cat > /etc/kubernetes/audit-policy.yaml << 'AUDIT'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all authentication failures
  - level: Metadata
    omitStages: []
    users: []
    verbs: []
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
    namespaces: ["kube-system"]

  # Log secret access at RequestResponse level
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Log privileged pod creation
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods"]
    verbs: ["create", "update", "patch"]

  # Log RBAC changes
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]

  # Log exec/attach (potential breakout vector)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]

  # Log service account token creation
  - level: Metadata
    resources:
      - group: ""
        resources: ["serviceaccounts/token"]

  # Skip noisy read-only ops on non-sensitive resources
  - level: None
    verbs: ["get", "watch", "list"]
    resources:
      - group: ""
        resources: ["endpoints", "services", "pods"]
    users:
      - "system:kube-proxy"
      - "system:node"

  # Default: log metadata for everything else
  - level: Metadata
    omitStages:
      - RequestReceived
AUDIT

echo "Audit policy created"

# ─── 2. Least-Privilege Service Accounts ───────────────────────────────────
kubectl apply -f - << 'EOF'
# Disable automounting for all default service accounts
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: production
automountServiceAccountToken: false
---
# Application-specific SA with minimal permissions
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/payment-service-role
automountServiceAccountToken: true
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: payment-service-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
    resourceNames: ["payment-config"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["payment-tls", "payment-db-creds"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payment-service-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: payment-service
    namespace: production
roleRef:
  kind: Role
  name: payment-service-role
  apiGroup: rbac.authorization.k8s.io
EOF

# ─── 3. Pod Security Standards ─────────────────────────────────────────────
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted PSS
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
EOF

# ─── 4. ClusterRole Aggregation with Minimal Permissions ───────────────────
kubectl apply -f - << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-read-only
  labels:
    rbac.example.com/aggregate-to-developer: "true"
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-deploy
  labels:
    rbac.example.com/aggregate-to-developer: "true"
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "update", "patch", "create"]
---
# Aggregate role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.example.com/aggregate-to-developer: "true"
rules: []
EOF

# ─── 5. RBAC Analyzer ──────────────────────────────────────────────────────
cat > /usr/local/bin/rbac-analyzer.py << 'PYEOF'
#!/usr/bin/env python3
"""Analyze Kubernetes RBAC for privilege escalation paths."""

import subprocess
import json
from collections import defaultdict

def get_rolebindings():
    result = subprocess.run(
        ["kubectl", "get", "clusterrolebindings", "-o", "json"],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)["items"]

def get_clusterroles():
    result = subprocess.run(
        ["kubectl", "get", "clusterroles", "-o", "json"],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)["items"]

DANGEROUS_VERBS = {
    "secrets": ["get", "list", "watch", "*"],
    "pods/exec": ["create", "*"],
    "clusterrolebindings": ["create", "update", "patch", "*"],
    "clusterroles": ["create", "update", "patch", "*"],
    "nodes": ["create", "update", "patch", "delete", "*"],
}

def analyze_role(role):
    risks = []
    for rule in role.get("rules", []):
        resources = rule.get("resources", [])
        verbs = rule.get("verbs", [])
        for resource in resources:
            if resource in DANGEROUS_VERBS:
                dangerous = DANGEROUS_VERBS[resource]
                overlap = set(verbs) & set(dangerous)
                if overlap:
                    risks.append({
                        "resource": resource,
                        "verbs": list(overlap),
                        "severity": "HIGH" if resource in ["secrets", "pods/exec"] else "MEDIUM"
                    })
    return risks

def main():
    print("=== RBAC Security Analysis ===\n")
    roles = get_clusterroles()
    bindings = get_rolebindings()

    role_map = {r["metadata"]["name"]: r for r in roles}
    
    # Find roles with dangerous permissions
    risky_roles = {}
    for role in roles:
        risks = analyze_role(role)
        if risks:
            risky_roles[role["metadata"]["name"]] = risks

    # Find who has risky roles
    print("Privilege Escalation Risks:")
    print("-" * 60)
    for binding in bindings:
        role_ref = binding["roleRef"]["name"]
        if role_ref in risky_roles:
            subjects = binding.get("subjects", [])
            risks = risky_roles[role_ref]
            for risk in risks:
                if risk["severity"] == "HIGH":
                    print(f"[HIGH] ClusterRole '{role_ref}'")
                    print(f"  Resource: {risk['resource']}, Verbs: {risk['verbs']}")
                    print(f"  Bound to: {[s.get('name','?') for s in subjects]}")
                    print()

    # Check for wildcard permissions
    print("\nWildcard Permission Analysis:")
    print("-" * 60)
    for role in roles:
        for rule in role.get("rules", []):
            if "*" in rule.get("verbs", []) or "*" in rule.get("resources", []):
                print(f"[WARN] Role '{role['metadata']['name']}' has wildcard permissions")
                print(f"  Rules: {rule}")

if __name__ == "__main__":
    main()
PYEOF
chmod +x /usr/local/bin/rbac-analyzer.py

echo "Running RBAC Analysis..."
python3 /usr/local/bin/rbac-analyzer.py 2>/dev/null || echo "RBAC analysis complete (kubectl required in production)"

echo "=== Step 597 Complete: RBAC Hardening and Audit Logging ==="
SCRIPT
chmod +x advanced-rbac-hardening.sh
echo "Script created: advanced-rbac-hardening.sh"
```

**สิ่งที่เรียนรู้:**
- Kubernetes Audit Policy ระดับ RequestResponse สำหรับ secrets/exec/RBAC changes
- Least-privilege Service Accounts พร้อม IRSA (IAM Roles for Service Accounts)
- Pod Security Standards: enforce `restricted` ใน production namespace
- ClusterRole Aggregation pattern สำหรับ developer permissions
- RBAC Analyzer Python script ตรวจจับ privilege escalation paths

---

## Step 598: Supply Chain Security — SLSA Level 3 + Sigstore

```bash
cat > supply-chain-security.sh << 'SCRIPT'
#!/bin/bash
# Supply Chain Security: SLSA Level 3 + Sigstore + SBOM

set -euo pipefail

echo "=== Supply Chain Security Platform ==="

# ─── 1. Install Sigstore tools ─────────────────────────────────────────────
install_sigstore_tools() {
    echo "Installing Sigstore tools..."
    
    # cosign
    COSIGN_VERSION="v2.4.1"
    curl -sLO "https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64"
    chmod +x cosign-linux-amd64
    mv cosign-linux-amd64 /usr/local/bin/cosign
    
    # syft (SBOM generator)
    curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
    
    # grype (vulnerability scanner)
    curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin
    
    # slsa-verifier
    SLSA_VERSION="v2.6.0"
    curl -sLO "https://github.com/slsa-framework/slsa-verifier/releases/download/${SLSA_VERSION}/slsa-verifier-linux-amd64"
    chmod +x slsa-verifier-linux-amd64
    mv slsa-verifier-linux-amd64 /usr/local/bin/slsa-verifier
    
    echo "Sigstore tools installed"
}

# ─── 2. GitHub Actions SLSA Builder workflow ───────────────────────────────
mkdir -p .github/workflows

cat > .github/workflows/slsa-build.yaml << 'EOF'
name: SLSA Build and Sign

on:
  push:
    tags: ["v*"]
  pull_request:
    branches: [main]

permissions:
  id-token: write      # For OIDC token (keyless signing)
  contents: read
  packages: write
  attestations: write

jobs:
  build:
    name: Build and Generate SBOM
    runs-on: ubuntu-latest
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-uri: ${{ steps.build.outputs.uri }}
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for provenance

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push (with provenance + SBOM)
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          provenance: mode=max      # SLSA Level 3 provenance
          sbom: true                # Attach SBOM to image
          cache-from: type=gha
          cache-to: type=gha,mode=max

  sign:
    name: Sign with Cosign (Keyless)
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign the container image (keyless via OIDC)
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${{ needs.build.outputs.image-digest }}
        env:
          COSIGN_EXPERIMENTAL: "1"

      - name: Generate and attach SBOM
        run: |
          # Generate SBOM with syft
          syft ghcr.io/${{ github.repository }}@${{ needs.build.outputs.image-digest }} \
            -o spdx-json=sbom.spdx.json \
            -o cyclonedx-json=sbom.cyclonedx.json
          
          # Attest SBOM with cosign
          cosign attest --yes \
            --predicate sbom.spdx.json \
            --type spdxjson \
            ghcr.io/${{ github.repository }}@${{ needs.build.outputs.image-digest }}

      - name: Scan for vulnerabilities
        run: |
          grype ghcr.io/${{ github.repository }}@${{ needs.build.outputs.image-digest }} \
            --fail-on high \
            --output table
        continue-on-error: false

  verify-slsa:
    name: Verify SLSA Provenance
    needs: [build, sign]
    runs-on: ubuntu-latest

    steps:
      - name: Install slsa-verifier
        run: |
          curl -sLO https://github.com/slsa-framework/slsa-verifier/releases/download/v2.6.0/slsa-verifier-linux-amd64
          chmod +x slsa-verifier-linux-amd64
          sudo mv slsa-verifier-linux-amd64 /usr/local/bin/slsa-verifier

      - name: Verify SLSA provenance
        run: |
          slsa-verifier verify-image \
            ghcr.io/${{ github.repository }}@${{ needs.build.outputs.image-digest }} \
            --source-uri github.com/${{ github.repository }} \
            --source-branch main
        env:
          COSIGN_EXPERIMENTAL: "1"
EOF

# ─── 3. Admission Controller: Verify signatures before deploy ──────────────
cat > cosign-admission-webhook.yaml << 'EOF'
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionWebhook
metadata:
  name: cosign-verify
webhooks:
  - name: cosign.verify.io
    admissionReviewVersions: ["v1"]
    clientConfig:
      service:
        name: cosign-webhook
        namespace: cosign-system
        path: /validate
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
        scope: "Namespaced"
    namespaceSelector:
      matchLabels:
        cosign-verify: enabled
    failurePolicy: Fail
    sideEffects: None
---
# Policy Contoller from Sigstore
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
    - glob: "ghcr.io/myorg/**"
  authorities:
    - keyless:
        url: https://fulcio.sigstore.dev
        identities:
          - issuer: https://token.actions.githubusercontent.com
            subjectRegExp: "^https://github.com/myorg/.*/.github/workflows/.*@refs/heads/main$"
      ctlog:
        url: https://rekor.sigstore.dev
EOF

# ─── 4. in-toto Link metadata (build provenance chain) ─────────────────────
cat > in-toto-layout.py << 'PYEOF'
#!/usr/bin/env python3
"""in-toto supply chain layout definition."""

# pip install in-toto
from in_toto.models.layout import Layout
from in_toto.models.link import Link
from in_toto.models.inspection import Inspection
from securesystemslib.interface import import_rsa_privatekey_from_file

def create_layout():
    layout = Layout()
    layout.expires = "2026-12-31T00:00:00Z"
    
    # Step 1: Source code checkout
    step_checkout = {
        "name": "checkout",
        "functionaries": [{"type": "pgp", "id": "developer-key-id"}],
        "expected_materials": [],
        "expected_products": [
            ["CREATE", "src/*"],
            ["CREATE", "Dockerfile"],
        ],
        "pubkeys": ["developer_pubkey.pub"],
    }
    
    # Step 2: Build container
    step_build = {
        "name": "build",
        "functionaries": [{"type": "pgp", "id": "ci-key-id"}],
        "expected_materials": [
            ["MATCH", "src/*", "WITH", "PRODUCTS", "FROM", "checkout"],
            ["MATCH", "Dockerfile", "WITH", "PRODUCTS", "FROM", "checkout"],
        ],
        "expected_products": [
            ["CREATE", "image.tar"],
        ],
    }
    
    # Step 3: Scan
    step_scan = {
        "name": "scan",
        "functionaries": [{"type": "pgp", "id": "scanner-key-id"}],
        "expected_materials": [
            ["MATCH", "image.tar", "WITH", "PRODUCTS", "FROM", "build"],
        ],
        "expected_products": [
            ["CREATE", "scan-report.json"],
        ],
    }
    
    # Inspection: Verify no HIGH vulnerabilities
    inspection = {
        "name": "no-high-vulns",
        "run": ["python3", "-c",
            "import json; r=json.load(open('scan-report.json')); "
            "assert not any(m['severity']=='HIGH' for m in r.get('matches',[])), 'HIGH vulns found'"
        ],
    }
    
    print("in-toto layout defined")
    print("Steps:", ["checkout", "build", "scan"])
    print("Inspections:", ["no-high-vulns"])
    return layout

if __name__ == "__main__":
    layout = create_layout()
PYEOF

# ─── 5. SBOM Policy Enforcement ────────────────────────────────────────────
cat > sbom-policy.py << 'PYEOF'
#!/usr/bin/env python3
"""Enforce SBOM policies: license compliance, vulnerability thresholds."""

import json
import sys
from pathlib import Path
from dataclasses import dataclass
from typing import List

@dataclass
class PolicyResult:
    passed: bool
    violations: List[str]
    warnings: List[str]

FORBIDDEN_LICENSES = {
    "GPL-2.0", "GPL-3.0", "AGPL-3.0",
    "CDDL-1.0", "EPL-1.0", "EPL-2.0",
}

ALLOWED_LICENSES = {
    "MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause",
    "ISC", "Unlicense", "0BSD",
}

class SBOMPolicyEnforcer:
    def __init__(self, sbom_path: str, grype_report_path: str):
        with open(sbom_path) as f:
            self.sbom = json.load(f)
        with open(grype_report_path) as f:
            self.grype = json.load(f)

    def check_licenses(self) -> PolicyResult:
        violations = []
        warnings = []
        packages = self.sbom.get("packages", [])
        
        for pkg in packages:
            for lic_expr in pkg.get("licenseConcluded", "").split(" AND "):
                lic = lic_expr.strip()
                if lic in FORBIDDEN_LICENSES:
                    violations.append(
                        f"Package '{pkg['name']}' uses forbidden license: {lic}"
                    )
                elif lic not in ALLOWED_LICENSES and lic not in ("NOASSERTION", "NONE", ""):
                    warnings.append(
                        f"Package '{pkg['name']}' uses unreviewed license: {lic}"
                    )
        
        return PolicyResult(
            passed=len(violations) == 0,
            violations=violations,
            warnings=warnings,
        )

    def check_vulnerabilities(self, max_critical=0, max_high=5) -> PolicyResult:
        violations = []
        warnings = []
        matches = self.grype.get("matches", [])
        
        critical_count = sum(
            1 for m in matches
            if m.get("vulnerability", {}).get("severity") == "Critical"
        )
        high_count = sum(
            1 for m in matches
            if m.get("vulnerability", {}).get("severity") == "High"
        )
        
        if critical_count > max_critical:
            violations.append(
                f"Found {critical_count} CRITICAL vulnerabilities (max: {max_critical})"
            )
        if high_count > max_high:
            violations.append(
                f"Found {high_count} HIGH vulnerabilities (max: {max_high})"
            )
        
        # List critical CVEs
        for match in matches:
            vuln = match.get("vulnerability", {})
            if vuln.get("severity") == "Critical":
                pkg = match.get("artifact", {})
                violations.append(
                    f"  CVE: {vuln.get('id')} in {pkg.get('name')}@{pkg.get('version')}"
                )
        
        return PolicyResult(
            passed=len(violations) == 0,
            violations=violations,
            warnings=warnings,
        )

    def enforce(self) -> bool:
        print("=== SBOM Policy Enforcement ===\n")
        
        all_passed = True
        
        # License check
        lic_result = self.check_licenses()
        print(f"License Policy: {'PASS' if lic_result.passed else 'FAIL'}")
        for v in lic_result.violations:
            print(f"  [VIOLATION] {v}")
        for w in lic_result.warnings:
            print(f"  [WARN] {w}")
        
        # Vulnerability check
        vuln_result = self.check_vulnerabilities()
        print(f"\nVulnerability Policy: {'PASS' if vuln_result.passed else 'FAIL'}")
        for v in vuln_result.violations:
            print(f"  [VIOLATION] {v}")
        
        all_passed = lic_result.passed and vuln_result.passed
        
        print(f"\n{'✓ All policies passed' if all_passed else '✗ Policy violations found'}")
        return all_passed

# Demo
import tempfile, os

demo_sbom = {
    "spdxVersion": "SPDX-2.3",
    "packages": [
        {"name": "requests", "licenseConcluded": "Apache-2.0", "versionInfo": "2.31.0"},
        {"name": "cryptography", "licenseConcluded": "Apache-2.0", "versionInfo": "41.0.0"},
        {"name": "old-gpl-lib", "licenseConcluded": "GPL-2.0", "versionInfo": "1.0.0"},
    ]
}

demo_grype = {
    "matches": [
        {
            "vulnerability": {"id": "CVE-2023-9999", "severity": "High"},
            "artifact": {"name": "old-lib", "version": "1.0.0"}
        }
    ]
}

with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False) as sf:
    json.dump(demo_sbom, sf)
    sbom_file = sf.name

with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False) as gf:
    json.dump(demo_grype, gf)
    grype_file = gf.name

enforcer = SBOMPolicyEnforcer(sbom_file, grype_file)
passed = enforcer.enforce()
os.unlink(sbom_file)
os.unlink(grype_file)
sys.exit(0 if passed else 1)
PYEOF

python3 sbom-policy.py || true

echo "=== Step 598 Complete: Supply Chain Security SLSA Level 3 ==="
SCRIPT
chmod +x supply-chain-security.sh
bash supply-chain-security.sh 2>/dev/null || true
echo "Script created: supply-chain-security.sh"
```

**สิ่งที่เรียนรู้:**
- SLSA Level 3 provenance ด้วย GitHub Actions `docker/build-push-action` provenance=max
- Keyless signing ด้วย Cosign + Sigstore Fulcio/Rekor (OIDC-based)
- SBOM generation (SPDX + CycloneDX) และ attestation
- Sigstore Policy Controller: `ClusterImagePolicy` ที่ enforce เฉพาะ images จาก verified workflow
- in-toto supply chain layout สำหรับ provenance chain
- SBOM Policy Enforcer: license compliance + vulnerability thresholds

---

## Step 599: Runtime Security — Advanced Falco Rules + eBPF

```bash
cat > runtime-security.sh << 'SCRIPT'
#!/bin/bash
# Runtime Security: Advanced Falco Rules + eBPF Monitoring

set -euo pipefail

echo "=== Runtime Security Platform ==="

# ─── 1. Install Falco with eBPF probe ──────────────────────────────────────
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm upgrade --install falco falcosecurity/falco \
  --namespace falco-system \
  --create-namespace \
  --values - << 'EOF'
driver:
  kind: ebpf           # Use eBPF (preferred over kernel module)
  ebpf:
    leastPrivileged: true

falco:
  grpc:
    enabled: true
  grpcOutput:
    enabled: true
  jsonOutput: true
  logLevel: info

falcoctl:
  artifact:
    install:
      enabled: true
      refs:
        - falco-rules:3         # Official ruleset v3
        - falco-incubating-rules:3

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi

tolerations:
  - operator: Exists   # Run on all nodes including masters
EOF

# ─── 2. Custom Falco Rules ─────────────────────────────────────────────────
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-custom-rules
  namespace: falco-system
data:
  custom-rules.yaml: |
    # ── Cryptocurrency Mining Detection ─────────────────────────────────────
    - macro: crypto_miners
      condition: >
        proc.name in (xmrig, minerd, cpuminer, ethminer, t-rex, nbminer,
                      lolminer, nanominer, gminer, bzminer)

    - rule: Cryptomining Activity Detected
      desc: Process known for cryptocurrency mining started
      condition: spawned_process and crypto_miners
      output: >
        Cryptominer process started
        (user=%user.name user_loginuid=%user.loginuid
         command=%proc.cmdline pid=%proc.pid
         container=%container.name image=%container.image.repository
         k8s_ns=%k8s.ns.name k8s_pod=%k8s.pod.name)
      priority: CRITICAL
      tags: [cryptomining, T1496]

    # ── Container Escape Attempts ────────────────────────────────────────────
    - rule: Container Namespace Escape Attempt
      desc: Process attempting to escape container via namespace manipulation
      condition: >
        (syscall.type = setns or syscall.type = unshare) and
        container and not proc.name in (runc, containerd-shim, dockerd)
      output: >
        Namespace escape attempt
        (syscall=%syscall.type user=%user.name
         container=%container.name image=%container.image.repository
         proc=%proc.cmdline k8s_pod=%k8s.pod.name)
      priority: CRITICAL
      tags: [escape, T1611]

    - rule: Mount Sensitive Host Path
      desc: Container mounting sensitive host filesystem paths
      condition: >
        evt.type = mount and
        evt.dir = < and
        container and
        (evt.arg.dev contains "/proc" or
         evt.arg.dev contains "/sys" or
         evt.arg.dev contains "/host" or
         fd.name startswith "/dev/sd")
      output: >
        Sensitive host path mounted
        (path=%fd.name container=%container.name k8s_pod=%k8s.pod.name)
      priority: HIGH
      tags: [escape, mount]

    # ── Data Exfiltration ────────────────────────────────────────────────────
    - macro: exfil_tools
      condition: proc.name in (curl, wget, nc, ncat, netcat, socat, scp, rsync)

    - rule: Sensitive File Read and Exfiltration
      desc: Sensitive file read followed by network connection
      condition: >
        open_read and
        fd.name pmatch (/etc/passwd, /etc/shadow, /etc/kubernetes/,
                         /var/run/secrets/, /.ssh/, /root/.ssh/) and
        container
      output: >
        Sensitive file read in container
        (file=%fd.name user=%user.name
         command=%proc.cmdline container=%container.name
         image=%container.image.repository k8s_pod=%k8s.pod.name)
      priority: HIGH
      tags: [exfil, T1552]

    # ── Reverse Shell Detection ──────────────────────────────────────────────
    - rule: Reverse Shell Detected
      desc: Shell spawned with stdin/stdout redirected to network socket
      condition: >
        evt.type = execve and
        proc.name in (bash, sh, zsh, fish, dash) and
        (fd.num < 3 and fd.typechar = 4) and  # fd type 4 = socket
        container
      output: >
        Reverse shell spawned
        (shell=%proc.name pid=%proc.pid parent=%proc.pname
         container=%container.name image=%container.image.repository
         k8s_ns=%k8s.ns.name k8s_pod=%k8s.pod.name)
      priority: CRITICAL
      tags: [shell, T1059]

    # ── Kubernetes API Abuse ─────────────────────────────────────────────────
    - rule: K8s Secret Enumeration from Pod
      desc: kubectl or API call enumerating secrets from inside a pod
      condition: >
        spawned_process and
        (proc.name = kubectl or
         (proc.name in (curl, wget) and proc.cmdline contains "/api/v1/secrets")) and
        container
      output: >
        Secret enumeration from pod
        (command=%proc.cmdline user=%user.name
         container=%container.name k8s_pod=%k8s.pod.name)
      priority: HIGH
      tags: [k8s, secrets, T1552.007]

    # ── Malware Indicators ───────────────────────────────────────────────────
    - rule: Linux Kernel Module Injection
      desc: Attempt to load kernel module from container
      condition: >
        (evt.type = init_module or evt.type = finit_module) and container
      output: >
        Kernel module injection attempt
        (proc=%proc.cmdline user=%user.name
         container=%container.name k8s_pod=%k8s.pod.name)
      priority: CRITICAL
      tags: [rootkit, T1215]

    - rule: Fileless Execution via memfd
      desc: Execution of code via anonymous memory (fileless malware pattern)
      condition: >
        evt.type = memfd_create and container
      output: >
        Fileless execution via memfd_create
        (proc=%proc.cmdline user=%user.name
         container=%container.name k8s_pod=%k8s.pod.name)
      priority: CRITICAL
      tags: [malware, fileless, T1620]
EOF

# ─── 3. Falco Sidekick — Alert Routing ─────────────────────────────────────
helm upgrade --install falcosidekick falcosecurity/falcosidekick \
  --namespace falco-system \
  --set config.slack.webhookurl="https://hooks.slack.com/services/XXX/YYY/ZZZ" \
  --set config.slack.minimumpriority=high \
  --set config.pagerduty.routingkey="${PAGERDUTY_KEY:-demo}" \
  --set config.pagerduty.minimumpriority=critical \
  --set config.elasticsearch.hostport="http://elasticsearch:9200" \
  --set config.elasticsearch.index=falco \
  --set webui.enabled=true

# ─── 4. Tetragon — eBPF Runtime Enforcement ────────────────────────────────
helm repo add cilium https://helm.cilium.io
helm upgrade --install tetragon cilium/tetragon \
  --namespace kube-system \
  --set tetragon.enablePolicyFilter=true

kubectl apply -f - << 'EOF'
# Prevent write to /etc from any process in pods
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: block-etc-writes
spec:
  kprobes:
    - call: "security_file_permission"
      syscall: false
      args:
        - index: 0
          type: "file"
        - index: 1
          type: "int"
      selectors:
        - matchArgs:
            - index: 0
              operator: "Prefix"
              values:
                - "/etc"
            - index: 1
              operator: "Equal"
              values:
                - "2"  # MAY_WRITE
          matchNamespaces:
            - namespace: NetNS
              operator: NotIn
              values:
                - "host_net_ns"
          matchActions:
            - action: Sigkill  # Kill process attempting write
---
# Prevent ptrace (debugger/injection attacks)
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: block-ptrace
spec:
  kprobes:
    - call: "sys_ptrace"
      syscall: true
      args:
        - index: 0
          type: "int"
      selectors:
        - matchArgs:
            - index: 0
              operator: "NotIn"
              values:
                - "0"  # PTRACE_TRACEME (allow child→parent)
          matchCapabilities:
            - type: Effective
              operator: NotIn
              values:
                - "CAP_SYS_PTRACE"
          matchActions:
            - action: Sigkill
EOF

# ─── 5. Runtime Security Aggregator ────────────────────────────────────────
cat > runtime-security-aggregator.py << 'PYEOF'
#!/usr/bin/env python3
"""Aggregate Falco + Tetragon alerts and correlate into incidents."""

import json
import time
import asyncio
import logging
from datetime import datetime, timedelta
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Dict, List, Optional

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class SecurityEvent:
    timestamp: datetime
    priority: str
    rule: str
    source: str  # falco or tetragon
    pod: str
    namespace: str
    container: str
    details: Dict
    tags: List[str] = field(default_factory=list)

@dataclass
class SecurityIncident:
    id: str
    start_time: datetime
    events: List[SecurityEvent] = field(default_factory=list)
    pod: str = ""
    namespace: str = ""
    severity: str = "LOW"
    attack_chain: List[str] = field(default_factory=list)
    auto_remediated: bool = False

# MITRE ATT&CK kill-chain patterns
ATTACK_CHAINS = {
    "container_escape": [
        "Mount Sensitive Host Path",
        "Container Namespace Escape Attempt",
        "Linux Kernel Module Injection",
    ],
    "data_exfiltration": [
        "Sensitive File Read and Exfiltration",
        "K8s Secret Enumeration from Pod",
    ],
    "cryptomining": [
        "Cryptomining Activity Detected",
    ],
    "reverse_shell": [
        "Reverse Shell Detected",
        "Sensitive File Read and Exfiltration",
    ],
}

class RuntimeSecurityAggregator:
    def __init__(self):
        self.events_by_pod: Dict[str, List[SecurityEvent]] = defaultdict(list)
        self.incidents: Dict[str, SecurityIncident] = {}
        self.correlation_window = timedelta(minutes=15)

    def ingest_falco_event(self, raw_event: Dict) -> SecurityEvent:
        event = SecurityEvent(
            timestamp=datetime.utcnow(),
            priority=raw_event.get("priority", "INFO"),
            rule=raw_event.get("rule", ""),
            source="falco",
            pod=raw_event.get("output_fields", {}).get("k8s.pod.name", "unknown"),
            namespace=raw_event.get("output_fields", {}).get("k8s.ns.name", "default"),
            container=raw_event.get("output_fields", {}).get("container.name", ""),
            details=raw_event.get("output_fields", {}),
            tags=raw_event.get("tags", []),
        )
        return self._process_event(event)

    def _process_event(self, event: SecurityEvent) -> SecurityEvent:
        key = f"{event.namespace}/{event.pod}"
        self.events_by_pod[key].append(event)
        
        # Prune old events
        cutoff = datetime.utcnow() - self.correlation_window
        self.events_by_pod[key] = [
            e for e in self.events_by_pod[key]
            if e.timestamp > cutoff
        ]
        
        # Check for attack chain patterns
        pod_rules = {e.rule for e in self.events_by_pod[key]}
        for chain_name, chain_rules in ATTACK_CHAINS.items():
            matched = [r for r in chain_rules if r in pod_rules]
            if len(matched) >= 2:
                self._create_or_update_incident(event, chain_name, matched)
        
        # Immediate critical actions
        if event.priority == "CRITICAL":
            self._handle_critical_event(event)
        
        return event

    def _handle_critical_event(self, event: SecurityEvent):
        logger.warning(
            f"CRITICAL security event: {event.rule} in {event.namespace}/{event.pod}"
        )
        
        if event.rule in ["Cryptomining Activity Detected", "Reverse Shell Detected"]:
            logger.critical(
                f"AUTO-REMEDIATION: Isolating pod {event.pod} in namespace {event.namespace}"
            )
            self._isolate_pod(event.namespace, event.pod)

    def _isolate_pod(self, namespace: str, pod: str):
        """Apply deny-all NetworkPolicy to isolate compromised pod."""
        import subprocess
        
        # Label pod for isolation
        subprocess.run([
            "kubectl", "label", "pod", pod,
            "-n", namespace,
            "security.example.com/isolated=true",
            "--overwrite"
        ], capture_output=True)
        
        # Apply deny-all NetworkPolicy for labeled pods
        np = {
            "apiVersion": "networking.k8s.io/v1",
            "kind": "NetworkPolicy",
            "metadata": {
                "name": f"isolate-{pod}",
                "namespace": namespace,
            },
            "spec": {
                "podSelector": {
                    "matchLabels": {"security.example.com/isolated": "true"}
                },
                "policyTypes": ["Ingress", "Egress"],
            }
        }
        
        import tempfile, os
        with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False) as f:
            json.dump(np, f)
            tmp = f.name
        
        subprocess.run(["kubectl", "apply", "-f", tmp], capture_output=True)
        os.unlink(tmp)
        logger.info(f"Pod {pod} isolated with deny-all NetworkPolicy")

    def _create_or_update_incident(
        self,
        event: SecurityEvent,
        chain_name: str,
        matched_rules: List[str]
    ):
        incident_id = f"{event.namespace}/{event.pod}/{chain_name}"
        
        if incident_id not in self.incidents:
            incident = SecurityIncident(
                id=incident_id,
                start_time=datetime.utcnow(),
                pod=event.pod,
                namespace=event.namespace,
                severity="CRITICAL" if chain_name in ["container_escape", "reverse_shell"] else "HIGH",
                attack_chain=[chain_name],
            )
            self.incidents[incident_id] = incident
            logger.critical(
                f"ATTACK CHAIN DETECTED: {chain_name} in {event.namespace}/{event.pod} "
                f"rules_matched={matched_rules}"
            )

    def get_incident_summary(self) -> Dict:
        return {
            "total_incidents": len(self.incidents),
            "incidents": [
                {
                    "id": i.id,
                    "severity": i.severity,
                    "attack_chain": i.attack_chain,
                    "start_time": i.start_time.isoformat(),
                }
                for i in self.incidents.values()
            ]
        }

# Demo
aggregator = RuntimeSecurityAggregator()

# Simulate events
demo_events = [
    {
        "rule": "Mount Sensitive Host Path",
        "priority": "HIGH",
        "output_fields": {
            "k8s.pod.name": "compromised-pod-xyz",
            "k8s.ns.name": "production",
            "container.name": "app",
            "fd.name": "/proc/1/mem",
        },
        "tags": ["escape", "mount"],
    },
    {
        "rule": "Container Namespace Escape Attempt",
        "priority": "CRITICAL",
        "output_fields": {
            "k8s.pod.name": "compromised-pod-xyz",
            "k8s.ns.name": "production",
            "container.name": "app",
        },
        "tags": ["escape"],
    },
    {
        "rule": "Cryptomining Activity Detected",
        "priority": "CRITICAL",
        "output_fields": {
            "k8s.pod.name": "miner-pod-abc",
            "k8s.ns.name": "default",
            "container.name": "worker",
            "proc.name": "xmrig",
        },
        "tags": ["cryptomining"],
    },
]

print("=== Runtime Security Aggregator Demo ===\n")
for evt in demo_events:
    aggregator.ingest_falco_event(evt)

summary = aggregator.get_incident_summary()
print(f"\nIncident Summary: {json.dumps(summary, indent=2)}")
PYEOF

python3 runtime-security-aggregator.py

echo "=== Step 599 Complete: Runtime Security with Falco + eBPF ==="
SCRIPT
chmod +x runtime-security.sh
echo "Script created: runtime-security.sh"
```

**สิ่งที่เรียนรู้:**
- Falco eBPF probe (preferred over kernel module) ด้วย leastPrivileged mode
- Custom Falco rules: cryptomining, container escape, reverse shell, fileless execution, ptrace blocking
- Falco Sidekick alert routing ไปยัง Slack/PagerDuty/Elasticsearch
- Tetragon TracingPolicy: Sigkill process ที่พยายาม write /etc หรือใช้ ptrace
- RuntimeSecurityAggregator: MITRE ATT&CK kill-chain correlation (15-min window), auto-isolate compromised pods ด้วย NetworkPolicy

---

## Step 600: Chaos Engineering — Production-Grade Resilience Testing

```bash
cat > chaos-engineering.sh << 'SCRIPT'
#!/bin/bash
# Chaos Engineering: LitmusChaos + Custom Experiments

set -euo pipefail

echo "=== Chaos Engineering Platform (Step 600 - Milestone!) ==="

# ─── 1. Install LitmusChaos ────────────────────────────────────────────────
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update

helm upgrade --install litmus litmuschaos/litmus \
  --namespace litmus \
  --create-namespace \
  --set portal.frontend.service.type=ClusterIP \
  --set portal.server.graphqlServer.replicaCount=2

# Install Chaos Operator
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v3.11.0.yaml

# Install ChaosHub experiments
kubectl apply -f - << 'EOF'
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosHub
metadata:
  name: litmus-hub
  namespace: litmus
spec:
  connectorType: PUBLIC
  repoURL: https://github.com/litmuschaos/chaos-charts
  repoBranch: master
  name: Litmus ChaosHub
EOF

# ─── 2. Pod Chaos Experiments ─────────────────────────────────────────────
kubectl apply -f - << 'EOF'
# Experiment: Kill payment service pods
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: payment-service-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: app=payment-service
    appkind: deployment
  chaosServiceAccount: litmus-admin
  monitoring: true
  annotationCheck: "true"
  engineState: active
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"        # 1 minute
            - name: CHAOS_INTERVAL
              value: "10"        # Kill a pod every 10 seconds
            - name: FORCE
              value: "false"
            - name: PODS_AFFECTED_PERC
              value: "50"        # Kill 50% of pods
        probe:
          - name: check-payment-availability
            type: httpProbe
            httpProbe/inputs:
              url: "http://payment-service.production.svc.cluster.local/health"
              insecureSkipVerify: false
              method:
                get:
                  criteria: "=="
                  responseCode: "200"
            mode: Continuous
            runProperties:
              probeTimeout: 5
              interval: 2
              retry: 1
              probePollingInterval: 2
---
# Experiment: Network latency for database connections
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: db-network-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: app=payment-service
    appkind: deployment
  chaosServiceAccount: litmus-admin
  engineState: active
  experiments:
    - name: pod-network-latency
      spec:
        components:
          env:
            - name: NETWORK_INTERFACE
              value: eth0
            - name: TARGET_CONTAINER
              value: payment-service
            - name: NETWORK_LATENCY
              value: "500"       # 500ms latency
            - name: JITTER
              value: "100"       # ±100ms jitter
            - name: TOTAL_CHAOS_DURATION
              value: "120"
            - name: DESTINATION_HOSTS
              value: "postgresql.production.svc.cluster.local"
        probe:
          - name: check-timeout-handling
            type: httpProbe
            httpProbe/inputs:
              url: "http://payment-service.production.svc.cluster.local/api/v1/health/db"
              method:
                get:
                  criteria: "!="
                  responseCode: "500"
            mode: Continuous
            runProperties:
              probeTimeout: 2
              interval: 5
              retry: 2
---
# Experiment: CPU stress to test HPA
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: cpu-stress-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: app=api-gateway
    appkind: deployment
  chaosServiceAccount: litmus-admin
  engineState: active
  experiments:
    - name: pod-cpu-hog
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "180"
            - name: CPU_CORES
              value: "2"
            - name: PODS_AFFECTED_PERC
              value: "25"
        probe:
          - name: check-hpa-scaling
            type: k8sProbe
            k8sProbe/inputs:
              group: apps
              version: v1
              resource: deployments
              namespace: production
              resourceNames: api-gateway
              fieldSelector: "status.readyReplicas>=3"
              operation: present
            mode: Edge
            runProperties:
              probeTimeout: 60
              interval: 15
              retry: 3
EOF

# ─── 3. GameDay Scenario Orchestrator ──────────────────────────────────────
cat > gameday-orchestrator.py << 'PYEOF'
#!/usr/bin/env python3
"""Orchestrate complex multi-failure GameDay scenarios."""

import asyncio
import json
import time
import logging
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import List, Callable, Optional
from enum import Enum

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
logger = logging.getLogger(__name__)

class ChaosType(Enum):
    POD_DELETE = "pod-delete"
    NETWORK_LATENCY = "network-latency"
    CPU_HOG = "cpu-hog"
    MEMORY_HOG = "memory-hog"
    DISK_FILL = "disk-fill"
    NODE_DRAIN = "node-drain"
    AZ_FAILURE = "az-failure"

@dataclass
class ChaosScenario:
    name: str
    chaos_type: ChaosType
    target_namespace: str
    target_label: str
    duration_seconds: int
    intensity: float  # 0.0 - 1.0
    expected_recovery_seconds: int
    slo_threshold: float  # minimum acceptable availability

@dataclass
class GameDayResult:
    scenario: ChaosScenario
    start_time: datetime
    end_time: Optional[datetime] = None
    max_latency_ms: float = 0
    min_availability: float = 1.0
    error_rate_peak: float = 0.0
    recovery_time_seconds: float = 0.0
    slo_breached: bool = False
    issues_found: List[str] = field(default_factory=list)

class GameDayOrchestrator:
    def __init__(self, prometheus_url: str = "http://prometheus:9090"):
        self.prometheus_url = prometheus_url
        self.results: List[GameDayResult] = []

    async def run_scenario(self, scenario: ChaosScenario) -> GameDayResult:
        result = GameDayResult(
            scenario=scenario,
            start_time=datetime.utcnow(),
        )
        
        logger.info(f"[GAMEDAY] Starting scenario: {scenario.name}")
        logger.info(f"  Type: {scenario.chaos_type.value}")
        logger.info(f"  Target: {scenario.target_namespace}/{scenario.target_label}")
        logger.info(f"  Duration: {scenario.duration_seconds}s")
        logger.info(f"  SLO threshold: {scenario.slo_threshold*100:.1f}%")
        
        # Phase 1: Establish baseline
        logger.info("Phase 1: Establishing baseline metrics (60s)...")
        baseline = await self._collect_metrics(scenario, duration=10)
        logger.info(f"  Baseline availability: {baseline['availability']:.3f}")
        logger.info(f"  Baseline p99 latency: {baseline['p99_latency_ms']:.1f}ms")
        
        # Phase 2: Inject chaos
        logger.info(f"Phase 2: Injecting chaos ({scenario.chaos_type.value})...")
        chaos_start = time.time()
        await self._inject_chaos(scenario)
        
        # Phase 3: Monitor during chaos
        logger.info("Phase 3: Monitoring during chaos...")
        monitoring_interval = 5  # seconds
        elapsed = 0
        
        while elapsed < scenario.duration_seconds:
            metrics = await self._collect_metrics(scenario, duration=monitoring_interval)
            
            result.min_availability = min(result.min_availability, metrics["availability"])
            result.max_latency_ms = max(result.max_latency_ms, metrics["p99_latency_ms"])
            result.error_rate_peak = max(result.error_rate_peak, metrics["error_rate"])
            
            if metrics["availability"] < scenario.slo_threshold:
                result.slo_breached = True
                result.issues_found.append(
                    f"SLO breached at t+{elapsed}s: "
                    f"availability={metrics['availability']:.3f} < {scenario.slo_threshold}"
                )
            
            if metrics["p99_latency_ms"] > 1000:
                result.issues_found.append(
                    f"High latency at t+{elapsed}s: p99={metrics['p99_latency_ms']:.0f}ms"
                )
            
            elapsed += monitoring_interval
            await asyncio.sleep(monitoring_interval)
        
        # Phase 4: Stop chaos and measure recovery
        logger.info("Phase 4: Stopping chaos, measuring recovery...")
        await self._stop_chaos(scenario)
        recovery_start = time.time()
        
        for _ in range(scenario.expected_recovery_seconds // 5):
            await asyncio.sleep(5)
            metrics = await self._collect_metrics(scenario, duration=5)
            if metrics["availability"] >= 0.999:
                result.recovery_time_seconds = time.time() - recovery_start
                break
        else:
            result.issues_found.append(
                f"Recovery timeout: service did not recover within "
                f"{scenario.expected_recovery_seconds}s"
            )
            result.recovery_time_seconds = scenario.expected_recovery_seconds
        
        result.end_time = datetime.utcnow()
        self._log_result(result)
        self.results.append(result)
        return result

    async def _collect_metrics(self, scenario: ChaosScenario, duration: int) -> dict:
        """Simulate metrics collection (real impl queries Prometheus)."""
        import random
        # In production: query Prometheus API
        return {
            "availability": random.uniform(0.98, 1.0),
            "p99_latency_ms": random.uniform(50, 500),
            "error_rate": random.uniform(0, 0.02),
        }

    async def _inject_chaos(self, scenario: ChaosScenario):
        """Inject chaos via LitmusChaos API or kubectl."""
        import subprocess
        logger.info(f"Injecting {scenario.chaos_type.value} into {scenario.target_namespace}")
        # Real impl: kubectl apply ChaosEngine YAML or call LitmusChaos GraphQL API

    async def _stop_chaos(self, scenario: ChaosScenario):
        """Stop chaos experiment."""
        logger.info("Stopping chaos injection")
        # Real impl: kubectl patch ChaosEngine engineState=stop

    def _log_result(self, result: GameDayResult):
        status = "PASSED" if not result.slo_breached else "FAILED"
        logger.info(f"\n{'='*60}")
        logger.info(f"GameDay Result: {result.scenario.name} — {status}")
        logger.info(f"  Min availability: {result.min_availability:.4f}")
        logger.info(f"  Max p99 latency: {result.max_latency_ms:.1f}ms")
        logger.info(f"  Peak error rate: {result.error_rate_peak:.4f}")
        logger.info(f"  Recovery time: {result.recovery_time_seconds:.1f}s")
        if result.issues_found:
            logger.info(f"  Issues found ({len(result.issues_found)}):")
            for issue in result.issues_found:
                logger.info(f"    - {issue}")

    def generate_report(self) -> dict:
        return {
            "gameday_date": datetime.utcnow().isoformat(),
            "total_scenarios": len(self.results),
            "passed": sum(1 for r in self.results if not r.slo_breached),
            "failed": sum(1 for r in self.results if r.slo_breached),
            "scenarios": [
                {
                    "name": r.scenario.name,
                    "status": "PASSED" if not r.slo_breached else "FAILED",
                    "min_availability": r.min_availability,
                    "max_latency_ms": r.max_latency_ms,
                    "recovery_seconds": r.recovery_time_seconds,
                    "issues": r.issues_found,
                }
                for r in self.results
            ]
        }

# Define GameDay scenarios
GAMEDAY_SCENARIOS = [
    ChaosScenario(
        name="Payment Service Pod Failure",
        chaos_type=ChaosType.POD_DELETE,
        target_namespace="production",
        target_label="app=payment-service",
        duration_seconds=60,
        intensity=0.5,
        expected_recovery_seconds=30,
        slo_threshold=0.999,
    ),
    ChaosScenario(
        name="Database Network Partition",
        chaos_type=ChaosType.NETWORK_LATENCY,
        target_namespace="production",
        target_label="app=postgresql",
        duration_seconds=120,
        intensity=0.8,
        expected_recovery_seconds=60,
        slo_threshold=0.99,
    ),
    ChaosScenario(
        name="API Gateway CPU Exhaustion",
        chaos_type=ChaosType.CPU_HOG,
        target_namespace="production",
        target_label="app=api-gateway",
        duration_seconds=180,
        intensity=0.75,
        expected_recovery_seconds=90,
        slo_threshold=0.995,
    ),
]

async def main():
    orchestrator = GameDayOrchestrator()
    
    print("=== GameDay Chaos Engineering Exercise ===\n")
    print(f"Scenarios planned: {len(GAMEDAY_SCENARIOS)}")
    
    for scenario in GAMEDAY_SCENARIOS:
        await orchestrator.run_scenario(scenario)
        await asyncio.sleep(30)  # Cool-down between scenarios
    
    report = orchestrator.generate_report()
    print(f"\n{'='*60}")
    print(f"GAMEDAY SUMMARY")
    print(f"  Total scenarios: {report['total_scenarios']}")
    print(f"  Passed: {report['passed']}")
    print(f"  Failed: {report['failed']}")
    print(f"{'='*60}")

asyncio.run(main())
PYEOF

python3 gameday-orchestrator.py

echo "=== Step 600 Complete: Chaos Engineering Platform (MILESTONE!) ==="
SCRIPT
chmod +x chaos-engineering.sh
echo "Script created: chaos-engineering.sh"
```

**สิ่งที่เรียนรู้:**
- LitmusChaos: ChaosEngine CRD, httpProbe (Continuous), k8sProbe (Edge)
- Experiments: pod-delete (50% pods), network-latency (500ms+100ms jitter ไป PostgreSQL), cpu-hog (25% pods, test HPA)
- GameDay Orchestrator: 4-phase (baseline → inject → monitor → recovery), SLO threshold enforcement
- Recovery time measurement, multi-scenario report generation

---

## สรุป Part 62

| Step | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|----------------|
| 597 | Advanced RBAC Hardening | Audit Policy, Least-Privilege SA, Pod Security Standards, RBAC Analyzer |
| 598 | Supply Chain Security | SLSA Level 3, Cosign Keyless, SBOM Policy, in-toto layout |
| 599 | Runtime Security | Falco eBPF, Tetragon Sigkill, Alert Correlation, Pod Isolation |
| 600 | Chaos Engineering | LitmusChaos, GameDay Orchestrator, SLO-aware testing |

**ขั้นตอนต่อไป: Part 63 — Platform Engineering, Developer Experience และ Internal Developer Platform (IDP)**
