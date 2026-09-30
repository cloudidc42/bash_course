# Part 06: Loop - for, while, until
## หลักสูตร Bash Script - ขั้นตอนที่ 141-170

---

## ขั้นตอนที่ 141: for Loop

### 141.1 for Loop พื้นฐาน

```bash
#!/bin/bash

# for loop แบบ list
for item in apple banana cherry; do
    echo "Fruit: $item"
done

# for loop แบบ array
fruits=("apple" "banana" "cherry" "date")
for fruit in "${fruits[@]}"; do
    echo "Fruit: $fruit"
done

# for loop แบบ range (C-style)
for ((i=1; i<=5; i++)); do
    echo "Count: $i"
done

# for loop แบบ seq
for i in $(seq 1 5); do
    echo "Seq: $i"
done

# for loop แบบ brace expansion
for i in {1..5}; do
    echo "Brace: $i"
done

# for loop กับ step
for i in {0..20..5}; do    # 0, 5, 10, 15, 20
    echo "Step: $i"
done

# for loop แบบ C-style ที่ยืดหยุ่น
for ((i=0, j=10; i<5; i++, j-=2)); do
    echo "i=$i, j=$j"
done
```

### 141.2 for Loop กับ Files

```bash
#!/bin/bash

# Loop ผ่านไฟล์ใน directory
for file in /etc/*.conf; do
    if [ -f "$file" ]; then
        echo "Config: $file ($(wc -l < "$file") lines)"
    fi
done

# Loop ผ่านไฟล์แบบ glob patterns
for file in *.{sh,py,js} 2>/dev/null; do
    [ -f "$file" ] || continue
    echo "Script: $file"
done

# Loop ผ่านไฟล์แบบปลอดภัย (รองรับ spaces)
while IFS= read -r -d $'\0' file; do
    echo "File: $file"
done < <(find /tmp -type f -print0)

# Rename ไฟล์ทั้งหมด
for file in *.txt; do
    [ -f "$file" ] || continue
    newname="${file%.txt}.bak"
    mv "$file" "$newname"
    echo "Renamed: $file → $newname"
done

# ประมวลผลแต่ละบรรทัดในไฟล์
for line in $(cat /etc/passwd | cut -d: -f1); do
    echo "User: $line"
done
# แต่วิธีนี้ไม่ดีถ้า lines มี spaces ใช้ while read แทน
```

### 141.3 Nested for Loops

```bash
#!/bin/bash

# สร้างตาราง multiplication
echo "=== ตารางสูตรคูณ ==="
printf "%4s" ""
for ((j=1; j<=9; j++)); do
    printf "%4d" "$j"
done
echo ""

for ((i=1; i<=9; i++)); do
    printf "%4d" "$i"
    for ((j=1; j<=9; j++)); do
        printf "%4d" "$((i * j))"
    done
    echo ""
done

# สร้าง patterns
echo ""
echo "=== Triangle Pattern ==="
for ((i=1; i<=5; i++)); do
    for ((j=1; j<=i; j++)); do
        printf "*"
    done
    echo ""
done

echo ""
echo "=== Diamond Pattern ==="
n=5
for ((i=1; i<=n; i++)); do
    printf "%$((n-i))s"
    for ((j=1; j<=2*i-1; j++)); do
        printf "*"
    done
    echo ""
done
for ((i=n-1; i>=1; i--)); do
    printf "%$((n-i))s"
    for ((j=1; j<=2*i-1; j++)); do
        printf "*"
    done
    echo ""
done
```

---

## ขั้นตอนที่ 142: while Loop

### 142.1 while Loop พื้นฐาน

```bash
#!/bin/bash

# while loop พื้นฐาน
count=1
while [ "$count" -le 5 ]; do
    echo "Count: $count"
    ((count++))
done

# while loop แบบ arithmetic
i=0
while (( i < 10 )); do
    echo "i = $i"
    (( i += 2 ))
done

# Infinite loop
while true; do
    echo "กำลังทำงาน..."
    sleep 1
    # break เมื่อต้องการออก
    break
done

# while loop กับ condition ที่ซับซ้อน
retries=0
max_retries=3
success=false

while [ "$retries" -lt "$max_retries" ] && [ "$success" = false ]; do
    echo "Attempt $((retries+1))/$max_retries"
    
    # จำลองการลองเชื่อมต่อ
    if (( RANDOM % 3 == 0 )); then
        success=true
        echo "สำเร็จ!"
    else
        echo "ล้มเหลว, รอ..."
        sleep 1
        ((retries++))
    fi
done

if [ "$success" = false ]; then
    echo "ERROR: ลองครบ $max_retries ครั้งแล้วไม่สำเร็จ"
fi
```

