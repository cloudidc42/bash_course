# Part 39: Performance Engineering และ Load Testing

## Module 3: Advanced Level - Performance Engineering

---

## ขั้นตอนที่ 513: Performance Testing Framework

```bash
#!/bin/bash
# performance-testing.sh - Comprehensive Performance Testing Framework

set -euo pipefail

RESULTS_DIR="${RESULTS_DIR:-./perf-results}"
REPORT_DIR="${REPORT_DIR:-./perf-reports}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Setup performance testing tools
setup_perf_tools() {
    log "Setting up performance testing tools..."
    
    mkdir -p "$RESULTS_DIR" "$REPORT_DIR"
    
    # Install k6
    if ! command -v k6 &>/dev/null; then
        log "Installing k6..."
        curl -L https://github.com/grafana/k6/releases/latest/download/k6-linux-amd64.tar.gz | \
            tar xz --wildcards '*/k6' -O > /usr/local/bin/k6
        chmod +x /usr/local/bin/k6
    fi
    
    # Install hey (simple HTTP load testing)
    if ! command -v hey &>/dev/null; then
        log "Installing hey..."
        curl -L https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64 \
            -o /usr/local/bin/hey
        chmod +x /usr/local/bin/hey
    fi
    
    # Install wrk
    if ! command -v wrk &>/dev/null; then
        apt-get install -y wrk 2>/dev/null || \
            yum install -y wrk 2>/dev/null || \
            log "Install wrk manually: https://github.com/wg/wrk"
    fi
    
    success "Performance testing tools ready"
}

# k6 load test script generator
generate_k6_script() {
    local target_url="${1:-http://localhost:3000}"
    local script_name="${2:-load-test}"
    local duration="${3:-5m}"
    local vus="${4:-50}"
    local ramp_up="${5:-1m}"
    
    log "Generating k6 load test script: $script_name"
    
    cat <<EOF > "${script_name}.js"
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter, Gauge } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const apiResponseTime = new Trend('api_response_time');
const requestCount = new Counter('requests');
const activeUsers = new Gauge('active_users');

// Test configuration
export const options = {
  stages: [
    // Ramp up
    { duration: '${ramp_up}', target: ${vus} },
    // Sustain
    { duration: '${duration}', target: ${vus} },
    // Ramp down
    { duration: '30s', target: 0 },
  ],
  thresholds: {
    // 95% of requests must complete below 500ms
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    // Error rate below 1%
    errors: ['rate<0.01'],
    // Request count
    requests: ['count>100'],
  },
};

// Setup (runs once per VU)
export function setup() {
  return {
    baseUrl: '${target_url}',
  };
}

// Main test function (runs repeatedly)
export default function (data) {
  activeUsers.add(1);
  
  group('API Tests', () => {
    // Health check
    group('Health Check', () => {
      const res = http.get(\`\${data.baseUrl}/health\`);
      check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 100ms': (r) => r.timings.duration < 100,
      });
      errorRate.add(res.status !== 200);
      requestCount.add(1);
    });
    
    // GET endpoint
    group('GET /api/items', () => {
      const params = {
        headers: {
          'Content-Type': 'application/json',
          'Authorization': 'Bearer test-token',
        },
      };
      const res = http.get(\`\${data.baseUrl}/api/items\`, params);
      
      apiResponseTime.add(res.timings.duration);
      
      check(res, {
        'status is 200': (r) => r.status === 200,
        'has items': (r) => {
          try {
            const body = JSON.parse(r.body);
            return body.items !== undefined;
          } catch {
            return false;
          }
        },
      });
      
      errorRate.add(res.status !== 200);
      requestCount.add(1);
    });
    
    // POST endpoint
    group('POST /api/items', () => {
      const payload = JSON.stringify({
        name: \`item-\${__VU}-\${__ITER}\`,
        value: Math.floor(Math.random() * 100),
        timestamp: new Date().toISOString(),
      });
      
      const params = {
        headers: {
          'Content-Type': 'application/json',
        },
      };
      
      const res = http.post(\`\${data.baseUrl}/api/items\`, payload, params);
      
      check(res, {
        'status is 201': (r) => r.status === 201,
        'has id': (r) => {
          try {
            const body = JSON.parse(r.body);
            return body.id !== undefined;
          } catch {
            return false;
          }
        },
      });
      
      errorRate.add(res.status !== 201);
      requestCount.add(1);
    });
  });
  
  activeUsers.add(-1);
  sleep(1);
}

// Teardown (runs once after test)
export function teardown(data) {
  console.log('Test completed. Base URL:', data.baseUrl);
}
EOF
    
    success "k6 script created: ${script_name}.js"
}

# Run k6 load test
run_k6_test() {
    local script="${1:-load-test.js}"
    local output_format="${2:-json}"
    local test_name="${3:-$(date +%Y%m%d_%H%M%S)}"
    
    if [[ ! -f "$script" ]]; then
        error "Script not found: $script"
        return 1
    fi
    
    log "Running k6 load test: $script"
    
    local output_file="${RESULTS_DIR}/${test_name}.json"
    
    k6 run \
        --out "json=${output_file}" \
        --summary-trend-stats "avg,min,med,max,p(90),p(95),p(99)" \
        "$script" 2>&1 | tee "${RESULTS_DIR}/${test_name}.log"
    
    local exit_code=$?
    
    if [[ $exit_code -eq 0 ]]; then
        success "Load test passed: $test_name"
    else
        error "Load test failed with exit code: $exit_code"
    fi
    
    return $exit_code
}

# Analyze k6 results
analyze_k6_results() {
    local results_file="${1:-}"
    
    if [[ -z "$results_file" || ! -f "$results_file" ]]; then
        error "Valid results file required"
        return 1
    fi
    
    log "Analyzing k6 results: $results_file"
    
    echo "=== Performance Summary ==="
    
    # Parse metrics from JSON output
    jq -r '
        select(.type == "Metric") |
        select(.data.name | test("http_req_duration|error_rate|requests")) |
        "\(.data.name): avg=\(.data.sample.avg)ms p95=\(.data.sample["p(95)"])ms"
    ' "$results_file" 2>/dev/null | sort -u
    
    echo ""
    echo "=== Error Analysis ==="
    jq -r '
        select(.type == "Point") |
        select(.metric == "http_req_failed") |
        select(.data.value == 1) |
        .data.tags | "Status: \(.status) URL: \(.url)"
    ' "$results_file" 2>/dev/null | sort | uniq -c | sort -rn | head -10
    
    echo ""
    echo "=== Threshold Results ==="
    jq -r '
        select(.type == "Metric") |
        select(.data.thresholds != null) |
        "\(.data.name): \(if .data.thresholds | to_entries | any(.value.ok == false) then "FAILED ✗" else "PASSED ✓" end)"
    ' "$results_file" 2>/dev/null | sort -u
}

# Spike test
run_spike_test() {
    local url="${1:-http://localhost:3000}"
    local baseline_vus="${2:-10}"
    local spike_vus="${3:-200}"
    local script_name="${4:-spike-test}"
    
    log "Generating spike test: ${baseline_vus} -> ${spike_vus} VUs"
    
    cat <<EOF > "${script_name}.js"
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m', target: ${baseline_vus} },   // Baseline
    { duration: '30s', target: ${spike_vus} },      // Spike up
    { duration: '3m', target: ${spike_vus} },       // Stay at spike
    { duration: '30s', target: ${baseline_vus} },   // Back to baseline
    { duration: '2m', target: ${baseline_vus} },    // Recover
    { duration: '30s', target: 0 },                 // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<5000'],  // 5s for spike
    errors: ['rate<0.1'],               // 10% error rate during spike
  },
};

export default function() {
  const res = http.get('${url}/api/test');
  errorRate.add(res.status !== 200);
  check(res, { 'status < 500': (r) => r.status < 500 });
  sleep(1);
}
EOF
    
    success "Spike test script created: ${script_name}.js"
}

# Soak/endurance test
run_soak_test() {
    local url="${1:-http://localhost:3000}"
    local vus="${2:-50}"
    local duration="${3:-4h}"
    local script_name="${4:-soak-test}"
    
    log "Generating soak test: ${vus} VUs for ${duration}"
    
    cat <<EOF > "${script_name}.js"
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const latencyTrend = new Trend('latency_trend');

export const options = {
  stages: [
    { duration: '5m', target: ${vus} },   // Ramp up
    { duration: '${duration}', target: ${vus} }, // Soak
    { duration: '5m', target: 0 },         // Ramp down
  ],
  thresholds: {
    // Latency should stay consistent throughout soak
    http_req_duration: ['p(95)<1000', 'p(99)<2000'],
    errors: ['rate<0.01'],
  },
};

export default function() {
  const res = http.get('${url}/api/test');
  
  latencyTrend.add(res.timings.duration);
  errorRate.add(res.status !== 200);
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'no memory leak indicator': (r) => r.timings.duration < 2000,
  });
  
  sleep(1);
}
EOF
    
    success "Soak test script created: ${script_name}.js"
}

# Main
case "${1:-help}" in
    setup) setup_perf_tools ;;
    generate-k6) generate_k6_script "${2:-http://localhost:3000}" "${3:-load-test}" "${4:-5m}" "${5:-50}" "${6:-1m}" ;;
    run-k6) run_k6_test "${2:-load-test.js}" "${3:-json}" "${4:-$(date +%Y%m%d_%H%M%S)}" ;;
    analyze) analyze_k6_results "${2:-}" ;;
    spike) run_spike_test "${2:-http://localhost:3000}" "${3:-10}" "${4:-200}" "${5:-spike-test}" ;;
    soak) run_soak_test "${2:-http://localhost:3000}" "${3:-50}" "${4:-4h}" "${5:-soak-test}" ;;
    *)
        echo "Usage: $0 {setup|generate-k6|run-k6|analyze|spike|soak}"
        ;;
esac
```

