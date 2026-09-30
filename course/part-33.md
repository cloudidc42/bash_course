# Part 33: Site Reliability Engineering (SRE) Practices

## Module 3: Advanced Level — การทำงานระดับสูง

---

## ขั้นตอนที่ 490: SLI, SLO, และ SLA Management

SRE (Site Reliability Engineering) คือวิธีการบริหารจัดการระบบที่มีความน่าเชื่อถือสูง

```bash
#!/bin/bash
# sre_slo_manager.sh - จัดการ SLI/SLO/SLA

# SLI = Service Level Indicator (ค่าวัดจริง)
# SLO = Service Level Objective (เป้าหมาย)
# SLA = Service Level Agreement (ข้อตกลงกับลูกค้า)
# Error Budget = เวลาที่ยอมให้ระบบล้มเหลวได้

PROMETHEUS_URL="${PROMETHEUS_URL:-http://localhost:9090}"
SLO_CONFIG="/etc/sre/slos.yaml"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# ===  SLO Definitions ===

# คำนวณ Error Budget
calculate_error_budget() {
    local slo_percent="$1"       # เช่น 99.9
    local window_days="${2:-30}"  # หน้าต่างเวลา
    
    # Error budget = เวลาที่ยอมให้ล้มเหลว
    local budget_percent=$(echo "scale=5; 100 - ${slo_percent}" | bc)
    local window_minutes=$((window_days * 24 * 60))
    local budget_minutes=$(echo "scale=2; ${window_minutes} * ${budget_percent} / 100" | bc)
    local budget_seconds=$(echo "scale=0; ${budget_minutes} * 60" | bc)
    
    echo ""
    echo "=== Error Budget Calculation ==="
    echo "SLO: ${slo_percent}%"
    echo "Window: ${window_days} days"
    echo ""
    echo "Error Budget:"
    echo "  Percentage: ${budget_percent}%"
    echo "  Time:       ${budget_minutes} minutes"
    printf "  Time:       %02d:%02d:%02d (hh:mm:ss)\n" \
        $(echo "${budget_minutes}/60" | bc) \
        $(echo "${budget_minutes}%60" | bc) \
        0
}

# คำนวณ SLI จาก Prometheus
calculate_sli() {
    local sli_type="$1"    # availability, latency, throughput, error_rate
    local service="$2"
    local window="${3:-5m}"
    
    local query=""
    
    case "$sli_type" in
        availability)
            # Availability = successful requests / total requests
            query="sum(rate(http_requests_total{service=\"${service}\",status!~\"5..\"}[${window}])) / sum(rate(http_requests_total{service=\"${service}\"}[${window}])) * 100"
            ;;
        latency)
            # Latency P99
            query="histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service=\"${service}\"}[${window}])) by (le)) * 1000"
            ;;
        throughput)
            # Request rate
            query="sum(rate(http_requests_total{service=\"${service}\"}[${window}]))"
            ;;
        error_rate)
            # Error rate percentage
            query="sum(rate(http_requests_total{service=\"${service}\",status=~\"5..\"}[${window}])) / sum(rate(http_requests_total{service=\"${service}\"}[${window}])) * 100"
            ;;
    esac
    
    if [[ -z "$query" ]]; then
        echo "Unknown SLI type: ${sli_type}"
        return 1
    fi
    
    # Query Prometheus
    local result
    result=$(curl -sf \
        "${PROMETHEUS_URL}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${query}'))" 2>/dev/null)" \
        2>/dev/null | \
        python3 -c "
import json, sys
data = json.load(sys.stdin)
result = data.get('data', {}).get('result', [])
if result:
    print(result[0].get('value', ['', ''])[1])
else:
    print('N/A')
" 2>/dev/null || echo "N/A")
    
    echo "$result"
}

# ตรวจสอบ SLO compliance
check_slo_compliance() {
    local service="$1"
    
    echo "=== SLO Compliance: ${service} ==="
    echo "Time: $(date)"
    echo ""
    
    # กำหนด SLOs
    declare -A SLOs=(
        ["availability"]="99.9"
        ["latency_p99_ms"]="500"
        ["error_rate_max"]="0.1"
    )
    
    # ตรวจสอบแต่ละ SLO
    local all_met=true
    
    # Availability SLO
    local avail
    avail=$(calculate_sli "availability" "$service" "30m")
    
    if [[ "$avail" != "N/A" ]]; then
        local avail_slo="${SLOs[availability]}"
        local avail_ok
        avail_ok=$(python3 -c "print('yes' if float('${avail}') >= ${avail_slo} else 'no')" 2>/dev/null)
        
        if [[ "$avail_ok" == "yes" ]]; then
            echo -e "  ${GREEN}✓ Availability: ${avail}% (SLO: ${avail_slo}%)${NC}"
        else
            echo -e "  ${RED}✗ Availability: ${avail}% (SLO: ${avail_slo}%)${NC}"
            all_met=false
        fi
    else
        echo -e "  ${YELLOW}? Availability: cannot measure${NC}"
    fi
    
    # Latency SLO
    local latency
    latency=$(calculate_sli "latency" "$service" "30m")
    
    if [[ "$latency" != "N/A" ]]; then
        local lat_slo="${SLOs[latency_p99_ms]}"
        local lat_ok
        lat_ok=$(python3 -c "print('yes' if float('${latency}') <= ${lat_slo} else 'no')" 2>/dev/null)
        
        if [[ "$lat_ok" == "yes" ]]; then
            echo -e "  ${GREEN}✓ Latency P99: ${latency}ms (SLO: <${lat_slo}ms)${NC}"
        else
            echo -e "  ${RED}✗ Latency P99: ${latency}ms (SLO: <${lat_slo}ms)${NC}"
            all_met=false
        fi
    fi
    
    # Error Rate SLO
    local error_rate
    error_rate=$(calculate_sli "error_rate" "$service" "30m")
    
    if [[ "$error_rate" != "N/A" ]]; then
        local err_slo="${SLOs[error_rate_max]}"
        local err_ok
        err_ok=$(python3 -c "print('yes' if float('${error_rate}') <= ${err_slo} else 'no')" 2>/dev/null)
        
        if [[ "$err_ok" == "yes" ]]; then
            echo -e "  ${GREEN}✓ Error Rate: ${error_rate}% (SLO: <${err_slo}%)${NC}"
        else
            echo -e "  ${RED}✗ Error Rate: ${error_rate}% (SLO: <${err_slo}%)${NC}"
            all_met=false
        fi
    fi
    
    echo ""
    if $all_met; then
        echo -e "${GREEN}✓ All SLOs are MET${NC}"
        return 0
    else
        echo -e "${RED}✗ One or more SLOs are VIOLATED${NC}"
        return 1
    fi
}

# Error Budget tracking
track_error_budget() {
    local service="$1"
    local slo="${2:-99.9}"
    local window_days="${3:-30}"
    
    echo "=== Error Budget: ${service} ==="
    
    # คำนวณ budget total
    local window_minutes=$((window_days * 24 * 60))
    local budget_percent=$(echo "scale=5; 100 - ${slo}" | bc)
    local total_budget_minutes=$(echo "scale=2; ${window_minutes} * ${budget_percent} / 100" | bc)
    
    # ดึงข้อมูลจาก Prometheus (downtime ใน window)
    local downtime_query="sum_over_time((1 - avg(up{job=\"${service}\"}))[(${window_days}d:1m)]) * 1"
    
    local downtime_minutes
    downtime_minutes=$(curl -sf \
        "${PROMETHEUS_URL}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${downtime_query}'))" 2>/dev/null)" \
        2>/dev/null | \
        python3 -c "
import json, sys
data = json.load(sys.stdin)
result = data.get('data', {}).get('result', [])
if result:
    print(float(result[0].get('value', ['', '0'])[1]))
else:
    print(0)
" 2>/dev/null || echo "0")
    
    local remaining_budget
    remaining_budget=$(python3 -c "
budget = ${total_budget_minutes}
used = ${downtime_minutes}
remaining = budget - used
pct = (remaining / budget * 100) if budget > 0 else 0
print(f'{remaining:.2f}:{pct:.1f}')
" 2>/dev/null || echo "0:100")
    
    local remaining_minutes="${remaining_budget%:*}"
    local remaining_pct="${remaining_budget#*:}"
    
    echo "SLO Target:     ${slo}%"
    echo "Window:         ${window_days} days"
    echo "Total Budget:   ${total_budget_minutes} minutes"
    echo "Budget Used:    ${downtime_minutes} minutes"
    echo "Budget Left:    ${remaining_minutes} minutes (${remaining_pct}%)"
    
    # แสดง burn rate
    local burn_rate
    burn_rate=$(python3 -c "
used = ${downtime_minutes}
total = ${total_budget_minutes}
if total > 0:
    # burn rate = actual consumption / expected consumption
    expected = total / (${window_days} * 24 * 60) * 60  # per hour
    actual_per_hour = used / (${window_days} * 24)
    rate = actual_per_hour / expected if expected > 0 else 0
    print(f'{rate:.2f}')
else:
    print('0')
" 2>/dev/null || echo "0")
    
    echo "Burn Rate:      ${burn_rate}x (1.0 = normal)"
    
    local remaining_float="${remaining_pct%.*}"
    
    if [[ "${remaining_float:-100}" -lt 10 ]]; then
        echo -e "${RED}⚠ CRITICAL: Error budget nearly exhausted!${NC}"
    elif [[ "${remaining_float:-100}" -lt 25 ]]; then
        echo -e "${YELLOW}⚠ WARNING: Error budget running low${NC}"
    else
        echo -e "${GREEN}✓ Error budget healthy${NC}"
    fi
}

# Incident Response
declare_incident() {
    local service="$1"
    local severity="${2:-SEV2}"  # SEV1 (critical) - SEV4 (low)
    local description="$3"
    
    local incident_id="INC-$(date +%Y%m%d)-$$"
    local timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    
    echo "=== INCIDENT DECLARED ==="
    echo "Incident ID:  ${incident_id}"
    echo "Service:      ${service}"
    echo "Severity:     ${severity}"
    echo "Time:         ${timestamp}"
    echo "Description:  ${description}"
    echo ""
    
    # บันทึก incident
    local incident_dir="/var/log/incidents"
    mkdir -p "$incident_dir"
    
    cat > "${incident_dir}/${incident_id}.json" << EOF
{
    "id": "${incident_id}",
    "service": "${service}",
    "severity": "${severity}",
    "status": "open",
    "declared_at": "${timestamp}",
    "description": "${description}",
    "timeline": [
        {
            "time": "${timestamp}",
            "event": "Incident declared",
            "author": "$(whoami)"
        }
    ]
}
EOF
    
    # Notify on-call
    case "$severity" in
        SEV1)
            echo "🚨 SEV1: Paging on-call immediately"
            # PagerDuty/OpsGenie API call ที่นี่
            ;;
        SEV2)
            echo "⚠️  SEV2: Notifying on-call"
            ;;
        SEV3)
            echo "ℹ️  SEV3: Sending notification"
            ;;
    esac
    
    echo ""
    echo "Incident logged: ${incident_dir}/${incident_id}.json"
    echo "Runbook: https://runbooks.example.com/${service}/incidents"
}

# Post-mortem generator
generate_postmortem() {
    local incident_id="$1"
    local output_file="${2:-/tmp/postmortem-${incident_id}.md}"
    
    cat > "$output_file" << MARKDOWN
# Post-mortem: ${incident_id}

**Date:** $(date +%Y-%m-%d)
**Author:** $(whoami)
**Status:** Draft

## Summary
[สรุปเหตุการณ์ใน 2-3 ประโยค]

## Impact
- **Duration:** X hours Y minutes
- **Affected Users:** ~X users
- **Services Affected:** [รายการ services]
- **Error Budget Impact:** X% consumed

## Timeline

| Time | Event | Who |
|------|-------|-----|
| HH:MM | Incident detected by [monitoring/user report] | |
| HH:MM | On-call engineer paged | |
| HH:MM | Incident bridge opened | |
| HH:MM | Root cause identified | |
| HH:MM | Mitigation applied | |
| HH:MM | Service restored | |
| HH:MM | All-clear declared | |

## Root Cause
[อธิบาย root cause ทางเทคนิคอย่างละเอียด]

## Contributing Factors
1. [ปัจจัยที่ 1]
2. [ปัจจัยที่ 2]
3. [ปัจจัยที่ 3]

## What Went Well
- [สิ่งที่ทำได้ดี]
- [การตอบสนองที่รวดเร็ว]

## What Could Be Improved
- [สิ่งที่ควรปรับปรุง]
- [กระบวนการที่ไม่มีประสิทธิภาพ]

## Action Items

| Action | Owner | Priority | Due Date |
|--------|-------|----------|----------|
| [แก้ไข root cause] | [ชื่อ] | High | [วันที่] |
| [เพิ่ม monitoring] | [ชื่อ] | Medium | [วันที่] |
| [อัพเดท runbook] | [ชื่อ] | Low | [วันที่] |

## Lessons Learned
[บทเรียนที่ได้รับ]

---
*Post-mortem follows Google SRE blameless culture guidelines*
MARKDOWN
    
    echo "✓ Post-mortem template created: ${output_file}"
}

# Runbook automation
create_runbook() {
    local service="$1"
    local alert_name="$2"
    local output_file="${3:-/tmp/runbook-${service}-${alert_name}.md}"
    
    cat > "$output_file" << MARKDOWN
# Runbook: ${service} - ${alert_name}

## Overview
**Alert:** ${alert_name}
**Service:** ${service}
**Severity:** [SEV1/SEV2/SEV3]
**On-call:** [ทีมที่รับผิดชอบ]

## Alert Description
[อธิบายว่า alert นี้หมายถึงอะไร]

## Impact Assessment
- [ ] ตรวจสอบ user-facing impact
- [ ] ตรวจสอบ error rate ใน Grafana
- [ ] ตรวจสอบ dependent services

## Diagnosis Steps

### Step 1: Check service health
\`\`\`bash
kubectl get pods -n production -l app=${service}
kubectl logs -n production -l app=${service} --tail=100
\`\`\`

### Step 2: Check metrics
\`\`\`bash
# Open Grafana dashboard
# URL: https://grafana.example.com/d/${service}-overview
\`\`\`

### Step 3: Check recent deployments
\`\`\`bash
kubectl rollout history deployment/${service} -n production
\`\`\`

## Mitigation

### If recent deployment caused issue:
\`\`\`bash
kubectl rollout undo deployment/${service} -n production
kubectl rollout status deployment/${service} -n production
\`\`\`

### If pod is crash-looping:
\`\`\`bash
kubectl delete pod <pod-name> -n production
# Pod will restart automatically
\`\`\`

### If database issue:
\`\`\`bash
# Check database connection
kubectl exec -it <app-pod> -n production -- psql \$DATABASE_URL -c "SELECT 1"
\`\`\`

## Escalation
- If unresolved in 30 minutes → escalate to SEV1
- Contact: [ชื่อผู้เชี่ยวชาญ] via Slack: @[username]

## Post-Incident
- [ ] Update this runbook if needed
- [ ] File post-mortem if SEV1/SEV2
- [ ] Create ticket for permanent fix

---
*Last updated: $(date +%Y-%m-%d)*
MARKDOWN
    
    echo "✓ Runbook created: ${output_file}"
}

# Main
case "${1:-}" in
    budget-calc)   calculate_error_budget "${2:-99.9}" "${3:-30}" ;;
    sli)           calculate_sli "${2}" "${3}" "${4:-5m}" ;;
    slo-check)     check_slo_compliance "${2}" ;;
    budget-track)  track_error_budget "${2}" "${3:-99.9}" "${4:-30}" ;;
    incident)      declare_incident "${2}" "${3:-SEV2}" "${4:-Incident}" ;;
    postmortem)    generate_postmortem "${2}" "${3:-}" ;;
    runbook)       create_runbook "${2}" "${3}" "${4:-}" ;;
    *)
        echo "Usage: $0 {budget-calc|sli|slo-check|budget-track|incident|postmortem|runbook} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 491: Chaos Engineering

```bash
#!/bin/bash
# chaos_engineering.sh - Chaos Engineering tools