### 142.2 while read - อ่านไฟล์

```bash
#!/bin/bash

# อ่านไฟล์ทีละบรรทัด (วิธีที่ถูกต้อง)
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/hostname

# อ่านพร้อม parse
while IFS=: read -r user password uid gid comment home shell; do
    printf "User: %-15s UID: %-6s Home: %s\n" "$user" "$uid" "$home"
done < /etc/passwd | head -5

# อ่านจาก command output
while IFS= read -r process; do
    echo "Process: $process"
done < <(ps aux | grep nginx | grep -v grep)

# อ่าน CSV
while IFS=, read -r name email age; do
    echo "Name: $name, Email: $email, Age: $age"
done << 'CSV'
สมชาย,somchai@example.com,25
สมหญิง,somying@example.com,30
สมศักดิ์,somsak@example.com,35
CSV

# อ่านแบบมี header
read_csv_with_header() {
    local file="$1"
    local -n result="$2"
    
    # อ่าน header
    IFS= read -r header < "$file"
    IFS=',' read -ra headers <<< "$header"
    
    # อ่าน data
    local line_num=0
    while IFS= read -r line; do
        [ $line_num -eq 0 ] && { ((line_num++)); continue; }
        
        IFS=',' read -ra values <<< "$line"
        declare -A row
        
        for i in "${!headers[@]}"; do
            row["${headers[$i]}"]="${values[$i]}"
        done
        
        # เพิ่มใน result
        result+=("$(declare -p row)")
        ((line_num++))
    done < "$file"
}
```

### 142.3 until Loop

```bash
#!/bin/bash

# until = loop จนกว่า condition จะเป็น true
# (ตรงข้ามกับ while)

count=1
until [ "$count" -gt 5 ]; do
    echo "Count: $count"
    ((count++))
done

# until loop สำหรับ retry
attempt=0
until ping -c1 -W1 google.com &>/dev/null; do
    echo "Waiting for network... (attempt $((attempt+1)))"
    sleep 2
    ((attempt++))
    if [ "$attempt" -ge 30 ]; then
        echo "Network timeout!"
        exit 1
    fi
done
echo "Network connected!"

# until loop รอ service
wait_for_service() {
    local host="$1"
    local port="$2"
    local timeout="${3:-30}"
    local elapsed=0
    
    echo "Waiting for $host:$port..."
    
    until nc -z "$host" "$port" 2>/dev/null; do
        if [ "$elapsed" -ge "$timeout" ]; then
            echo "Timeout waiting for $host:$port"
            return 1
        fi
        sleep 1
        ((elapsed++))
    done
    
    echo "$host:$port is ready! (after ${elapsed}s)"
    return 0
}

# wait_for_service localhost 5432  # รอ PostgreSQL
```

---

## ขั้นตอนที่ 143: Loop Control

### 143.1 break, continue, return

```bash
#!/bin/bash

# break - ออกจาก loop
for i in {1..10}; do
    if [ "$i" -eq 5 ]; then
        echo "Break at $i"
        break
    fi
    echo "i = $i"
done

# break N - ออก N ระดับ
for i in {1..3}; do
    for j in {1..3}; do
        if [ "$i" -eq 2 ] && [ "$j" -eq 2 ]; then
            echo "Break 2 levels at i=$i, j=$j"
            break 2    # ออก 2 ระดับ
        fi
        echo "i=$i, j=$j"
    done
done

# continue - ข้ามไปรอบถัดไป
for i in {1..10}; do
    if (( i % 2 == 0 )); then
        continue    # ข้ามเลขคู่
    fi
    echo "Odd: $i"
done

# continue N - ข้าม N ระดับ
for i in {1..3}; do
    for j in {1..3}; do
        if [ "$j" -eq 2 ]; then
            continue 2    # ข้ามไปรอบถัดไปของ outer loop
        fi
        echo "i=$i, j=$j"
    done
done
```

