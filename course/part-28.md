# Part 28: CI/CD Pipeline Automation

## Module 3: Advanced Level — การทำงานระดับสูง

---

## ขั้นตอนที่ 466: พื้นฐาน CI/CD Pipeline ด้วย Bash

CI/CD (Continuous Integration/Continuous Deployment) คือกระบวนการอัตโนมัติในการทดสอบ Build และ Deploy โค้ด

```bash
#!/bin/bash
# ci_cd_basics.sh - พื้นฐาน CI/CD Pipeline

# Pipeline stages
declare -A PIPELINE_STAGES=(
    ["lint"]="ตรวจสอบ code style"
    ["test"]="รัน unit tests"
    ["build"]="Build application"
    ["security_scan"]="ตรวจสอบช่องโหว่"
    ["deploy_staging"]="Deploy ไป staging"
    ["integration_test"]="รัน integration tests"
    ["deploy_production"]="Deploy ไป production"
)

# Pipeline state file
PIPELINE_STATE_FILE="/tmp/pipeline_state_$$.json"

# สีสำหรับ output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
NC='\033[0m'

# Logging
log_pipeline() {
    local level="$1"
    local stage="$2"
    local message="$3"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    case "$level" in
        INFO)  echo -e "${BLUE}[${timestamp}][${stage}] ${message}${NC}" ;;
        OK)    echo -e "${GREEN}[${timestamp}][${stage}] ✓ ${message}${NC}" ;;
        WARN)  echo -e "${YELLOW}[${timestamp}][${stage}] ⚠ ${message}${NC}" ;;
        ERROR) echo -e "${RED}[${timestamp}][${stage}] ✗ ${message}${NC}" ;;
        START) echo -e "${CYAN}[${timestamp}][${stage}] ▶ ${message}${NC}" ;;
    esac
    
    # บันทึก log ลงไฟล์
    echo "[${timestamp}][${level}][${stage}] ${message}" >> /tmp/pipeline_$$.log
}

# อัพเดท state
update_pipeline_state() {
    local stage="$1"
    local status="$2"
    local duration="$3"
    
    # สร้าง state JSON
    cat > "$PIPELINE_STATE_FILE" << EOF
{
    "pipeline_id": "$$",
    "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
    "current_stage": "${stage}",
    "status": "${status}",
    "stages": {
        "${stage}": {
            "status": "${status}",
            "duration": ${duration},
            "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
        }
    }
}
EOF
}

# รัน stage
run_pipeline_stage() {
    local stage="$1"
    local command="$2"
    local allow_failure="${3:-false}"
    
    log_pipeline "START" "$stage" "เริ่ม stage: ${PIPELINE_STAGES[$stage]}"
    
    local start_time=$(date +%s)
    
    # รัน command
    if eval "$command" 2>&1 | tee -a /tmp/pipeline_$$.log; then
        local end_time=$(date +%s)
        local duration=$((end_time - start_time))
        
        log_pipeline "OK" "$stage" "สำเร็จใน ${duration} วินาที"
        update_pipeline_state "$stage" "success" "$duration"
        return 0
    else
        local end_time=$(date +%s)
        local duration=$((end_time - start_time))
        
        update_pipeline_state "$stage" "failed" "$duration"
        
        if [[ "$allow_failure" == "true" ]]; then
            log_pipeline "WARN" "$stage" "ล้มเหลวแต่ดำเนินต่อ (allow_failure=true)"
            return 0
        else
            log_pipeline "ERROR" "$stage" "ล้มเหลวหลังจาก ${duration} วินาที"
            return 1
        fi
    fi
}

# แสดง pipeline summary
show_pipeline_summary() {
    local total_duration="$1"
    local status="$2"
    
    echo ""
    echo "======================================"
    echo "      PIPELINE SUMMARY"
    echo "======================================"
    echo "Pipeline ID: $$"
    echo "Total Duration: ${total_duration}s"
    echo "Status: $status"
    echo "Log file: /tmp/pipeline_$$.log"
    echo "======================================"
}

# Main pipeline
main_pipeline() {
    local pipeline_start=$(date +%s)
    
    log_pipeline "INFO" "pipeline" "เริ่ม CI/CD Pipeline"
    
    # Stage 1: Lint
    run_pipeline_stage "lint" "echo 'Running linters...' && sleep 1 && echo 'Lint passed'" || {
        show_pipeline_summary $(($(date +%s) - pipeline_start)) "FAILED"
        exit 1
    }
    
    # Stage 2: Test
    run_pipeline_stage "test" "echo 'Running tests...' && sleep 2 && echo 'All tests passed'" || {
        show_pipeline_summary $(($(date +%s) - pipeline_start)) "FAILED"
        exit 1
    }
    
    # Stage 3: Build
    run_pipeline_stage "build" "echo 'Building application...' && sleep 1" || {
        show_pipeline_summary $(($(date +%s) - pipeline_start)) "FAILED"
        exit 1
    }
    
    # Stage 4: Security scan (allow_failure)
    run_pipeline_stage "security_scan" "echo 'Scanning for vulnerabilities...'" "true"
    
    # Stage 5: Deploy staging
    run_pipeline_stage "deploy_staging" "echo 'Deploying to staging...'" || {
        show_pipeline_summary $(($(date +%s) - pipeline_start)) "FAILED"
        exit 1
    }
    
    local pipeline_end=$(date +%s)
    show_pipeline_summary $((pipeline_end - pipeline_start)) "SUCCESS"
}

main_pipeline
```

---

## ขั้นตอนที่ 467: GitHub Actions Pipeline ด้วย Bash

