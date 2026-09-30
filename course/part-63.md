# Part 63: Platform Engineering, Developer Experience และ Internal Developer Platform (IDP)

## ภาพรวม
Part นี้ครอบคลุมการสร้าง Internal Developer Platform (IDP) ระดับ World-Class ด้วย Backstage, Port, และ Platform Automation

---

## Step 601: Backstage IDP — Service Catalog + Templates

```bash
cat > backstage-idp.sh << 'SCRIPT'
#!/bin/bash
# Backstage Internal Developer Platform

set -euo pipefail

echo "=== Backstage Internal Developer Platform ==="

# ─── 1. Create Backstage App ───────────────────────────────────────────────
npx @backstage/create-app@latest --name platform-idp --path ./backstage 2>/dev/null || \
  echo "Backstage creation skipped (npx required)"

# ─── 2. App Configuration ─────────────────────────────────────────────────
mkdir -p backstage/app-config

cat > backstage/app-config/app-config.production.yaml << 'EOF'
app:
  title: Platform IDP
  baseUrl: https://backstage.internal.example.com

organization:
  name: Engineering Platform

backend:
  baseUrl: https://backstage.internal.example.com
  listen:
    port: 7007
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: 5432
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}

auth:
  environment: production
  providers:
    github:
      production:
        clientId: ${AUTH_GITHUB_CLIENT_ID}
        clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}
    google:
      production:
        clientId: ${AUTH_GOOGLE_CLIENT_ID}
        clientSecret: ${AUTH_GOOGLE_CLIENT_SECRET}

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

catalog:
  import:
    entityFilename: catalog-info.yaml
    pullRequestBranchName: backstage-integration
  rules:
    - allow:
        [Component, System, API, Resource, Location, Template, User, Group]
  locations:
    # Platform-owned services
    - type: url
      target: https://github.com/myorg/platform-catalog/blob/main/catalog-info.yaml
      rules:
        - allow: [Location, Component, System]
    # Teams
    - type: url
      target: https://github.com/myorg/org-catalog/blob/main/users.yaml

kubernetes:
  serviceLocatorMethod:
    type: multiTenant
  clusterLocatorMethods:
    - type: config
      clusters:
        - url: https://k8s-prod.internal.example.com
          name: production
          authProvider: serviceAccount
          serviceAccountToken: ${K8S_PROD_TOKEN}
        - url: https://k8s-staging.internal.example.com
          name: staging
          authProvider: serviceAccount
          serviceAccountToken: ${K8S_STAGING_TOKEN}

techdocs:
  builder: external
  publisher:
    type: awsS3
    awsS3:
      bucketName: backstage-techdocs-prod
      region: ap-southeast-1

costInsights:
  engineerCost: 200000    # USD per engineer per year
  products:
    computeEngine:
      name: Compute Engine
      icon: compute
    cloudStorage:
      name: Cloud Storage
      icon: storage
EOF

# ─── 3. Service Catalog Entity ─────────────────────────────────────────────
cat > catalog-info-template.yaml << 'EOF'
# catalog-info.yaml for a microservice
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  title: Payment Service
  description: Handles all payment processing, refunds, and fraud detection
  annotations:
    # GitHub Actions integration
    github.com/project-slug: myorg/payment-service
    # Kubernetes workload
    backstage.io/kubernetes-id: payment-service
    backstage.io/kubernetes-namespace: production
    # Prometheus dashboards
    grafana/dashboard-selector: "title=Payment Service"
    # PagerDuty
    pagerduty.com/service-id: P123ABC
    # ArgoCD
    argocd/app-name: payment-service
    # SonarQube
    sonarqube.org/project-key: payment-service
    # Sentry
    sentry.io/project-slug: payment-service
  tags:
    - java
    - kafka
    - postgresql
    - payments
  links:
    - url: https://grafana.internal.example.com/d/payments
      title: Grafana Dashboard
      icon: dashboard
    - url: https://runbooks.internal.example.com/payment-service
      title: Runbook
      icon: docs
spec:
  type: service
  lifecycle: production
  owner: group:payments-team
  system: payment-platform
  dependsOn:
    - component:postgresql-cluster
    - component:kafka-cluster
    - component:fraud-detection-service
  providesApis:
    - payment-api-v2
  consumesApis:
    - user-profile-api
    - fraud-score-api
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payment-api-v2
  title: Payment API v2
  description: REST API for payment processing
spec:
  type: openapi
  lifecycle: production
  owner: group:payments-team
  definition: |
    openapi: "3.0.0"
    info:
      title: Payment API
      version: "2.0.0"
    paths:
      /api/v2/payments:
        post:
          summary: Create payment
          operationId: createPayment
          requestBody:
            required: true
            content:
              application/json:
                schema:
                  $ref: '#/components/schemas/PaymentRequest'
          responses:
            '201':
              description: Payment created
    components:
      schemas:
        PaymentRequest:
          type: object
          required: [amount, currency, method]
          properties:
            amount:
              type: number
            currency:
              type: string
              enum: [USD, EUR, THB]
            method:
              type: string
              enum: [card, bank_transfer, wallet]
---
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: payment-platform
  title: Payment Platform
  description: Complete payment processing ecosystem
spec:
  owner: group:payments-team
  domain: financial
EOF

# ─── 4. Software Templates (Scaffolder) ────────────────────────────────────
cat > backstage-template-microservice.yaml << 'EOF'
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: microservice-template
  title: New Microservice
  description: Create a production-ready microservice with K8s, CI/CD, monitoring
  tags:
    - recommended
    - microservice
spec:
  owner: group:platform-team
  type: service

  parameters:
    - title: Service Information
      required: [name, description, owner, language]
      properties:
        name:
          title: Service Name
          type: string
          pattern: "^[a-z][a-z0-9-]*$"
          ui:autofocus: true
        description:
          title: Description
          type: string
        owner:
          title: Owner Team
          type: string
          ui:field: OwnerPicker
          ui:options:
            allowedKinds: [Group]
        language:
          title: Language
          type: string
          enum: [go, python, java, node]
          default: go
        
    - title: Infrastructure
      required: [region, tier]
      properties:
        region:
          title: Primary Region
          type: string
          enum: [ap-southeast-1, us-east-1, eu-west-1]
          default: ap-southeast-1
        tier:
          title: Service Tier
          type: string
          enum: [critical, standard, dev]
          default: standard
        database:
          title: Database Type
          type: string
          enum: [none, postgresql, mysql, mongodb]
          default: none
        cache:
          title: Cache
          type: boolean
          default: false
          
    - title: Repository
      required: [repoUrl]
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts: [github.com]

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
          language: ${{ parameters.language }}
          region: ${{ parameters.region }}
          tier: ${{ parameters.tier }}
          database: ${{ parameters.database }}

    - id: create-github-repo
      name: Create GitHub Repository
      action: github:repo:create
      input:
        repoUrl: ${{ parameters.repoUrl }}
        description: ${{ parameters.description }}
        topics: [${{ parameters.language }}, microservice, ${{ parameters.tier }}]
        defaultBranch: main
        repoVisibility: internal

    - id: push-code
      name: Push Initial Code
      action: github:repo:push
      input:
        repoUrl: ${{ parameters.repoUrl }}
        defaultBranch: main
        sourcePath: .

    - id: create-argocd-app
      name: Create ArgoCD Application
      action: argocd:create-resources
      input:
        appName: ${{ parameters.name }}
        argoInstance: production
        namespace: ${{ parameters.name }}
        repoUrl: https://github.com/myorg/${{ parameters.name }}
        path: kubernetes/overlays/production

    - id: create-pagerduty-service
      name: Create PagerDuty Service
      action: pagerduty:service:create
      input:
        name: ${{ parameters.name }}
        escalationPolicyId: P123ABC
        alertCreation: create_alerts_and_incidents

    - id: register-catalog
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['create-github-repo'].output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml

  output:
    links:
      - title: Repository
        url: ${{ steps['create-github-repo'].output.remoteUrl }}
      - title: ArgoCD Application
        url: https://argocd.internal.example.com/applications/${{ parameters.name }}
      - title: Backstage Component
        entityRef: ${{ steps['register-catalog'].output.entityRef }}
EOF

echo "Backstage configuration created"
echo "=== Step 601 Complete: Backstage IDP ==="
SCRIPT
chmod +x backstage-idp.sh
echo "Script created: backstage-idp.sh"
```