NAMESPACE="${NAMESPACE:-production}"
KUBECTL="${KUBECTL:-kubectl}"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# Safety checks ก่อนรัน chaos
chaos_safety_check() {
    local target="$1"
    
    echo "=== Chaos Safety Check ==="
    
    # ตรวจสอบ environment
    local env="${CHAOS_ENV:-staging}"
    
    if [[ "$env" == "production" ]]; then
        echo -e "${RED}WARNING: Running chaos in PRODUCTION${NC}"
        read -rp "Type 'CONFIRM' to proceed: " confirm
        [[ "$confirm" != "CONFIRM" ]] && { echo "Aborted"; return 1; }
    fi
    
    # ตรวจสอบ deployment มีอยู่จริง
    if ! $KUBECTL get deployment "$target" -n "$NAMESPACE" &>/dev/null; then
        echo -e "${RED}ERROR: Deployment '${target}' not found${NC}"
        return 1
    fi
    
    # ตรวจสอบจำนวน replicas
    local replicas
    replicas=$($KUBECTL get deployment "$target" -n "$NAMESPACE" \
        -o jsonpath='{.spec.replicas}' 2>/dev/null || echo "0")
    
    if [[ $replicas -lt 2 ]]; then
        echo -e "${RED}ERROR: Need at least 2 replicas to run chaos safely (found: ${replicas})${NC}"
        return 1
    fi
    
    echo -e "${GREEN}✓ Safety check passed${NC}"
    echo "  Target: ${target}"
    echo "  Namespace: ${NAMESPACE}"
    echo "  Replicas: ${replicas}"
    return 0
}

