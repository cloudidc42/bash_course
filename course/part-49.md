# Part 49: Advanced CI/CD and DevSecOps Pipeline

## Module 4: Professional Level — DevSecOps

### ขั้นตอนที่ 546: Jenkins Enterprise Pipeline

**Jenkins** enterprise pipeline พร้อม security scanning และ quality gates

```bash
#!/bin/bash
# jenkins-enterprise-pipeline.sh - Jenkins Enterprise CI/CD

set -euo pipefail

JENKINS_NAMESPACE="${JENKINS_NAMESPACE:-jenkins}"
JENKINS_RELEASE="${JENKINS_RELEASE:-jenkins}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_jenkins_enterprise() {
    log "Installing Jenkins Enterprise with Kubernetes plugin..."
    
    helm repo add jenkins https://charts.jenkins.io
    helm repo update
    
    kubectl create namespace "${JENKINS_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF > /tmp/jenkins-values.yaml
controller:
  image: jenkins/jenkins
  tag: "2.440.2-lts"
  
  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "8Gi"
  
  javaOpts: >-
    -Xmx6g -Xms2g
    -Djava.awt.headless=true
    -Dhudson.model.DirectoryBrowserSupport.CSP=
    -XX:+UseG1GC
    -XX:MaxGCPauseMillis=100
    -XX:+ExplicitGCInvokesConcurrent
    -Dhudson.slaves.NodeProvisioner.initialDelay=0
    -Dhudson.slaves.NodeProvisioner.MARGIN=50
    -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
  
  numExecutors: 2
  
  installPlugins:
    - kubernetes:3900.va_dce992317b_4
    - workflow-aggregator:596.v8c21c963d92d
    - git:5.2.1
    - credentials-binding:657.v2b_19db_7d6e6d
    - sonar:2.17.2
    - blueocean:1.27.9
    - pipeline-stage-view:2.34
    - github:1.37.3.1
    - docker-workflow:572.v950f58993843
    - aqua-security-scanner:3.0.18
    - dependency-check-jenkins-plugin:5.4.3
    - warnings-ng:11.3.0
    - code-coverage-api:4.0.0
    - slack:682.v7b_5a_e6a_1ea_3d
    - email-ext:1762.v27de362ea_6ce
    - generic-webhook-trigger:1.88
    - envinject:2.906.v9a_bcd3e39a_d6
    - configuration-as-code:1810.v9b_c30a_249a_4c
    
  JCasC:
    defaultConfig: true
    configScripts:
      jenkins-config: |
        jenkins:
          systemMessage: "Jenkins Enterprise - DevSecOps Platform"
          numExecutors: 2
          scmCheckoutRetryCount: 2
          mode: NORMAL
          
          clouds:
            - kubernetes:
                name: kubernetes
                serverUrl: https://kubernetes.default
                namespace: ${JENKINS_NAMESPACE}
                jenkinsUrl: http://jenkins-svc.${JENKINS_NAMESPACE}:8080
                jenkinsTunnel: jenkins-svc.${JENKINS_NAMESPACE}:50000
                templates:
                  - name: jnlp-agent
                    namespace: ${JENKINS_NAMESPACE}
                    label: k8s-agent
                    containers:
                      - name: jnlp
                        image: jenkins/inbound-agent:latest
                        resourceRequestCpu: "500m"
                        resourceLimitCpu: "2"
                        resourceRequestMemory: "512Mi"
                        resourceLimitMemory: "2Gi"
                    volumes:
                      - hostPathVolume:
                          mountPath: /var/run/docker.sock
                          hostPath: /var/run/docker.sock
                    serviceAccount: jenkins
                    yamlMergeStrategy: merge
        
          securityRealm:
            ldap:
              configurations:
                - server: ldap://ldap.company.com:389
                  rootDN: dc=company,dc=com
                  userSearchBase: ou=users
                  userSearch: uid={0}
                  groupSearchBase: ou=groups
                  groupSearchFilter: (member={0})
                  managerDN: cn=jenkins,dc=company,dc=com
                  managerPasswordSecret: jenkins-ldap-password
          
          authorizationStrategy:
            roleBased:
              roles:
                global:
                  - name: admin
                    permissions:
                      - Overall/Administer
                    assignments:
                      - jenkins-admins
                  - name: developer
                    permissions:
                      - Overall/Read
                      - Job/Build
                      - Job/Cancel
                      - Job/Read
                    assignments:
                      - developers
        
        unclassified:
          slackNotifier:
            baseUrl: https://company.slack.com/services/hooks/jenkins-ci/
            teamDomain: company
            tokenCredentialId: slack-token
EOF
    
    helm upgrade --install "${JENKINS_RELEASE}" jenkins/jenkins \
        --namespace "${JENKINS_NAMESPACE}" \
        -f /tmp/jenkins-values.yaml \
        --wait --timeout=600s
    
    log "Jenkins Enterprise installed"
}

create_devsecops_pipeline() {
    local app_name="${1:-my-app}"
    local repo_url="${2:-https://github.com/company/my-app.git}"
    
    log "Creating DevSecOps Pipeline for: ${app_name}"
    
    cat <<'GROOVY' > "/tmp/${app_name}-Jenkinsfile"
#!/usr/bin/env groovy
/**
 * Enterprise DevSecOps Pipeline
 * Includes: Build, Security Scan, Test, Deploy with Quality Gates
 */

def DOCKER_REGISTRY = "registry.company.com"
def APP_NAME = "my-app"
def SONAR_URL = "https://sonar.company.com"
def TRIVY_SEVERITY = "HIGH,CRITICAL"

pipeline {
    agent {
        kubernetes {
            label "devsecops-agent-${BUILD_NUMBER}"
            yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
    - name: docker
      image: docker:24-dind
      securityContext:
        privileged: true
      volumeMounts:
        - name: docker-storage
          mountPath: /var/lib/docker
    - name: kubectl
      image: bitnami/kubectl:latest
      command: ["cat"]
      tty: true
    - name: sonar
      image: sonarsource/sonar-scanner-cli:latest
      command: ["cat"]
      tty: true
    - name: trivy
      image: aquasec/trivy:latest
      command: ["cat"]
      tty: true
    - name: helm
      image: alpine/helm:latest
      command: ["cat"]
      tty: true
  volumes:
    - name: docker-storage
      emptyDir: {}
"""
        }
    }
    
    environment {
        DOCKER_REGISTRY = "${DOCKER_REGISTRY}"
        IMAGE_TAG = "${GIT_COMMIT.take(7)}"
        FULL_IMAGE = "${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
        SONAR_TOKEN = credentials('sonar-token')
        REGISTRY_CREDS = credentials('registry-credentials')
        SNYK_TOKEN = credentials('snyk-token')
    }
    
    options {
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        ansiColor('xterm')
    }
    
    stages {
        stage('🔍 Checkout & Validate') {
            steps {
                checkout scm
                script {
                    env.GIT_BRANCH_SHORT = sh(
                        script: "echo ${GIT_BRANCH} | sed 's|origin/||g'",
                        returnStdout: true
                    ).trim()
                    
                    sh "git log --oneline -5"
                    sh "ls -la"
                }
            }
        }
        
        stage('🔒 Secret Scanning') {
            parallel {
                stage('GitLeaks') {
                    steps {
                        container('docker') {
                            sh """
                                docker run --rm \
                                    -v "\$(pwd):/path" \
                                    zricethezav/gitleaks:latest detect \
                                    --source /path \
                                    --report-format json \
                                    --report-path /path/gitleaks-report.json \
                                    --exit-code 0
                            """
                            archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                        }
                    }
                }
                stage('Dependency Audit') {
                    steps {
                        container('docker') {
                            sh """
                                docker run --rm \
                                    -v "\$(pwd):/app" \
                                    -w /app \
                                    node:20-alpine \
                                    sh -c "npm audit --json > npm-audit.json 2>&1 || true"
                            """
                            archiveArtifacts artifacts: 'npm-audit.json', allowEmptyArchive: true
                        }
                    }
                }
            }
        }
        
        stage('🏗️ Build') {
            steps {
                container('docker') {
                    sh """
                        docker build \
                            --build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
                            --build-arg GIT_COMMIT=${GIT_COMMIT} \
                            --build-arg VERSION=${IMAGE_TAG} \
                            --cache-from ${DOCKER_REGISTRY}/${APP_NAME}:latest \
                            --tag ${FULL_IMAGE} \
                            --tag ${DOCKER_REGISTRY}/${APP_NAME}:latest \
                            --file Dockerfile \
                            .
                    """
                }
            }
        }
        
        stage('🔍 Code Quality') {
            parallel {
                stage('SonarQube Analysis') {
                    steps {
                        container('sonar') {
                            withSonarQubeEnv('SonarQube') {
                                sh """
                                    sonar-scanner \
                                        -Dsonar.projectKey=${APP_NAME} \
                                        -Dsonar.projectName="${APP_NAME}" \
                                        -Dsonar.projectVersion=${IMAGE_TAG} \
                                        -Dsonar.sources=src \
                                        -Dsonar.tests=tests \
                                        -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                                        -Dsonar.coverage.exclusions=**/*.test.*,**/mocks/**
                                """
                            }
                        }
                    }
                }
                stage('Unit Tests') {
                    steps {
                        container('docker') {
                            sh """
                                docker run --rm \
                                    -v "\$(pwd):/app" -w /app \
                                    node:20-alpine \
                                    sh -c "npm ci && npm test -- --coverage --ci --reporters=jest-junit"
                            """
                            junit 'junit.xml'
                            publishHTML([
                                reportDir: 'coverage/lcov-report',
                                reportFiles: 'index.html',
                                reportName: 'Coverage Report'
                            ])
                        }
                    }
                }
            }
        }
        
        stage('🔒 Security Scanning') {
            parallel {
                stage('Trivy Image Scan') {
                    steps {
                        container('trivy') {
                            sh """
                                trivy image \
                                    --exit-code 1 \
                                    --severity ${TRIVY_SEVERITY} \
                                    --format template \
                                    --template "@/contrib/html.tpl" \
                                    --output trivy-report.html \
                                    ${FULL_IMAGE}
                            """
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'trivy-report.html', allowEmptyArchive: true
                        }
                    }
                }
                stage('OWASP ZAP Scan') {
                    when {
                        branch 'main'
                    }
                    steps {
                        container('docker') {
                            sh """
                                docker run --rm \
                                    -v \$(pwd):/zap/wrk \
                                    owasp/zap2docker-stable:latest \
                                    zap-api-scan.py \
                                    -t https://staging-api.company.com/openapi.json \
                                    -f openapi \
                                    -r zap-report.html \
                                    -x zap-report.xml || true
                            """
                            archiveArtifacts artifacts: 'zap-report.*', allowEmptyArchive: true
                        }
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('📦 Push Image') {
            steps {
                container('docker') {
                    sh """
                        echo "${REGISTRY_CREDS_PSW}" | docker login ${DOCKER_REGISTRY} \
                            -u "${REGISTRY_CREDS_USR}" --password-stdin
                        docker push ${FULL_IMAGE}
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest
                    """
                }
            }
        }
        
        stage('🚀 Deploy') {
            when {
                anyOf {
                    branch 'main'
                    branch 'release/*'
                }
            }
            parallel {
                stage('Deploy to Staging') {
                    when { branch 'main' }
                    steps {
                        container('helm') {
                            sh """
                                helm upgrade --install ${APP_NAME} ./chart \
                                    --namespace staging \
                                    --set image.tag=${IMAGE_TAG} \
                                    --set image.repository=${DOCKER_REGISTRY}/${APP_NAME} \
                                    --set environment=staging \
                                    --atomic \
                                    --timeout 300s \
                                    --wait
                            """
                        }
                    }
                }
                stage('Integration Tests') {
                    when { branch 'main' }
                    steps {
                        container('docker') {
                            sh """
                                docker run --rm \
                                    -e BASE_URL=https://staging.company.com \
                                    -e ENVIRONMENT=staging \
                                    ${DOCKER_REGISTRY}/integration-tests:latest \
                                    npm run test:integration
                            """
                        }
                    }
                }
            }
        }
        
        stage('📊 Generate SBOM') {
            steps {
                container('trivy') {
                    sh """
                        trivy image \
                            --format cyclonedx \
                            --output sbom.json \
                            ${FULL_IMAGE}
                    """
                    archiveArtifacts artifacts: 'sbom.json'
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ *${APP_NAME}* deployed successfully\n" +
                         "Version: `${IMAGE_TAG}` | Branch: `${GIT_BRANCH_SHORT}`\n" +
                         "Build: ${BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: '#alerts',
                color: 'danger',
                message: "❌ *${APP_NAME}* pipeline FAILED\n" +
                         "Branch: `${GIT_BRANCH_SHORT}` | Stage: `${currentBuild.currentResult}`\n" +
                         "Build: ${BUILD_URL}"
            )
        }
    }
}
GROOVY
    
    log "DevSecOps pipeline created: /tmp/${app_name}-Jenkinsfile"
}

case "${1:-help}" in
    "install") install_jenkins_enterprise ;;
    "create-pipeline") create_devsecops_pipeline "${2:-my-app}" "${3:-}" ;;
    *) echo "Usage: $0 {install|create-pipeline}" ;;
esac
```