**สิ่งที่เรียนรู้:**
- Backstage `catalog-info.yaml`: Component/API/System entities พร้อม annotations (GitHub, K8s, Grafana, PagerDuty, ArgoCD)
- Software Template: multi-step scaffolder สร้าง repo+ArgoCD app+PagerDuty service อัตโนมัติ
- Kubernetes plugin: multi-cluster support (production+staging)
- TechDocs: external builder → S3 publisher
- Cost Insights integration

---

## Step 602: Golden Path Templates + Developer Self-Service

```bash
cat > golden-path-templates.sh << 'SCRIPT'
#!/bin/bash
# Golden Path: Developer Self-Service Automation

set -euo pipefail

echo "=== Golden Path Templates + Developer Self-Service ==="

# ─── 1. Microservice Skeleton (Go) ─────────────────────────────────────────
mkdir -p service-skeleton-go/{cmd/server,internal/{handler,service,repository},kubernetes/{base,overlays/{dev,staging,production}},docs,.github/workflows}

cat > service-skeleton-go/cmd/server/main.go << 'GOEOF'
package main

import (
	"context"
	"fmt"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
	"go.opentelemetry.io/otel/sdk/trace"
	"go.uber.org/zap"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
	// Logger
	log, _ := zap.NewProduction()
	defer log.Sync()

	// OpenTelemetry tracer
	ctx := context.Background()
	exporter, err := otlptracehttp.New(ctx,
		otlptracehttp.WithEndpoint(os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")),
	)
	if err != nil {
		log.Fatal("failed to create tracer", zap.Error(err))
	}
	tp := trace.NewTracerProvider(
		trace.WithBatcher(exporter),
		trace.WithSampler(trace.AlwaysSample()),
	)
	otel.SetTracerProvider(tp)
	defer tp.Shutdown(ctx)

	// HTTP server
	mux := http.NewServeMux()
	mux.Handle("/metrics", promhttp.Handler())
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "ok")
	})
	mux.HandleFunc("/readyz", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "ok")
	})

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	srv := &http.Server{
		Addr:         ":" + port,
		Handler:      mux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	// Graceful shutdown
	done := make(chan os.Signal, 1)
	signal.Notify(done, syscall.SIGTERM, syscall.SIGINT)

	go func() {
		log.Info("server starting", zap.String("port", port))
		if err := srv.ListenAndServe(); err != http.ErrServerClosed {
			log.Fatal("server error", zap.Error(err))
		}
	}()

	<-done
	log.Info("shutting down gracefully...")
	shutdownCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
	defer cancel()
	srv.Shutdown(shutdownCtx)
}
GOEOF

# ─── 2. Kubernetes Base Manifests ──────────────────────────────────────────
cat > service-skeleton-go/kubernetes/base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: SERVICE_NAME
  labels:
    app: SERVICE_NAME
    version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: SERVICE_NAME
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: SERVICE_NAME
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: SERVICE_NAME
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: SERVICE_NAME
      containers:
        - name: SERVICE_NAME
          image: ghcr.io/myorg/SERVICE_NAME:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          env:
            - name: PORT
              value: "8080"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector:4318"
            - name: OTEL_SERVICE_NAME
              value: "SERVICE_NAME"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
            readOnlyRootFilesystem: true
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: SERVICE_NAME
spec:
  selector:
    app: SERVICE_NAME
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: SERVICE_NAME-pdb
spec:
  minAvailable: "50%"
  selector:
    matchLabels:
      app: SERVICE_NAME
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: SERVICE_NAME-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: SERVICE_NAME
  minReplicas: 2
  maxReplicas: 20
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

# ─── 3. GitHub Actions Golden Path CI ──────────────────────────────────────
cat > service-skeleton-go/.github/workflows/ci-cd.yaml << 'EOF'
name: CI/CD Golden Path

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write
  contents: read
  packages: write
  security-events: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  quality-gate:
    name: Quality Gate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.23"
          cache: true

      - name: Lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest

      - name: Unit Tests
        run: |
          go test -v -race -coverprofile=coverage.out ./...
          go tool cover -func=coverage.out

      - name: Coverage Check (80% minimum)
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
          echo "Coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage ${COVERAGE}% is below minimum 80%"
            exit 1
          fi

      - name: Security Scan (govulncheck)
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

  build-sign:
    name: Build and Sign
    needs: quality-gate
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and push with SLSA provenance
        id: build
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          provenance: mode=max
          sbom: true

      - name: Sign image
        uses: sigstore/cosign-installer@v3
      
      - run: |
          cosign sign --yes \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
        env:
          COSIGN_EXPERIMENTAL: "1"

  deploy-staging:
    name: Deploy to Staging
    needs: build-sign
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Update staging image
        run: |
          # ArgoCD image updater handles this via annotation
          echo "ArgoCD image updater will detect new image and sync"

  integration-tests:
    name: Integration Tests
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run integration tests
        run: go test -v -tags=integration ./test/integration/...
        env:
          API_BASE_URL: https://staging.internal.example.com

  deploy-production:
    name: Deploy to Production
    needs: integration-tests
    runs-on: ubuntu-latest
    environment: production     # Requires manual approval
    steps:
      - name: Trigger ArgoCD sync
        run: |
          argocd app sync ${{ github.event.repository.name }} \
            --server argocd.internal.example.com \
            --auth-token ${{ secrets.ARGOCD_TOKEN }}
EOF

# ─── 4. Platform CLI (Golden Path enforcer) ────────────────────────────────
cat > platform-cli.py << 'PYEOF'
#!/usr/bin/env python3
"""Platform CLI: enforce golden path standards for new services."""

import os
import sys
import json
import subprocess
from pathlib import Path
from dataclasses import dataclass
from typing import List, Tuple

@dataclass
class Violation:
    severity: str  # ERROR, WARN
    rule: str
    message: str
    fix: str

class GoldenPathValidator:
    def __init__(self, service_path: str):
        self.path = Path(service_path)
        self.violations: List[Violation] = []

    def check(self, condition: bool, severity: str, rule: str, message: str, fix: str):
        if not condition:
            self.violations.append(Violation(severity, rule, message, fix))

    def validate_structure(self):
        required_files = [
            ("catalog-info.yaml", "Backstage catalog entry"),
            (".github/workflows/ci-cd.yaml", "Golden path CI/CD"),
            ("kubernetes/base/deployment.yaml", "K8s deployment manifest"),
            ("Dockerfile", "Container image definition"),
            ("docs/README.md", "Service documentation"),
        ]
        for filepath, desc in required_files:
            self.check(
                (self.path / filepath).exists(),
                "ERROR", "required-files",
                f"Missing {filepath} ({desc})",
                f"Create {filepath} using platform template",
            )

    def validate_dockerfile(self):
        dockerfile = self.path / "Dockerfile"
        if not dockerfile.exists():
            return
        
        content = dockerfile.read_text()
        
        self.check(
            "FROM" in content and "distroless" in content.lower() or "scratch" in content.lower() or "alpine" in content.lower(),
            "WARN", "minimal-base-image",
            "Dockerfile should use minimal base image (distroless/alpine/scratch)",
            "Change FROM to gcr.io/distroless/static or alpine:3",
        )
        
        self.check(
            "USER" in content,
            "ERROR", "non-root-user",
            "Dockerfile must set non-root USER",
            "Add 'USER 65534' before ENTRYPOINT",
        )

    def validate_k8s_manifests(self):
        deployment = self.path / "kubernetes/base/deployment.yaml"
        if not deployment.exists():
            return
        
        try:
            import yaml
            with open(deployment) as f:
                docs = list(yaml.safe_load_all(f))
        except Exception:
            return
        
        for doc in docs:
            if doc and doc.get("kind") == "Deployment":
                spec = doc.get("spec", {})
                template = spec.get("template", {})
                pod_spec = template.get("spec", {})
                
                self.check(
                    pod_spec.get("securityContext", {}).get("runAsNonRoot"),
                    "ERROR", "run-as-non-root",
                    "Deployment must set securityContext.runAsNonRoot=true",
                    "Add securityContext.runAsNonRoot: true to pod spec",
                )
                
                containers = pod_spec.get("containers", [])
                for container in containers:
                    resources = container.get("resources", {})
                    self.check(
                        bool(resources.get("requests")),
                        "ERROR", "resource-requests",
                        f"Container '{container.get('name')}' missing resource requests",
                        "Add resources.requests.cpu and resources.requests.memory",
                    )
                    self.check(
                        bool(resources.get("limits")),
                        "ERROR", "resource-limits",
                        f"Container '{container.get('name')}' missing resource limits",
                        "Add resources.limits.cpu and resources.limits.memory",
                    )
                    
                    probes = container.get("readinessProbe") or container.get("livenessProbe")
                    self.check(
                        bool(probes),
                        "ERROR", "health-probes",
                        f"Container '{container.get('name')}' missing health probes",
                        "Add readinessProbe and livenessProbe",
                    )
                
                # Check PDB exists
                has_pdb = any(
                    d and d.get("kind") == "PodDisruptionBudget"
                    for d in docs
                )
                self.check(
                    has_pdb,
                    "WARN", "pdb-required",
                    "No PodDisruptionBudget found",
                    "Add PDB with minAvailable: 50%",
                )

    def validate_catalog(self):
        catalog = self.path / "catalog-info.yaml"
        if not catalog.exists():
            return
        
        content = catalog.read_text()
        
        required_annotations = [
            "github.com/project-slug",
            "backstage.io/kubernetes-id",
            "grafana/dashboard-selector",
            "pagerduty.com/service-id",
        ]
        for ann in required_annotations:
            self.check(
                ann in content,
                "WARN", "catalog-annotations",
                f"catalog-info.yaml missing annotation: {ann}",
                f"Add annotation {ann} to catalog-info.yaml",
            )

    def run(self) -> bool:
        print(f"\n=== Golden Path Validation: {self.path} ===\n")
        
        self.validate_structure()
        self.validate_dockerfile()
        self.validate_k8s_manifests()
        self.validate_catalog()
        
        errors = [v for v in self.violations if v.severity == "ERROR"]
        warnings = [v for v in self.violations if v.severity == "WARN"]
        
        if errors:
            print(f"ERRORS ({len(errors)}):")
            for v in errors:
                print(f"  ✗ [{v.rule}] {v.message}")
                print(f"    Fix: {v.fix}")
        
        if warnings:
            print(f"\nWARNINGS ({len(warnings)}):")
            for v in warnings:
                print(f"  ⚠ [{v.rule}] {v.message}")
                print(f"    Fix: {v.fix}")
        
        if not self.violations:
            print("✓ All golden path checks passed!")
        
        print(f"\nResult: {len(errors)} errors, {len(warnings)} warnings")
        return len(errors) == 0

# Validate the skeleton we just created
validator = GoldenPathValidator("service-skeleton-go")
passed = validator.run()
print(f"\nValidation {'passed' if passed else 'failed (expected for skeleton demo)'}")
PYEOF

pip install pyyaml 2>/dev/null | tail -1
python3 platform-cli.py

echo "=== Step 602 Complete: Golden Path Templates + Self-Service ==="
SCRIPT
chmod +x golden-path-templates.sh
bash golden-path-templates.sh 2>/dev/null || true
echo "Script created: golden-path-templates.sh"
```