# Kill random pod
chaos_kill_pod() {
    local deployment="$1"
    local count="${2:-1}"
    
    echo "=== Chaos: Kill Pod (${deployment} x${count}) ==="
    
    chaos_safety_check "$deployment" || return 1
    
    # หา pods
    local pods
    pods=($($KUBECTL get pods -n "$NAMESPACE" \
        -l "app=${deployment}" \
        --no-headers \
        -o custom-columns=":metadata.name" 2>/dev/null))
    
    if [[ ${#pods[@]} -eq 0 ]]; then
        echo "No pods found for deployment: ${deployment}"
        return 1
    fi
    
    # เลือก pods แบบ random
    local killed=0
    local killed_pods=()
    
    for ((i=0; i<count && i<${#pods[@]}; i++)); do
        local random_idx=$((RANDOM % ${#pods[@]}))
        local pod="${pods[$random_idx]}"
        
        # ตรวจสอบ pod ยังไม่ถูก kill
        if [[ " ${killed_pods[*]} " != *" ${pod} "* ]]; then
            echo "  Killing pod: ${pod}"
            $KUBECTL delete pod "$pod" -n "$NAMESPACE" --grace-period=0 &>/dev/null
            killed_pods+=("$pod")
            ((killed++))
        fi
    done
    
    echo "✓ Killed ${killed} pod(s)"
    
    # Monitor recovery
    echo "Monitoring recovery..."
    sleep 5
    
    $KUBECTL rollout status deployment/"$deployment" -n "$NAMESPACE" --timeout=60s 2>/dev/null && \
    echo -e "${GREEN}✓ Deployment recovered${NC}" || \
    echo -e "${RED}✗ Recovery taking too long${NC}"
}

# Network latency injection (ด้วย tc)
chaos_inject_latency() {
    local deployment="$1"
    local latency_ms="${2:-100}"
    local jitter_ms="${3:-20}"
    local duration="${4:-60}"
    
    echo "=== Chaos: Inject Network Latency ==="
    echo "  Deployment: ${deployment}"
    echo "  Latency: ${latency_ms}ms ±${jitter_ms}ms"
    echo "  Duration: ${duration}s"
    
    chaos_safety_check "$deployment" || return 1
    
    # ใช้ Chaos Mesh (ถ้ามี)
    if $KUBECTL get crd networkchaos.chaos-mesh.org &>/dev/null; then
        cat << YAML | $KUBECTL apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: ${deployment}-latency
  namespace: ${NAMESPACE}
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - ${NAMESPACE}
    labelSelectors:
      app: ${deployment}
  delay:
    latency: ${latency_ms}ms
    jitter: ${jitter_ms}ms
    correlation: "25"
  duration: ${duration}s
YAML
        
        echo "✓ Latency injection applied via Chaos Mesh"
        echo "  Will automatically stop after ${duration}s"
        
        # Wait and cleanup
        sleep "$duration"
        $KUBECTL delete networkchaos "${deployment}-latency" -n "$NAMESPACE" &>/dev/null
        echo "✓ Latency injection removed"
        
    else
        echo "Chaos Mesh not available, using tc (requires privileged access)..."
        
        # ใช้ tc ผ่าน kubectl exec
        local pods
        pods=($($KUBECTL get pods -n "$NAMESPACE" \
            -l "app=${deployment}" --no-headers \
            -o custom-columns=":metadata.name" 2>/dev/null))
        
        for pod in "${pods[@]}"; do
            $KUBECTL exec "$pod" -n "$NAMESPACE" -- \
                tc qdisc add dev eth0 root netem delay "${latency_ms}ms" "${jitter_ms}ms" \
                2>/dev/null && echo "  Applied to: ${pod}" || echo "  Failed for: ${pod}"
        done
        
        echo "Waiting ${duration}s..."
        sleep "$duration"
        
        # Remove tc rules
        for pod in "${pods[@]}"; do
            $KUBECTL exec "$pod" -n "$NAMESPACE" -- \
                tc qdisc del dev eth0 root 2>/dev/null || true
        done
        
        echo "✓ Latency injection removed"
    fi
}

# Memory pressure
chaos_memory_pressure() {
    local deployment="$1"
    local memory_mb="${2:-256}"
    local duration="${3:-60}"
    
    echo "=== Chaos: Memory Pressure ==="
    echo "  Deployment: ${deployment}"
    echo "  Memory: ${memory_mb}MB"
    echo "  Duration: ${duration}s"
    
    chaos_safety_check "$deployment" || return 1
    
    # ใช้ Chaos Mesh
    if $KUBECTL get crd stresschaos.chaos-mesh.org &>/dev/null; then
        cat << YAML | $KUBECTL apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: ${deployment}-mem-stress
  namespace: ${NAMESPACE}
spec:
  mode: one
  selector:
    namespaces:
      - ${NAMESPACE}
    labelSelectors:
      app: ${deployment}
  stressors:
    memory:
      workers: 1
      size: ${memory_mb}MB
  duration: ${duration}s
YAML
        
        echo "✓ Memory stress applied"
        sleep "$duration"
        $KUBECTL delete stresschaos "${deployment}-mem-stress" -n "$NAMESPACE" &>/dev/null
        echo "✓ Memory stress removed"
    else
        echo "Chaos Mesh not available"
        return 1
    fi
}

# CPU stress
chaos_cpu_stress() {
    local deployment="$1"
    local cpu_workers="${2:-2}"
    local duration="${3:-60}"
    
    echo "=== Chaos: CPU Stress ==="
    
    chaos_safety_check "$deployment" || return 1
    
    if $KUBECTL get crd stresschaos.chaos-mesh.org &>/dev/null; then
        cat << YAML | $KUBECTL apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: ${deployment}-cpu-stress
  namespace: ${NAMESPACE}
spec:
  mode: one
  selector:
    namespaces:
      - ${NAMESPACE}
    labelSelectors:
      app: ${deployment}
  stressors:
    cpu:
      workers: ${cpu_workers}
  duration: ${duration}s
YAML
        
        echo "✓ CPU stress applied (${cpu_workers} workers, ${duration}s)"
        sleep "$duration"
        $KUBECTL delete stresschaos "${deployment}-cpu-stress" -n "$NAMESPACE" &>/dev/null
        echo "✓ CPU stress removed"
    fi
}

# Chaos experiment scheduler
run_chaos_experiment() {
    local experiment_name="$1"
    local deployment="$2"
    
    echo "=== Running Chaos Experiment: ${experiment_name} ==="
    echo "Start: $(date)"
    
    local experiment_log="/tmp/chaos-${experiment_name}-$$.log"
    
    case "$experiment_name" in
        kill-pods)
            chaos_kill_pod "$deployment" 2 2>&1 | tee "$experiment_log"
            ;;
        latency)
            chaos_inject_latency "$deployment" 200 50 120 2>&1 | tee "$experiment_log"
            ;;
        memory)
            chaos_memory_pressure "$deployment" 512 60 2>&1 | tee "$experiment_log"
            ;;
        cpu)
            chaos_cpu_stress "$deployment" 4 60 2>&1 | tee "$experiment_log"
            ;;
        *)
            echo "Unknown experiment: ${experiment_name}"
            return 1
            ;;
    esac
    
    echo ""
    echo "End: $(date)"
    echo "Log: ${experiment_log}"
}

# Main
case "${1:-}" in
    kill-pod)   chaos_kill_pod "${2}" "${3:-1}" ;;
    latency)    chaos_inject_latency "${2}" "${3:-100}" "${4:-20}" "${5:-60}" ;;
    memory)     chaos_memory_pressure "${2}" "${3:-256}" "${4:-60}" ;;
    cpu)        chaos_cpu_stress "${2}" "${3:-2}" "${4:-60}" ;;
    experiment) run_chaos_experiment "${2}" "${3}" ;;
    *)
        echo "Usage: $0 {kill-pod|latency|memory|cpu|experiment} <deployment> [options]"
        ;;
