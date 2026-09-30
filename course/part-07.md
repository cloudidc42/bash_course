# Part 07: Functions (ฟังก์ชัน)
## หลักสูตร Bash Script - ขั้นตอนที่ 171-200

---

## ขั้นตอนที่ 171: Function พื้นฐาน

### 171.1 การประกาศ Function

```bash
#!/bin/bash

# วิธีที่ 1: function keyword
function greet() {
    echo "Hello, $1!"
}

# วิธีที่ 2: ไม่ใช้ function keyword (แนะนำ - POSIX compatible)
greet() {
    echo "Hello, $1!"
}

# เรียกใช้
greet "World"
greet "สมชาย"

# Function ที่ซับซ้อนขึ้น
calculate_bmi() {
    local weight="$1"    # น้ำหนัก kg
    local height="$2"    # ส่วนสูง m
    
    local bmi
    bmi=$(echo "scale=1; $weight / ($height * $height)" | bc)
    
    local category
    if (( $(echo "$bmi < 18.5" | bc -l) )); then
        category="น้ำหนักน้อย"
    elif (( $(echo "$bmi < 25" | bc -l) )); then
        category="ปกติ"
    elif (( $(echo "$bmi < 30" | bc -l) )); then
        category="น้ำหนักเกิน"
    else
        category="อ้วน"
    fi
    
    echo "BMI: $bmi ($category)"
}

calculate_bmi 70 1.75
calculate_bmi 50 1.65
calculate_bmi 90 1.70
```

### 171.2 Parameters และ Return Values

```bash
#!/bin/bash

# Parameters: $1, $2, ..., $@, $#
show_params() {
    echo "Function: ${FUNCNAME[0]}"
    echo "Num params: $#"
    echo "Param 1: $1"
    echo "Param 2: $2"
    echo "All params: $@"
    
    # Loop ผ่าน parameters
    for param in "$@"; do
        echo "  - $param"
    done
}

show_params "apple" "banana" "cherry"

# Return value
# Bash functions return exit code (0-255) เท่านั้น
# สำหรับ return ค่าอื่น ใช้:

# วิธี 1: echo + command substitution
get_full_name() {
    local first="$1"
    local last="$2"
    echo "$first $last"    # "return" ผ่าน stdout
}

full_name=$(get_full_name "สมชาย" "ใจดี")
echo "Full name: $full_name"

# วิธี 2: global variable
calculate_area() {
    local width="$1"
    local height="$2"
    RESULT=$((width * height))    # เก็บใน global
}

calculate_area 5 8
echo "Area: $RESULT"

# วิธี 3: nameref (Bash 4.3+)
calculate_area_v2() {
    local width="$1"
    local height="$2"
    declare -n _result="$3"    # nameref ไปยัง variable ที่ส่งมา
    _result=$((width * height))
}

calculate_area_v2 5 8 my_area
echo "Area: $my_area"

# Return exit code
is_even() {
    local num="$1"
    (( num % 2 == 0 ))    # 0 = true, 1 = false
}

if is_even 4; then
    echo "4 เป็นเลขคู่"
fi

if ! is_even 7; then
    echo "7 เป็นเลขคี่"
fi
```

### 171.3 Local Variables

```bash
#!/bin/bash

# local ป้องกันการ leak ไปยัง global scope
outer_var="global"

test_scope() {
    local inner_var="local"    # เฉพาะใน function นี้
    outer_var="modified"       # แก้ global!
    
    echo "Inside: $inner_var"
    echo "Inside: $outer_var"
}

echo "Before: $outer_var"
test_scope
echo "After:  $outer_var"    # ถูกแก้!

# inner_var ไม่มีข้างนอก
echo "Inner: ${inner_var:-undefined}"

# Best practice: ประกาศ local ทุกตัวแปรใน function
good_function() {
    local name="${1:-default}"
    local count=0
    local result=""
    
    for item in "${@:2}"; do
        result+="$item "
        ((count++))
    done
    
    echo "Name: $name, Count: $count, Items: ${result% }"
}

good_function "test" "a" "b" "c"
```

---

## ขั้นตอนที่ 172: Advanced Function Techniques

### 172.1 Recursive Functions