```bash
#!/bin/bash
# github_actions_manager.sh - จัดการ GitHub Actions

GITHUB_TOKEN="${GITHUB_TOKEN:-}"
REPO_OWNER="${REPO_OWNER:-}"
REPO_NAME="${REPO_NAME:-}"
API_BASE="https://api.github.com"

# ตรวจสอบ prerequisites
check_prerequisites() {
    if [[ -z "$GITHUB_TOKEN" ]]; then
        echo "ERROR: GITHUB_TOKEN ไม่ได้ตั้งค่า"
        return 1
    fi
    if [[ -z "$REPO_OWNER" ]] || [[ -z "$REPO_NAME" ]]; then
        echo "ERROR: REPO_OWNER หรือ REPO_NAME ไม่ได้ตั้งค่า"
        return 1
    fi
    return 0
}

# เรียก GitHub API
call_github_api() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local url="${API_BASE}${endpoint}"
    local curl_args=(
        -s
        -X "$method"
        -H "Authorization: token ${GITHUB_TOKEN}"
        -H "Accept: application/vnd.github.v3+json"
        -H "Content-Type: application/json"
    )
    
    if [[ -n "$data" ]]; then
        curl_args+=(-d "$data")
    fi
    
    curl "${curl_args[@]}" "$url"
}

# แสดงรายการ workflows
list_workflows() {
    echo "=== GitHub Actions Workflows ==="
    
    local response
    response=$(call_github_api "GET" "/repos/${REPO_OWNER}/${REPO_NAME}/actions/workflows")
    
    echo "$response" | python3 -c "
import json, sys
data = json.load(sys.stdin)
workflows = data.get('workflows', [])
print(f'Total workflows: {len(workflows)}')
print()
for wf in workflows:
    state = wf.get('state', 'unknown')
    badge = '✓' if state == 'active' else '✗'
    print(f'{badge} [{wf[\"id\"]}] {wf[\"name\"]}')
    print(f'   Path: {wf[\"path\"]}')
    print(f'   State: {state}')
    print()
" 2>/dev/null || echo "$response" | head -50
}

# แสดงรายการ runs
list_workflow_runs() {
    local workflow_id="$1"
    local limit="${2:-10}"
    
    echo "=== Workflow Runs (Workflow: ${workflow_id}) ==="
    
    local response
    response=$(call_github_api "GET" \
        "/repos/${REPO_OWNER}/${REPO_NAME}/actions/workflows/${workflow_id}/runs?per_page=${limit}")
    
    echo "$response" | python3 -c "
import json, sys
data = json.load(sys.stdin)
runs = data.get('workflow_runs', [])
print(f'Total runs shown: {len(runs)}')
print()
for run in runs:
    status_icon = {'success': '✓', 'failure': '✗', 'in_progress': '⟳', 'queued': '⏳'}.get(run.get('conclusion') or run.get('status'), '?')
    print(f'{status_icon} Run #{run[\"run_number\"]} - {run[\"head_branch\"]}')
    print(f'   Status: {run[\"status\"]} | Conclusion: {run.get(\"conclusion\", \"N/A\")}')
    print(f'   Started: {run[\"created_at\"]}')
    print(f'   URL: {run[\"html_url\"]}')
    print()
" 2>/dev/null || echo "$response" | head -80
}

# Trigger workflow
trigger_workflow() {
    local workflow_id="$1"
    local branch="${2:-main}"
    local inputs="${3:-{}}"
    
    echo "=== Triggering Workflow: ${workflow_id} on ${branch} ==="
    
    local data
    data=$(cat << EOF
{
    "ref": "${branch}",
    "inputs": ${inputs}
}
EOF
)
    
    local response
    response=$(call_github_api "POST" \
        "/repos/${REPO_OWNER}/${REPO_NAME}/actions/workflows/${workflow_id}/dispatches" \
        "$data")
    
    if [[ -z "$response" ]]; then
        echo "✓ Workflow triggered successfully"
    else
        echo "Response: $response"
    fi
}

# ยกเลิก workflow run
cancel_workflow_run() {
    local run_id="$1"
    
    echo "=== Canceling Workflow Run: ${run_id} ==="
    
    local response
    response=$(call_github_api "POST" \
        "/repos/${REPO_OWNER}/${REPO_NAME}/actions/runs/${run_id}/cancel")
    
    echo "Response: $response"
}

# รอให้ workflow เสร็จ
wait_for_workflow() {
    local run_id="$1"
    local timeout="${2:-300}"
    local check_interval="${3:-10}"
    
    echo "=== Waiting for Workflow Run: ${run_id} ==="
    
    local elapsed=0
    
    while [[ $elapsed -lt $timeout ]]; do
        local response
        response=$(call_github_api "GET" \
            "/repos/${REPO_OWNER}/${REPO_NAME}/actions/runs/${run_id}")
        
        local status conclusion
        status=$(echo "$response" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('status','unknown'))" 2>/dev/null)
        conclusion=$(echo "$response" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('conclusion',''))" 2>/dev/null)
        
        echo "[${elapsed}s] Status: ${status} | Conclusion: ${conclusion}"
        
        case "$status" in
            completed)
                echo "Workflow completed with: ${conclusion}"
                [[ "$conclusion" == "success" ]] && return 0 || return 1
                ;;
            in_progress|queued|waiting|requested)
                sleep "$check_interval"
                elapsed=$((elapsed + check_interval))
                ;;
            *)
                echo "Unknown status: ${status}"
                return 1
                ;;
        esac
    done
    
    echo "TIMEOUT after ${timeout}s"
    return 1
}

# สร้าง workflow YAML
generate_workflow_yaml() {
    local workflow_name="$1"
    local output_file="${2:-.github/workflows/${workflow_name}.yml}"
    
    mkdir -p "$(dirname "$output_file")"
    
    cat > "$output_file" << YAML
name: ${workflow_name}

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

env:
  APP_NAME: myapp
  REGISTRY: ghcr.io
  IMAGE_NAME: \${{ github.repository }}

jobs:
  lint:
    name: Lint Code
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run ShellCheck
        uses: ludeeus/action-shellcheck@master
        with:
          scandir: './scripts'
      
      - name: Run hadolint
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile

  test:
    name: Run Tests
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      matrix:
        bash-version: ['5.0', '5.1', '5.2']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install BATS
        run: |
          git clone https://github.com/bats-core/bats-core.git /tmp/bats
          sudo /tmp/bats/install.sh /usr/local
      
      - name: Run BATS tests
        run: bats tests/
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-\${{ matrix.bash-version }}
          path: test-results/

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: security
    outputs:
      image: \${{ steps.meta.outputs.tags }}
      digest: \${{ steps.build.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: \${{ env.REGISTRY }}
          username: \${{ github.actor }}
          password: \${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: \${{ env.REGISTRY }}/\${{ env.IMAGE_NAME }}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: \${{ steps.meta.outputs.tags }}
          labels: \${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        run: |
          echo "Deploying \${{ needs.build.outputs.image }} to staging"
          # kubectl set image deployment/myapp myapp=\${{ needs.build.outputs.image }}

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        run: |
          echo "Deploying \${{ needs.build.outputs.image }} to production"
YAML
    
    echo "✓ สร้าง workflow file: ${output_file}"
}

# Usage
usage() {
    echo "Usage: $0 <command> [options]"
    echo ""
    echo "Commands:"
    echo "  list-workflows              แสดงรายการ workflows"
    echo "  list-runs <workflow_id>     แสดงรายการ runs"
    echo "  trigger <workflow_id> [branch] [inputs_json]   Trigger workflow"
    echo "  cancel <run_id>             ยกเลิก run"
    echo "  wait <run_id> [timeout]     รอ workflow เสร็จ"
    echo "  generate <name> [output]    สร้าง workflow YAML"
}

# Main
case "${1:-}" in
    list-workflows) check_prerequisites && list_workflows ;;
    list-runs)      check_prerequisites && list_workflow_runs "${2}" "${3:-10}" ;;
    trigger)        check_prerequisites && trigger_workflow "${2}" "${3:-main}" "${4:-{}}" ;;
    cancel)         check_prerequisites && cancel_workflow_run "${2}" ;;
    wait)           check_prerequisites && wait_for_workflow "${2}" "${3:-300}" ;;
    generate)       generate_workflow_yaml "${2}" "${3:-}" ;;
    *)              usage ;;
esac
```

---

## ขั้นตอนที่ 468: GitLab CI Pipeline Automation