esac
```

---

## ขั้นตอนที่ 492: Capacity Planning

```bash
#!/bin/bash
# capacity_planning.sh - วางแผน Capacity

PROMETHEUS_URL="${PROMETHEUS_URL:-http://localhost:9090}"

# ดึงข้อมูล resource usage
get_resource_trends() {
    local service="$1"
    local days="${2:-30}"
    
    echo "=== Resource Trends: ${service} (${days} days) ==="
    
    # CPU trend
    local cpu_query="avg(rate(container_cpu_usage_seconds_total{container=\"${service}\"}[1h]))"
    
    # Memory trend
    local mem_query="avg(container_memory_usage_bytes{container=\"${service}\"})"
    
    # Traffic trend
    local req_query="sum(rate(http_requests_total{service=\"${service}\"}[1h]))"
    
    # ดึงข้อมูล (จำลอง)
    echo "CPU Usage (avg last ${days}d):"
    echo "  Current: $(shuf -i 20-60 -n 1)%"
    echo "  Peak:    $(shuf -i 60-90 -n 1)%"
    echo "  Trend:   ↑ +$(shuf -i 2-10 -n 1)% per week"
    
    echo ""
    echo "Memory Usage (avg last ${days}d):"
    echo "  Current: $(shuf -i 300-600 -n 1)MB"
    echo "  Peak:    $(shuf -i 700-900 -n 1)MB"
    echo "  Trend:   → stable"
    
    echo ""
    echo "Request Rate (avg last ${days}d):"
    echo "  Current: $(shuf -i 100-500 -n 1) rps"
    echo "  Peak:    $(shuf -i 800-1500 -n 1) rps"
    echo "  Trend:   ↑ +$(shuf -i 5-20 -n 1)% per week"
}