```bash
#!/bin/bash

# Factorial
factorial() {
    local n="$1"
    
    if [ "$n" -le 1 ]; then
        echo 1
        return
    fi
    
    local sub
    sub=$(factorial $((n - 1)))
    echo $((n * sub))
}

echo "5! = $(factorial 5)"
echo "10! = $(factorial 10)"

# Fibonacci
fibonacci() {
    local n="$1"
    
    if [ "$n" -le 1 ]; then
        echo "$n"
        return
    fi
    
    local a b
    a=$(fibonacci $((n - 1)))
    b=$(fibonacci $((n - 2)))
    echo $((a + b))
}

# Fibonacci แบบ iterative (เร็วกว่า)
fibonacci_iter() {
    local n="$1"
    local a=0
    local b=1
    local temp
    
    for ((i=0; i<n; i++)); do
        temp=$((a + b))
        a=$b
        b=$temp
    done
    echo "$a"
}

echo "Fibonacci 10 = $(fibonacci_iter 10)"

# Tree traversal
process_directory() {
    local dir="$1"
    local depth="${2:-0}"
    local indent
    indent=$(printf '%*s' $((depth * 2)) '')
    
    echo "${indent}📁 $(basename "$dir")"
    
    for item in "$dir"/*; do
        [ -e "$item" ] || continue
        
        if [ -d "$item" ]; then
            process_directory "$item" $((depth + 1))
        else
            echo "${indent}  📄 $(basename "$item")"
        fi
    done
}

# process_directory /etc/apt
```

### 172.2 Function Libraries

```bash
#!/bin/bash
# lib_utils.sh - Utility Functions Library

# ============================
# String Functions
# ============================

# Trim whitespace
str_trim() {
    local str="$1"
    str="${str#"${str%%[![:space:]]*}"}"
    str="${str%"${str##*[![:space:]]}"}"
    echo "$str"
}

# Convert to uppercase
str_upper() { echo "${1^^}"; }

# Convert to lowercase
str_lower() { echo "${1,,}"; }

# Capitalize first letter
str_capitalize() {
    local str="$1"
    echo "${str^}"
}

# Repeat string
str_repeat() {
    local str="$1"
    local n="$2"
    printf "%0.s$str" $(seq 1 "$n")
}

# Pad string
str_pad_left() {
    local str="$1"
    local width="$2"
    local pad="${3:- }"
    printf "%${width}s" "$str" | tr ' ' "$pad"
}

str_pad_right() {
    local str="$1"
    local width="$2"
    local pad="${3:- }"
    printf "%-${width}s" "$str" | tr ' ' "$pad"
}

# Contains substring
str_contains() {
    local str="$1"
    local sub="$2"
    [[ "$str" == *"$sub"* ]]
}

# Starts with
str_starts_with() {
    local str="$1"
    local prefix="$2"
    [[ "$str" == "$prefix"* ]]
}

# Ends with
str_ends_with() {
    local str="$1"
    local suffix="$2"
    [[ "$str" == *"$suffix" ]]
}

# Split string
str_split() {
    local str="$1"
    local delim="${2:-,}"
    local -n result_ref="$3"
    
    IFS="$delim" read -ra result_ref <<< "$str"
}

# Join array
arr_join() {
    local delim="$1"
    shift
    local first=true
    local result=""
    
    for item in "$@"; do
        if [ "$first" = true ]; then
            result="$item"
            first=false
        else
            result="${result}${delim}${item}"
        fi
    done
    
    echo "$result"
}

# ============================
# Number Functions
# ============================

# ตรวจสอบว่าเป็น number
num_is_integer() { [[ "$1" =~ ^-?[0-9]+$ ]]; }
num_is_float()   { [[ "$1" =~ ^-?[0-9]*\.?[0-9]+$ ]]; }
num_is_positive(){ [[ "$1" =~ ^[0-9]+$ ]] && [ "$1" -gt 0 ]; }

# Clamp value between min and max
num_clamp() {
    local val="$1"
    local min="$2"
    local max="$3"
    
    if (( val < min )); then echo "$min"
    elif (( val > max )); then echo "$max"
    else echo "$val"
    fi
}

# Round number
num_round() {
    local num="$1"
    local precision="${2:-0}"
    printf "%.${precision}f" "$num"
}

# Min/Max
num_min() {
    local min="$1"
    shift
    for n in "$@"; do
        (( n < min )) && min="$n"
    done
    echo "$min"
}

num_max() {
    local max="$1"
    shift
    for n in "$@"; do
        (( n > max )) && max="$n"
    done
    echo "$max"
}

# ============================
# Array Functions
# ============================

# ตรวจสอบว่า element อยู่ใน array
arr_contains() {
    local element="$1"
    shift
    for item in "$@"; do
        [[ "$item" == "$element" ]] && return 0
    done
    return 1
}

# ลบ duplicates
arr_unique() {
    local -a result=()
    for item in "$@"; do
        arr_contains "$item" "${result[@]}" || result+=("$item")
    done
    printf '%s\n' "${result[@]}"
}

# Flatten (ใช้กับ nested arrays ไม่ได้ใน bash แต่ทำ workaround ได้)
arr_reverse() {
    local -a arr=("$@")
    local -a result=()
    for ((i=${#arr[@]}-1; i>=0; i--)); do
        result+=("${arr[$i]}")
    done
    printf '%s\n' "${result[@]}"
}

# Filter
arr_filter() {
    local func="$1"
    shift
    for item in "$@"; do
        "$func" "$item" && echo "$item"
    done
}

# Map
arr_map() {
    local func="$1"
    shift
    for item in "$@"; do
        "$func" "$item"
    done
}

# ============================
# File Functions
# ============================

# ตรวจสอบและสร้าง directory
ensure_dir() {
    local dir="$1"
    [ -d "$dir" ] || mkdir -p "$dir"
}

# ตรวจสอบว่าไฟล์ว่าง
file_is_empty() { [ ! -s "$1" ]; }

# นับบรรทัด
file_line_count() { wc -l < "${1:?}"; }

# อ่านค่าจาก key=value file
read_config() {
    local file="$1"
    local key="$2"
    
    grep "^${key}=" "$file" 2>/dev/null | head -1 | cut -d= -f2-
}

# เขียนค่าลง key=value file
write_config() {
    local file="$1"
    local key="$2"
    local value="$3"
    
    if grep -q "^${key}=" "$file" 2>/dev/null; then
        sed -i "s|^${key}=.*|${key}=${value}|" "$file"
    else
        echo "${key}=${value}" >> "$file"
    fi
}

# ============================
# ทดสอบ Library
# ============================

# String tests
echo "=== String Tests ==="
echo "trim: '$(str_trim "  hello  ")'"
echo "upper: $(str_upper "hello")"
echo "lower: $(str_lower "HELLO")"
echo "repeat: $(str_repeat "ab" 3)"
echo "pad_right: $(str_pad_right "test" 10 "-")"
echo "contains 'world': $(str_contains "hello world" "world" && echo yes || echo no)"

# Number tests
echo ""
echo "=== Number Tests ==="
echo "clamp(15, 0, 10): $(num_clamp 15 0 10)"
echo "min(5,3,8,1): $(num_min 5 3 8 1)"
echo "max(5,3,8,1): $(num_max 5 3 8 1)"

# Array tests
echo ""
echo "=== Array Tests ==="
arr_unique "a" "b" "a" "c" "b" "d"
echo "reverse:"
arr_reverse "1" "2" "3" "4" "5"
```