### 143.2 Loop with exit codes

```bash
#!/bin/bash

# ใช้ exit code เพื่อควบคุม loop
find_first() {
    local pattern="$1"
    local dir="${2:-.}"
    
    while IFS= read -r -d $'\0' file; do
        if [[ "$(basename "$file")" == $pattern ]]; then
            echo "$file"
            return 0    # พบแล้ว
        fi
    done < <(find "$dir" -type f -print0)
    
    return 1    # ไม่พบ
}

if result=$(find_first "*.log" /var/log); then
    echo "Found: $result"
else
    echo "Not found"
fi
```

---

## ขั้นตอนที่ 144: Advanced Loop Patterns

### 144.1 Loop แบบ Parallel

```bash
#!/bin/bash

# รัน loops แบบ parallel (background jobs)
urls=(
    "http://example.com"
    "http://example.org"
    "http://example.net"
)

# รัน parallel และรอผล
pids=()
results=()

for url in "${urls[@]}"; do
    (
        status=$(curl -o /dev/null -s -w "%{http_code}" --max-time 5 "$url" 2>/dev/null)
        echo "$url: $status"
    ) &
    pids+=($!)
done

# รอทุก background job
for pid in "${pids[@]}"; do
    wait "$pid"
done

echo "All checks complete"

# Parallel ด้วย xargs
# ls *.log | xargs -P 4 -I {} gzip {}

# Parallel ด้วย GNU parallel (ถ้ามี)
# find . -name "*.jpg" | parallel -j4 convert {} {.}.png
```

### 144.2 Loop Optimization

```bash
#!/bin/bash

# ไม่ดี: เรียก external command ทุก iteration
for i in {1..1000}; do
    date  # เรียก date 1000 ครั้ง
done

# ดีกว่า: เรียกครั้งเดียว ถ้าค่าไม่เปลี่ยน
current_date=$(date)
for i in {1..1000}; do
    echo "$current_date $i"  # ใช้ค่าที่คำนวณแล้ว
done

# ไม่ดี: ใช้ cat ใน loop (useless cat)
for file in *.txt; do
    cat "$file" | grep "error"
done

# ดีกว่า: ไม่ใช้ cat
for file in *.txt; do
    grep "error" "$file"
done

# ไม่ดี: subshell ใน loop
for i in {1..100}; do
    result=$(echo "$i * 2" | bc)  # spawn process ทุกครั้ง
    echo "$result"
done

# ดีกว่า: arithmetic ใน bash
for i in {1..100}; do
    echo $((i * 2))
done

# ไม่ดี: for loop กับ ls
for file in $(ls /tmp/*.txt); do
    echo "$file"
done

# ดีกว่า: ใช้ glob โดยตรง
for file in /tmp/*.txt; do
    [ -f "$file" ] && echo "$file"
done
```

### 144.3 Loop Patterns ที่ใช้บ่อย