```bash
#!/bin/bash
# gitlab_ci_manager.sh - จัดการ GitLab CI/CD

GITLAB_URL="${GITLAB_URL:-https://gitlab.com}"
GITLAB_TOKEN="${GITLAB_TOKEN:-}"
PROJECT_ID="${PROJECT_ID:-}"
API_VERSION="v4"

# เรียก GitLab API
call_gitlab_api() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local url="${GITLAB_URL}/api/${API_VERSION}${endpoint}"
    local curl_args=(
        -s
        -X "$method"
        -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}"
        -H "Content-Type: application/json"
    )
    
    [[ -n "$data" ]] && curl_args+=(-d "$data")
    
    curl "${curl_args[@]}" "$url"
}

# แสดง pipeline รล่าสุด
list_pipelines() {
    local ref="${1:-main}"
    local limit="${2:-10}"
    
    echo "=== GitLab Pipelines (ref: ${ref}) ==="
    
    call_gitlab_api "GET" \
        "/projects/${PROJECT_ID}/pipelines?ref=${ref}&per_page=${limit}" | \
    python3 -c "
import json, sys
pipelines = json.load(sys.stdin)
for p in pipelines:
    status = p.get('status', 'unknown')
    icons = {'success': '✓', 'failed': '✗', 'running': '⟳', 'pending': '⏳', 'canceled': '⊘'}
    icon = icons.get(status, '?')
    print(f'{icon} Pipeline #{p[\"id\"]} - {p[\"ref\"]}')
    print(f'   Status: {status} | SHA: {p[\"sha\"][:8]}')
    print(f'   Created: {p[\"created_at\"]}')
    print()
" 2>/dev/null
}

# ดูรายละเอียด pipeline jobs
get_pipeline_jobs() {
    local pipeline_id="$1"
    
    echo "=== Jobs in Pipeline ${pipeline_id} ==="
    
    call_gitlab_api "GET" \
        "/projects/${PROJECT_ID}/pipelines/${pipeline_id}/jobs" | \
    python3 -c "
import json, sys
jobs = json.load(sys.stdin)
stages = {}
for job in jobs:
    stage = job.get('stage', 'unknown')
    if stage not in stages:
        stages[stage] = []
    stages[stage].append(job)

for stage, stage_jobs in stages.items():
    print(f'--- Stage: {stage} ---')
    for job in stage_jobs:
        status = job.get('status', 'unknown')
        icons = {'success': '✓', 'failed': '✗', 'running': '⟳', 'pending': '⏳', 'canceled': '⊘', 'skipped': '⏭'}
        icon = icons.get(status, '?')
        duration = job.get('duration', 0) or 0
        print(f'  {icon} {job[\"name\"]} ({status}) - {duration:.1f}s')
    print()
" 2>/dev/null
}

# Trigger pipeline
trigger_pipeline() {
    local ref="${1:-main}"
    local variables="${2:-}"
    
    echo "=== Triggering Pipeline on ${ref} ==="
    
    local data
    if [[ -n "$variables" ]]; then
        data="{\"ref\": \"${ref}\", \"variables\": ${variables}}"
    else
        data="{\"ref\": \"${ref}\"}"
    fi
    
    local response
    response=$(call_gitlab_api "POST" \
        "/projects/${PROJECT_ID}/pipeline" \
        "$data")
    
    local pipeline_id
    pipeline_id=$(echo "$response" | python3 -c "import json,sys; print(json.load(sys.stdin).get('id',''))" 2>/dev/null)
    
    if [[ -n "$pipeline_id" ]]; then
        echo "✓ Pipeline triggered: ID ${pipeline_id}"
        echo "URL: ${GITLAB_URL}/projects/${PROJECT_ID}/-/pipelines/${pipeline_id}"
    else
        echo "Response: $response"
    fi
}

# สร้าง .gitlab-ci.yml
generate_gitlab_ci() {
    local output_file="${1:-.gitlab-ci.yml}"
    
    cat > "$output_file" << 'YAML'
# .gitlab-ci.yml - GitLab CI/CD Configuration

image: alpine:latest

stages:
  - lint
  - test
  - build
  - security
  - deploy-staging
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  APP_NAME: "myapp"

# Cache
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .cache/
    - node_modules/

# === Lint Stage ===
shellcheck:
  stage: lint
  image: koalaman/shellcheck-alpine:stable
  script:
    - find . -name "*.sh" -exec shellcheck {} \;
  allow_failure: false

yaml-lint:
  stage: lint
  image: cytopia/yamllint:latest
  script:
    - yamllint -c .yamllint.yml .
  allow_failure: true

# === Test Stage ===
unit-tests:
  stage: test
  image: bash:5.2
  before_script:
    - apk add --no-cache bats
  script:
    - bats tests/unit/
  artifacts:
    when: always
    reports:
      junit: test-results.xml
    paths:
      - test-results/
    expire_in: 1 week

integration-tests:
  stage: test
  image: bash:5.2
  services:
    - postgres:15-alpine
    - redis:7-alpine
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
  script:
    - ./tests/run_integration_tests.sh
  allow_failure: false

# === Build Stage ===
build-docker:
  stage: build
  image: docker:24-dind
  services:
    - docker:24-dind
  variables:
    IMAGE_TAG: ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
    - docker tag $IMAGE_TAG ${CI_REGISTRY_IMAGE}:latest
    - docker push ${CI_REGISTRY_IMAGE}:latest
  only:
    - main
    - develop
    - tags

# === Security Stage ===
container-scanning:
  stage: security
  image: docker:24-dind
  services:
    - docker:24-dind
  variables:
    DOCKER_IMAGE: ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}
  script:
    - docker run --rm aquasec/trivy image $DOCKER_IMAGE
  allow_failure: true

sast:
  stage: security
  include:
    - template: Security/SAST.gitlab-ci.yml

# === Deploy Staging ===
deploy-staging:
  stage: deploy-staging
  image: bitnami/kubectl:latest
  environment:
    name: staging
    url: https://staging.myapp.com
  script:
    - kubectl config use-context staging
    - kubectl set image deployment/${APP_NAME} ${APP_NAME}=${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA} -n staging
    - kubectl rollout status deployment/${APP_NAME} -n staging
  only:
    - develop
    - main

# === Deploy Production ===
deploy-production:
  stage: deploy-production
  image: bitnami/kubectl:latest
  environment:
    name: production
    url: https://myapp.com
  script:
    - kubectl config use-context production
    - kubectl set image deployment/${APP_NAME} ${APP_NAME}=${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA} -n production
    - kubectl rollout status deployment/${APP_NAME} -n production
  when: manual
  only:
    - main
    - tags

# Notification
notify-slack:
  stage: .post
  image: curlimages/curl:latest
  script:
    - |
      STATUS="${CI_JOB_STATUS:-unknown}"
      COLOR="good"
      [[ "$STATUS" == "failed" ]] && COLOR="danger"
      curl -X POST "$SLACK_WEBHOOK_URL" \
        -H 'Content-Type: application/json' \
        -d "{
          \"attachments\": [{
            \"color\": \"$COLOR\",
            \"title\": \"Pipeline ${STATUS}: ${CI_PROJECT_NAME}\",
            \"text\": \"Branch: ${CI_COMMIT_REF_NAME} | Commit: ${CI_COMMIT_SHORT_SHA}\",
            \"footer\": \"GitLab CI\"
          }]
        }"
  when: always
  allow_failure: true
YAML
    
    echo "✓ สร้าง .gitlab-ci.yml"
}

# Main
case "${1:-}" in
    list)     list_pipelines "${2:-main}" "${3:-10}" ;;
    jobs)     get_pipeline_jobs "${2}" ;;
    trigger)  trigger_pipeline "${2:-main}" "${3:-}" ;;
    generate) generate_gitlab_ci "${2:-.gitlab-ci.yml}" ;;
    *)
        echo "Usage: $0 {list|jobs|trigger|generate} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 469: Jenkins Pipeline Automation

```bash
#!/bin/bash
# jenkins_manager.sh - จัดการ Jenkins

JENKINS_URL="${JENKINS_URL:-http://localhost:8080}"
JENKINS_USER="${JENKINS_USER:-admin}"
JENKINS_TOKEN="${JENKINS_TOKEN:-}"

# เรียก Jenkins API
call_jenkins_api() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local url="${JENKINS_URL}${endpoint}"
    local curl_args=(
        -s
        -X "$method"
        -u "${JENKINS_USER}:${JENKINS_TOKEN}"
    )
    
    if [[ -n "$data" ]]; then
        curl_args+=(-H "Content-Type: application/x-www-form-urlencoded" -d "$data")
    fi
    
    curl "${curl_args[@]}" "$url"
}

# รับ CSRF crumb
get_crumb() {
    call_jenkins_api "GET" "/crumbIssuer/api/json" | \
        python3 -c "import json,sys; d=json.load(sys.stdin); print(f'{d[\"crumbRequestField\"]}:{d[\"crumb\"]}')" 2>/dev/null
}

# แสดงรายการ jobs
list_jobs() {
    echo "=== Jenkins Jobs ==="
    
    call_jenkins_api "GET" "/api/json?tree=jobs[name,color,url]" | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
jobs = data.get('jobs', [])
color_icons = {
    'blue': '✓',
    'red': '✗',
    'yellow': '⚠',
    'grey': '○',
    'disabled': '⊘',
    'notbuilt': '?'
}
for job in jobs:
    color = job.get('color', 'unknown')
    # Remove _anime suffix for running jobs
    base_color = color.replace('_anime', '')
    icon = color_icons.get(base_color, '?')
    running = ' (running)' if '_anime' in color else ''
    print(f'{icon} {job[\"name\"]}{running}')
" 2>/dev/null
}

# Build job
trigger_build() {
    local job_name="$1"
    local params="${@:2}"
    
    echo "=== Triggering Build: ${job_name} ==="
    
    local crumb
    crumb=$(get_crumb)
    
    local endpoint="/job/${job_name}/build"
    local param_str=""
    
    if [[ -n "$params" ]]; then
        endpoint="/job/${job_name}/buildWithParameters"
        param_str="$params"
    fi
    
    local response_code
    response_code=$(curl -s -o /dev/null -w "%{http_code}" \
        -X POST \
        -u "${JENKINS_USER}:${JENKINS_TOKEN}" \
        -H "${crumb}" \
        "${JENKINS_URL}${endpoint}" \
        ${param_str:+-d "$param_str"})
    
    if [[ "$response_code" == "201" ]]; then
        echo "✓ Build triggered successfully"
    else
        echo "✗ Build trigger failed (HTTP: ${response_code})"
    fi
}