---

## ขั้นตอนที่ 173: Function Design Patterns

### 173.1 Command Pattern

```bash
#!/bin/bash

# Command pattern: แยก commands เป็น functions
# แล้วใช้ dispatch table

# Commands
cmd_start()  { echo "Starting service..."; }
cmd_stop()   { echo "Stopping service..."; }
cmd_status() { echo "Service status: running"; }
cmd_restart(){ cmd_stop; cmd_start; }

# Dispatch
dispatch() {
    local command="$1"
    
    if declare -f "cmd_${command}" &>/dev/null; then
        "cmd_${command}"
    else
        echo "Unknown command: $command" >&2
        echo "Available: start, stop, status, restart"
        return 1
    fi
}

dispatch "start"
dispatch "status"
dispatch "restart"
dispatch "unknown"
```

### 173.2 Builder Pattern

```bash
#!/bin/bash

# Builder pattern สำหรับ HTTP requests
declare -A HTTP_REQUEST=(
    [method]="GET"
    [url]=""
    [headers]=""
    [body]=""
    [timeout]="30"
)

http_set_method()  { HTTP_REQUEST[method]="$1"; }
http_set_url()     { HTTP_REQUEST[url]="$1"; }
http_add_header()  { HTTP_REQUEST[headers]+=" -H '$1: $2'"; }
http_set_body()    { HTTP_REQUEST[body]="$1"; }
http_set_timeout() { HTTP_REQUEST[timeout]="$1"; }

http_build() {
    local cmd="curl"
    cmd+=" -s"
    cmd+=" -X ${HTTP_REQUEST[method]}"
    cmd+=" --max-time ${HTTP_REQUEST[timeout]}"
    
    if [ -n "${HTTP_REQUEST[headers]}" ]; then
        cmd+=" ${HTTP_REQUEST[headers]}"
    fi
    
    if [ -n "${HTTP_REQUEST[body]}" ]; then
        cmd+=" -d '${HTTP_REQUEST[body]}'"
    fi
    
    cmd+=" '${HTTP_REQUEST[url]}'"
    echo "$cmd"
}

# ใช้งาน
http_set_method "POST"
http_set_url "http://api.example.com/users"
http_add_header "Content-Type" "application/json"
http_add_header "Authorization" "Bearer TOKEN"
http_set_body '{"name": "John"}'

echo "Command: $(http_build)"
```