### ขั้นตอนที่ 547: GitHub Actions Enterprise Workflows

**GitHub Actions** สำหรับ CI/CD ระดับ enterprise

```bash
#!/bin/bash
# github-actions-enterprise.sh - GitHub Actions Enterprise Configuration

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

create_reusable_workflows() {
    local repo_path="${1:-.}"
    
    log "Creating reusable GitHub Actions workflows..."
    
    mkdir -p "${repo_path}/.github/workflows"
    
    cat <<'YAML' > "${repo_path}/.github/workflows/reusable-build.yml"
name: Reusable Build Workflow

on:
  workflow_call:
    inputs:
      app-name:
        required: true
        type: string
      registry:
        required: false
        type: string
        default: ghcr.io
      image-tag:
        required: false
        type: string
        default: ${{ github.sha }}
      dockerfile:
        required: false
        type: string
        default: Dockerfile
      push-image:
        required: false
        type: boolean
        default: true
    outputs:
      image-digest:
        description: Docker image digest
        value: ${{ jobs.build.outputs.image-digest }}
      image-tag:
        description: Docker image tag
        value: ${{ jobs.build.outputs.image-tag }}
    secrets:
      registry-token:
        required: true

jobs:
  build:
    name: Build and Push Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write
      id-token: write
    
    outputs:
      image-digest: ${{ steps.docker-build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          platforms: linux/amd64,linux/arm64
      
      - name: Log in to Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ inputs.registry }}
          username: ${{ github.actor }}
          password: ${{ secrets.registry-token }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ inputs.registry }}/${{ github.repository_owner }}/${{ inputs.app-name }}
          tags: |
            type=sha,prefix=,suffix=,format=short
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=raw,value=latest,enable={{is_default_branch}}
          labels: |
            org.opencontainers.image.title=${{ inputs.app-name }}
            org.opencontainers.image.revision=${{ github.sha }}
      
      - name: Build and Push
        id: docker-build
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ${{ inputs.dockerfile }}
          platforms: linux/amd64,linux/arm64
          push: ${{ inputs.push-image }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          sbom: true
          provenance: true
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_COMMIT=${{ github.sha }}
            VERSION=${{ github.ref_name }}
      
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ${{ inputs.registry }}/${{ github.repository_owner }}/${{ inputs.app-name }}:${{ github.sha }}
          artifact-name: sbom-${{ inputs.app-name }}.spdx
          output-file: sbom.spdx
      
      - name: Sign Image with Cosign
        uses: sigstore/cosign-installer@v3
        with:
          cosign-release: v2.2.2
      
      - run: |
          cosign sign --yes \
            ${{ inputs.registry }}/${{ github.repository_owner }}/${{ inputs.app-name }}@${{ steps.docker-build.outputs.digest }}
YAML
    
    cat <<'YAML' > "${repo_path}/.github/workflows/reusable-security-scan.yml"
name: Reusable Security Scan

on:
  workflow_call:
    inputs:
      image:
        required: true
        type: string
      severity:
        required: false
        type: string
        default: HIGH,CRITICAL
      fail-on-findings:
        required: false
        type: boolean
        default: true
    outputs:
      vulnerabilities-found:
        description: Number of vulnerabilities found
        value: ${{ jobs.scan.outputs.vuln-count }}

jobs:
  scan:
    name: Security Scan
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    
    outputs:
      vuln-count: ${{ steps.count-vulns.outputs.count }}
    
    steps:
      - name: Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ inputs.image }}
          format: sarif
          output: trivy-results.sarif
          severity: ${{ inputs.severity }}
          exit-code: ${{ inputs.fail-on-findings && '1' || '0' }}
      
      - name: Upload scan results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
      
      - name: Count vulnerabilities
        id: count-vulns
        run: |
          count=$(jq '[.runs[].results[] | select(.level == "error")] | length' trivy-results.sarif)
          echo "count=${count}" >> $GITHUB_OUTPUT
          echo "Found ${count} high/critical vulnerabilities"
      
      - name: Grype scan
        uses: anchore/scan-action@v3
        with:
          image: ${{ inputs.image }}
          fail-build: ${{ inputs.fail-on-findings }}
          severity-cutoff: high
          output-format: sarif
YAML
    
    cat <<'YAML' > "${repo_path}/.github/workflows/reusable-deploy.yml"
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
      app-name:
        required: true
        type: string
      helm-chart-path:
        required: false
        type: string
        default: ./chart
      namespace:
        required: false
        type: string
        default: default
      values-file:
        required: false
        type: string
        default: ""
    secrets:
      kubeconfig:
        required: true

jobs:
  deploy:
    name: Deploy to ${{ inputs.environment }}
    runs-on: ubuntu-latest
    environment:
      name: ${{ inputs.environment }}
      url: https://${{ inputs.app-name }}.${{ inputs.environment }}.company.com
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Setup Helm
        uses: azure/setup-helm@v3
      
      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.kubeconfig }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config
      
      - name: Helm deploy
        run: |
          VALUES_ARGS=""
          if [ -n "${{ inputs.values-file }}" ]; then
            VALUES_ARGS="-f ${{ inputs.values-file }}"
          fi
          
          helm upgrade --install ${{ inputs.app-name }} ${{ inputs.helm-chart-path }} \
            --namespace ${{ inputs.namespace }} \
            --create-namespace \
            --set image.tag=${{ inputs.image-tag }} \
            --set environment=${{ inputs.environment }} \
            ${VALUES_ARGS} \
            --atomic \
            --timeout 300s \
            --wait
      
      - name: Verify deployment
        run: |
          kubectl rollout status deployment/${{ inputs.app-name }} \
            -n ${{ inputs.namespace }} \
            --timeout=300s
          
          kubectl get pods -n ${{ inputs.namespace }} \
            -l app=${{ inputs.app-name }} \
            --field-selector status.phase=Running
      
      - name: Run smoke tests
        run: |
          echo "Running smoke tests..."
          curl -f https://${{ inputs.app-name }}.${{ inputs.environment }}.company.com/health || exit 1
          echo "Smoke tests passed!"
YAML
    
    log "Reusable workflows created in: ${repo_path}/.github/workflows/"
}

create_main_pipeline() {
    local repo_path="${1:-.}"
    local app_name="${2:-my-app}"
    
    log "Creating main CI/CD pipeline for: ${app_name}"
    
    cat <<YAML > "${repo_path}/.github/workflows/ci-cd.yml"
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop, "release/*"]
    tags: ["v*"]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment
        required: true
        type: choice
        options: [staging, production]
      image-tag:
        description: Image tag to deploy
        required: false

concurrency:
  group: \${{ github.workflow }}-\${{ github.ref }}
  cancel-in-progress: \${{ github.ref != 'refs/heads/main' }}

permissions:
  contents: read
  packages: write
  security-events: write
  pull-requests: write
  id-token: write

jobs:
  validate:
    name: Validate Code
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm
      
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run test:unit -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage/lcov.info
          flags: unit-tests
  
  secret-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: GitLeaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: \${{ secrets.GITHUB_TOKEN }}
  
  sast:
    name: SAST Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: CodeQL Analysis
        uses: github/codeql-action/init@v3
        with:
          languages: javascript, typescript
      
      - name: Build
        run: npm ci && npm run build
      
      - name: Analyze
        uses: github/codeql-action/analyze@v3
      
      - name: Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten
          generateSarif: "1"
      
      - name: Upload Semgrep SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif
  
  build:
    name: Build Image
    needs: [validate, secret-scan]
    uses: ./.github/workflows/reusable-build.yml
    with:
      app-name: ${app_name}
    secrets:
      registry-token: \${{ secrets.GITHUB_TOKEN }}
  
  security-scan:
    name: Security Scan
    needs: build
    uses: ./.github/workflows/reusable-security-scan.yml
    with:
      image: ghcr.io/\${{ github.repository_owner }}/${app_name}:\${{ github.sha }}
      severity: HIGH,CRITICAL
      fail-on-findings: \${{ github.ref == 'refs/heads/main' }}
  
  deploy-staging:
    name: Deploy to Staging
    needs: [build, security-scan]
    if: github.ref == 'refs/heads/main'
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      image-tag: \${{ github.sha }}
      app-name: ${app_name}
      namespace: staging
    secrets:
      kubeconfig: \${{ secrets.STAGING_KUBECONFIG }}
  
  integration-tests:
    name: Integration Tests
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Run integration tests
        run: |
          npm ci
          BASE_URL=https://${app_name}.staging.company.com \
          npm run test:integration
  
  deploy-production:
    name: Deploy to Production
    needs: [deploy-staging, integration-tests]
    if: |
      github.ref == 'refs/heads/main' &&
      github.event_name == 'push'
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      image-tag: \${{ github.sha }}
      app-name: ${app_name}
      namespace: production
    secrets:
      kubeconfig: \${{ secrets.PRODUCTION_KUBECONFIG }}

YAML
    
    log "Main CI/CD pipeline created: ${repo_path}/.github/workflows/ci-cd.yml"
}

case "${1:-help}" in
    "reusable-workflows") create_reusable_workflows "${2:-.}" ;;
    "main-pipeline") create_main_pipeline "${2:-.}" "${3:-my-app}" ;;
    *) echo "Usage: $0 {reusable-workflows|main-pipeline}" ;;
esac
```