---

## ขั้นตอนที่ 514: Application Performance Profiling

```bash
#!/bin/bash
# app-performance-profiling.sh - Application Performance Profiling

set -euo pipefail

PROFILE_DIR="${PROFILE_DIR:-./profiles}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Node.js CPU profiling
profile_nodejs_cpu() {
    local pid="${1:-}"
    local duration="${2:-30}"
    local output_file="${3:-${PROFILE_DIR}/nodejs-cpu-$(date +%Y%m%d%H%M%S).cpuprofile}"
    
    if [[ -z "$pid" ]]; then
        error "Node.js PID required"
        return 1
    fi
    
    mkdir -p "$PROFILE_DIR"
    
    log "Profiling Node.js process PID: $pid for ${duration}s"
    
    # Send SIGUSR1 to enable profiler via node inspector
    kill -SIGUSR1 "$pid" 2>/dev/null || error "Cannot send signal to PID $pid"
    
    log "Inspector enabled - connect to chrome://inspect or use clinic.js"
    log "Run: node --inspect --inspect-brk app.js for startup profiling"
    
    # Using clinic.js if available
    if command -v clinic &>/dev/null; then
        log "Using clinic.js for profiling..."
        clinic doctor --on-port 'autocannon localhost:{PORT}' -- node app.js &
        local clinic_pid=$!
        sleep "$duration"
        kill "$clinic_pid" 2>/dev/null || true
    fi
}

# Python profiling
profile_python() {
    local script="${1:-app.py}"
    local duration="${2:-30}"
    local output_file="${3:-${PROFILE_DIR}/python-profile-$(date +%Y%m%d%H%M%S)}"
    
    mkdir -p "$PROFILE_DIR"
    
    log "Profiling Python script: $script"
    
    if command -v py-spy &>/dev/null; then
        # Find Python process
        local python_pid
        python_pid=$(pgrep -f "$script" | head -1)
        
        if [[ -n "$python_pid" ]]; then
            log "Profiling PID: $python_pid"
            
            # CPU profile
            py-spy record \
                --pid "$python_pid" \
                --output "${output_file}.svg" \
                --duration "$duration" \
                --native
            
            # Top output
            py-spy top --pid "$python_pid" --duration 5
        else
            # Profile from start
            python -m cProfile \
                -o "${output_file}.prof" \
                "$script"
            
            # Generate report
            python -c "
import pstats
p = pstats.Stats('${output_file}.prof')
p.sort_stats('cumulative')
p.print_stats(20)
"
        fi
    else
        log "Installing py-spy..."
        pip install py-spy
        profile_python "$script" "$duration" "$output_file"
    fi
}

# Go profiling
profile_go() {
    local url="${1:-http://localhost:8080}"
    local duration="${2:-30}"
    local output_dir="${3:-$PROFILE_DIR/go}"
    
    mkdir -p "$output_dir"
    
    log "Profiling Go application: $url"
    
    # CPU profile
    log "Collecting CPU profile for ${duration}s..."
    curl -s "${url}/debug/pprof/profile?seconds=${duration}" \
        -o "${output_dir}/cpu.pprof"
    
    # Memory (heap) profile
    log "Collecting heap profile..."
    curl -s "${url}/debug/pprof/heap" \
        -o "${output_dir}/heap.pprof"
    
    # Goroutine profile
    log "Collecting goroutine profile..."
    curl -s "${url}/debug/pprof/goroutine" \
        -o "${output_dir}/goroutine.pprof"
    
    # Mutex profile
    log "Collecting mutex profile..."
    curl -s "${url}/debug/pprof/mutex" \
        -o "${output_dir}/mutex.pprof"
    
    # Analyze if go tool is available
    if command -v go &>/dev/null; then
        log "Analyzing CPU profile..."
        go tool pprof -top "${output_dir}/cpu.pprof" 2>/dev/null | head -20
        
        log "Generating CPU flame graph..."
        go tool pprof -svg "${output_dir}/cpu.pprof" > "${output_dir}/cpu-flamegraph.svg" 2>/dev/null
        
        success "Go profiles saved to: $output_dir"
    fi
}

# JVM profiling  
profile_jvm() {
    local pid="${1:-}"
    local duration="${2:-30}"
    local output_dir="${3:-$PROFILE_DIR/jvm}"
    
    if [[ -z "$pid" ]]; then
        error "JVM PID required (use: jps to list)"
        return 1
    fi
    
    mkdir -p "$output_dir"
    
    log "Profiling JVM process PID: $pid"
    
    # Thread dump
    log "Collecting thread dump..."
    kill -3 "$pid" 2>/dev/null || jstack "$pid" > "${output_dir}/thread-dump.txt" 2>/dev/null
    
    # Heap dump
    log "Collecting heap dump..."
    jmap -dump:format=b,file="${output_dir}/heap.hprof" "$pid" 2>/dev/null || \
        log "jmap not available - use JVM flag: -XX:+HeapDumpOnOutOfMemoryError"
    
    # GC stats
    log "Collecting GC statistics for ${duration}s..."
    jstat -gcutil "$pid" 1000 "$duration" > "${output_dir}/gc-stats.txt" 2>/dev/null || \
        log "jstat not available"
    
    # JFR recording (Java Flight Recorder)
    if jcmd "$pid" VM.version 2>/dev/null | grep -q "HotSpot"; then
        log "Starting JFR recording..."
        jcmd "$pid" JFR.start \
            name=ProfileRecording \
            duration="${duration}s" \
            filename="${output_dir}/recording.jfr" \
            settings=profile 2>/dev/null
        
        sleep "$duration"
        
        log "JFR recording saved: ${output_dir}/recording.jfr"
    fi
    
    success "JVM profiles collected in: $output_dir"
}

# System-level profiling with perf
profile_system() {
    local pid="${1:-}"
    local duration="${2:-10}"
    local output_file="${3:-${PROFILE_DIR}/perf-$(date +%Y%m%d%H%M%S)}"
    
    mkdir -p "$PROFILE_DIR"
    
    log "System-level profiling for ${duration}s..."
    
    if ! command -v perf &>/dev/null; then
        error "perf not available - install: apt-get install linux-tools-generic"
        return 1
    fi
    
    local pid_arg=""
    if [[ -n "$pid" ]]; then
        pid_arg="-p $pid"
    fi
    
    # CPU performance counter stats
    perf stat $pid_arg sleep "$duration" 2>&1 | tee "${output_file}.stat"
    
    # Flame graph
    if [[ -n "$pid" ]]; then
        log "Collecting perf samples for flame graph..."
        perf record -g -F 99 -p "$pid" sleep "$duration" -o "${output_file}.data" 2>/dev/null
        
        if [[ -f "${output_file}.data" ]]; then
            perf script -i "${output_file}.data" > "${output_file}.perf"
            
            # Generate flamegraph if FlameGraph scripts are available
            if [[ -d /opt/flamegraph ]]; then
                /opt/flamegraph/stackcollapse-perf.pl "${output_file}.perf" | \
                    /opt/flamegraph/flamegraph.pl > "${output_file}.svg"
                log "Flame graph: ${output_file}.svg"
            fi
        fi
    fi
    
    success "System profile complete"
}

# Database query profiling
profile_database_queries() {
    local db_host="${1:-localhost}"
    local db_port="${2:-5432}"
    local db_name="${3:-myapp}"
    local db_user="${4:-postgres}"
    local duration="${5:-60}"
    
    log "Profiling database queries for ${duration}s..."
    
    # Enable pg_stat_statements
    PGPASSWORD="${PGPASSWORD:-}" psql \
        -h "$db_host" -p "$db_port" -U "$db_user" -d "$db_name" \
        -c "CREATE EXTENSION IF NOT EXISTS pg_stat_statements;" 2>/dev/null || true
    
    sleep "$duration"
    
    echo "=== Top Slow Queries ==="
    PGPASSWORD="${PGPASSWORD:-}" psql \
        -h "$db_host" -p "$db_port" -U "$db_user" -d "$db_name" \
        -x -c "
SELECT 
    round(total_exec_time::numeric, 2) AS total_exec_ms,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_exec_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS percentage,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;" 2>/dev/null
    
    echo ""
    echo "=== Most Called Queries ==="
    PGPASSWORD="${PGPASSWORD:-}" psql \
        -h "$db_host" -p "$db_port" -U "$db_user" -d "$db_name" \
        -x -c "
SELECT 
    calls,
    round(mean_exec_time::numeric, 2) AS mean_exec_ms,
    query
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;" 2>/dev/null
    
    echo ""
    echo "=== Index Usage ==="
    PGPASSWORD="${PGPASSWORD:-}" psql \
        -h "$db_host" -p "$db_port" -U "$db_user" -d "$db_name" \
        -x -c "
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC
LIMIT 20;" 2>/dev/null
}

# Performance regression detection
detect_performance_regression() {
    local baseline_file="${1:-}"
    local current_file="${2:-}"
    local threshold_percentage="${3:-10}"
    
    if [[ -z "$baseline_file" || -z "$current_file" ]]; then
        error "Usage: detect_performance_regression <baseline> <current> [threshold%]"
        return 1
    fi
    
    if [[ ! -f "$baseline_file" || ! -f "$current_file" ]]; then
        error "Result files not found"
        return 1
    fi
    
    log "Detecting performance regressions (threshold: ${threshold_percentage}%)..."
    
    local baseline_p95
    baseline_p95=$(jq -r 'select(.type == "Metric") | select(.data.name == "http_req_duration") | .data.sample["p(95)"]' \
        "$baseline_file" 2>/dev/null | tail -1)
    
    local current_p95
    current_p95=$(jq -r 'select(.type == "Metric") | select(.data.name == "http_req_duration") | .data.sample["p(95)"]' \
        "$current_file" 2>/dev/null | tail -1)
    
    if [[ -n "$baseline_p95" && -n "$current_p95" ]]; then
        local regression
        regression=$(echo "scale=2; (($current_p95 - $baseline_p95) / $baseline_p95) * 100" | bc)
        
        echo "P95 Latency:"
        echo "  Baseline: ${baseline_p95}ms"
        echo "  Current:  ${current_p95}ms"
        echo "  Change:   ${regression}%"
        
        if (( $(echo "$regression > $threshold_percentage" | bc -l) )); then
            error "REGRESSION DETECTED: P95 latency increased by ${regression}%"
            return 1
        else
            success "No regression: P95 latency change is ${regression}%"
        fi
    else
        log "Could not extract metrics from files"
    fi
}

# Main
case "${1:-help}" in
    setup) setup_perf_tools ;;
    nodejs-cpu) profile_nodejs_cpu "${2:-}" "${3:-30}" "${4:-}" ;;
    python) profile_python "${2:-app.py}" "${3:-30}" "${4:-}" ;;
    go) profile_go "${2:-http://localhost:8080}" "${3:-30}" "${4:-}" ;;
    jvm) profile_jvm "${2:-}" "${3:-30}" "${4:-}" ;;
    system) profile_system "${2:-}" "${3:-10}" "${4:-}" ;;
    db) profile_database_queries "${2:-localhost}" "${3:-5432}" "${4:-myapp}" "${5:-postgres}" "${6:-60}" ;;
    regression) detect_performance_regression "${2:-}" "${3:-}" "${4:-10}" ;;
    *)
        echo "Usage: $0 {setup|nodejs-cpu|python|go|jvm|system|db|regression}"
        ;;
esac
```