### 173.3 Middleware / Decorator Pattern

```bash
#!/bin/bash

# Decorator: เพิ่ม functionality รอบๆ function

# Timing decorator
timed() {
    local func="$1"
    shift
    
    local start_ns end_ns elapsed
    start_ns=$(date +%s%N 2>/dev/null || echo 0)
    
    "$func" "$@"
    local exit_code=$?
    
    end_ns=$(date +%s%N 2>/dev/null || echo 0)
    
    if [ "$start_ns" -ne 0 ]; then
        elapsed=$(( (end_ns - start_ns) / 1000000 ))
        echo "[TIMING] ${func}: ${elapsed}ms" >&2
    fi
    
    return "$exit_code"
}

# Logging decorator
logged() {
    local func="$1"
    shift
    
    echo "[LOG] Calling: $func $*" >&2
    "$func" "$@"
    local exit_code=$?
    echo "[LOG] $func returned: $exit_code" >&2
    
    return "$exit_code"
}

# Retry decorator
with_retry() {
    local max_attempts="$1"
    local func="$2"
    shift 2
    
    local attempt=0
    while [ "$attempt" -lt "$max_attempts" ]; do
        if "$func" "$@"; then
            return 0
        fi
        
        ((attempt++))
        [ "$attempt" -lt "$max_attempts" ] && {
            echo "[RETRY] Attempt $attempt failed, retrying..." >&2
            sleep "$attempt"
        }
    done
    
    echo "[RETRY] All $max_attempts attempts failed" >&2
    return 1
}

# ฟังก์ชันตัวอย่าง
my_function() {
    local n="$1"
    echo "Processing $n"
    sleep 0.1
}

unreliable_function() {
    (( RANDOM % 3 == 0 )) && return 0 || return 1
}

# ใช้ decorators
timed my_function "test"
logged my_function "hello"
with_retry 3 unreliable_function
```

---

## ขั้นตอนที่ 174: Error Handling ใน Functions

### 174.1 Error Propagation

```bash
#!/bin/bash
set -euo pipefail

# Function ที่ return error อย่างถูกต้อง
read_config_file() {
    local config_file="$1"
    
    # Validate input
    if [ -z "$config_file" ]; then
        echo "ERROR: ต้องระบุ config file" >&2
        return 1
    fi
    
    if [ ! -f "$config_file" ]; then
        echo "ERROR: ไม่พบไฟล์: $config_file" >&2
        return 1
    fi
    
    if [ ! -r "$config_file" ]; then
        echo "ERROR: ไม่มีสิทธิ์อ่าน: $config_file" >&2
        return 1
    fi
    
    # อ่านและ export config
    while IFS='=' read -r key value; do
        [[ "$key" =~ ^[[:space:]]*# ]] && continue
        [ -z "$key" ] && continue
        
        key=$(echo "$key" | tr -d '[:space:]')
        value=$(echo "$value" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
        
        export "$key=$value"
        echo "  Loaded: $key"
    done < "$config_file"
    
    return 0
}

# ใช้งานพร้อม error handling
config_file="/tmp/myapp.conf"

cat > "$config_file" << 'EOF'
# Application Config
APP_NAME=MyApp
APP_PORT=8080
APP_ENV=production
EOF

if read_config_file "$config_file"; then
    echo "Config loaded: APP_NAME=$APP_NAME, PORT=$APP_PORT"
else
    echo "Failed to load config"
    exit 1
fi
```

### 174.2 Cleanup Pattern