**สิ่งที่เรียนรู้:**
- Backstage Software Template: multi-step scaffolder (GitHub repo + ArgoCD app + PagerDuty + catalog registration)
- Golden Path Go skeleton: graceful shutdown, OTEL tracing, Prometheus metrics, structured logging
- K8s base manifests: PSS security context, topology spread, HPA, PDB — production-ready defaults
- GitHub Actions golden path CI: lint→test→coverage(80%)→govulncheck→SLSA build→staging→integration→prod
- Platform CLI validator: 12 golden path checks (structure, Dockerfile, K8s security, catalog annotations)

---

## Step 603: Internal Developer Platform — Metrics + Developer Productivity

```bash
cat > developer-productivity.sh << 'SCRIPT'
#!/bin/bash
# Developer Productivity Metrics (DORA + SPACE)

set -euo pipefail

echo "=== Developer Productivity Platform ==="

# ─── 1. DORA Metrics Collector ─────────────────────────────────────────────
cat > dora-metrics.py << 'PYEOF'
#!/usr/bin/env python3
"""
DORA Metrics: Deployment Frequency, Lead Time, Change Failure Rate, MTTR
SPACE Framework: Satisfaction, Performance, Activity, Communication, Efficiency
"""

import json
import random
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from statistics import mean, median

@dataclass
class Deployment:
    id: str
    service: str
    team: str
    timestamp: datetime
    commit_sha: str
    lead_time_hours: float    # commit → production
    is_rollback: bool = False
    incident_caused: bool = False

@dataclass
class Incident:
    id: str
    service: str
    team: str
    start_time: datetime
    resolved_time: Optional[datetime]
    severity: str
    caused_by_deployment: bool = False

    @property
    def mttr_minutes(self) -> float:
        if not self.resolved_time:
            return float("inf")
        return (self.resolved_time - self.start_time).total_seconds() / 60

@dataclass
class DORAMetrics:
    team: str
    period_days: int
    deployment_frequency_per_day: float
    lead_time_hours: float
    change_failure_rate: float  # percentage
    mttr_minutes: float

    @property
    def deployment_frequency_rating(self) -> str:
        if self.deployment_frequency_per_day >= 1:
            return "Elite"
        elif self.deployment_frequency_per_day >= 1/7:
            return "High"
        elif self.deployment_frequency_per_day >= 1/30:
            return "Medium"
        return "Low"

    @property
    def lead_time_rating(self) -> str:
        if self.lead_time_hours <= 1:
            return "Elite"
        elif self.lead_time_hours <= 24:
            return "High"
        elif self.lead_time_hours <= 168:  # 1 week
            return "Medium"
        return "Low"

    @property
    def cfr_rating(self) -> str:
        if self.change_failure_rate <= 5:
            return "Elite"
        elif self.change_failure_rate <= 15:
            return "High"
        elif self.change_failure_rate <= 30:
            return "Medium"
        return "Low"

    @property
    def mttr_rating(self) -> str:
        if self.mttr_minutes <= 60:
            return "Elite"
        elif self.mttr_minutes <= 24 * 60:
            return "High"
        elif self.mttr_minutes <= 7 * 24 * 60:
            return "Medium"
        return "Low"

    @property
    def overall_rating(self) -> str:
        ratings = [
            self.deployment_frequency_rating,
            self.lead_time_rating,
            self.cfr_rating,
            self.mttr_rating,
        ]
        elite_count = ratings.count("Elite")
        high_count = ratings.count("High")
        
        if elite_count >= 3:
            return "Elite"
        elif elite_count + high_count >= 3:
            return "High"
        elif "Low" not in ratings:
            return "Medium"
        return "Low"

class DORACalculator:
    def __init__(self):
        self.deployments: List[Deployment] = []
        self.incidents: List[Incident] = []

    def add_deployment(self, deployment: Deployment):
        self.deployments.append(deployment)

    def add_incident(self, incident: Incident):
        self.incidents.append(incident)

    def calculate_team_metrics(self, team: str, period_days: int = 30) -> DORAMetrics:
        cutoff = datetime.utcnow() - timedelta(days=period_days)
        
        team_deployments = [
            d for d in self.deployments
            if d.team == team and d.timestamp >= cutoff
        ]
        
        team_incidents = [
            i for i in self.incidents
            if i.team == team and i.start_time >= cutoff
        ]
        
        # Deployment Frequency
        deploy_freq = len(team_deployments) / period_days
        
        # Lead Time (median)
        lead_times = [d.lead_time_hours for d in team_deployments]
        avg_lead_time = median(lead_times) if lead_times else 0
        
        # Change Failure Rate
        failed = sum(1 for d in team_deployments if d.incident_caused)
        cfr = (failed / len(team_deployments) * 100) if team_deployments else 0
        
        # MTTR
        deployment_incidents = [i for i in team_incidents if i.caused_by_deployment]
        mttr_values = [i.mttr_minutes for i in deployment_incidents if i.resolved_time]
        avg_mttr = mean(mttr_values) if mttr_values else 0
        
        return DORAMetrics(
            team=team,
            period_days=period_days,
            deployment_frequency_per_day=deploy_freq,
            lead_time_hours=avg_lead_time,
            change_failure_rate=cfr,
            mttr_minutes=avg_mttr,
        )

    def generate_team_report(self, teams: List[str]) -> Dict:
        report = {
            "generated_at": datetime.utcnow().isoformat(),
            "teams": {}
        }
        
        for team in teams:
            metrics = self.calculate_team_metrics(team)
            report["teams"][team] = {
                "overall_rating": metrics.overall_rating,
                "deployment_frequency": {
                    "value": round(metrics.deployment_frequency_per_day, 3),
                    "unit": "per_day",
                    "rating": metrics.deployment_frequency_rating,
                },
                "lead_time": {
                    "value": round(metrics.lead_time_hours, 1),
                    "unit": "hours",
                    "rating": metrics.lead_time_rating,
                },
                "change_failure_rate": {
                    "value": round(metrics.change_failure_rate, 1),
                    "unit": "percent",
                    "rating": metrics.cfr_rating,
                },
                "mttr": {
                    "value": round(metrics.mttr_minutes, 1),
                    "unit": "minutes",
                    "rating": metrics.mttr_rating,
                },
            }
        
        return report

# Generate sample data
calculator = DORACalculator()
teams = ["payments-team", "platform-team", "order-team"]
now = datetime.utcnow()

random.seed(42)
for _ in range(200):
    team = random.choice(teams)
    days_ago = random.uniform(0, 30)
    is_incident = random.random() < 0.05  # 5% failure rate
    
    deploy = Deployment(
        id=f"deploy-{random.randint(10000, 99999)}",
        service=f"{team}-service",
        team=team,
        timestamp=now - timedelta(days=days_ago),
        commit_sha=f"{random.randint(0, 0xFFFFFF):06x}",
        lead_time_hours=random.uniform(0.5, 48),
        incident_caused=is_incident,
    )
    calculator.add_deployment(deploy)
    
    if is_incident:
        incident = Incident(
            id=f"inc-{random.randint(1000, 9999)}",
            service=deploy.service,
            team=team,
            start_time=deploy.timestamp + timedelta(minutes=random.randint(5, 60)),
            resolved_time=deploy.timestamp + timedelta(minutes=random.randint(30, 180)),
            severity=random.choice(["P1", "P2", "P3"]),
            caused_by_deployment=True,
        )
        calculator.add_incident(incident)

report = calculator.generate_team_report(teams)
print("=== DORA Metrics Report ===\n")
for team, data in report["teams"].items():
    print(f"Team: {team} — {data['overall_rating']}")
    print(f"  Deploy Freq: {data['deployment_frequency']['value']}/day ({data['deployment_frequency']['rating']})")
    print(f"  Lead Time: {data['lead_time']['value']}h ({data['lead_time']['rating']})")
    print(f"  Change Failure Rate: {data['change_failure_rate']['value']}% ({data['change_failure_rate']['rating']})")
    print(f"  MTTR: {data['mttr']['value']}min ({data['mttr']['rating']})")
    print()
PYEOF

python3 dora-metrics.py

# ─── 2. Developer Experience Scorecard ─────────────────────────────────────
cat > dx-scorecard.py << 'PYEOF'
#!/usr/bin/env python3
"""Developer Experience Scorecard based on SPACE framework."""

from dataclasses import dataclass
from typing import Dict

@dataclass
class DXScore:
    satisfaction: float        # S: Developer satisfaction survey (1-10)
    performance: float         # P: Code quality metrics (test coverage, review time)
    activity: float            # A: PR merged, commits, deploys per week
    communication: float       # C: PR review time, doc coverage, on-call burden
    efficiency: float          # E: CI time, env setup time, toil reduction

    @property
    def overall(self) -> float:
        weights = {
            "satisfaction": 0.25,
            "performance": 0.20,
            "activity": 0.15,
            "communication": 0.20,
            "efficiency": 0.20,
        }
        return (
            self.satisfaction * weights["satisfaction"] +
            self.performance * weights["performance"] +
            self.activity * weights["activity"] +
            self.communication * weights["communication"] +
            self.efficiency * weights["efficiency"]
        )

    @property
    def rating(self) -> str:
        score = self.overall
        if score >= 8:
            return "Excellent"
        elif score >= 6:
            return "Good"
        elif score >= 4:
            return "Needs Improvement"
        return "Poor"

def collect_dx_scores() -> Dict[str, DXScore]:
    """Aggregate DX scores from various sources."""
    return {
        "payments-team": DXScore(
            satisfaction=8.2,   # From quarterly survey
            performance=7.5,    # Test coverage 87%, avg review time 4h
            activity=6.8,       # 8 PRs/week per engineer
            communication=7.9,  # Review time <24h, docs 92%
            efficiency=7.2,     # CI 8min, setup 30min, toil 15%
        ),
        "platform-team": DXScore(
            satisfaction=9.1,
            performance=8.8,
            activity=9.0,
            communication=8.5,
            efficiency=9.2,
        ),
        "order-team": DXScore(
            satisfaction=6.5,
            performance=6.2,
            activity=5.8,
            communication=6.0,
            efficiency=5.5,
        ),
    }

scores = collect_dx_scores()
print("\n=== Developer Experience Scorecard (SPACE Framework) ===\n")
for team, score in scores.items():
    print(f"Team: {team} — {score.rating} ({score.overall:.1f}/10)")
    print(f"  S (Satisfaction): {score.satisfaction}/10")
    print(f"  P (Performance):  {score.performance}/10")
    print(f"  A (Activity):     {score.activity}/10")
    print(f"  C (Communication):{score.communication}/10")
    print(f"  E (Efficiency):   {score.efficiency}/10")
    print()
PYEOF

python3 dx-scorecard.py

echo "=== Step 603 Complete: Developer Productivity Metrics ==="
SCRIPT
chmod +x developer-productivity.sh
bash developer-productivity.sh
echo "Script created: developer-productivity.sh"
```

