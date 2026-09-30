# Part 22: Logging และ Monitoring

## Module 2: Intermediate Level
### ขั้นตอนที่ 425-436: ระบบ Logging และ Monitoring

---

## ขั้นตอนที่ 425: Structured Logging Framework

```bash
#!/usr/bin/env bash
# logging_framework.sh

# ===== Logging Framework =====
LOG_FILE="${LOG_FILE:-/tmp/app_$$.log}"
LOG_LEVEL="${LOG_LEVEL:-INFO}"
LOG_FORMAT="${LOG_FORMAT:-text}"  # text | json | csv

declare -A LOG_LEVELS=([DEBUG]=0 [INFO]=1 [WARN]=2 [ERROR]=3 [FATAL]=4)

# สี
declare -A LOG_COLORS=(
    [DEBUG]="\033[36m"   # Cyan
    [INFO]="\033[32m"    # Green
    [WARN]="\033[33m"    # Yellow
    [ERROR]="\033[31m"   # Red
    [FATAL]="\033[35m"   # Magenta
)
NC="\033[0m"

# เปิดใช้ file logging
LOG_TO_FILE=true
LOG_TO_STDOUT=true

# ฟังก์ชัน log หลัก
_log() {
    local level="$1"
    local message="$2"
    local caller="${BASH_SOURCE[2]:-unknown}:${BASH_LINENO[1]:-0}"
    
    # ตรวจสอบ level
    local level_num="${LOG_LEVELS[$level]:-1}"
    local min_level_num="${LOG_LEVELS[$LOG_LEVEL]:-1}"
    
    [[ $level_num -lt $min_level_num ]] && return
    
    local timestamp
    timestamp=$(date '+%Y-%m-%dT%H:%M:%S%z')
    
    # Format output
    local formatted_message
    case "$LOG_FORMAT" in
        json)
            formatted_message=$(printf '{"timestamp":"%s","level":"%s","message":"%s","caller":"%s"}\n' \
                "$timestamp" "$level" "${message//\"/\\\"}" "$caller")
            ;;
        csv)
            formatted_message=$(printf '"%s","%s","%s","%s"\n' \
                "$timestamp" "$level" "$message" "$caller")
            ;;
        *)  # text
            formatted_message=$(printf '[%s] [%s] %s (%s)\n' \
                "$timestamp" "$level" "$message" "$caller")
            ;;
    esac
    
    # Output
    if $LOG_TO_STDOUT; then
        local color="${LOG_COLORS[$level]:-}"
        printf "${color}%s${NC}\n" "$formatted_message"
    fi
    
    if $LOG_TO_FILE; then
        echo "$formatted_message" >> "$LOG_FILE"
    fi
    
    # Fatal: exit after logging
    if [[ "$level" == "FATAL" ]]; then
        exit 1
    fi
}

# Public logging functions
log_debug() { _log "DEBUG" "$*"; }
log_info()  { _log "INFO"  "$*"; }
log_warn()  { _log "WARN"  "$*"; }
log_error() { _log "ERROR" "$*"; }
log_fatal() { _log "FATAL" "$*"; }

# Convenience aliases
log()       { log_info  "$*"; }
warn()      { log_warn  "$*"; }
error()     { log_error "$*"; }

# Context logging
declare -A LOG_CONTEXT=()

log_with_context() {
    local level="$1"
    local message="$2"
    
    local ctx_str=""
    for key in "${!LOG_CONTEXT[@]}"; do
        ctx_str+=" $key=${LOG_CONTEXT[$key]}"
    done
    
    _log "$level" "${message}${ctx_str}"
}

set_log_context() {
    LOG_CONTEXT["$1"]="$2"
}

clear_log_context() {
    LOG_CONTEXT=()
}

# Log rotation
rotate_log() {
    local log_file="${1:-$LOG_FILE}"
    local max_size_mb="${2:-10}"
    local keep_files="${3:-5}"
    
    if [[ ! -f "$log_file" ]]; then return; fi
    
    local size_kb
    size_kb=$(du -k "$log_file" | cut -f1)
    
    if [[ $size_kb -gt $((max_size_mb * 1024)) ]]; then
        # Rotate
        for ((i=keep_files-1; i>=1; i--)); do
            [[ -f "${log_file}.$i" ]] && mv "${log_file}.$i" "${log_file}.$((i+1))"
        done
        
        gzip -c "$log_file" > "${log_file}.1.gz"
        > "$log_file"  # Truncate current log
        
        log_info "Log rotated: $log_file"
    fi
}

# Timer logging
declare -A _log_timers=()

log_timer_start() {
    local name="$1"
    _log_timers[$name]=$(date +%s%N)
    log_debug "Timer started: $name"
}

log_timer_stop() {
    local name="$1"
    local start="${_log_timers[$name]}"
    
    if [[ -z "$start" ]]; then
        log_warn "Timer '$name' was not started"
        return
    fi
    
    local end now
    now=$(date +%s%N)
    local elapsed_ms=$(( (now - start) / 1000000 ))
    
    log_info "Timer $name: ${elapsed_ms}ms"
    unset '_log_timers[$name]'
}

# Demo
echo "=== Logging Framework Demo ==="
LOG_LEVEL=DEBUG

log_debug "This is a debug message"
log_info "Application started"
log_warn "Low memory warning"
log_error "Connection failed"

echo ""
echo "=== JSON Logging ==="
LOG_FORMAT=json
log_info "User logged in"
log_error "Authentication failed"
LOG_FORMAT=text

echo ""
echo "=== Context Logging ==="
set_log_context "user_id" "12345"
set_log_context "request_id" "req-abc-123"
log_with_context "INFO" "Processing request"
clear_log_context

echo ""
echo "=== Timer ==="
log_timer_start "database_query"
sleep 0.1
log_timer_stop "database_query"

echo ""
echo "Log file: $LOG_FILE"
```