# รอ build เสร็จ
wait_for_build() {
    local job_name="$1"
    local build_number="$2"
    local timeout="${3:-300}"
    
    echo "=== Waiting for Build: ${job_name} #${build_number} ==="
    
    local elapsed=0
    
    while [[ $elapsed -lt $timeout ]]; do
        local response
        response=$(call_jenkins_api "GET" "/job/${job_name}/${build_number}/api/json")
        
        local building result
        building=$(echo "$response" | python3 -c "import json,sys; print(json.load(sys.stdin).get('building', True))" 2>/dev/null)
        result=$(echo "$response" | python3 -c "import json,sys; print(json.load(sys.stdin).get('result', 'null'))" 2>/dev/null)
        
        echo "[${elapsed}s] Building: ${building} | Result: ${result}"
        
        if [[ "$building" == "False" ]]; then
            echo "Build completed: ${result}"
            [[ "$result" == "SUCCESS" ]] && return 0 || return 1
        fi
        
        sleep 10
        elapsed=$((elapsed + 10))
    done
    
    echo "TIMEOUT after ${timeout}s"
    return 1
}

# สร้าง Jenkinsfile
generate_jenkinsfile() {
    local output_file="${1:-Jenkinsfile}"
    
    cat > "$output_file" << 'GROOVY'
#!/usr/bin/env groovy
// Jenkinsfile - Declarative Pipeline

pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: bash
    image: bash:5.2
    command: ['cat']
    tty: true
  - name: docker
    image: docker:24-dind
    command: ['cat']
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
'''
        }
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }
    
    environment {
        APP_NAME = 'myapp'
        REGISTRY = 'registry.example.com'
        IMAGE_TAG = "${env.GIT_COMMIT.take(8)}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log --oneline -5'
            }
        }
        
        stage('Lint') {
            steps {
                container('bash') {
                    sh '''
                        find . -name "*.sh" | head -20 | while read f; do
                            echo "Checking: $f"
                            bash -n "$f" || exit 1
                        done
                    '''
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('bash') {
                            sh '''
                                apk add --no-cache bats
                                bats tests/unit/
                            '''
                        }
                    }
                    post {
                        always {
                            junit 'test-results/*.xml'
                        }
                    }
                }
                stage('Shell Check') {
                    steps {
                        container('bash') {
                            sh 'shellcheck scripts/*.sh || true'
                        }
                    }
                }
            }
        }
        
        stage('Build') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'registry-credentials',
                        usernameVariable: 'REGISTRY_USER',
                        passwordVariable: 'REGISTRY_PASS'
                    )]) {
                        sh '''
                            docker login -u $REGISTRY_USER -p $REGISTRY_PASS $REGISTRY
                            docker build -t ${REGISTRY}/${APP_NAME}:${IMAGE_TAG} .
                            docker push ${REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                        '''
                    }
                }
            }
        }
        
        stage('Deploy Staging') {
            when { branch 'develop' }
            steps {
                sh './scripts/deploy.sh staging ${IMAGE_TAG}'
            }
        }
        
        stage('Deploy Production') {
            when { branch 'main' }
            input {
                message "Deploy to production?"
                ok "Deploy"
                parameters {
                    choice(name: 'STRATEGY', choices: ['rolling', 'blue-green', 'canary'], description: 'Deployment strategy')
                }
            }
            steps {
                sh './scripts/deploy.sh production ${IMAGE_TAG} ${STRATEGY}'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend channel: '#deploys', color: 'good',
                message: "✓ Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            slackSend channel: '#deploys', color: 'danger',
                message: "✗ Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            emailext subject: "Jenkins Build Failed: ${env.JOB_NAME}",
                body: "Build ${env.BUILD_NUMBER} failed. See ${env.BUILD_URL}",
                to: "team@example.com"
        }
    }
}
GROOVY
    
    echo "✓ สร้าง ${output_file}"
}

# Main
case "${1:-}" in
    list)      list_jobs ;;
    build)     trigger_build "${2}" "${@:3}" ;;
    wait)      wait_for_build "${2}" "${3}" "${4:-300}" ;;
    generate)  generate_jenkinsfile "${2:-Jenkinsfile}" ;;
    *)
        echo "Usage: $0 {list|build|wait|generate} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 470: ArgoCD GitOps Automation

```bash
#!/bin/bash
# argocd_manager.sh - จัดการ ArgoCD สำหรับ GitOps

ARGOCD_SERVER="${ARGOCD_SERVER:-argocd.example.com}"
ARGOCD_TOKEN="${ARGOCD_TOKEN:-}"
API_BASE="https://${ARGOCD_SERVER}/api/v1"

# เรียก ArgoCD API
call_argocd_api() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local curl_args=(
        -s
        -X "$method"
        -H "Authorization: Bearer ${ARGOCD_TOKEN}"
        -H "Content-Type: application/json"
        --insecure
    )
    
    [[ -n "$data" ]] && curl_args+=(-d "$data")
    
    curl "${curl_args[@]}" "${API_BASE}${endpoint}"
}

# แสดงรายการ applications
list_applications() {
    echo "=== ArgoCD Applications ==="
    
    call_argocd_api "GET" "/applications" | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
apps = data.get('items', [])
print(f'Total: {len(apps)} applications')
print()
for app in apps:
    meta = app.get('metadata', {})
    status = app.get('status', {})
    health = status.get('health', {}).get('status', 'Unknown')
    sync = status.get('sync', {}).get('status', 'Unknown')
    
    health_icons = {'Healthy': '✓', 'Degraded': '✗', 'Progressing': '⟳', 'Suspended': '⊘'}
    sync_icons = {'Synced': '✓', 'OutOfSync': '↑', 'Unknown': '?'}
    
    h_icon = health_icons.get(health, '?')
    s_icon = sync_icons.get(sync, '?')
    
    print(f'{h_icon} {meta[\"name\"]}')
    print(f'   Health: {health} | Sync: {sync}')
    dest = status.get('sync', {}).get('compareWith', 'N/A')
    source = app.get('spec', {}).get('source', {})
    print(f'   Repo: {source.get(\"repoURL\", \"N/A\")} @ {source.get(\"targetRevision\", \"HEAD\")}')
    print()
" 2>/dev/null
}

# Sync application
sync_application() {
    local app_name="$1"
    local prune="${2:-false}"
    local dry_run="${3:-false}"
    
    echo "=== Syncing Application: ${app_name} ==="
    
    local data
    data=$(cat << EOF
{
    "name": "${app_name}",
    "prune": ${prune},
    "dryRun": ${dry_run}
}
EOF
)
    
    local response
    response=$(call_argocd_api "POST" "/applications/${app_name}/sync" "$data")
    
    echo "$response" | python3 -c "
import json, sys
data = json.load(sys.stdin)
status = data.get('status', {})
phase = status.get('operationState', {}).get('phase', 'Unknown')
print(f'Sync initiated - Phase: {phase}')
" 2>/dev/null || echo "Response: $response"
}

# รอ sync เสร็จ
wait_for_sync() {
    local app_name="$1"
    local timeout="${2:-120}"
    
    echo "=== Waiting for Sync: ${app_name} ==="
    
    local elapsed=0
    
    while [[ $elapsed -lt $timeout ]]; do
        local response
        response=$(call_argocd_api "GET" "/applications/${app_name}")
        
        local health sync
        health=$(echo "$response" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('status',{}).get('health',{}).get('status','Unknown'))" 2>/dev/null)
        sync=$(echo "$response" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('status',{}).get('sync',{}).get('status','Unknown'))" 2>/dev/null)
        
        echo "[${elapsed}s] Health: ${health} | Sync: ${sync}"
        
        if [[ "$sync" == "Synced" ]] && [[ "$health" == "Healthy" ]]; then
            echo "✓ Application synced and healthy"
            return 0
        elif [[ "$health" == "Degraded" ]]; then
            echo "✗ Application is degraded"
            return 1
        fi
        
        sleep 5
        elapsed=$((elapsed + 5))
    done
    
    echo "TIMEOUT after ${timeout}s"
    return 1
}

# สร้าง ArgoCD Application manifest
create_app_manifest() {
    local app_name="$1"
    local repo_url="$2"
    local path="$3"
    local namespace="${4:-default}"
    local cluster="${5:-https://kubernetes.default.svc}"
    
    cat << YAML
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ${app_name}
  namespace: argocd
  labels:
    app.kubernetes.io/name: ${app_name}
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: ${repo_url}
    targetRevision: HEAD
    path: ${path}
    helm:
      valueFiles:
        - values.yaml
        - values-production.yaml
  destination:
    server: ${cluster}
    namespace: ${namespace}
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10
YAML
}

# GitOps workflow - อัพเดท image tag
gitops_update_image() {
    local repo_path="$1"
    local service="$2"
    local new_tag="$3"
    local values_file="${4:-helm/values.yaml}"
    
    echo "=== GitOps: Updating ${service} to ${new_tag} ==="
    
    # อัพเดท values.yaml
    if [[ -f "${repo_path}/${values_file}" ]]; then
        # ใช้ sed อัพเดท image tag
        sed -i "s|tag:.*${service}.*|tag: ${new_tag}|g" "${repo_path}/${values_file}"
        
        # หรือใช้ yq ถ้ามี
        if command -v yq &>/dev/null; then
            yq -i ".${service}.image.tag = \"${new_tag}\"" "${repo_path}/${values_file}"
        fi
    fi
    
    # Commit และ push
    cd "$repo_path" || return 1
    
    git config user.email "cicd@example.com"
    git config user.name "CI/CD Bot"
    
    git add "$values_file"
    git commit -m "chore: update ${service} to ${new_tag}

[skip ci]

Automated update by CI/CD pipeline.
Service: ${service}
New tag: ${new_tag}
Triggered at: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
    
    git push origin HEAD
    
    echo "✓ GitOps update pushed"
}

# Main
case "${1:-}" in
    list)       list_applications ;;
    sync)       sync_application "${2}" "${3:-false}" "${4:-false}" ;;
    wait)       wait_for_sync "${2}" "${3:-120}" ;;
    manifest)   create_app_manifest "${2}" "${3}" "${4}" "${5:-default}" ;;
    update)     gitops_update_image "${2}" "${3}" "${4}" "${5:-}" ;;
    *)
        echo "Usage: $0 {list|sync|wait|manifest|update} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 471: Pipeline Orchestration และ Parallel Execution

```bash
#!/bin/bash
# pipeline_orchestrator.sh - ควบคุม pipeline แบบ parallel

# Pipeline configuration
declare -A JOB_DEPS         # dependency map
declare -A JOB_STATUS       # status: pending/running/done/failed
declare -A JOB_PIDS         # process IDs
declare -A JOB_LOGS         # log files
declare -a JOB_ORDER        # execution order

# สีสำหรับ output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
CYAN='\033[0;36m'
NC='\033[0m'

# Log directory
LOG_DIR="/tmp/pipeline_$$"
mkdir -p "$LOG_DIR"

# เพิ่ม job เข้า pipeline
add_job() {
    local name="$1"
    local command="$2"
    local deps="${3:-}"       # comma-separated dependencies
    
    JOB_STATUS[$name]="pending"
    JOB_DEPS[$name]="$deps"
    JOB_LOGS[$name]="${LOG_DIR}/${name}.log"
    JOB_ORDER+=("$name")
}

# ตรวจสอบว่า dependencies เสร็จหมดแล้วหรือยัง
deps_satisfied() {
    local job="$1"
    local deps="${JOB_DEPS[$job]}"
    
    [[ -z "$deps" ]] && return 0
    
    IFS=',' read -ra dep_list <<< "$deps"
    for dep in "${dep_list[@]}"; do
        dep="${dep// /}"  # trim spaces
        if [[ "${JOB_STATUS[$dep]}" != "done" ]]; then
            return 1
        fi
    done
    return 0
}

# รัน job ใน background
run_job_async() {
    local name="$1"
    local command="$2"
    
    JOB_STATUS[$name]="running"
    
    # รัน job และบันทึก log
    (
        echo "=== Job: ${name} started at $(date) ==="
        eval "$command"
        exit_code=$?
        echo "=== Job: ${name} finished at $(date) (exit: $exit_code) ==="
        exit $exit_code
    ) > "${JOB_LOGS[$name]}" 2>&1 &
    
    JOB_PIDS[$name]=$!
}

# ตรวจสอบ job ที่กำลังรัน
check_running_jobs() {
    local all_done=true
    
    for name in "${JOB_ORDER[@]}"; do
        if [[ "${JOB_STATUS[$name]}" == "running" ]]; then
            local pid="${JOB_PIDS[$name]}"
            
            if kill -0 "$pid" 2>/dev/null; then
                all_done=false
            else
                # Job เสร็จแล้ว ตรวจสอบ exit code
                wait "$pid"
                local exit_code=$?
                
                if [[ $exit_code -eq 0 ]]; then
                    JOB_STATUS[$name]="done"
                    echo -e "${GREEN}✓ Job '${name}' completed${NC}"
                else
                    JOB_STATUS[$name]="failed"
                    echo -e "${RED}✗ Job '${name}' failed (exit: ${exit_code})${NC}"
                    echo "  Log: ${JOB_LOGS[$name]}"
                fi
            fi
        fi
    done
    
    return $(if $all_done; then echo 0; else echo 1; fi)
}

# แสดง pipeline status
show_pipeline_status() {
    local clear_screen="${1:-false}"
    
    [[ "$clear_screen" == "true" ]] && clear
    
    echo ""
    echo "======================================"
    echo "         PIPELINE STATUS"
    echo "======================================"
    
    local pending=0 running=0 done=0 failed=0
    
    for name in "${JOB_ORDER[@]}"; do
        local status="${JOB_STATUS[$name]:-pending}"
        local deps="${JOB_DEPS[$name]:-none}"
        
        case "$status" in
            pending) echo -e "  ${YELLOW}⏳ ${name}${NC} (deps: ${deps})" ; ((pending++)) ;;
            running) echo -e "  ${BLUE}⟳ ${name}${NC} (PID: ${JOB_PIDS[$name]})" ; ((running++)) ;;
            done)    echo -e "  ${GREEN}✓ ${name}${NC}" ; ((done++)) ;;
            failed)  echo -e "  ${RED}✗ ${name}${NC}" ; ((failed++)) ;;
        esac
    done
    
    echo ""
    echo "Pending: ${pending} | Running: ${running} | Done: ${done} | Failed: ${failed}"
    echo "======================================"
}

# รัน pipeline
run_pipeline() {
    local max_parallel="${1:-4}"
    local fail_fast="${2:-true}"
    
    echo -e "${CYAN}=== Starting Pipeline (max_parallel: ${max_parallel}) ===${NC}"
    
    local pipeline_start=$(date +%s)
    local pipeline_failed=false
    
    while true; do
        # ตรวจสอบ running jobs
        local running_count=0
        for name in "${JOB_ORDER[@]}"; do
            [[ "${JOB_STATUS[$name]}" == "running" ]] && ((running_count++))
        done
        
        # ตรวจสอบว่ามี failed jobs ไหม
        local has_failed=false
        for name in "${JOB_ORDER[@]}"; do
            if [[ "${JOB_STATUS[$name]}" == "failed" ]]; then
                has_failed=true
                pipeline_failed=true
                break
            fi
        done
        
        # Fail fast - หยุดถ้ามี job ล้มเหลว
        if $has_failed && [[ "$fail_fast" == "true" ]]; then
            echo -e "${RED}Pipeline failed! (fail_fast=true)${NC}"
            
            # Kill running jobs
            for name in "${JOB_ORDER[@]}"; do
                if [[ "${JOB_STATUS[$name]}" == "running" ]]; then
                    kill "${JOB_PIDS[$name]}" 2>/dev/null
                    JOB_STATUS[$name]="canceled"
                fi
            done
            break
        fi
        
        # Start new jobs ถ้า slots ว่าง
        local pending_exists=false
        for name in "${JOB_ORDER[@]}"; do
            if [[ "${JOB_STATUS[$name]}" == "pending" ]]; then
                pending_exists=true
                
                if [[ $running_count -lt $max_parallel ]]; then
                    if deps_satisfied "$name"; then
                        echo -e "${BLUE}▶ Starting job: ${name}${NC}"
                        
                        # หา command จาก array
                        local cmd="${JOB_COMMANDS[$name]}"
                        run_job_async "$name" "$cmd"
                        ((running_count++))
                    fi
                fi
            fi
        done
        
        # ตรวจสอบ running jobs
        check_running_jobs
        
        # Pipeline เสร็จถ้าไม่มี pending หรือ running
        local active_count=0
        for name in "${JOB_ORDER[@]}"; do
            local s="${JOB_STATUS[$name]}"
            [[ "$s" == "pending" || "$s" == "running" ]] && ((active_count++))
        done
        
        [[ $active_count -eq 0 ]] && break
        
        sleep 1
    done
    
    local pipeline_end=$(date +%s)
    local pipeline_duration=$((pipeline_end - pipeline_start))
    
    show_pipeline_status
    
    echo ""
    echo "Pipeline duration: ${pipeline_duration}s"
    
    if $pipeline_failed; then
        echo -e "${RED}Pipeline FAILED${NC}"
        
        # แสดง failed job logs
        for name in "${JOB_ORDER[@]}"; do
            if [[ "${JOB_STATUS[$name]}" == "failed" ]]; then
                echo ""
                echo -e "${RED}=== Failed Job: ${name} ===${NC}"
                cat "${JOB_LOGS[$name]}" | tail -20
            fi
        done
        return 1
    else
        echo -e "${GREEN}Pipeline SUCCESS${NC}"
        return 0
    fi
}

# ตัวอย่างการใช้งาน
demo_pipeline() {
    # กำหนด commands
    declare -g -A JOB_COMMANDS
    
    JOB_COMMANDS["checkout"]="echo 'Checking out code...' && sleep 2"
    JOB_COMMANDS["lint"]="echo 'Running linters...' && sleep 3"
    JOB_COMMANDS["unit_tests"]="echo 'Running unit tests...' && sleep 4"
    JOB_COMMANDS["build_app"]="echo 'Building application...' && sleep 3"
    JOB_COMMANDS["build_docker"]="echo 'Building Docker image...' && sleep 5"
    JOB_COMMANDS["security_scan"]="echo 'Running security scan...' && sleep 2"
    JOB_COMMANDS["deploy_staging"]="echo 'Deploying to staging...' && sleep 3"
    JOB_COMMANDS["smoke_test"]="echo 'Running smoke tests...' && sleep 2"
    JOB_COMMANDS["deploy_prod"]="echo 'Deploying to production...' && sleep 4"
    
    # กำหนด jobs และ dependencies
    add_job "checkout" "${JOB_COMMANDS[checkout]}" ""
    add_job "lint" "${JOB_COMMANDS[lint]}" "checkout"
    add_job "unit_tests" "${JOB_COMMANDS[unit_tests]}" "checkout"
    add_job "build_app" "${JOB_COMMANDS[build_app]}" "lint,unit_tests"
    add_job "build_docker" "${JOB_COMMANDS[build_docker]}" "build_app"
    add_job "security_scan" "${JOB_COMMANDS[security_scan]}" "build_docker"
    add_job "deploy_staging" "${JOB_COMMANDS[deploy_staging]}" "security_scan"
    add_job "smoke_test" "${JOB_COMMANDS[smoke_test]}" "deploy_staging"
    add_job "deploy_prod" "${JOB_COMMANDS[deploy_prod]}" "smoke_test"
    
    # รัน pipeline
    run_pipeline 3 true
}

demo_pipeline
```

---

## ขั้นตอนที่ 472: Deployment Strategies — Blue-Green และ Canary

```bash
#!/bin/bash
# deployment_strategies.sh - Blue-Green และ Canary Deployments

KUBECTL="${KUBECTL:-kubectl}"
NAMESPACE="${NAMESPACE:-production}"

# === Blue-Green Deployment ===
blue_green_deploy() {
    local app_name="$1"
    local new_image="$2"
    local service_name="${3:-${app_name}}"
    
    echo "=== Blue-Green Deployment: ${app_name} ==="
    
    # ระบุว่า active อยู่ที่ blue หรือ green
    local current_color
    current_color=$($KUBECTL get service "$service_name" \
        -n "$NAMESPACE" \
        -o jsonpath='{.spec.selector.version}' 2>/dev/null || echo "blue")
    
    local new_color
    [[ "$current_color" == "blue" ]] && new_color="green" || new_color="blue"
    
    echo "Current: ${current_color} → New: ${new_color}"
    
    # Deploy new version
    echo "Deploying new ${new_color} version..."
    
    cat << EOF | $KUBECTL apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${app_name}-${new_color}
  namespace: ${NAMESPACE}
  labels:
    app: ${app_name}
    version: ${new_color}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ${app_name}
      version: ${new_color}
  template:
    metadata:
      labels:
        app: ${app_name}
        version: ${new_color}
    spec:
      containers:
      - name: ${app_name}
        image: ${new_image}
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
EOF
    
    # รอให้ deployment พร้อม
    echo "Waiting for ${new_color} deployment to be ready..."
    $KUBECTL rollout status deployment/${app_name}-${new_color} \
        -n "$NAMESPACE" --timeout=300s
    
    # Smoke test
    echo "Running smoke test on ${new_color}..."
    local new_pod
    new_pod=$($KUBECTL get pods \
        -n "$NAMESPACE" \
        -l "app=${app_name},version=${new_color}" \
        -o jsonpath='{.items[0].metadata.name}')
    
    local test_result
    if $KUBECTL exec "$new_pod" -n "$NAMESPACE" -- curl -sf localhost:8080/health &>/dev/null; then
        echo "✓ Smoke test passed"
    else
        echo "✗ Smoke test failed, rolling back"
        $KUBECTL delete deployment ${app_name}-${new_color} -n "$NAMESPACE"
        return 1
    fi
    
    # Switch traffic
    echo "Switching traffic to ${new_color}..."
    $KUBECTL patch service "$service_name" \
        -n "$NAMESPACE" \
        -p "{\"spec\":{\"selector\":{\"app\":\"${app_name}\",\"version\":\"${new_color}\"}}}"
    
    echo "✓ Traffic switched to ${new_color}"
    
    # ลบ old deployment หลัง delay
    echo "Waiting 60s before removing ${current_color}..."
    sleep 60
    $KUBECTL delete deployment ${app_name}-${current_color} -n "$NAMESPACE" --ignore-not-found
    
    echo "✓ Blue-Green deployment complete"
}

# === Canary Deployment ===
canary_deploy() {
    local app_name="$1"
    local new_image="$2"
    local canary_weight="${3:-10}"  # เปอร์เซ็นต์ traffic ที่ไปหา canary
    local stable_replicas="${4:-9}"
    
    echo "=== Canary Deployment: ${app_name} (${canary_weight}% canary) ==="
    
    # คำนวณจำนวน replicas
    local total_replicas=$((stable_replicas + (stable_replicas * canary_weight / (100 - canary_weight)) + 1))
    local canary_replicas=$((total_replicas - stable_replicas))
    
    echo "Stable replicas: ${stable_replicas} | Canary replicas: ${canary_replicas}"
    
    # Deploy canary
    cat << EOF | $KUBECTL apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${app_name}-canary
  namespace: ${NAMESPACE}
  labels:
    app: ${app_name}
    track: canary
  annotations:
    deployment.kubernetes.io/canary: "true"
    deployment.kubernetes.io/canary-weight: "${canary_weight}"
spec:
  replicas: ${canary_replicas}
  selector:
    matchLabels:
      app: ${app_name}
      track: canary
  template:
    metadata:
      labels:
        app: ${app_name}
        track: canary
    spec:
      containers:
      - name: ${app_name}
        image: ${new_image}
        ports:
        - containerPort: 8080
EOF
    
    # รอให้ canary พร้อม
    $KUBECTL rollout status deployment/${app_name}-canary \
        -n "$NAMESPACE" --timeout=120s
    
    echo "✓ Canary deployed (${canary_weight}% traffic)"
    echo ""
    echo "Monitor canary for issues..."
    echo "  - Check error rates"
    echo "  - Check latency"
    echo "  - Check logs"
    echo ""
    
    # Canary analysis (ตัวอย่าง)
    local canary_ok=true
    local max_errors=5
    local error_count=0
    
    for i in $(seq 1 10); do
        echo "Checking canary metrics (${i}/10)..."
        
        # ตรวจสอบ error logs
        local recent_errors
        recent_errors=$($KUBECTL logs \
            -l "app=${app_name},track=canary" \
            -n "$NAMESPACE" \
            --since=30s 2>/dev/null | grep -c "ERROR" || echo "0")
        
        error_count=$((error_count + recent_errors))
        
        if [[ $error_count -gt $max_errors ]]; then
            canary_ok=false
            break
        fi
        
        sleep 30
    done
    
    if $canary_ok; then
        echo "✓ Canary analysis passed, promoting..."
        promote_canary "$app_name" "$new_image" "$stable_replicas"
    else
        echo "✗ Canary analysis failed (${error_count} errors), rolling back..."
        rollback_canary "$app_name"
        return 1
    fi
}

# โปรโมท canary เป็น stable
promote_canary() {
    local app_name="$1"
    local new_image="$2"
    local replicas="${3:-3}"
    
    echo "=== Promoting Canary to Stable ==="
    
    # อัพเดท stable deployment
    $KUBECTL set image deployment/${app_name} \
        ${app_name}=${new_image} \
        -n "$NAMESPACE"
    
    $KUBECTL rollout status deployment/${app_name} \
        -n "$NAMESPACE" --timeout=300s
    
    # ลบ canary
    $KUBECTL delete deployment ${app_name}-canary -n "$NAMESPACE"
    
    echo "✓ Canary promoted to stable"
}

# Rollback canary
rollback_canary() {
    local app_name="$1"
    
    echo "=== Rolling Back Canary ==="
    $KUBECTL delete deployment ${app_name}-canary -n "$NAMESPACE" --ignore-not-found
    echo "✓ Canary rolled back"
}

# === Rolling Deployment ===
rolling_deploy() {
    local app_name="$1"
    local new_image="$2"
    local max_surge="${3:-1}"
    local max_unavailable="${4:-0}"
    
    echo "=== Rolling Deployment: ${app_name} ==="
    
    # อัพเดท strategy
    $KUBECTL patch deployment "$app_name" \
        -n "$NAMESPACE" \
        -p "{
            \"spec\": {
                \"strategy\": {
                    \"type\": \"RollingUpdate\",
                    \"rollingUpdate\": {
                        \"maxSurge\": ${max_surge},
                        \"maxUnavailable\": ${max_unavailable}
                    }
                }
            }
        }"
    
    # อัพเดท image
    $KUBECTL set image deployment/${app_name} \
        ${app_name}=${new_image} \
        -n "$NAMESPACE"
    
    # รอให้เสร็จ
    $KUBECTL rollout status deployment/${app_name} \
        -n "$NAMESPACE" --timeout=300s
    
    echo "✓ Rolling deployment complete"
    
    # แสดงประวัติ
    $KUBECTL rollout history deployment/${app_name} -n "$NAMESPACE"
}

# Main
case "${1:-}" in
    blue-green) blue_green_deploy "${2}" "${3}" "${4:-}" ;;
    canary)     canary_deploy "${2}" "${3}" "${4:-10}" "${5:-9}" ;;
    rolling)    rolling_deploy "${2}" "${3}" "${4:-1}" "${5:-0}" ;;
    rollback)
        APP="${2}"
        $KUBECTL rollout undo deployment/${APP} -n "$NAMESPACE"
        $KUBECTL rollout status deployment/${APP} -n "$NAMESPACE"
        ;;
    *)
        echo "Usage: $0 {blue-green|canary|rolling|rollback} <app> <image> [options]"
        ;;
esac
```

---

## Workshop: CI/CD Pipeline แบบ Complete

```bash
#!/bin/bash
# complete_cicd_pipeline.sh - Workshop: Full CI/CD Pipeline

set -euo pipefail

# === Configuration ===
APP_NAME="${APP_NAME:-myapp}"
REPO_URL="${REPO_URL:-https://github.com/example/myapp}"
REGISTRY="${REGISTRY:-registry.example.com}"
ENVIRONMENTS=("development" "staging" "production")

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
NC='\033[0m'

# === Pipeline State ===
PIPELINE_ID="$(date +%Y%m%d-%H%M%S)-$$"
PIPELINE_LOG="/tmp/pipeline-${PIPELINE_ID}.log"
PIPELINE_START=$(date +%s)

log() { echo "[$(date +%H:%M:%S)] $*" | tee -a "$PIPELINE_LOG"; }
log_ok()    { echo -e "${GREEN}[$(date +%H:%M:%S)] ✓ $*${NC}" | tee -a "$PIPELINE_LOG"; }
log_err()   { echo -e "${RED}[$(date +%H:%M:%S)] ✗ $*${NC}" | tee -a "$PIPELINE_LOG"; }
log_info()  { echo -e "${BLUE}[$(date +%H:%M:%S)] ℹ $*${NC}" | tee -a "$PIPELINE_LOG"; }
log_warn()  { echo -e "${YELLOW}[$(date +%H:%M:%S)] ⚠ $*${NC}" | tee -a "$PIPELINE_LOG"; }
log_stage() { echo -e "\n${CYAN}${BOLD}[$(date +%H:%M:%S)] ▶ STAGE: $*${NC}" | tee -a "$PIPELINE_LOG"; }

# Banner
print_banner() {
    echo -e "${BOLD}"
    cat << 'BANNER'
 ██████╗██╗      ██████╗██████╗ 
██╔════╝██║     ██╔════╝██╔══██╗
██║     ██║     ██║     ██║  ██║
██║     ██║     ██║     ██║  ██║
╚██████╗███████╗╚██████╗██████╔╝
 ╚═════╝╚══════╝ ╚═════╝╚═════╝ 
BANNER
    echo -e "${NC}"
    echo "Pipeline ID: ${PIPELINE_ID}"
    echo "Application: ${APP_NAME}"
    echo "Repository:  ${REPO_URL}"
    echo ""
}

# === Stage Implementations ===

stage_checkout() {
    log_stage "CHECKOUT"
    
    log_info "Fetching latest code..."
    
    # จำลอง checkout
    local commit_sha
    commit_sha=$(git -C "${WORKSPACE:-/tmp}" rev-parse --short HEAD 2>/dev/null || echo "abc1234")
    
    local branch
    branch=$(git -C "${WORKSPACE:-/tmp}" branch --show-current 2>/dev/null || echo "main")
    
    log_ok "Checked out branch: ${branch} @ ${commit_sha}"
    
    # Export สำหรับ stages ถัดไป
    export GIT_COMMIT="${commit_sha}"
    export GIT_BRANCH="${branch}"
    export BUILD_TAG="${APP_NAME}:${commit_sha}"
}

stage_lint() {
    log_stage "LINT & CODE QUALITY"
    
    local checks_passed=0
    local checks_failed=0
    
    # ShellCheck
    log_info "Running ShellCheck..."
    if find "${WORKSPACE:-.}" -name "*.sh" -exec bash -n {} \; 2>/dev/null; then
        log_ok "ShellCheck passed"
        ((checks_passed++))
    else
        log_warn "ShellCheck found issues (non-blocking)"
    fi
    
    # YAML lint
    log_info "Checking YAML files..."
    local yaml_errors=0
    while IFS= read -r yaml_file; do
        if ! python3 -c "import yaml; yaml.safe_load(open('${yaml_file}'))" 2>/dev/null; then
            ((yaml_errors++))
        fi
    done < <(find "${WORKSPACE:-.}" -name "*.yml" -o -name "*.yaml" 2>/dev/null | head -20)
    
    if [[ $yaml_errors -eq 0 ]]; then
        log_ok "YAML validation passed"
        ((checks_passed++))
    else
        log_err "${yaml_errors} YAML files have errors"
        ((checks_failed++))
    fi
    
    # Markdown links (ตัวอย่าง)
    log_info "Checking Markdown files..."
    log_ok "Markdown check passed"
    ((checks_passed++))
    
    log_ok "Lint completed: ${checks_passed} passed, ${checks_failed} failed"
    
    [[ $checks_failed -gt 0 ]] && return 1 || return 0
}

stage_test() {
    log_stage "TESTS"
    
    local test_results_dir="/tmp/test-results-${PIPELINE_ID}"
    mkdir -p "$test_results_dir"
    
    # Unit Tests
    log_info "Running unit tests..."
    local unit_passed=0
    local unit_failed=0
    
    # จำลอง unit tests
    local test_functions=("test_config_loading" "test_logging" "test_retry_logic" "test_api_calls" "test_data_validation")
    for func in "${test_functions[@]}"; do
        if [[ $((RANDOM % 10)) -gt 1 ]]; then  # 90% pass rate
            ((unit_passed++))
        else
            ((unit_failed++))
            log_warn "Test failed: ${func}"
        fi
    done
    
    log_ok "Unit tests: ${unit_passed} passed, ${unit_failed} failed"
    
    # สร้าง JUnit XML
    cat > "${test_results_dir}/unit-tests.xml" << XML
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="Unit Tests" tests="$((unit_passed + unit_failed))" failures="${unit_failed}" timestamp="$(date -u +%Y-%m-%dT%H:%M:%S)">
    $(for f in "${test_functions[@]}"; do echo "  <testcase name=\"${f}\" classname=\"UnitTest\" time=\"0.1\"/>"; done)
</testsuite>
XML
    
    # Integration Tests (ถ้า environment พร้อม)
    log_info "Running integration tests..."
    log_ok "Integration tests: 5 passed, 0 failed"
    
    # Coverage
    log_info "Calculating code coverage..."
    local coverage=$((75 + RANDOM % 20))
    log_ok "Code coverage: ${coverage}%"
    
    if [[ $coverage -lt 70 ]]; then
        log_err "Code coverage below minimum (70%)"
        return 1
    fi
    
    [[ $unit_failed -gt 0 ]] && return 1 || return 0
}

stage_build() {
    log_stage "BUILD"
    
    log_info "Building application..."
    
    # จำลอง build
    local build_start=$(date +%s)
    
    # Build steps
    local steps=("Downloading dependencies" "Compiling sources" "Running asset pipeline" "Creating artifacts")
    for step in "${steps[@]}"; do
        log_info "${step}..."
        sleep 0.5  # จำลองเวลา
    done
    
    local build_duration=$(($(date +%s) - build_start))
    
    # สร้าง build artifacts
    local artifact_path="/tmp/artifacts-${PIPELINE_ID}"
    mkdir -p "$artifact_path"
    
    echo "VERSION=${GIT_COMMIT:-unknown}" > "${artifact_path}/build.env"
    echo "BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> "${artifact_path}/build.env"
    echo "APP_NAME=${APP_NAME}" >> "${artifact_path}/build.env"
    
    log_ok "Build completed in ${build_duration}s"
    log_ok "Artifacts: ${artifact_path}"
    
    export ARTIFACT_PATH="$artifact_path"
}

stage_security_scan() {
    log_stage "SECURITY SCAN"
    
    local issues_critical=0
    local issues_high=0
    local issues_medium=0
    
    # SAST (Static Application Security Testing)
    log_info "Running SAST scan..."
    
    # ตรวจสอบ secrets ใน code
    log_info "Scanning for secrets..."
    local potential_secrets=0
    while IFS= read -r file; do
        if grep -qE "(password|secret|api_key|token)\s*=\s*['\"][^'\"]+['\"]" "$file" 2>/dev/null; then
            ((potential_secrets++))
            log_warn "Potential secret found in: ${file}"
        fi
    done < <(find "${WORKSPACE:-.}" -name "*.sh" -o -name "*.env" 2>/dev/null | head -20)
    
    if [[ $potential_secrets -gt 0 ]]; then
        log_warn "${potential_secrets} potential secrets found"
        ((issues_high++))
    else
        log_ok "No secrets detected"
    fi
    
    # Dependency vulnerabilities (จำลอง)
    log_info "Scanning dependencies..."
    log_ok "No critical vulnerabilities found"
    
    # สรุป
    echo ""
    echo "Security Scan Summary:"
    echo "  Critical: ${issues_critical}"
    echo "  High:     ${issues_high}"
    echo "  Medium:   ${issues_medium}"
    
    if [[ $issues_critical -gt 0 ]]; then
        log_err "Critical security issues found!"
        return 1
    fi
    
    log_ok "Security scan passed"
}

stage_deploy() {
    local env="$1"
    log_stage "DEPLOY: ${env^^}"
    
    local env_config
    case "$env" in
        development)
            env_config="replicas=1 cpu_request=100m memory_request=128Mi"
            ;;
        staging)
            env_config="replicas=2 cpu_request=250m memory_request=256Mi"
            ;;
        production)
            env_config="replicas=5 cpu_request=500m memory_request=512Mi"
            ;;
    esac
    
    log_info "Deploying to ${env}..."
    log_info "Config: ${env_config}"
    
    # Deploy steps
    log_info "Applying Kubernetes manifests..."
    sleep 1
    
    log_info "Waiting for rollout..."
    sleep 1
    
    log_info "Running health checks..."
    
    # Health check
    local health_ok=true
    for i in $(seq 1 3); do
        log_info "Health check ${i}/3..."
        sleep 0.5
    done
    
    if $health_ok; then
        log_ok "Deployment to ${env} successful"
    else
        log_err "Health check failed"
        return 1
    fi
    
    # ส่ง notification
    log_info "Sending deployment notification..."
    log_ok "Notification sent"
}

stage_smoke_test() {
    local env="$1"
    log_stage "SMOKE TEST: ${env^^}"
    
    local tests=("GET /health" "GET /api/v1/status" "POST /api/v1/ping" "GET /metrics")
    local passed=0
    local failed=0
    
    for test in "${tests[@]}"; do
        log_info "Testing: ${test}"
        
        # จำลองการทดสอบ
        if [[ $((RANDOM % 10)) -gt 0 ]]; then
            log_ok "${test} → 200 OK"
            ((passed++))
        else
            log_err "${test} → 500 Error"
            ((failed++))
        fi
    done
    
    log_ok "Smoke tests: ${passed} passed, ${failed} failed"
    
    [[ $failed -gt 0 ]] && return 1 || return 0
}

# === Main Pipeline ===
main() {
    print_banner
    
    log_info "Starting CI/CD Pipeline"
    log_info "Pipeline ID: ${PIPELINE_ID}"
    log_info "Application: ${APP_NAME}"
    echo ""
    
    # Pipeline stages
    local stages=(
        "stage_checkout"
        "stage_lint"
        "stage_test"
        "stage_build"
        "stage_security_scan"
    )
    
    # รัน stages
    for stage in "${stages[@]}"; do
        if ! $stage; then
            log_err "Pipeline failed at stage: ${stage}"
            send_failure_notification "${stage}"
            exit 1
        fi
    done
    
    # Deploy stages
    for env in "development" "staging"; do
        if ! stage_deploy "$env"; then
            log_err "Deployment failed: ${env}"
            exit 1
        fi
        
        if ! stage_smoke_test "$env"; then
            log_err "Smoke test failed: ${env}"
            exit 1
        fi
    done
    
    # Production deploy (manual approval ในที่จริง)
    if [[ "${AUTO_DEPLOY_PROD:-false}" == "true" ]]; then
        if ! stage_deploy "production"; then
            log_err "Production deployment failed"
            exit 1
        fi
        
        if ! stage_smoke_test "production"; then
            log_err "Production smoke test failed"
            exit 1
        fi
    fi
    
    # สรุป pipeline
    local pipeline_duration=$(($(date +%s) - PIPELINE_START))
    
    echo ""
    echo "======================================"
    echo -e "${GREEN}${BOLD}  PIPELINE COMPLETED SUCCESSFULLY${NC}"
    echo "======================================"
    echo "Pipeline ID: ${PIPELINE_ID}"
    echo "Duration:    ${pipeline_duration}s"
    echo "Log file:    ${PIPELINE_LOG}"
    echo "======================================"
    
    send_success_notification
}

send_success_notification() {
    log_info "Sending success notification..."
    # ส่ง Slack/email notification ที่นี่
}

send_failure_notification() {
    local failed_stage="$1"
    log_info "Sending failure notification for stage: ${failed_stage}..."
    # ส่ง Slack/email notification ที่นี่
}

# รัน pipeline
main "$@"
```

---

## สรุป Part 28

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | เนื้อหา |
|--------|---------|
| **CI/CD Basics** | Pipeline stages, logging, state management |
| **GitHub Actions** | Workflow management, triggering, YAML generation |
| **GitLab CI** | Pipeline API, .gitlab-ci.yml generation |
| **Jenkins** | Job management, Jenkinsfile generation |
| **ArgoCD** | GitOps workflow, sync management |
| **Parallel Execution** | Job orchestration, dependency management |
| **Blue-Green Deploy** | Zero-downtime deployment |
| **Canary Deploy** | Gradual rollout with analysis |
| **Complete Pipeline** | Workshop: full CI/CD workflow |

**ขั้นตอนต่อไป**: Part 29 - Infrastructure as Code (Terraform, Pulumi)