```bash
#!/bin/bash

# Pattern: Process files in batches
process_in_batches() {
    local batch_size=10
    local -a batch=()
    
    while IFS= read -r item; do
        batch+=("$item")
        
        if [ "${#batch[@]}" -ge "$batch_size" ]; then
            # ประมวลผล batch
            echo "Processing batch of ${#batch[@]}: ${batch[*]}"
            batch=()
        fi
    done
    
    # ประมวลผล batch สุดท้าย
    if [ "${#batch[@]}" -gt 0 ]; then
        echo "Processing final batch: ${batch[*]}"
    fi
}

# ใช้งาน
seq 1 25 | process_in_batches

# Pattern: Retry with exponential backoff
retry_with_backoff() {
    local max_attempts="${1:-3}"
    local base_delay="${2:-1}"
    shift 2
    local command=("$@")
    
    local attempt=0
    local delay="$base_delay"
    
    while [ "$attempt" -lt "$max_attempts" ]; do
        if "${command[@]}"; then
            return 0
        fi
        
        ((attempt++))
        if [ "$attempt" -lt "$max_attempts" ]; then
            echo "Retry $attempt/$max_attempts in ${delay}s..."
            sleep "$delay"
            delay=$((delay * 2))
        fi
    done
    
    echo "Failed after $max_attempts attempts"
    return 1
}

# ใช้งาน
# retry_with_backoff 3 2 curl -sf http://example.com

# Pattern: Timeout loop
timeout_loop() {
    local timeout="$1"
    local interval="${2:-1}"
    local start_time
    start_time=$(date +%s)
    
    while true; do
        # ทำงาน
        echo "Working..."
        
        # ตรวจสอบ timeout
        local current_time
        current_time=$(date +%s)
        local elapsed=$((current_time - start_time))
        
        if [ "$elapsed" -ge "$timeout" ]; then
            echo "Timeout after ${timeout}s"
            return 1
        fi
        
        sleep "$interval"
    done
}

# Pattern: Generator pattern
generate_range() {
    local start="$1"
    local end="$2"
    local step="${3:-1}"
    
    for ((i=start; i<=end; i+=step)); do
        echo "$i"
    done
}

# ใช้งาน
while IFS= read -r num; do
    echo "Number: $num"
done < <(generate_range 1 10 2)
```

---

## ขั้นตอนที่ 145: Loop กับ Data Processing

### 145.1 Processing CSV และ Data Files

```bash
#!/bin/bash

# ประมวลผล CSV
process_csv() {
    local csv_file="$1"
    local -a headers=()
    local line_num=0
    local total=0
    local count=0
    
    while IFS=',' read -ra fields; do
        if [ "$line_num" -eq 0 ]; then
            headers=("${fields[@]}")
            ((line_num++))
            continue
        fi
        
        # แสดงข้อมูลแบบ formatted
        echo "Record $line_num:"
        for i in "${!headers[@]}"; do
            printf "  %-15s: %s\n" "${headers[$i]}" "${fields[$i]:-N/A}"
        done
        
        # คำนวณสถิติ (ถ้า field 3 เป็นตัวเลข)
        if [[ "${fields[2]:-0}" =~ ^[0-9.]+$ ]]; then
            total=$(echo "$total + ${fields[2]}" | bc)
            ((count++))
        fi
        
        ((line_num++))
    done < "$csv_file"
    
    if [ "$count" -gt 0 ]; then
        local avg
        avg=$(echo "scale=2; $total / $count" | bc)
        echo ""
        echo "Statistics (field 3):"
        echo "  Total: $total"
        echo "  Count: $count"
        echo "  Average: $avg"
    fi
}

# สร้างไฟล์ทดสอบ
cat > /tmp/test_data.csv << 'EOF'
Name,Department,Salary
สมชาย,Engineering,75000
สมหญิง,Marketing,65000
สมศักดิ์,Engineering,80000
สมใจ,HR,60000
สมหมาย,Marketing,70000
EOF

process_csv /tmp/test_data.csv

# ประมวลผล log file
analyze_log() {
    local logfile="$1"
    declare -A status_counts
    declare -A ip_counts
    local total=0
    
    while IFS= read -r line; do
        # parse Apache/Nginx combined log format
        # 192.168.1.1 - - [01/Jan/2026:12:00:00 +0700] "GET /path HTTP/1.1" 200 1234
        
        local ip status
        ip=$(echo "$line" | awk '{print $1}')
        status=$(echo "$line" | awk '{print $9}')
        
        if [[ "$status" =~ ^[0-9]+$ ]]; then
            ((status_counts[$status]++))
            ((ip_counts[$ip]++))
            ((total++))
        fi
    done < "$logfile"
    
    echo "Log Analysis: $logfile"
    echo "Total requests: $total"
    echo ""
    echo "Status codes:"
    for code in $(echo "${!status_counts[@]}" | tr ' ' '\n' | sort -n); do
        printf "  %3s: %d (%.1f%%)\n" "$code" "${status_counts[$code]}" \
            "$(echo "scale=1; ${status_counts[$code]} * 100 / $total" | bc)"
    done
    
    echo ""
    echo "Top 10 IPs:"
    for ip in "${!ip_counts[@]}"; do
        echo "${ip_counts[$ip]} $ip"
    done | sort -rn | head -10 | while read -r count ip; do
        printf "  %-20s %d requests\n" "$ip" "$count"
    done
}
```