---

## ขั้นตอนที่ 426: System Monitoring Dashboard

```bash
#!/usr/bin/env bash
# system_monitor.sh

# ===== System Monitoring =====

# ใช้ tput สำหรับ terminal control
COLUMNS=${COLUMNS:-$(tput cols 2>/dev/null || echo 80)}
LINES=${LINES:-$(tput lines 2>/dev/null || echo 24)}

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
NC='\033[0m'
BOLD='\033[1m'

# Draw progress bar
draw_bar() {
    local value="$1"
    local max="${2:-100}"
    local width="${3:-40}"
    local label="${4:-}"
    
    local filled=$(( value * width / max ))
    local empty=$(( width - filled ))
    
    local color
    if [[ $value -ge 90 ]]; then
        color="$RED"
    elif [[ $value -ge 70 ]]; then
        color="$YELLOW"
    else
        color="$GREEN"
    fi
    
    printf "${color}["
    printf "%${filled}s" | tr ' ' '█'
    printf "${NC}%${empty}s] %3d%% %s\n" '' "$value" "$label"
}

# Get CPU usage
get_cpu_usage() {
    if [[ -f /proc/stat ]]; then
        local cpu_line1 cpu_line2
        cpu_line1=$(grep '^cpu ' /proc/stat)
        sleep 0.1
        cpu_line2=$(grep '^cpu ' /proc/stat)
        
        local idle1 idle2 total1 total2
        read -ra arr1 <<< "$cpu_line1"
        read -ra arr2 <<< "$cpu_line2"
        
        idle1=${arr1[4]}
        idle2=${arr2[4]}
        
        total1=0
        total2=0
        for val in "${arr1[@]:1}"; do total1=$((total1+val)); done
        for val in "${arr2[@]:1}"; do total2=$((total2+val)); done
        
        local total_diff=$(( total2 - total1 ))
        local idle_diff=$(( idle2 - idle1 ))
        
        if [[ $total_diff -gt 0 ]]; then
            echo $(( 100 * (total_diff - idle_diff) / total_diff ))
        else
            echo 0
        fi
    else
        echo 0
    fi
}

# Get memory usage
get_memory_info() {
    if [[ -f /proc/meminfo ]]; then
        local total free available buffers cached
        total=$(grep '^MemTotal:' /proc/meminfo | awk '{print $2}')
        available=$(grep '^MemAvailable:' /proc/meminfo | awk '{print $2}')
        
        local used=$(( total - available ))
        local percent=$(( used * 100 / total ))
        
        echo "$percent $((total/1024)) $((used/1024)) $((available/1024))"
    else
        echo "0 0 0 0"
    fi
}

# Get disk usage
get_disk_info() {
    local mount="${1:-/}"
    df -h "$mount" | awk 'NR==2 {
        gsub(/%/,"",$5)
        print $5, $2, $3, $4
    }'
}

# Get network stats
get_network_stats() {
    if [[ -f /proc/net/dev ]]; then
        awk 'NR>2 && !/lo/ {
            gsub(/:/, " ")
            printf "%s RX:%s TX:%s\n", $1, $2, $10
        }' /proc/net/dev | head -3
    fi
}

# Get top processes
get_top_processes() {
    ps aux --sort=-%cpu 2>/dev/null | awk 'NR>1 && NR<=6 {
        printf "%-20s %5s%% %5s%%\n", $11, $3, $4
    }' | head -5
}

# Get system uptime
get_uptime() {
    if [[ -f /proc/uptime ]]; then
        local uptime_secs
        uptime_secs=$(awk '{print int($1)}' /proc/uptime)
        local days=$((uptime_secs / 86400))
        local hours=$(( (uptime_secs % 86400) / 3600 ))
        local mins=$(( (uptime_secs % 3600) / 60 ))
        echo "${days}d ${hours}h ${mins}m"
    else
        uptime -p 2>/dev/null | sed 's/up //' || echo "Unknown"
    fi
}

# Draw dashboard
draw_dashboard() {
    clear
    
    local cpu_usage
    cpu_usage=$(get_cpu_usage)
    
    local mem_info
    read -r mem_pct mem_total mem_used mem_avail <<< "$(get_memory_info)"
    
    local disk_info
    read -r disk_pct disk_total disk_used disk_avail <<< "$(get_disk_info /)"
    
    echo -e "${BOLD}${BLUE}╔══════════════════════════════════════════════════╗${NC}"
    echo -e "${BOLD}${BLUE}║          System Monitor Dashboard                ║${NC}"
    echo -e "${BOLD}${BLUE}╚══════════════════════════════════════════════════╝${NC}"
    echo ""
    
    printf "${BOLD}%-15s${NC} %s\n" "Hostname:" "$(hostname)"
    printf "${BOLD}%-15s${NC} %s\n" "Uptime:" "$(get_uptime)"
    printf "${BOLD}%-15s${NC} %s\n" "Date/Time:" "$(date '+%Y-%m-%d %H:%M:%S')"
    echo ""
    
    echo -e "${BOLD}CPU Usage:${NC}"
    draw_bar "$cpu_usage" 100 40 "${cpu_usage}%"
    echo ""
    
    echo -e "${BOLD}Memory Usage:${NC} ${mem_used}MB / ${mem_total}MB"
    draw_bar "$mem_pct" 100 40 "${mem_pct}% (${mem_avail}MB free)"
    echo ""
    
    echo -e "${BOLD}Disk Usage (/):${NC} ${disk_used} / ${disk_total}"
    draw_bar "${disk_pct:-0}" 100 40 "${disk_pct}% (${disk_avail} free)"
    echo ""
    
    echo -e "${BOLD}Network:${NC}"
    get_network_stats | while read -r line; do
        echo "  $line"
    done
    echo ""
    
    echo -e "${BOLD}Top Processes:${NC}"
    printf "  %-20s %6s %6s\n" "COMMAND" "CPU%" "MEM%"
    get_top_processes | while read -r line; do
        echo "  $line"
    done
    
    echo ""
    echo -e "${CYAN}Press Ctrl+C to exit | Refreshing every 2s${NC}"
}

# Run dashboard
if [[ "${1:-}" == "--once" ]]; then
    draw_dashboard
else
    while true; do
        draw_dashboard
        sleep 2
    done
fi
```