# คำนวณ capacity requirements
calculate_capacity() {
    local current_rps="$1"
    local growth_rate="$2"        # % per month
    local months_ahead="${3:-6}"
    local cpu_per_rps="${4:-0.01}" # CPU cores per RPS
    local mem_per_rps="${5:-2}"    # MB per RPS
    
    echo "=== Capacity Planning ==="
    echo "Current RPS:  ${current_rps}"
    echo "Growth Rate:  ${growth_rate}% per month"
    echo "Planning:     ${months_ahead} months"
    echo ""
    
    python3 << PYTHON
current = ${current_rps}
growth = ${growth_rate} / 100
months = ${months_ahead}
cpu_per_rps = ${cpu_per_rps}
mem_per_rps = ${mem_per_rps}

print("Month | RPS      | CPU Cores | Memory (GB)")
print("------|----------|-----------|------------")

for m in range(months + 1):
    projected_rps = current * ((1 + growth) ** m)
    cpu = projected_rps * cpu_per_rps
    mem = projected_rps * mem_per_rps / 1024
    
    label = f"+{m}m" if m > 0 else "Now "
    print(f"{label:5} | {projected_rps:8.0f} | {cpu:9.1f} | {mem:.1f} GB")

# Recommendation
final_rps = current * ((1 + growth) ** months)
safety_margin = 1.3  # 30% buffer
recommended_cpu = final_rps * cpu_per_rps * safety_margin
recommended_mem = final_rps * mem_per_rps / 1024 * safety_margin

print()
print("=== Recommendations (with 30% safety margin) ===")
print(f"CPU Cores: {recommended_cpu:.1f}")
print(f"Memory:    {recommended_mem:.1f} GB")
print(f"For {months} months with {${growth_rate}}% monthly growth")
PYTHON
}