### 145.2 Loop กับ JSON (ด้วย jq)

```bash
#!/bin/bash

# ต้องติดตั้ง jq: sudo apt install jq

# ตัวอย่าง JSON data
JSON_DATA='{
    "users": [
        {"id": 1, "name": "สมชาย", "email": "somchai@example.com", "active": true},
        {"id": 2, "name": "สมหญิง", "email": "somying@example.com", "active": false},
        {"id": 3, "name": "สมศักดิ์", "email": "somsak@example.com", "active": true}
    ]
}'

# อ่านทุก users
echo "All users:"
while IFS= read -r user; do
    id=$(echo "$user" | jq -r '.id')
    name=$(echo "$user" | jq -r '.name')
    email=$(echo "$user" | jq -r '.email')
    active=$(echo "$user" | jq -r '.active')
    
    printf "  [%d] %-15s %-30s %s\n" "$id" "$name" "$email" \
        "$([[ "$active" == "true" ]] && echo "Active" || echo "Inactive")"
done < <(echo "$JSON_DATA" | jq -c '.users[]')

# กรองเฉพาะ active users
echo ""
echo "Active users:"
while IFS= read -r name; do
    echo "  - $name"
done < <(echo "$JSON_DATA" | jq -r '.users[] | select(.active == true) | .name')

# สร้าง summary
total=$(echo "$JSON_DATA" | jq '.users | length')
active=$(echo "$JSON_DATA" | jq '[.users[] | select(.active == true)] | length')
echo ""
echo "Summary: $active/$total active"
```

---

## ขั้นตอนที่ 146: Workshop 06 - Data Processing Pipeline

### Workshop: Log Analysis Tool