---

## ขั้นตอนที่ 427: Alert System

```bash
#!/usr/bin/env bash
# alert_system.sh

# ===== Alert System =====
ALERT_LOG="/tmp/alerts_$$.log"
ALERT_COOLDOWN=300  # 5 นาที ระหว่าง alerts เดียวกัน

declare -A ALERT_LAST_SENT=()
declare -A ALERT_THRESHOLDS=(
    [cpu_critical]=90
    [cpu_warning]=75
    [mem_critical]=95
    [mem_warning]=80
    [disk_critical]=90
    [disk_warning]=80
)

# Alert channels
send_alert_log() {
    local severity="$1"
    local subject="$2"
    local message="$3"
    
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] [$severity] $subject: $message" | tee -a "$ALERT_LOG"
}

send_alert_email() {
    local severity="$1"
    local subject="$2"
    local message="$3"
    local recipient="${ALERT_EMAIL:-admin@example.com}"
    
    if command -v sendmail &>/dev/null; then
        echo "Subject: [$severity] $subject
From: monitoring@$(hostname)
To: $recipient

$message

--
Sent from $(hostname) at $(date)" | sendmail "$recipient"
        echo "Email alert sent to $recipient"
    else
        send_alert_log "$severity" "$subject" "$message (email not available)"
    fi
}

send_alert_slack() {
    local severity="$1"
    local subject="$2"
    local message="$3"
    local webhook_url="${SLACK_WEBHOOK_URL:-}"
    
    if [[ -z "$webhook_url" ]]; then
        send_alert_log "$severity" "$subject" "$message (slack not configured)"
        return
    fi
    
    local emoji
    case "$severity" in
        CRITICAL) emoji=":red_circle:" ;;
        WARNING)  emoji=":warning:" ;;
        INFO)     emoji=":information_source:" ;;
        *) emoji=":bell:" ;;
    esac
    
    local payload
    payload=$(printf '{"text": "%s *[%s]* %s\n%s"}' \
        "$emoji" "$severity" "$subject" "$message")
    
    curl -s -X POST -H 'Content-type: application/json' \
        --data "$payload" "$webhook_url" > /dev/null
}

# Cooldown check
should_send_alert() {
    local alert_key="$1"
    local now
    now=$(date +%s)
    local last_sent="${ALERT_LAST_SENT[$alert_key]:-0}"
    
    if (( now - last_sent > ALERT_COOLDOWN )); then
        ALERT_LAST_SENT[$alert_key]=$now
        return 0
    fi
    return 1
}

# Send alert via configured channels
send_alert() {
    local severity="$1"
    local subject="$2"
    local message="$3"
    local alert_key="${4:-${subject// /_}}"
    
    if ! should_send_alert "$alert_key"; then
        echo "Alert suppressed (cooldown): $subject"
        return
    fi
    
    # Always log
    send_alert_log "$severity" "$subject" "$message"
    
    # Email for critical
    if [[ "$severity" == "CRITICAL" ]]; then
        send_alert_email "$severity" "$subject" "$message"
    fi
    
    # Slack for warning+
    send_alert_slack "$severity" "$subject" "$message"
}

# ===== Monitoring Checks =====
check_cpu() {
    local cpu_usage
    cpu_usage=$(top -bn1 2>/dev/null | grep "Cpu(s)" | awk '{print int($2)}' || echo 0)
    
    if [[ $cpu_usage -ge ${ALERT_THRESHOLDS[cpu_critical]} ]]; then
        send_alert "CRITICAL" "High CPU Usage" \
            "CPU usage is ${cpu_usage}% on $(hostname)" "cpu_critical"
    elif [[ $cpu_usage -ge ${ALERT_THRESHOLDS[cpu_warning]} ]]; then
        send_alert "WARNING" "CPU Usage Warning" \
            "CPU usage is ${cpu_usage}% on $(hostname)" "cpu_warning"
    fi
    
    echo "CPU: ${cpu_usage}%"
}

check_memory() {
    local mem_pct
    if [[ -f /proc/meminfo ]]; then
        local total available
        total=$(grep '^MemTotal:' /proc/meminfo | awk '{print $2}')
        available=$(grep '^MemAvailable:' /proc/meminfo | awk '{print $2}')
        local used=$(( total - available ))
        mem_pct=$(( used * 100 / total ))
    else
        mem_pct=0
    fi
    
    if [[ $mem_pct -ge ${ALERT_THRESHOLDS[mem_critical]} ]]; then
        send_alert "CRITICAL" "Critical Memory Usage" \
            "Memory usage is ${mem_pct}% on $(hostname)" "mem_critical"
    elif [[ $mem_pct -ge ${ALERT_THRESHOLDS[mem_warning]} ]]; then
        send_alert "WARNING" "Memory Usage Warning" \
            "Memory usage is ${mem_pct}% on $(hostname)" "mem_warning"
    fi
    
    echo "Memory: ${mem_pct}%"
}

check_disk() {
    while IFS= read -r line; do
        local mount pct
        mount=$(echo "$line" | awk '{print $NF}')
        pct=$(echo "$line" | awk '{gsub(/%/,"",$5); print $5}')
        
        if [[ $pct -ge ${ALERT_THRESHOLDS[disk_critical]} ]]; then
            send_alert "CRITICAL" "Critical Disk Usage" \
                "Disk $mount is ${pct}% full on $(hostname)" "disk_critical_$mount"
        elif [[ $pct -ge ${ALERT_THRESHOLDS[disk_warning]} ]]; then
            send_alert "WARNING" "Disk Usage Warning" \
                "Disk $mount is ${pct}% full on $(hostname)" "disk_warning_$mount"
        fi
        
        echo "Disk $mount: ${pct}%"
    done < <(df -h | awk 'NR>1 && /^\// {print}')
}

check_service() {
    local service="$1"
    
    if ! systemctl is-active --quiet "$service" 2>/dev/null; then
        send_alert "CRITICAL" "Service Down" \
            "Service '$service' is not running on $(hostname)" "service_$service"
        echo "Service $service: DOWN"
        return 1
    fi
    
    echo "Service $service: OK"
    return 0
}

check_port() {
    local host="${1:-localhost}"
    local port="$2"
    local service_name="${3:-port_$port}"
    
    if ! timeout 5 bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null; then
        send_alert "CRITICAL" "Port Unreachable" \
            "Cannot connect to $host:$port ($service_name)" "port_$port"
        echo "Port $host:$port ($service_name): CLOSED"
        return 1
    fi
    
    echo "Port $host:$port ($service_name): OPEN"
    return 0
}

check_log_errors() {
    local log_file="$1"
    local threshold="${2:-10}"
    local window_mins="${3:-5}"
    
    if [[ ! -f "$log_file" ]]; then
        echo "Log file not found: $log_file"
        return
    fi
    
    local error_count
    error_count=$(awk -v mins="$window_mins" '
        /ERROR|CRITICAL|FATAL/ {
            cmd = "date -d \"" $1 "T" $2 "\" +%s 2>/dev/null"
            if ((system("test -n \"" $1 "\"")) == 0) count++
        }
        END {print count+0}
    ' "$log_file" 2>/dev/null || grep -c "ERROR\|CRITICAL\|FATAL" "$log_file" 2>/dev/null || echo 0)
    
    if [[ $error_count -ge $threshold ]]; then
        send_alert "WARNING" "High Error Rate" \
            "$error_count errors found in $log_file (last ${window_mins}min)" "log_errors"
    fi
    
    echo "Log errors (${window_mins}min): $error_count"
}

# ===== Run all checks =====
run_monitoring() {
    echo "=== System Health Check: $(date) ==="
    echo ""
    
    check_cpu
    check_memory
    check_disk
    
    echo ""
    echo "Alert log: $ALERT_LOG"
}

run_monitoring
```