```bash
#!/bin/bash

# Cleanup stack - เพิ่ม cleanup functions และรันกลับด้าน
declare -a CLEANUP_STACK=()

add_cleanup() {
    CLEANUP_STACK+=("$1")
}

run_cleanup() {
    local exit_code=$?
    
    echo "Running cleanup (${#CLEANUP_STACK[@]} tasks)..."
    
    # รันกลับด้าน (stack order)
    for ((i=${#CLEANUP_STACK[@]}-1; i>=0; i--)); do
        eval "${CLEANUP_STACK[$i]}" || true
    done
    
    exit "$exit_code"
}

trap run_cleanup EXIT INT TERM

# ใช้งาน
temp_dir=$(mktemp -d)
add_cleanup "rm -rf '$temp_dir'"
echo "Created: $temp_dir"

temp_file=$(mktemp)
add_cleanup "rm -f '$temp_file'"
echo "Created: $temp_file"

# ทำงาน...
echo "data" > "$temp_file"
cp "$temp_file" "$temp_dir/"

echo "Work complete"
# cleanup จะรันอัตโนมัติ
```

---

## ขั้นตอนที่ 175: Workshop 07 - Script Framework

### Workshop: สร้าง Application Framework

```bash
#!/usr/bin/env bash
# =============================================================================
# Workshop 07: app_framework.sh
# Bash Application Framework
# =============================================================================

# ============================
# Framework Core
# ============================

# Version
readonly FRAMEWORK_VERSION="1.0.0"

# Color scheme
readonly COLOR_PRIMARY='\033[0;34m'
readonly COLOR_SUCCESS='\033[0;32m'
readonly COLOR_WARNING='\033[1;33m'
readonly COLOR_ERROR='\033[0;31m'
readonly COLOR_INFO='\033[0;36m'
readonly COLOR_DIM='\033[0;37m'
readonly COLOR_BOLD='\033[1m'
readonly COLOR_RESET='\033[0m'

# ============================
# Logging System
# ============================

declare -g LOG_LEVEL="${LOG_LEVEL:-INFO}"
declare -g LOG_FILE="${LOG_FILE:-}"
declare -g APP_NAME="${APP_NAME:-app}"

_log() {
    local level="$1"; shift
    local message="$*"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    local levels=(TRACE DEBUG INFO WARNING ERROR CRITICAL)
    local level_num=2
    local current_num=2
    
    for i in "${!levels[@]}"; do
        [ "${levels[$i]}" = "$level" ] && level_num=$i
        [ "${levels[$i]}" = "$LOG_LEVEL" ] && current_num=$i
    done
    
    [ "$level_num" -lt "$current_num" ] && return 0
    
    local color
    case "$level" in
        DEBUG)    color="$COLOR_DIM" ;;
        INFO)     color="$COLOR_INFO" ;;
        WARNING)  color="$COLOR_WARNING" ;;
        ERROR)    color="$COLOR_ERROR" ;;
        CRITICAL) color="${COLOR_ERROR}${COLOR_BOLD}" ;;
        *)        color="$COLOR_RESET" ;;
    esac
    
    local entry="[$timestamp] [$APP_NAME] [$level] $message"
    
    if [ "$level_num" -ge 4 ]; then
        echo -e "${color}${entry}${COLOR_RESET}" >&2
    else
        echo -e "${color}${entry}${COLOR_RESET}"
    fi
    
    [ -n "$LOG_FILE" ] && echo "$entry" >> "$LOG_FILE"
    
    return 0
}

log.trace()    { _log TRACE "$@"; }
log.debug()    { _log DEBUG "$@"; }
log.info()     { _log INFO "$@"; }
log.warning()  { _log WARNING "$@"; }
log.error()    { _log ERROR "$@"; }
log.critical() { _log CRITICAL "$@"; }

# ============================
# Configuration
# ============================

declare -A _CONFIG=()

config.set() { _CONFIG["$1"]="$2"; }
config.get() { echo "${_CONFIG[$1]:-${2:-}}"; }
config.has() { [[ "${_CONFIG[$1]+exists}" ]]; }

config.load() {
    local file="$1"
    [ -f "$file" ] || { log.error "Config file not found: $file"; return 1; }
    
    while IFS='=' read -r key value; do
        [[ "$key" =~ ^[[:space:]]*[#] ]] && continue
        [[ -z "$key" ]] && continue
        
        key="${key// /}"
        value="${value#"${value%%[![:space:]]*}"}"
        value="${value%"${value##*[![:space:]]}"}"
        
        _CONFIG["$key"]="$value"
    done < "$file"
    
    log.debug "Loaded config: $file (${#_CONFIG[@]} entries)"
}

config.dump() {
    echo "=== Configuration ==="
    for key in $(echo "${!_CONFIG[@]}" | tr ' ' '\n' | sort); do
        printf "  %-30s = %s\n" "$key" "${_CONFIG[$key]}"
    done
}

# ============================
# Event System
# ============================

declare -A _LISTENERS=()

event.on() {
    local event="$1"
    local callback="$2"
    _LISTENERS["$event"]+=" $callback"
}

event.emit() {
    local event="$1"; shift
    local listeners="${_LISTENERS[$event]:-}"
    
    for listener in $listeners; do
        "$listener" "$@"
    done
}

# ============================
# Plugin System
# ============================

declare -a _PLUGINS=()

plugin.load() {
    local plugin_file="$1"
    
    if [ ! -f "$plugin_file" ]; then
        log.error "Plugin not found: $plugin_file"
        return 1
    fi
    
    # shellcheck source=/dev/null
    source "$plugin_file"
    _PLUGINS+=("$plugin_file")
    log.info "Plugin loaded: $(basename "$plugin_file")"
}

plugin.list() {
    echo "Loaded plugins:"
    for p in "${_PLUGINS[@]}"; do
        echo "  - $(basename "$p")"
    done
}

# ============================
# Task System
# ============================

declare -A _TASKS=()
declare -A _TASK_DEPS=()

task.define() {
    local name="$1"
    local deps="${2:-}"
    local func="${3:-}"
    
    _TASKS["$name"]="${func:-$name}"
    _TASK_DEPS["$name"]="$deps"
}

task.run() {
    local task_name="$1"
    
    if [ -z "${_TASKS[$task_name]+exists}" ]; then
        log.error "Task not found: $task_name"
        return 1
    fi
    
    # รัน dependencies ก่อน
    local deps="${_TASK_DEPS[$task_name]}"
    for dep in $deps; do
        log.debug "Running dependency: $dep"
        task.run "$dep" || return 1
    done
    
    # รัน task
    local func="${_TASKS[$task_name]}"
    log.info "Running task: $task_name"
    
    local start_time
    start_time=$(date +%s)
    
    if "$func"; then
        local elapsed=$(( $(date +%s) - start_time ))
        log.info "Task '$task_name' completed in ${elapsed}s"
        event.emit "task.complete" "$task_name"
    else
        log.error "Task '$task_name' failed"
        event.emit "task.failed" "$task_name"
        return 1
    fi
}

# ============================
# ตัวอย่างใช้งาน Framework
# ============================

# Application setup
APP_NAME="MyApp"
LOG_LEVEL="DEBUG"
LOG_FILE="/tmp/myapp.log"

# Listen to events
task_completed() { log.info "✓ Task done: $1"; }
task_failed()    { log.error "✗ Task failed: $1"; }

event.on "task.complete" task_completed
event.on "task.failed"   task_failed

# Config
config.set "APP_VERSION" "1.0.0"
config.set "APP_ENV" "development"
config.set "DB_HOST" "localhost"
config.set "DB_PORT" "5432"

# Define tasks
setup_env() {
    log.info "Setting up environment..."
    mkdir -p /tmp/myapp
    return 0
}

build_app() {
    log.info "Building application..."
    sleep 0.5
    return 0
}

run_tests() {
    log.info "Running tests..."
    sleep 0.3
    return 0
}

deploy() {
    log.info "Deploying..."
    sleep 0.2
    return 0
}

# Register tasks
task.define "setup"  ""             setup_env
task.define "build"  "setup"        build_app
task.define "test"   "build"        run_tests
task.define "deploy" "test"         deploy

# Run
echo ""
log.info "Starting deployment pipeline..."
echo ""

task.run "deploy"

echo ""
config.dump
```

---

## สรุป Part 07

### สิ่งที่เรียนรู้

1. ✅ การประกาศและเรียกใช้ functions
2. ✅ Parameters และ return values
3. ✅ Local variables
4. ✅ Recursive functions
5. ✅ Function libraries
6. ✅ Design patterns: Command, Builder, Decorator
7. ✅ Error handling ใน functions
8. ✅ Application framework

---

**ต่อไป:** [Part 08 - Arrays (อาร์เรย์)](part-08.md)

*Part 07 จบแล้ว! พร้อมเรียน Part 08 →*