# Auto-scaling recommendation
recommend_autoscaling() {
    local service="$1"
    local current_replicas="${2:-3}"
    local avg_cpu="${3:-65}"
    local target_cpu="${4:-70}"
    
    echo "=== Auto-scaling Recommendation: ${service} ==="
    
    python3 << PYTHON
import math

service = "${service}"
replicas = ${current_replicas}
avg_cpu = ${avg_cpu}
target_cpu = ${target_cpu}

# HPA calculation
desired_replicas = math.ceil(replicas * avg_cpu / target_cpu)

print(f"Current replicas: {replicas}")
print(f"Current CPU avg:  {avg_cpu}%")
print(f"Target CPU:       {target_cpu}%")
print(f"Recommended:      {desired_replicas} replicas")
print()

# HPA configuration
print("Recommended HPA configuration:")
print(f"""
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {service}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {service}
  minReplicas: {max(2, replicas - 2)}
  maxReplicas: {replicas * 3}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {target_cpu}
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120
""")
PYTHON
}

# Main
case "${1:-}" in
    trends)    get_resource_trends "${2}" "${3:-30}" ;;
    calculate) calculate_capacity "${2}" "${3}" "${4:-6}" "${5:-0.01}" "${6:-2}" ;;
    autoscale) recommend_autoscaling "${2}" "${3:-3}" "${4:-65}" "${5:-70}" ;;
    *)
        echo "Usage: $0 {trends|calculate|autoscale} [args]"
        ;;
esac
```

---

## สรุป Part 33

| หัวข้อ | เนื้อหา |
|--------|---------|
| **SLI/SLO/SLA** | Availability, latency, error rate measurement |
| **Error Budget** | Calculation, tracking, burn rate |
| **Incident Response** | Incident declaration, severity levels, notification |
| **Post-mortem** | Template generator, blameless culture |
| **Runbook** | Automated runbook generation |
| **Chaos Engineering** | Pod killing, latency injection, memory/CPU stress |
| **Capacity Planning** | Resource trends, growth projection, HPA recommendations |

**ขั้นตอนต่อไป**: Part 34 - GitOps และ Platform Engineering