---

## ขั้นตอนที่ 428: Metrics Collection

```bash
#!/usr/bin/env bash
# metrics_collection.sh

# ===== Metrics Collection System =====
METRICS_FILE="/tmp/metrics_$$.tsv"
METRICS_INTERVAL=60

# Metric types: counter, gauge, histogram
declare -A _counters=()
declare -A _gauges=()
declare -A _histograms=()

# Increment counter
counter_inc() {
    local name="$1"
    local amount="${2:-1}"
    local labels="${3:-}"
    local key="${name}${labels:+{$labels}}"
    _counters[$key]=$(( ${_counters[$key]:-0} + amount ))
}

# Set gauge
gauge_set() {
    local name="$1"
    local value="$2"
    local labels="${3:-}"
    local key="${name}${labels:+{$labels}}"
    _gauges[$key]="$value"
}

# Record histogram value
histogram_observe() {
    local name="$1"
    local value="$2"
    local labels="${3:-}"
    local key="${name}${labels:+{$labels}}"
    _histograms[$key]+="${value} "
}

# Get histogram stats
histogram_stats() {
    local name="$1"
    local values=("${_histograms[$name]}")
    
    if [[ -z "${values[*]}" ]]; then
        echo "0 0 0 0 0"
        return
    fi
    
    echo "${values[*]}" | tr ' ' '\n' | grep -v '^$' | sort -n | awk '
    BEGIN { count=0; sum=0; min=""; max="" }
    {
        count++; sum+=$1
        if (min=="" || $1<min) min=$1
        if (max=="" || $1>max) max=$1
        vals[count]=$1
    }
    END {
        avg = (count>0) ? sum/count : 0
        p95_idx = int(count * 0.95)
        p95 = (p95_idx>0) ? vals[p95_idx] : 0
        printf "%d %.2f %.2f %.2f %.2f\n", count, avg, min, max, p95
    }'
}

# Collect system metrics
collect_system_metrics() {
    local timestamp
    timestamp=$(date +%s)
    
    # CPU
    local cpu_usage
    cpu_usage=$(top -bn1 2>/dev/null | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1 || echo 0)
    gauge_set "cpu_usage_percent" "${cpu_usage:-0}"
    
    # Memory
    if [[ -f /proc/meminfo ]]; then
        local total avail
        total=$(grep '^MemTotal:' /proc/meminfo | awk '{print $2}')
        avail=$(grep '^MemAvailable:' /proc/meminfo | awk '{print $2}')
        local used=$(( total - avail ))
        gauge_set "memory_total_kb" "$total"
        gauge_set "memory_used_kb" "$used"
        gauge_set "memory_available_kb" "$avail"
        gauge_set "memory_usage_percent" $(( used * 100 / total ))
    fi
    
    # Disk
    while IFS= read -r line; do
        local mount pct used_kb total_kb
        mount=$(echo "$line" | awk '{print $NF}')
        pct=$(echo "$line" | awk '{gsub(/%/,"",$5); print $5}')
        used_kb=$(echo "$line" | awk '{print $3}')
        total_kb=$(echo "$line" | awk '{print $2}')
        
        local mount_label
        mount_label=$(echo "$mount" | tr '/' '_' | sed 's/^_//')
        gauge_set "disk_usage_percent" "$pct" "mount=${mount_label}"
    done < <(df -h 2>/dev/null | awk 'NR>1 && /^\// {print}')
    
    # Load average
    if [[ -f /proc/loadavg ]]; then
        read -r load1 load5 load15 _ < /proc/loadavg
        gauge_set "load_1min" "$load1"
        gauge_set "load_5min" "$load5"
        gauge_set "load_15min" "$load15"
    fi
    
    # Process count
    local proc_count
    proc_count=$(ps aux | wc -l)
    gauge_set "process_count" "$proc_count"
}

# Export metrics
export_prometheus() {
    local output="${1:-/dev/stdout}"
    
    {
        echo "# Bash Script Metrics"
        echo "# $(date)"
        echo ""
        
        # Gauges
        for key in "${!_gauges[@]}"; do
            local name labels=""
            if [[ "$key" =~ ^([^{]+)\{(.*)\}$ ]]; then
                name="${BASH_REMATCH[1]}"
                labels="{${BASH_REMATCH[2]}}"
            else
                name="$key"
            fi
            echo "# TYPE ${name} gauge"
            echo "${name}${labels} ${_gauges[$key]}"
        done
        
        # Counters
        for key in "${!_counters[@]}"; do
            local name labels=""
            if [[ "$key" =~ ^([^{]+)\{(.*)\}$ ]]; then
                name="${BASH_REMATCH[1]}"
                labels="{${BASH_REMATCH[2]}}"
            else
                name="$key"
            fi
            echo "# TYPE ${name}_total counter"
            echo "${name}_total${labels} ${_counters[$key]}"
        done
    } > "$output"
}

export_json() {
    local output="${1:-/dev/stdout}"
    local timestamp
    timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    
    {
        echo "{"
        echo "  \"timestamp\": \"$timestamp\","
        echo "  \"gauges\": {"
        
        local first=true
        for key in "${!_gauges[@]}"; do
            $first || echo ","
            printf '    "%s": %s' "$key" "${_gauges[$key]}"
            first=false
        done
        
        echo ""
        echo "  },"
        echo "  \"counters\": {"
        
        first=true
        for key in "${!_counters[@]}"; do
            $first || echo ","
            printf '    "%s": %s' "$key" "${_counters[$key]}"
            first=false
        done
        
        echo ""
        echo "  }"
        echo "}"
    } > "$output"
}

# Demo
echo "=== Collecting Metrics ==="
collect_system_metrics

# Simulate some application metrics
counter_inc "http_requests_total" 100 'method=GET,status=200'
counter_inc "http_requests_total" 5 'method=GET,status=404'
counter_inc "http_requests_total" 2 'method=POST,status=500'

for duration in 50 120 85 200 95 150 300 45 175 210; do
    histogram_observe "request_duration_ms" "$duration"
done

echo ""
echo "=== Gauges ==="
for key in "${!_gauges[@]}"; do
    printf "  %-40s = %s\n" "$key" "${_gauges[$key]}"
done

echo ""
echo "=== Counters ==="
for key in "${!_counters[@]}"; do
    printf "  %-50s = %s\n" "$key" "${_counters[$key]}"
done

echo ""
echo "=== Request Duration Histogram ==="
read -r count avg min max p95 <<< "$(histogram_stats "request_duration_ms")"
printf "  Count: %s, Avg: %.2fms, Min: %s, Max: %s, P95: %.2fms\n" \
    "$count" "$avg" "$min" "$max" "$p95"

echo ""
echo "=== Prometheus Format ==="
export_prometheus

echo ""
echo "=== JSON Format ==="
export_json
```