```bash
#!/usr/bin/env bash
# =============================================================================
# Workshop 06: log_analyzer.sh
# เครื่องมือวิเคราะห์ Log files
# =============================================================================

set -euo pipefail

# Colors
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly BLUE='\033[0;34m'
readonly CYAN='\033[0;36m'
readonly NC='\033[0m'

# Configuration
readonly SCRIPT_NAME="$(basename "$0")"
LOG_FILE=""
TOP_N=10
SHOW_HOURLY=false
FILTER_STATUS=""
MIN_RESPONSE_TIME=0

# Statistics
declare -A STATUS_COUNTS=()
declare -A IP_COUNTS=()
declare -A URL_COUNTS=()
declare -A HOUR_COUNTS=()
declare -a SLOW_REQUESTS=()
TOTAL_REQUESTS=0
TOTAL_BYTES=0
ERROR_COUNT=0

usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS] LOGFILE

Options:
    -h, --help          แสดง help
    -n N                แสดง top N (default: 10)
    -s STATUS           กรอง HTTP status code
    -t MS               แสดง requests ที่ช้ากว่า MS milliseconds
    --hourly            แสดงสถิติรายชั่วโมง

Example:
    $SCRIPT_NAME /var/log/nginx/access.log
    $SCRIPT_NAME -n 20 -s 404 access.log
    $SCRIPT_NAME -t 1000 --hourly access.log
EOF
}

# Parse arguments
parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -h|--help) usage; exit 0 ;;
            -n) TOP_N="$2"; shift 2 ;;
            -s) FILTER_STATUS="$2"; shift 2 ;;
            -t) MIN_RESPONSE_TIME="$2"; shift 2 ;;
            --hourly) SHOW_HOURLY=true; shift ;;
            -*) echo "Unknown option: $1" >&2; usage; exit 1 ;;
            *)  LOG_FILE="$1"; shift ;;
        esac
    done
    
    if [ -z "$LOG_FILE" ]; then
        echo "Error: ต้องระบุ log file" >&2
        usage
        exit 1
    fi
    
    if [ ! -f "$LOG_FILE" ]; then
        echo "Error: ไม่พบไฟล์: $LOG_FILE" >&2
        exit 1
    fi
}

# Parse single log line (Combined Log Format)
# 192.168.1.1 - - [01/Jan/2026:12:00:00 +0700] "GET /path HTTP/1.1" 200 1234 "-" "Mozilla/5.0"
parse_log_line() {
    local line="$1"
    
    # ใช้ regex แยกส่วน
    local regex='^([0-9.]+) \S+ \S+ \[([^]]+)\] "(\S+) (\S+) \S+" ([0-9]+) ([0-9-]+)'
    
    if [[ "$line" =~ $regex ]]; then
        local ip="${BASH_REMATCH[1]}"
        local datetime="${BASH_REMATCH[2]}"
        local method="${BASH_REMATCH[3]}"
        local url="${BASH_REMATCH[4]}"
        local status="${BASH_REMATCH[5]}"
        local bytes="${BASH_REMATCH[6]}"
        
        # Extract hour
        local hour
        hour=$(echo "$datetime" | grep -oE '[0-9]+:[0-9]+:[0-9]+' | cut -d: -f1)
        
        # ตรวจสอบ filter
        if [ -n "$FILTER_STATUS" ] && [ "$status" != "$FILTER_STATUS" ]; then
            return 0
        fi
        
        # อัปเดต counters
        ((TOTAL_REQUESTS++))
        ((STATUS_COUNTS[$status]++)) 2>/dev/null || STATUS_COUNTS[$status]=1
        ((IP_COUNTS[$ip]++)) 2>/dev/null || IP_COUNTS[$ip]=1
        ((URL_COUNTS[$url]++)) 2>/dev/null || URL_COUNTS[$url]=1
        
        if [ -n "$hour" ]; then
            ((HOUR_COUNTS[$hour]++)) 2>/dev/null || HOUR_COUNTS[$hour]=1
        fi
        
        if [ "$bytes" != "-" ] && [[ "$bytes" =~ ^[0-9]+$ ]]; then
            TOTAL_BYTES=$((TOTAL_BYTES + bytes))
        fi
        
        if [[ "$status" =~ ^[45] ]]; then
            ((ERROR_COUNT++))
        fi
    fi
}

# Process log file
process_log() {
    echo -e "${CYAN}กำลังวิเคราะห์: $LOG_FILE${NC}"
    
    local line_count=0
    local total_lines
    total_lines=$(wc -l < "$LOG_FILE")
    
    while IFS= read -r line; do
        parse_log_line "$line"
        ((line_count++))
        
        # แสดง progress
        if (( line_count % 10000 == 0 )); then
            printf "\r  Progress: %d/%d lines (%.0f%%)" \
                "$line_count" "$total_lines" \
                "$(echo "scale=0; $line_count * 100 / $total_lines" | bc)"
        fi
    done < "$LOG_FILE"
    
    printf "\r  Processed: %d lines%s\n" "$line_count" "$(printf '%20s')"
}

# Format bytes
format_bytes() {
    local bytes="$1"
    if (( bytes >= 1073741824 )); then
        echo "$(echo "scale=2; $bytes / 1073741824" | bc)GB"
    elif (( bytes >= 1048576 )); then
        echo "$(echo "scale=2; $bytes / 1048576" | bc)MB"
    elif (( bytes >= 1024 )); then
        echo "$(echo "scale=2; $bytes / 1024" | bc)KB"
    else
        echo "${bytes}B"
    fi
}

# Print summary
print_summary() {
    echo ""
    echo -e "${BLUE}╔══════════════════════════════════════════════════╗${NC}"
    echo -e "${BLUE}║              Log Analysis Report                 ║${NC}"
    echo -e "${BLUE}╚══════════════════════════════════════════════════╝${NC}"
    echo ""
    
    echo -e "${YELLOW}📊 Summary${NC}"
    echo "  Log file:    $LOG_FILE"
    echo "  Total requests: $(printf '%,d' "$TOTAL_REQUESTS")"
    echo "  Total bytes: $(format_bytes $TOTAL_BYTES)"
    echo "  Error rate:  $(echo "scale=1; $ERROR_COUNT * 100 / $TOTAL_REQUESTS" | bc)%"
    echo ""
    
    echo -e "${YELLOW}📈 HTTP Status Codes${NC}"
    printf "  %-6s %8s %8s\n" "Status" "Count" "%"
    echo "  ──────────────────────"
    
    for status in $(echo "${!STATUS_COUNTS[@]}" | tr ' ' '\n' | sort -n); do
        local count="${STATUS_COUNTS[$status]}"
        local pct
        pct=$(echo "scale=1; $count * 100 / $TOTAL_REQUESTS" | bc)
        
        # Color by status
        local color="$NC"
        case "$status" in
            2*) color="$GREEN" ;;
            3*) color="$CYAN" ;;
            4*) color="$YELLOW" ;;
            5*) color="$RED" ;;
        esac
        
        printf "  ${color}%-6s${NC} %8d %7.1f%%\n" "$status" "$count" "$pct"
    done
    
    echo ""
    echo -e "${YELLOW}🔝 Top $TOP_N IP Addresses${NC}"
    printf "  %-20s %10s\n" "IP Address" "Requests"
    echo "  ──────────────────────────────"
    
    for ip in "${!IP_COUNTS[@]}"; do
        echo "${IP_COUNTS[$ip]} $ip"
    done | sort -rn | head "$TOP_N" | while read -r count ip; do
        printf "  %-20s %10d\n" "$ip" "$count"
    done
    
    echo ""
    echo -e "${YELLOW}🔗 Top $TOP_N URLs${NC}"
    printf "  %-50s %10s\n" "URL" "Requests"
    echo "  ──────────────────────────────────────────────────────────"
    
    for url in "${!URL_COUNTS[@]}"; do
        echo "${URL_COUNTS[$url]} $url"
    done | sort -rn | head "$TOP_N" | while read -r count url; do
        printf "  %-50s %10d\n" "${url:0:50}" "$count"
    done
    
    if [ "$SHOW_HOURLY" = true ] && [ "${#HOUR_COUNTS[@]}" -gt 0 ]; then
        echo ""
        echo -e "${YELLOW}⏰ Requests by Hour${NC}"
        printf "  %-6s %10s\n" "Hour" "Requests"
        echo "  ──────────────────"
        
        for hour in $(echo "${!HOUR_COUNTS[@]}" | tr ' ' '\n' | sort -n); do
            local count="${HOUR_COUNTS[$hour]}"
            local bar_len=$((count * 40 / TOTAL_REQUESTS))
            printf "  %02d:00  %5d |%s\n" "$hour" "$count" \
                "$(printf '█%.0s' $(seq 1 $((bar_len > 0 ? bar_len : 1))))"
        done
    fi
    
    echo ""
    echo -e "${GREEN}✓ Analysis complete${NC}"
}

# Main
main() {
    parse_args "$@"
    process_log
    print_summary
}

main "$@"
```