### ขั้นตอนที่ 548: Supply Chain Security (SLSA)

**SLSA** (Supply-chain Levels for Software Artifacts) สำหรับ software supply chain security

```bash
#!/bin/bash
# slsa-supply-chain-security.sh - Supply Chain Security Implementation

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

implement_slsa_level3() {
    log "Implementing SLSA Level 3 supply chain security..."
    
    cat <<'YAML' > /tmp/slsa-provenance-workflow.yml
name: SLSA Level 3 Build

on:
  push:
    branches: [main]
    tags: ["v*"]

permissions:
  id-token: write
  contents: read
  packages: write
  attestations: write

jobs:
  build:
    name: Build with SLSA Provenance
    runs-on: ubuntu-latest
    outputs:
      image-digest: ${{ steps.push.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and Push
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          provenance: true
          sbom: true
      
      - name: Attest Build Provenance
        uses: actions/attest-build-provenance@v1
        with:
          subject-name: ghcr.io/${{ github.repository }}
          subject-digest: ${{ steps.push.outputs.digest }}
          push-to-registry: true
  
  verify:
    name: Verify SLSA Provenance
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - name: Install cosign
        uses: sigstore/cosign-installer@v3
      
      - name: Verify image signature
        run: |
          cosign verify \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com \
            --certificate-identity-regexp "https://github.com/${{ github.repository }}" \
            ghcr.io/${{ github.repository }}@${{ needs.build.outputs.image-digest }}
      
      - name: Verify SBOM attestation
        run: |
          cosign verify-attestation \
            --type spdxjson \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com \
            --certificate-identity-regexp "https://github.com/${{ github.repository }}" \
            ghcr.io/${{ github.repository }}@${{ needs.build.outputs.image-digest }} | \
            jq '.payload | @base64d | fromjson | .subject[0]'
YAML
    
    log "SLSA Level 3 workflow created"
}

setup_cosign_policy() {
    log "Setting up Cosign image signature verification policy..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
    - glob: "ghcr.io/company/**"
    - glob: "registry.company.com/**"
  authorities:
    - keyless:
        url: https://fulcio.sigstore.dev
        trustRootRef: public-good
        identities:
          - issuer: https://token.actions.githubusercontent.com
            subjectRegExp: "https://github.com/company/.*/.github/workflows/.*@refs/heads/main"
EOF
    
    log "Cosign signature policy deployed"
}

generate_sbom_and_sign() {
    local image="${1:-registry.company.com/app:latest}"
    local output_dir="${2:-/tmp/sbom}"
    
    log "Generating SBOM and signing image: ${image}"
    
    mkdir -p "${output_dir}"
    
    syft "${image}" -o spdx-json > "${output_dir}/sbom.spdx.json"
    syft "${image}" -o cyclonedx-json > "${output_dir}/sbom.cyclonedx.json"
    
    grype sbom:"${output_dir}/sbom.spdx.json" \
        --output json \
        --file "${output_dir}/vulnerabilities.json"
    
    cosign attest \
        --predicate "${output_dir}/sbom.spdx.json" \
        --type spdxjson \
        "${image}"
    
    cosign sign "${image}"
    
    log "SBOM generated and image signed"
    log "SBOM (SPDX): ${output_dir}/sbom.spdx.json"
    log "SBOM (CycloneDX): ${output_dir}/sbom.cyclonedx.json"
    log "Vulnerabilities: ${output_dir}/vulnerabilities.json"
}

case "${1:-help}" in
    "slsa-level3") implement_slsa_level3 ;;
    "cosign-policy") setup_cosign_policy ;;
    "sbom-sign") generate_sbom_and_sign "${2:-}" "${3:-/tmp/sbom}" ;;
    *) echo "Usage: $0 {slsa-level3|cosign-policy|sbom-sign}" ;;
esac
```

---

## สรุป Part 49

ในส่วนนี้เราได้เรียนรู้:

| ขั้นตอน | หัวข้อ | เครื่องมือหลัก |
|---------|--------|----------------|
| 546 | Jenkins Enterprise Pipeline | Kubernetes Agent, DevSecOps stages, SonarQube, Trivy, OWASP ZAP |
| 547 | GitHub Actions Enterprise | Reusable workflows, SAST/CodeQL, Semgrep, Multi-env deploy |
| 548 | Supply Chain Security (SLSA) | SLSA Level 3, Cosign, SBOM generation, Sigstore, Attestation |

### ขั้นตอนต่อไป: Part 50 - Module 4 Completion and Review