---

## ขั้นตอนที่ 429: Log Aggregation และ Analysis

```bash
#!/usr/bin/env bash
# log_analysis.sh

# ===== Log Analysis Tools =====

# Parse log line
parse_log_line() {
    local line="$1"
    local -A parsed=()
    
    # Common formats: [timestamp] [level] message
    if [[ "$line" =~ ^\[([^\]]+)\][[:space:]]+\[([^\]]+)\][[:space:]]+(.+)$ ]]; then
        parsed[timestamp]="${BASH_REMATCH[1]}"
        parsed[level]="${BASH_REMATCH[2]}"
        parsed[message]="${BASH_REMATCH[3]}"
    # Nginx format: IP - - [date] "method path protocol" status bytes
    elif [[ "$line" =~ ^([0-9.]+)[[:space:]]+-[[:space:]]+-[[:space:]]+\[([^\]]+)\][[:space:]]+"([^"]+)"[[:space:]]+([0-9]+) ]]; then
        parsed[ip]="${BASH_REMATCH[1]}"
        parsed[timestamp]="${BASH_REMATCH[2]}"
        parsed[request]="${BASH_REMATCH[3]}"
        parsed[status]="${BASH_REMATCH[4]}"
    fi
    
    for key in "${!parsed[@]}"; do
        echo "$key=${parsed[$key]}"
    done
}

# Log statistics
analyze_log_file() {
    local log_file="$1"
    
    echo "=== Log Analysis: $log_file ==="
    echo "File size: $(du -sh "$log_file" 2>/dev/null | cut -f1)"
    echo "Total lines: $(wc -l < "$log_file")"
    echo ""
    
    echo "--- Level Distribution ---"
    grep -oE '\[(DEBUG|INFO|WARN|ERROR|FATAL|CRITICAL)\]' "$log_file" 2>/dev/null | \
        sort | uniq -c | sort -rn | \
        awk '{printf "  %-10s: %d\n", $2, $1}'
    
    echo ""
    echo "--- Errors per Hour ---"
    grep "ERROR\|CRITICAL\|FATAL" "$log_file" 2>/dev/null | \
        grep -oE '^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}' | \
        sort | uniq -c | \
        awk '{printf "  %s: %d errors\n", $2, $1}' | tail -24
    
    echo ""
    echo "--- Top Error Messages ---"
    grep "ERROR\|CRITICAL\|FATAL" "$log_file" 2>/dev/null | \
        sed 's/.*\] //' | \
        sort | uniq -c | sort -rn | head -10 | \
        awk '{printf "  %3d: %s\n", $1, substr($0, index($0,$2))}'
}

# Real-time log following with filtering
follow_log() {
    local log_file="$1"
    local filter="${2:-}"
    local highlight="${3:-ERROR}"
    
    tail -f "$log_file" 2>/dev/null | while IFS= read -r line; do
        # Filter
        if [[ -n "$filter" ]] && ! [[ "$line" == *"$filter"* ]]; then
            continue
        fi
        
        # Highlight
        if [[ "$line" == *"$highlight"* ]]; then
            echo -e "\033[31m$line\033[0m"
        elif [[ "$line" == *"WARN"* ]]; then
            echo -e "\033[33m$line\033[0m"
        elif [[ "$line" == *"INFO"* ]]; then
            echo -e "\033[32m$line\033[0m"
        else
            echo "$line"
        fi
    done
}

# Log search with context
log_search() {
    local log_file="$1"
    local pattern="$2"
    local context="${3:-2}"
    
    echo "=== Searching for: $pattern ==="
    grep -n -C "$context" "$pattern" "$log_file" 2>/dev/null | \
        head -50
}

# Extract and aggregate log metrics
extract_metrics_from_log() {
    local log_file="$1"
    
    echo "=== Metrics from Log ==="
    
    # Request durations (if present)
    if grep -q "duration=" "$log_file" 2>/dev/null; then
        echo "Request Duration Statistics:"
        grep -oE 'duration=[0-9]+' "$log_file" | \
            cut -d= -f2 | \
            awk '
            {sum+=$1; count++; if(min==""||$1<min) min=$1; if(max==""||$1>max) max=$1}
            END {
                if(count>0) printf "  Count: %d, Avg: %.0fms, Min: %dms, Max: %dms\n",
                    count, sum/count, min, max
            }'
    fi
    
    # Status codes
    if grep -q "status=" "$log_file" 2>/dev/null; then
        echo "Status Code Distribution:"
        grep -oE 'status=[0-9]+' "$log_file" | \
            sort | uniq -c | \
            awk '{printf "  %s: %d\n", $2, $1}'
    fi
    
    # IP addresses
    if grep -qE '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' "$log_file" 2>/dev/null; then
        echo "Top IP Addresses:"
        grep -oE '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' "$log_file" | \
            sort | uniq -c | sort -rn | head -10 | \
            awk '{printf "  %-15s: %d requests\n", $2, $1}'
    fi
}

# Generate sample log for demo
generate_sample_log() {
    local file="$1"
    local lines="${2:-200}"
    
    local levels=(DEBUG INFO INFO INFO WARN ERROR)
    local messages=(
        "Request processed successfully"
        "User logged in"
        "Database query executed"
        "Cache miss for key: user_123"
        "High memory usage detected"
        "Connection to database failed"
        "Authentication failed for user: unknown"
        "File not found: /var/data/missing.txt"
    )
    
    for ((i=1; i<=lines; i++)); do
        local timestamp
        timestamp=$(date -d "-$((RANDOM % 3600)) seconds" '+%Y-%m-%dT%H:%M:%S' 2>/dev/null || date '+%Y-%m-%dT%H:%M:%S')
        local level="${levels[$((RANDOM % ${#levels[@]}))]}"
        local message="${messages[$((RANDOM % ${#messages[@]}))]}"
        echo "[$timestamp] [$level] $message" >> "$file"
    done
}

# Demo
echo "=== Log Analysis Demo ==="
SAMPLE_LOG="/tmp/sample_$$.log"
generate_sample_log "$SAMPLE_LOG" 100
analyze_log_file "$SAMPLE_LOG"
echo ""
extract_metrics_from_log "$SAMPLE_LOG"
rm -f "$SAMPLE_LOG"
```