**สิ่งที่เรียนรู้:**
- DORA Metrics 4 indicators: Deployment Frequency, Lead Time, Change Failure Rate, MTTR
- Elite/High/Medium/Low ratings per DORA research benchmarks
- SPACE Framework: 5 dimensions ของ Developer Experience
- DORACalculator: เก็บ 200 sample deployments, คำนวณ team-level metrics
- DXScorecard: weighted scoring (satisfaction 25%, performance 20%, activity 15%, communication 20%, efficiency 20%)

---

## Step 604: GitOps Advanced — Flux v2 + Progressive Delivery

```bash
cat > gitops-advanced.sh << 'SCRIPT'
#!/bin/bash
# Advanced GitOps with Flux v2 + Progressive Delivery

set -euo pipefail

echo "=== Advanced GitOps: Flux v2 + Progressive Delivery ==="

# ─── 1. Flux v2 Bootstrap ──────────────────────────────────────────────────
flux bootstrap github \
  --owner=myorg \
  --repository=platform-gitops \
  --branch=main \
  --path=clusters/production \
  --personal 2>/dev/null || echo "Flux bootstrap (GitHub access required)"

# ─── 2. Multi-tenancy with Flux ────────────────────────────────────────────
kubectl apply -f - << 'EOF'
# Tenant: payments team
apiVersion: v1
kind: Namespace
metadata:
  name: payments-system
  labels:
    toolkit.fluxcd.io/tenant: payments-team
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-reconciler
  namespace: payments-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-reconciler
  namespace: payments-system
subjects:
  - kind: ServiceAccount
    name: payments-reconciler
    namespace: payments-system
roleRef:
  kind: ClusterRole
  name: cluster-admin       # Scoped to namespace via RoleBinding
  apiGroup: rbac.authorization.k8s.io
---
# Flux Kustomization for payments team
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: payments-apps
  namespace: flux-system
spec:
  interval: 5m
  path: ./teams/payments/apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: platform-gitops
  serviceAccountName: payments-reconciler
  targetNamespace: payments-system
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: payment-service
      namespace: payments-system
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars
      - kind: Secret
        name: cluster-secrets
EOF

# ─── 3. Flagger Progressive Delivery ──────────────────────────────────────
helm repo add flagger https://flagger.app
helm upgrade --install flagger flagger/flagger \
  --namespace istio-system \
  --set meshProvider=istio \
  --set metricsServer=http://prometheus:9090

kubectl apply -f - << 'EOF'
# Canary analysis for payment-service
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: payment-service
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  progressDeadlineSeconds: 900    # 15 min max per stage
  service:
    port: 80
    targetPort: 8080
    gateways:
      - public-gateway.istio-system.svc.cluster.local
    hosts:
      - payment.example.com
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL
    retries:
      attempts: 3
      perTryTimeout: 3s
      retryOn: "gateway-error,connect-failure,retriable-4xx"
  
  analysis:
    # Run canary every 1 minute for 10 steps = 10 minute analysis
    interval: 1m
    threshold: 5             # 5 consecutive failures → rollback
    maxWeight: 50            # Max 50% canary traffic
    stepWeight: 5            # Increase by 5% per interval
    
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99.5           # 99.5% success rate minimum
        interval: 1m

      - name: request-duration
        thresholdRange:
          max: 500            # p99 < 500ms
        interval: 30s

      - name: error-rate
        templateRef:
          name: error-rate
          namespace: flagger-system
        thresholdRange:
          max: 1              # < 1% errors
        interval: 1m

    webhooks:
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 5s
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 https://payment.example.com/api/v2/health"

      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        timeout: 30s
        metadata:
          cmd: "curl -s https://payment.example.com/api/v2/health | grep OK"

      - name: slack-notification
        type: rollout
        url: https://hooks.slack.com/services/XXX/YYY/ZZZ
        metadata:
          color: "#0076D7"
          username: Flagger
          template: |
            {{ if eq .type "rollout" }}
            Deployment {{ .metadata.name }} promoted to production at {{ .metadata.canaryWeight }}%
            {{ end }}
---
# Custom metric template: error rate
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: error-rate
  namespace: flagger-system
spec:
  provider:
    type: prometheus
    address: http://prometheus:9090
  query: |
    100 - sum(
      rate(http_requests_total{
        namespace="{{ namespace }}",
        pod=~"{{ target }}-[0-9a-zA-Z]+(-[0-9a-zA-Z]+)",
        status!~"5.."
      }[{{ interval }}])
    ) / sum(
      rate(http_requests_total{
        namespace="{{ namespace }}",
        pod=~"{{ target }}-[0-9a-zA-Z]+(-[0-9a-zA-Z]+)"
      }[{{ interval }}])
    ) * 100
EOF

# ─── 4. Image Automation (Flux Image Reflector) ────────────────────────────
kubectl apply -f - << 'EOF'
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: payment-service
  namespace: flux-system
spec:
  image: ghcr.io/myorg/payment-service
  interval: 1m
  secretRef:
    name: ghcr-credentials
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: payment-service
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: payment-service
  policy:
    semver:
      range: ">=1.0.0"     # Only stable releases
    filter:
      tagFilter: "^v"      # Only tags starting with v
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: platform-gitops
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxbot@example.com
        name: Flux Bot
      messageTemplate: |
        chore: update {{ range .Updated.Images -}}
        {{ .Repository }} to {{ .NewTag }}
        {{- end }}
    push:
      branch: main
  update:
    strategy: Setters     # Use image marker comments in YAML
    path: ./clusters/production
EOF

echo "=== Step 604 Complete: GitOps Advanced with Flagger ==="
SCRIPT
chmod +x gitops-advanced.sh
echo "Script created: gitops-advanced.sh"
```

**สิ่งที่เรียนรู้:**
- Flux v2 multi-tenancy: namespace-scoped ServiceAccount + Kustomization per team
- Flux postBuild substituteFrom: inject cluster-level ConfigMap/Secret ลงใน YAML
- Flagger Canary: progressive 5%→50% traffic shift, 1-min analysis interval
- Canary metrics: request-success-rate (min 99.5%), request-duration (p99 < 500ms), custom error-rate template
- Pre-rollout acceptance tests + load tests via webhooks
- Flux Image Automation: semver policy + ImageUpdateAutomation auto-commit

---

## สรุป Part 63

| Step | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|----------------|
| 601 | Backstage IDP | Catalog, Software Templates, Multi-cluster K8s plugin |
| 602 | Golden Path Templates | Go skeleton, K8s base manifests, CI/CD, Platform CLI validator |
| 603 | Developer Productivity | DORA Metrics, SPACE Framework, Team scorecards |
| 604 | Advanced GitOps | Flux v2 multi-tenancy, Flagger canary, Image Automation |

**ขั้นตอนต่อไป: Part 64 — Real-Time Data Streaming, Event-Driven Architecture 2.0 และ Stream Processing at Scale**