---

## ขั้นตอนที่ 515: Performance Optimization Automation

```bash
#!/bin/bash
# performance-optimizer.sh - Automated Performance Optimization

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Kubernetes resource rightsizing
rightsize_resources() {
    local namespace="${1:-default}"
    local days="${2:-7}"
    
    log "Analyzing resource usage for rightsizing (last ${days} days)..."
    
    # Get VPA recommendations
    local vpas
    vpas=$(kubectl get vpa -n "$namespace" -o json 2>/dev/null)
    
    if [[ -n "$vpas" ]]; then
        echo "=== VPA Recommendations ==="
        echo "$vpas" | jq -r '
            .items[] |
            "\(.metadata.name):" +
            "\n  CPU: target=\(.status.recommendation.containerRecommendations[0].target.cpu)" +
            "\n  Memory: target=\(.status.recommendation.containerRecommendations[0].target.memory)"
        ' 2>/dev/null
    fi
    
    # Analyze actual usage vs requests
    echo ""
    echo "=== Resource Efficiency Analysis ==="
    
    # Get all deployments
    local deployments
    deployments=$(kubectl get deployments -n "$namespace" -o json 2>/dev/null)
    
    echo "$deployments" | jq -r '.items[] | .metadata.name' 2>/dev/null | \
    while read -r deploy; do
        local cpu_request
        cpu_request=$(kubectl get deployment "$deploy" -n "$namespace" \
            -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}' 2>/dev/null || echo "unset")
        
        local mem_request
        mem_request=$(kubectl get deployment "$deploy" -n "$namespace" \
            -o jsonpath='{.spec.template.spec.containers[0].resources.requests.memory}' 2>/dev/null || echo "unset")
        
        local cpu_actual
        cpu_actual=$(kubectl top pods -n "$namespace" -l "app=$deploy" \
            --no-headers 2>/dev/null | awk '{print $2}' | head -1 || echo "N/A")
        
        echo "  $deploy: request=CPU:$cpu_request MEM:$mem_request actual=CPU:$cpu_actual"
    done
}

# Auto-tune JVM settings
optimize_jvm() {
    local heap_size="${1:-}"
    local container_memory="${2:-}"
    local gc_type="${3:-G1GC}"
    
    if [[ -z "$container_memory" ]]; then
        container_memory=$(free -m | awk '/Mem:/ {print $2}')
    fi
    
    if [[ -z "$heap_size" ]]; then
        # Use 75% of container memory for heap
        heap_size=$(echo "$container_memory * 0.75 / 1" | bc)
    fi
    
    log "Generating optimized JVM settings for ${container_memory}MB container..."
    
    cat <<EOF
# JVM Optimization Settings
# Container memory: ${container_memory}MB
# Heap size: ${heap_size}MB

JVM_OPTS="-Xms${heap_size}m -Xmx${heap_size}m"

# GC Configuration (${gc_type})
$(case "$gc_type" in
    G1GC)
        cat <<GCEOF
JVM_OPTS="\$JVM_OPTS -XX:+UseG1GC"
JVM_OPTS="\$JVM_OPTS -XX:MaxGCPauseMillis=200"
JVM_OPTS="\$JVM_OPTS -XX:G1HeapRegionSize=16m"
JVM_OPTS="\$JVM_OPTS -XX:G1NewSizePercent=20"
JVM_OPTS="\$JVM_OPTS -XX:G1MaxNewSizePercent=40"
JVM_OPTS="\$JVM_OPTS -XX:G1MixedGCCountTarget=8"
GCEOF
        ;;
    ZGC)
        cat <<GCEOF
JVM_OPTS="\$JVM_OPTS -XX:+UseZGC"
JVM_OPTS="\$JVM_OPTS -XX:+ZGenerational"
GCEOF
        ;;
    Shenandoah)
        cat <<GCEOF
JVM_OPTS="\$JVM_OPTS -XX:+UseShenandoahGC"
JVM_OPTS="\$JVM_OPTS -XX:ShenandoahGCMode=adaptive"
GCEOF
        ;;
esac)

# Memory settings
JVM_OPTS="\$JVM_OPTS -XX:+UseContainerSupport"
JVM_OPTS="\$JVM_OPTS -XX:MaxRAMPercentage=75.0"
JVM_OPTS="\$JVM_OPTS -XX:+ExitOnOutOfMemoryError"

# Observability
JVM_OPTS="\$JVM_OPTS -XX:+PrintGCDetails"
JVM_OPTS="\$JVM_OPTS -XX:+PrintGCDateStamps"
JVM_OPTS="\$JVM_OPTS -Xlog:gc*:file=/logs/gc.log:time,level,tags:filecount=5,filesize=20m"

# Performance monitoring
JVM_OPTS="\$JVM_OPTS -XX:+UnlockDiagnosticVMOptions"
JVM_OPTS="\$JVM_OPTS -XX:+FlightRecorder"
JVM_OPTS="\$JVM_OPTS -XX:StartFlightRecording=duration=60s,settings=profile,filename=/profiles/startup.jfr"

# JIT
JVM_OPTS="\$JVM_OPTS -server"
JVM_OPTS="\$JVM_OPTS -XX:+TieredCompilation"
JVM_OPTS="\$JVM_OPTS -XX:+OptimizeStringConcat"
JVM_OPTS="\$JVM_OPTS -XX:ReservedCodeCacheSize=256m"

export JVM_OPTS
EOF
}

# Auto-tune Node.js
optimize_nodejs() {
    local memory_limit_mb="${1:-512}"
    local cpu_count="${2:-$(nproc)}"
    
    log "Generating Node.js optimization settings for ${memory_limit_mb}MB, ${cpu_count} CPUs..."
    
    cat <<EOF
# Node.js Performance Optimization
# Memory: ${memory_limit_mb}MB
# CPUs: ${cpu_count}

# Heap size (80% of memory limit)
export NODE_OPTIONS="--max-old-space-size=$(echo "$memory_limit_mb * 0.8 / 1" | bc)"

# V8 optimization flags
export NODE_OPTIONS="\$NODE_OPTIONS --optimize-for-size"
export NODE_OPTIONS="\$NODE_OPTIONS --v8-pool-size=${cpu_count}"

# UV thread pool (for I/O)
export UV_THREADPOOL_SIZE=$((cpu_count * 4))

# Cluster mode configuration (cluster.js)
# Number of workers = CPU count
CLUSTER_WORKERS=${cpu_count}

# PM2 ecosystem.config.js
cat > ecosystem.config.js << 'PM2CONFIG'
module.exports = {
  apps: [{
    name: 'app',
    script: 'src/index.js',
    instances: ${cpu_count},
    exec_mode: 'cluster',
    max_memory_restart: '${memory_limit_mb}M',
    node_args: '--max-old-space-size=$(echo "$memory_limit_mb * 0.8 / 1" | bc)',
    env: {
      NODE_ENV: 'production',
    },
    watch: false,
    exp_backoff_restart_delay: 100,
    wait_ready: true,
    listen_timeout: 5000,
    kill_timeout: 5000,
  }]
};
PM2CONFIG
EOF
}

# Database connection pool optimization
optimize_db_pool() {
    local db_type="${1:-postgresql}"
    local max_connections="${2:-100}"
    local app_instances="${3:-3}"
    
    log "Optimizing database connection pool: $db_type"
    
    local pool_min=$((max_connections / (app_instances * 4)))
    local pool_max=$((max_connections / app_instances))
    
    echo "Database: $db_type"
    echo "Max DB connections: $max_connections"
    echo "App instances: $app_instances"
    echo ""
    echo "Recommended pool settings:"
    echo "  min: $pool_min"
    echo "  max: $pool_max"
    echo "  idle timeout: 30000ms"
    echo "  connection timeout: 5000ms"
    echo ""
    
    case "$db_type" in
        postgresql)
            cat <<EOF
# PostgreSQL connection pool (pg/knex)
const pool = {
  min: ${pool_min},
  max: ${pool_max},
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
  acquireConnectionTimeout: 60000,
  // Statement timeout
  statement_timeout: 30000,
  // Connection idle in transaction timeout
  idle_in_transaction_session_timeout: 60000,
};

# PgBouncer configuration (connection pooling proxy)
[pgbouncer]
listen_port = 6432
pool_mode = transaction
max_client_conn = ${max_connections}
default_pool_size = ${pool_max}
min_pool_size = ${pool_min}
server_idle_timeout = 600
client_idle_timeout = 0
autodb_idle_timeout = 3600
EOF
            ;;
        mysql)
            cat <<EOF
# MySQL connection pool (mysql2/knex)
const pool = {
  min: ${pool_min},
  max: ${pool_max},
  acquire: 60000,
  idle: 10000,
  evict: 1000,
};

# ProxySQL configuration
EOF
            ;;
        mongodb)
            cat <<EOF
# MongoDB connection pool (mongoose)
const mongooseOptions = {
  maxPoolSize: ${pool_max},
  minPoolSize: ${pool_min},
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  connectTimeoutMS: 10000,
};
EOF
            ;;
    esac
}

# Main
case "${1:-help}" in
    rightsize) rightsize_resources "${2:-default}" "${3:-7}" ;;
    optimize-jvm) optimize_jvm "${2:-}" "${3:-}" "${4:-G1GC}" ;;
    optimize-nodejs) optimize_nodejs "${2:-512}" "${3:-$(nproc)}" ;;
    optimize-db) optimize_db_pool "${2:-postgresql}" "${3:-100}" "${4:-3}" ;;
    *)
        echo "Usage: $0 {rightsize|optimize-jvm|optimize-nodejs|optimize-db}"
        ;;
esac
```

---

## สรุป Part 39

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | เครื่องมือ | ขั้นตอน |
|--------|-----------|---------|
| Performance Testing Framework | k6, hey, wrk, Spike/Soak tests | 513 |
| Application Performance Profiling | py-spy, pprof, JFR, perf | 514 |
| Performance Optimization Automation | Resource rightsizing, JVM/Node.js/DB tuning | 515 |

**เทคโนโลยีที่ใช้:**
- k6: Modern load testing tool
- py-spy: Python profiler
- pprof: Go profiler
- Java Flight Recorder: JVM profiler
- perf: Linux system profiler
- pg_stat_statements: PostgreSQL query profiler

ขั้นตอนต่อไป: Part 40 - Module 3 Completion - Advanced Patterns Workshop