---

## ขั้นตอนที่ 430: Health Check System

```bash
#!/usr/bin/env bash
# health_check.sh

# ===== Health Check System =====
HEALTH_STATUS="healthy"
declare -A HEALTH_CHECKS=()
declare -A HEALTH_DETAILS=()

# Register health check
register_check() {
    local name="$1"
    local check_fn="$2"
    HEALTH_CHECKS[$name]="$check_fn"
}

# Run a single health check
run_check() {
    local name="$1"
    local check_fn="${HEALTH_CHECKS[$name]}"
    
    if [[ -z "$check_fn" ]]; then
        echo "Unknown check: $name"
        return 2
    fi
    
    local start end duration
    start=$(date +%s%N)
    
    local output
    if output=$("$check_fn" 2>&1); then
        end=$(date +%s%N)
        duration=$(( (end - start) / 1000000 ))
        HEALTH_DETAILS[$name]="ok:${duration}ms:$output"
        echo "  ✓ $name (${duration}ms)"
        return 0
    else
        end=$(date +%s%N)
        duration=$(( (end - start) / 1000000 ))
        HEALTH_DETAILS[$name]="fail:${duration}ms:$output"
        echo "  ✗ $name (${duration}ms): $output"
        HEALTH_STATUS="unhealthy"
        return 1
    fi
}

# Run all health checks
run_all_checks() {
    echo "Health Checks: $(date)"
    echo "===================="
    
    for name in "${!HEALTH_CHECKS[@]}"; do
        run_check "$name"
    done
}

# Output JSON health report
health_json() {
    local overall="$HEALTH_STATUS"
    
    echo "{"
    echo "  \"status\": \"$overall\","
    echo "  \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\","
    echo "  \"checks\": {"
    
    local first=true
    for name in "${!HEALTH_DETAILS[@]}"; do
        $first || echo ","
        
        local status duration detail
        IFS=':' read -r status duration detail <<< "${HEALTH_DETAILS[$name]}"
        
        printf '    "%s": {"status": "%s", "duration": "%s", "message": "%s"}' \
            "$name" "$status" "$duration" "${detail//\"/\\\"}"
        
        first=false
    done
    
    echo ""
    echo "  }"
    echo "}"
}

# ===== Built-in Health Checks =====

check_disk_space() {
    local min_free_pct="${1:-10}"
    local usage
    usage=$(df / | awk 'NR==2{gsub(/%/,"",$5); print $5}')
    local free=$((100 - usage))
    
    if [[ $free -lt $min_free_pct ]]; then
        echo "Disk space critical: only ${free}% free"
        return 1
    fi
    echo "Disk OK: ${free}% free"
}

check_memory_available() {
    local min_free_pct="${1:-10}"
    
    if [[ -f /proc/meminfo ]]; then
        local total avail
        total=$(grep '^MemTotal:' /proc/meminfo | awk '{print $2}')
        avail=$(grep '^MemAvailable:' /proc/meminfo | awk '{print $2}')
        local free_pct=$(( avail * 100 / total ))
        
        if [[ $free_pct -lt $min_free_pct ]]; then
            echo "Memory critical: only ${free_pct}% available"
            return 1
        fi
        echo "Memory OK: ${free_pct}% available"
    fi
}

check_tcp_port() {
    local host="$1"
    local port="$2"
    local timeout="${3:-5}"
    
    if timeout "$timeout" bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null; then
        echo "$host:$port is reachable"
        return 0
    else
        echo "Cannot reach $host:$port"
        return 1
    fi
}

check_http_endpoint() {
    local url="$1"
    local expected_code="${2:-200}"
    local timeout="${3:-10}"
    
    if ! command -v curl &>/dev/null; then
        echo "curl not available"
        return 2
    fi
    
    local status_code
    status_code=$(curl -s -o /dev/null -w '%{http_code}' \
        --max-time "$timeout" "$url" 2>/dev/null)
    
    if [[ "$status_code" == "$expected_code" ]]; then
        echo "HTTP $url: $status_code OK"
        return 0
    else
        echo "HTTP $url: got $status_code, expected $expected_code"
        return 1
    fi
}

check_file_exists() {
    local file="$1"
    [[ -f "$file" ]] && echo "File exists: $file" || { echo "Missing: $file"; return 1; }
}

check_command_available() {
    local cmd="$1"
    command -v "$cmd" &>/dev/null && echo "$cmd found: $(command -v "$cmd")" || {
        echo "$cmd not found"
        return 1
    }
}

# ===== Register Checks =====
register_check "disk" check_disk_space
register_check "memory" check_memory_available
register_check "bash" "check_command_available bash"
register_check "curl" "check_command_available curl"

# ===== HTTP Health Endpoint (simple) =====
serve_health_endpoint() {
    local port="${1:-8080}"
    
    echo "Starting health endpoint on port $port..."
    
    while true; do
        # Simple HTTP server using bash
        {
            read -r request
            while read -r line && [[ -n "$line" ]]; do :; done
            
            HEALTH_STATUS="healthy"
            run_all_checks > /dev/null 2>&1
            
            local body
            body=$(health_json)
            
            echo -e "HTTP/1.1 200 OK\r"
            echo -e "Content-Type: application/json\r"
            echo -e "Content-Length: ${#body}\r"
            echo -e "\r"
            echo "$body"
        } | nc -l -p "$port" -q 1 2>/dev/null || break
    done
}

# Demo
echo "=== Health Check Demo ==="
run_all_checks

echo ""
echo "=== JSON Health Report ==="
health_json
```

---

## สรุป Part 22

### สิ่งที่เรียนรู้ (Steps 425-430):

| Step | หัวข้อ | ฟีเจอร์หลัก |
|------|--------|------------|
| 425 | Structured Logging | log levels, JSON/text format, rotation |
| 426 | System Dashboard | CPU/Memory/Disk bars, real-time display |
| 427 | Alert System | cooldown, email/Slack alerts, thresholds |
| 428 | Metrics Collection | counters, gauges, histograms, Prometheus |
| 429 | Log Analysis | parsing, statistics, real-time following |
| 430 | Health Checks | registered checks, JSON report, HTTP endpoint |

### เครื่องมือที่ใช้:
- `/proc/stat`, `/proc/meminfo`, `/proc/loadavg`
- `top`, `df`, `ps aux`
- `tail -f` สำหรับ log following
- `awk`, `grep` สำหรับ log analysis
- `nc` (netcat) สำหรับ simple HTTP server

**ขั้นตอนต่อไป**: Part 23 - Configuration Management