---

## สรุป Part 06

### สิ่งที่เรียนรู้

1. ✅ for loop ทุกรูปแบบ (list, array, C-style, brace expansion)
2. ✅ while loop และ until loop
3. ✅ while read สำหรับอ่านไฟล์
4. ✅ break, continue, break N, continue N
5. ✅ Parallel loops
6. ✅ Loop optimization
7. ✅ Retry patterns
8. ✅ Data processing กับ loops

### แบบฝึกหัด

**Easy:**
1. สร้าง script แสดง FizzBuzz (1-100)
2. สร้าง script หาผลรวมของ 1+2+...+N

**Medium:**
3. สร้าง script ที่อ่าน CSV และคำนวณสถิติ
4. สร้าง script ที่ retry HTTP request ด้วย exponential backoff

**Hard:**
5. สร้าง script วิเคราะห์ log file และสร้าง report

### เฉลย FizzBuzz

```bash
#!/bin/bash
for i in {1..100}; do
    if (( i % 15 == 0 )); then
        echo "FizzBuzz"
    elif (( i % 3 == 0 )); then
        echo "Fizz"
    elif (( i % 5 == 0 )); then
        echo "Buzz"
    else
        echo "$i"
    fi
done
```

---

**ต่อไป:** [Part 07 - Functions (ฟังก์ชัน)](part-07.md)

*Part 06 จบแล้ว! พร้อมเรียน Part 07 →*
