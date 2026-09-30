# Part 25: Advanced Scripting Patterns

## Module 2: Intermediate Level (จบ)
### ขั้นตอนที่ 442-455: Patterns ขั้นสูง

---

## ขั้นตอนที่ 442: Design Patterns สำหรับ Bash

```bash
#!/usr/bin/env bash
# design_patterns.sh

# ===== 1. Singleton Pattern =====
declare -g _SINGLETON_INITIALIZED=false

singleton_init() {
    if $_SINGLETON_INITIALIZED; then
        return 0
    fi
    
    # Initialize once
    echo "Singleton initializing..."
    _SINGLETON_INITIALIZED=true
    declare -gA _SINGLETON_DATA=()
    echo "Singleton ready"
}

singleton_set() {
    singleton_init
    _SINGLETON_DATA["$1"]="$2"
}

singleton_get() {
    singleton_init
    echo "${_SINGLETON_DATA[$1]:-}"
}

# ===== 2. Observer Pattern =====
declare -A _OBSERVERS=()

event_subscribe() {
    local event="$1"
    local handler="$2"
    _OBSERVERS["$event"]+="$handler "
}

event_emit() {
    local event="$1"
    shift
    local handlers="${_OBSERVERS[$event]:-}"
    
    for handler in $handlers; do
        if declare -f "$handler" > /dev/null 2>&1; then
            "$handler" "$event" "$@"
        fi
    done
}

# ===== 3. Chain of Responsibility =====
declare -a _MIDDLEWARE_CHAIN=()

middleware_add() {
    _MIDDLEWARE_CHAIN+=("$1")
}

middleware_run() {
    local request="$1"
    local current_idx=0
    
    run_next() {
        local idx=$current_idx
        ((current_idx++))
        
        if [[ $idx -lt ${#_MIDDLEWARE_CHAIN[@]} ]]; then
            local middleware="${_MIDDLEWARE_CHAIN[$idx]}"
            "$middleware" "$request" run_next
        fi
    }
    
    run_next
}

# ===== 4. Strategy Pattern =====
declare -A _STRATEGIES=()

register_strategy() {
    local name="$1"
    local fn="$2"
    _STRATEGIES[$name]="$fn"
}

use_strategy() {
    local strategy="$1"
    shift
    
    local fn="${_STRATEGIES[$strategy]}"
    if [[ -z "$fn" ]]; then
        echo "Strategy not found: $strategy" >&2
        return 1
    fi
    
    "$fn" "$@"
}

# ===== 5. Builder Pattern =====
declare -A _BUILDER=()

builder_new() {
    _BUILDER=()
    _BUILDER[type]="$1"
}

builder_set() {
    _BUILDER["$1"]="$2"
}

builder_build() {
    echo "Building ${_BUILDER[type]}:"
    for key in "${!_BUILDER[@]}"; do
        [[ "$key" == "type" ]] && continue
        echo "  $key=${_BUILDER[$key]}"
    done
}

# ===== 6. Command Pattern =====
declare -a _COMMAND_HISTORY=()

execute_command() {
    local cmd_name="$1"
    local cmd_fn="${cmd_name}_execute"
    local undo_fn="${cmd_name}_undo"
    shift
    
    if declare -f "$cmd_fn" > /dev/null 2>&1; then
        "$cmd_fn" "$@"
        _COMMAND_HISTORY+=("$cmd_name:$*")
    else
        echo "Command not found: $cmd_name" >&2
        return 1
    fi
}

undo_last() {
    if [[ ${#_COMMAND_HISTORY[@]} -eq 0 ]]; then
        echo "Nothing to undo"
        return 1
    fi
    
    local last="${_COMMAND_HISTORY[-1]}"
    unset '_COMMAND_HISTORY[-1]'
    
    local cmd_name="${last%%:*}"
    local args="${last#*:}"
    local undo_fn="${cmd_name}_undo"
    
    if declare -f "$undo_fn" > /dev/null 2>&1; then
        echo "Undoing: $cmd_name"
        $undo_fn $args
    fi
}

# ===== Demo =====
echo "=== Design Patterns Demo ==="

# Singleton
echo ""
echo "1. Singleton:"
singleton_set "config" "production"
singleton_set "version" "1.0.0"
echo "Config: $(singleton_get config)"
echo "Version: $(singleton_get version)"

# Observer
echo ""
echo "2. Observer:"
on_user_login() { echo "  Handler: User logged in - event=$1, user=$2"; }
on_user_login_audit() { echo "  Audit: Login recorded - user=$2"; }

event_subscribe "user.login" "on_user_login"
event_subscribe "user.login" "on_user_login_audit"
event_emit "user.login" "alice"

# Strategy
echo ""
echo "3. Strategy:"
compress_gzip() { echo "  Compressing with gzip: $1"; }
compress_bzip2() { echo "  Compressing with bzip2: $1"; }
compress_none() { echo "  No compression: $1"; }

register_strategy "gzip" compress_gzip
register_strategy "bzip2" compress_bzip2
register_strategy "none" compress_none

use_strategy "gzip" "myfile.tar"
use_strategy "bzip2" "myfile.tar"

# Builder
echo ""
echo "4. Builder:"
builder_new "HTTPRequest"
builder_set "method" "POST"
builder_set "url" "https://api.example.com/users"
builder_set "body" '{"name":"Alice"}'
builder_set "timeout" "30"
builder_build

# Command
echo ""
echo "5. Command pattern:"
create_file_execute() { touch "$1" && echo "  Created: $1"; }
create_file_undo() { rm -f "$1" && echo "  Deleted: $1"; }

execute_command "create_file" "/tmp/test_cmd_$$.txt"
ls /tmp/test_cmd_$$.txt && echo "  File exists"
undo_last
[[ ! -f /tmp/test_cmd_$$.txt ]] && echo "  File removed by undo"
```

---

## ขั้นตอนที่ 443: Functional Programming ใน Bash

```bash
#!/usr/bin/env bash
# functional_bash.sh

# ===== Functional Programming Concepts =====

# 1. Pure Functions (ไม่มี side effects)
pure_add() { echo $(( $1 + $2 )); }
pure_multiply() { echo $(( $1 * $2 )); }
pure_uppercase() { echo "${1^^}"; }

# 2. Higher-Order Functions
map() {
    local fn="$1"
    shift
    for item in "$@"; do
        "$fn" "$item"
    done
}

filter() {
    local predicate="$1"
    shift
    for item in "$@"; do
        if "$predicate" "$item" 2>/dev/null; then
            echo "$item"
        fi
    done
}

reduce() {
    local fn="$1"
    local accumulator="$2"
    shift 2
    
    for item in "$@"; do
        accumulator=$("$fn" "$accumulator" "$item")
    done
    
    echo "$accumulator"
}

# 3. Currying
curry() {
    local fn="$1"
    local partial_arg="$2"
    
    # สร้าง curried function
    eval "curried_${fn}_${partial_arg//[^a-zA-Z0-9]/_}() {
        $fn '$partial_arg' \"\$@\"
    }"
    echo "curried_${fn}_${partial_arg//[^a-zA-Z0-9]/_}"
}

# 4. Function Composition
compose() {
    local -a fns=("$@")
    
    composed() {
        local result="$1"
        for fn in "${fns[@]}"; do
            result=$("$fn" "$result")
        done
        echo "$result"
    }
    
    echo "composed"  # Return function name
}

# 5. Memoization
declare -A _memo=()

memoize_fn() {
    local fn="$1"
    shift
    local key="${fn}:$*"
    
    if [[ -n "${_memo[$key]+set}" ]]; then
        echo "${_memo[$key]}"
        return
    fi
    
    local result
    result=$("$fn" "$@")
    _memo[$key]="$result"
    echo "$result"
}

# 6. Lazy Evaluation
lazy_range() {
    local start="$1"
    local end="$2"
    local step="${3:-1}"
    
    for ((i=start; i<=end; i+=step)); do
        echo "$i"
    done
}

take() {
    local n="$1"
    local count=0
    while IFS= read -r line && [[ $count -lt $n ]]; do
        echo "$line"
        ((count++))
    done
}

# 7. Pipeline operations
pipeline() {
    local -a fns=("$@")
    local input
    
    # Read from stdin
    while IFS= read -r line; do
        local result="$line"
        for fn in "${fns[@]}"; do
            result=$("$fn" "$result")
        done
        echo "$result"
    done
}

# ===== Demo =====
echo "=== Functional Programming Demo ==="

# Map
echo ""
echo "map (uppercase):"
map pure_uppercase "hello" "world" "bash"

# Filter
is_even_num() { (( $1 % 2 == 0 )); }
echo ""
echo "filter (even numbers):"
filter is_even_num 1 2 3 4 5 6 7 8 9 10

# Reduce
add_fn() { echo $(( $1 + $2 )); }
echo ""
echo "reduce (sum 1-5):"
reduce add_fn 0 1 2 3 4 5

# Compose
double() { echo $(( $1 * 2 )); }
add_one() { echo $(( $1 + 1 )); }
echo ""
echo "compose (double then add_one of 5):"
double_then_add_one() {
    local x="$1"
    x=$(double "$x")
    x=$(add_one "$x")
    echo "$x"
}
double_then_add_one 5

# Lazy range
echo ""
echo "lazy take 5 from range(1..100):"
lazy_range 1 100 | take 5

# Pipeline
echo ""
echo "pipeline:"
trim_str() { echo "${1// /}"; }
printf '%s\n' "  hello  " "  world  " "  bash  " | \
    while IFS= read -r line; do
        trimmed="${line#"${line%%[! ]*}"}"
        trimmed="${trimmed%"${trimmed##*[! ]}"}"
        echo "${trimmed^^}"
    done
```

---

## ขั้นตอนที่ 444: Event-Driven Architecture

```bash
#!/usr/bin/env bash
# event_driven.sh

# ===== Event Bus =====
declare -A _EVENT_HANDLERS=()
declare -a _EVENT_QUEUE=()
EVENT_LOG="/tmp/events_$$.log"

# Subscribe to event
on() {
    local event="$1"
    local handler="$2"
    
    _EVENT_HANDLERS["$event"]+="${handler}|"
}

# Unsubscribe
off() {
    local event="$1"
    local handler="$2"
    
    _EVENT_HANDLERS["$event"]="${_EVENT_HANDLERS[$event]//$handler|/}"
}

# Emit event immediately
emit() {
    local event="$1"
    shift
    local payload="$*"
    
    # Log event
    echo "$(date +%s):$event:$payload" >> "$EVENT_LOG"
    
    # Get handlers
    local handlers="${_EVENT_HANDLERS[$event]:-}"
    local IFS='|'
    for handler in $handlers; do
        [[ -z "$handler" ]] && continue
        if declare -f "$handler" > /dev/null 2>&1; then
            "$handler" "$event" "$payload"
        fi
    done
}

# Emit async (queue for later)
emit_async() {
    local event="$1"
    shift
    _EVENT_QUEUE+=("$event:$*")
}

# Process event queue
process_events() {
    while [[ ${#_EVENT_QUEUE[@]} -gt 0 ]]; do
        local ev="${_EVENT_QUEUE[0]}"
        _EVENT_QUEUE=("${_EVENT_QUEUE[@]:1}")
        
        local event="${ev%%:*}"
        local payload="${ev#*:}"
        emit "$event" "$payload"
    done
}

# Once - trigger only one time
declare -A _ONCE_FIRED=()
once() {
    local event="$1"
    local handler="$2"
    local once_wrapper="__once_${event//[^a-zA-Z0-9]/_}_${handler}"
    
    eval "${once_wrapper}() {
        off '$event' '$once_wrapper'
        $handler \"\$@\"
    }"
    
    on "$event" "$once_wrapper"
}

# ===== State Machine =====
declare -A _STATE_TRANSITIONS=()
declare -g _CURRENT_STATE=""

state_define() {
    local from="$1"
    local event="$2"
    local to="$3"
    local action="${4:-}"
    
    _STATE_TRANSITIONS["${from}:${event}"]="${to}:${action}"
}

state_init() {
    _CURRENT_STATE="$1"
    echo "Initial state: $_CURRENT_STATE"
}

state_transition() {
    local event="$1"
    shift
    local key="${_CURRENT_STATE}:${event}"
    
    local transition="${_STATE_TRANSITIONS[$key]}"
    if [[ -z "$transition" ]]; then
        echo "Invalid transition: $event in state $_CURRENT_STATE" >&2
        return 1
    fi
    
    local new_state="${transition%%:*}"
    local action="${transition#*:}"
    
    echo "Transition: $_CURRENT_STATE --[$event]--> $new_state"
    
    # Run action
    if [[ -n "$action" ]] && declare -f "$action" > /dev/null 2>&1; then
        "$action" "$@"
    fi
    
    _CURRENT_STATE="$new_state"
    emit "state.changed" "$_CURRENT_STATE"
}

get_state() {
    echo "$_CURRENT_STATE"
}

# ===== Demo =====
echo "=== Event-Driven Demo ==="

# Event bus
echo ""
echo "--- Event Bus ---"

handle_user_login() {
    echo "  Handler: User logged in - $2"
}
handle_audit_login() {
    echo "  Audit: Login at $(date) - $2"
}
handle_welcome() {
    echo "  Welcome email sent to: $2"
}

on "user.login" "handle_user_login"
on "user.login" "handle_audit_login"
once "user.login" "handle_welcome"  # Only fires once

emit "user.login" "alice@example.com"
echo "  Second login:"
emit "user.login" "alice@example.com"  # welcome won't fire again

# State machine
echo ""
echo "--- State Machine (Order) ---"

action_process_payment() { echo "    Processing payment..."; }
action_ship_order() { echo "    Shipping order..."; }
action_cancel() { echo "    Cancelling order..."; }

state_define "pending" "pay" "paid" "action_process_payment"
state_define "paid" "ship" "shipped" "action_ship_order"
state_define "shipped" "deliver" "delivered" ""
state_define "pending" "cancel" "cancelled" "action_cancel"
state_define "paid" "cancel" "cancelled" "action_cancel"

on "state.changed" 'handle_state_change() { echo "  State now: $2"; }'
on "state.changed" "handle_state_change"

state_init "pending"
state_transition "pay"
state_transition "ship"
state_transition "deliver"

rm -f "$EVENT_LOG"
```

---

## ขั้นตอนที่ 445: Plugin System

```bash
#!/usr/bin/env bash
# plugin_system.sh

# ===== Plugin System =====
PLUGIN_DIR="${PLUGIN_DIR:-/tmp/plugins_$$}"
declare -A _LOADED_PLUGINS=()
declare -A _PLUGIN_HOOKS=()

# Plugin API
plugin_register_hook() {
    local hook_name="$1"
    local handler="$2"
    _PLUGIN_HOOKS["$hook_name"]+="${handler} "
    echo "Hook registered: $hook_name -> $handler"
}

plugin_run_hook() {
    local hook_name="$1"
    shift
    local handlers="${_PLUGIN_HOOKS[$hook_name]:-}"
    
    for handler in $handlers; do
        [[ -z "$handler" ]] && continue
        if declare -f "$handler" > /dev/null 2>&1; then
            echo "Running hook $hook_name: $handler"
            "$handler" "$@"
        fi
    done
}

# Load plugin
plugin_load() {
    local plugin_name="$1"
    local plugin_file="${PLUGIN_DIR}/${plugin_name}.sh"
    
    if [[ -n "${_LOADED_PLUGINS[$plugin_name]+set}" ]]; then
        echo "Plugin already loaded: $plugin_name"
        return 0
    fi
    
    if [[ ! -f "$plugin_file" ]]; then
        echo "Plugin not found: $plugin_name" >&2
        return 1
    fi
    
    # Load plugin
    source "$plugin_file"
    
    # Call plugin init if exists
    if declare -f "${plugin_name}_init" > /dev/null 2>&1; then
        "${plugin_name}_init"
    fi
    
    _LOADED_PLUGINS[$plugin_name]=1
    echo "Loaded plugin: $plugin_name"
}

# Unload plugin
plugin_unload() {
    local plugin_name="$1"
    
    if declare -f "${plugin_name}_cleanup" > /dev/null 2>&1; then
        "${plugin_name}_cleanup"
    fi
    
    unset "_LOADED_PLUGINS[$plugin_name]"
    echo "Unloaded: $plugin_name"
}

# List plugins
plugin_list() {
    echo "Loaded plugins:"
    for name in "${!_LOADED_PLUGINS[@]}"; do
        echo "  - $name"
    done
}

# ===== Create Sample Plugins =====
create_sample_plugins() {
    mkdir -p "$PLUGIN_DIR"
    
    # Logger plugin
    cat > "$PLUGIN_DIR/logger.sh" << 'EOF'
#!/usr/bin/env bash
# Logger Plugin

logger_init() {
    echo "  Logger plugin initialized"
    plugin_register_hook "app.start" "logger_on_start"
    plugin_register_hook "app.stop" "logger_on_stop"
    plugin_register_hook "request" "logger_on_request"
}

logger_on_start() {
    echo "  [Logger] Application started at $(date)"
}

logger_on_stop() {
    echo "  [Logger] Application stopped at $(date)"
}

logger_on_request() {
    echo "  [Logger] Request: $*"
}

logger_cleanup() {
    echo "  Logger plugin cleaned up"
}
EOF
    
    # Cache plugin
    cat > "$PLUGIN_DIR/cache.sh" << 'EOF'
#!/usr/bin/env bash
# Cache Plugin

declare -A _CACHE_STORE=()

cache_init() {
    echo "  Cache plugin initialized"
    plugin_register_hook "request" "cache_on_request"
    plugin_register_hook "app.stop" "cache_on_stop"
}

cache_on_request() {
    local key="$1"
    if [[ -n "${_CACHE_STORE[$key]+set}" ]]; then
        echo "  [Cache] HIT: $key"
    else
        echo "  [Cache] MISS: $key"
        _CACHE_STORE[$key]="cached_$(date +%s)"
    fi
}

cache_on_stop() {
    echo "  [Cache] Saved ${#_CACHE_STORE[@]} entries"
}

cache_cleanup() {
    _CACHE_STORE=()
}
EOF
    
    # Auth plugin
    cat > "$PLUGIN_DIR/auth.sh" << 'EOF'
#!/usr/bin/env bash
# Auth Plugin

declare -A _AUTH_TOKENS=()

auth_init() {
    echo "  Auth plugin initialized"
    plugin_register_hook "request" "auth_check"
    # Demo tokens
    _AUTH_TOKENS["token123"]="alice"
    _AUTH_TOKENS["token456"]="bob"
}

auth_check() {
    local request="$1"
    local token
    token=$(echo "$request" | grep -oE 'token=[^ ]+' | cut -d= -f2 || echo "")
    
    if [[ -n "$token" && -n "${_AUTH_TOKENS[$token]+set}" ]]; then
        echo "  [Auth] Authenticated: ${_AUTH_TOKENS[$token]}"
    else
        echo "  [Auth] Anonymous request"
    fi
}
EOF
}

# ===== Demo =====
echo "=== Plugin System Demo ==="

create_sample_plugins

echo ""
echo "Loading plugins..."
plugin_load "logger"
plugin_load "cache"
plugin_load "auth"

echo ""
plugin_list

echo ""
echo "Running hooks:"
plugin_run_hook "app.start"

echo ""
echo "Processing requests:"
plugin_run_hook "request" "/api/users token=token123"
plugin_run_hook "request" "/api/products"
plugin_run_hook "request" "/api/users token=token123"  # cache hit

echo ""
plugin_run_hook "app.stop"

rm -rf "$PLUGIN_DIR"
```

---

## ขั้นตอนที่ 446: DSL (Domain-Specific Language) ใน Bash

```bash
#!/usr/bin/env bash
# dsl_example.sh

# ===== เขียน DSL ใน Bash =====

# ===== HTTP Router DSL =====
declare -A _ROUTES=()
declare -A _MIDDLEWARES=()

# Route definitions
GET() {
    local path="$1"
    local handler="$2"
    _ROUTES["GET:$path"]="$handler"
}

POST() {
    local path="$1"
    local handler="$2"
    _ROUTES["POST:$path"]="$handler"
}

PUT() {
    local path="$1"
    local handler="$2"
    _ROUTES["PUT:$path"]="$handler"
}

DELETE() {
    local path="$1"
    local handler="$2"
    _ROUTES["DELETE:$path"]="$handler"
}

middleware() {
    local name="$1"
    local handler="$2"
    _MIDDLEWARES[$name]="$handler"
}

use() {
    local mw_name="$1"
    local route_handler="$2"
    
    echo "use_${mw_name}_${route_handler}() {
        ${_MIDDLEWARES[$mw_name]} || return 1
        $route_handler \"\$@\"
    }"
}

# Request dispatch
dispatch() {
    local method="$1"
    local path="$2"
    shift 2
    
    local handler="${_ROUTES["$method:$path"]}"
    
    if [[ -z "$handler" ]]; then
        echo "404 Not Found: $method $path"
        return 404
    fi
    
    "$handler" "$@"
}

# ===== Pipeline DSL =====
declare -a _PIPELINE_STEPS=()

pipeline_start() {
    _PIPELINE_STEPS=()
}

step() {
    local name="$1"
    local command="$2"
    _PIPELINE_STEPS+=("$name:$command")
}

pipeline_run() {
    local input="$1"
    local output="$input"
    
    echo "Pipeline starting with: $input"
    for step_def in "${_PIPELINE_STEPS[@]}"; do
        local step_name="${step_def%%:*}"
        local step_cmd="${step_def#*:}"
        
        echo "  [$step_name] processing..."
        output=$(echo "$output" | eval "$step_cmd")
        echo "  [$step_name] result: $output"
    done
    
    echo "Pipeline complete: $output"
}

# ===== Cron DSL =====
declare -a _SCHEDULED_TASKS=()

every() {
    local interval="$1"
    local unit="$2"
    local task_fn="$3"
    
    local seconds
    case "$unit" in
        second|seconds) seconds=$interval ;;
        minute|minutes) seconds=$((interval * 60)) ;;
        hour|hours) seconds=$((interval * 3600)) ;;
        day|days) seconds=$((interval * 86400)) ;;
        *) echo "Unknown unit: $unit" >&2; return 1 ;;
    esac
    
    _SCHEDULED_TASKS+=("$seconds:$task_fn")
    echo "Scheduled: $task_fn every $interval $unit"
}

at() {
    local time="$1"
    local task_fn="$2"
    _SCHEDULED_TASKS+=("at:$time:$task_fn")
    echo "Scheduled: $task_fn at $time"
}

run_scheduler() {
    local duration="${1:-10}"
    local start
    start=$(date +%s)
    local end=$((start + duration))
    
    echo "Scheduler running for ${duration}s..."
    
    while [[ $(date +%s) -lt $end ]]; do
        local now
        now=$(date +%s)
        
        for task in "${_SCHEDULED_TASKS[@]}"; do
            local interval="${task%%:*}"
            local fn="${task#*:}"
            
            if [[ "$interval" =~ ^[0-9]+$ ]]; then
                if (( (now - start) % interval == 0 )); then
                    echo "Running task: $fn"
                    declare -f "$fn" > /dev/null && "$fn"
                fi
            fi
        done
        
        sleep 1
    done
}

# ===== Demo =====
echo "=== DSL Demo ==="

# HTTP Router DSL
echo ""
echo "--- HTTP Router ---"

# Route handlers
handle_users_list() { echo "200 OK: [user1, user2, user3]"; }
handle_users_create() { echo "201 Created: new user"; }
handle_user_get() { echo "200 OK: user profile"; }
handle_user_update() { echo "200 OK: user updated"; }
handle_user_delete() { echo "204 No Content"; }
handle_home() { echo "200 OK: Welcome!"; }

# Define routes
GET "/" handle_home
GET "/api/users" handle_users_list
POST "/api/users" handle_users_create
GET "/api/users/1" handle_user_get
PUT "/api/users/1" handle_user_update
DELETE "/api/users/1" handle_user_delete

echo "Routes defined: ${#_ROUTES[@]}"
echo ""

dispatch "GET" "/"
dispatch "GET" "/api/users"
dispatch "POST" "/api/users"
dispatch "GET" "/api/unknown"

# Pipeline DSL
echo ""
echo "--- Pipeline DSL ---"
pipeline_start
step "trim" "sed 's/^[[:space:]]*//;s/[[:space:]]*$//'"
step "uppercase" "tr '[:lower:]' '[:upper:]'"
step "count_words" "wc -w"
pipeline_run "  hello world from bash  "

# Scheduler DSL
echo ""
echo "--- Scheduler DSL (3s demo) ---"
tick_task() { echo "  Tick: $(date +%H:%M:%S)"; }
every 1 second tick_task
run_scheduler 3
```

---

## ขั้นตอนที่ 447: Microservices Pattern ด้วย Bash

```bash
#!/usr/bin/env bash
# microservices_bash.sh

# ===== Microservice Architecture ใน Bash =====
# สำหรับ automation scripts ขนาดใหญ่

SERVICE_REGISTRY="/tmp/service_registry_$$.json"
SERVICE_LOG="/tmp/services_$$.log"

# ===== Service Registry =====
service_register() {
    local name="$1"
    local port="$2"
    local health_endpoint="${3:-/health}"
    
    echo "Registering service: $name on port $port"
    
    if command -v jq &>/dev/null; then
        local entry
        entry=$(printf '{"name":"%s","port":%s,"health":"%s","registered_at":"%s"}' \
            "$name" "$port" "$health_endpoint" "$(date -u +%Y-%m-%dT%H:%M:%SZ)")
        
        local current="{}"
        [[ -f "$SERVICE_REGISTRY" ]] && current=$(cat "$SERVICE_REGISTRY")
        
        echo "$current" | jq ". + {\"$name\": $entry}" > "$SERVICE_REGISTRY" 2>/dev/null || \
            echo "{\"$name\": $entry}" > "$SERVICE_REGISTRY"
    else
        echo "${name}=${port}:${health_endpoint}" >> "$SERVICE_REGISTRY"
    fi
}

service_discover() {
    local name="$1"
    
    if command -v jq &>/dev/null && [[ -f "$SERVICE_REGISTRY" ]]; then
        jq -r ".\"$name\".port // empty" "$SERVICE_REGISTRY" 2>/dev/null
    else
        grep "^${name}=" "$SERVICE_REGISTRY" 2>/dev/null | cut -d= -f2 | cut -d: -f1
    fi
}

service_list() {
    echo "=== Registered Services ==="
    if command -v jq &>/dev/null && [[ -f "$SERVICE_REGISTRY" ]]; then
        jq -r 'to_entries[] | "  \(.key): port \(.value.port)"' "$SERVICE_REGISTRY" 2>/dev/null
    else
        cat "$SERVICE_REGISTRY" 2>/dev/null
    fi
}

# ===== Service Template =====
create_service() {
    local service_name="$1"
    local service_port="$2"
    
    cat << SCRIPT
#!/usr/bin/env bash
# Microservice: $service_name
SERVICE_NAME="$service_name"
SERVICE_PORT="${service_port}"
SERVICE_LOG="$SERVICE_LOG"

log() { echo "\$(date '+%Y-%m-%d %H:%M:%S') [\$SERVICE_NAME] \$*" | tee -a "\$SERVICE_LOG"; }

handle_request() {
    local path="\$1"
    case "\$path" in
        /health)
            echo '{"status":"healthy","service":"'"\$SERVICE_NAME"'"}'
            ;;
        /metrics)
            echo '{"requests":0,"errors":0}'
            ;;
        *)
            echo '{"error":"not found","path":"'"\$path"'"}'
            ;;
    esac
}

log "Starting \$SERVICE_NAME on port \$SERVICE_PORT"
SCRIPT
}

# ===== Service Communication =====
call_service() {
    local service="$1"
    local path="${2:-/}"
    local method="${3:-GET}"
    local body="${4:-}"
    
    local port
    port=$(service_discover "$service")
    
    if [[ -z "$port" ]]; then
        echo "Service not found: $service" >&2
        return 1
    fi
    
    local url="http://localhost:${port}${path}"
    echo "Calling $service: $method $url"
    
    if command -v curl &>/dev/null; then
        if [[ -n "$body" ]]; then
            curl -s -X "$method" -H "Content-Type: application/json" \
                -d "$body" "$url" 2>/dev/null
        else
            curl -s -X "$method" "$url" 2>/dev/null
        fi
    else
        echo '{"error":"curl not available"}'
    fi
}

# ===== Circuit Breaker =====
declare -A _CIRCUIT_STATE=()
declare -A _CIRCUIT_FAILURES=()
declare -A _CIRCUIT_LAST_FAILURE=()

CIRCUIT_THRESHOLD=3
CIRCUIT_TIMEOUT=30

circuit_call() {
    local service="$1"
    shift
    
    local state="${_CIRCUIT_STATE[$service]:-closed}"
    
    case "$state" in
        open)
            local last_failure="${_CIRCUIT_LAST_FAILURE[$service]:-0}"
            local now
            now=$(date +%s)
            
            if (( now - last_failure > CIRCUIT_TIMEOUT )); then
                _CIRCUIT_STATE[$service]="half-open"
                echo "Circuit half-open for $service"
            else
                echo "Circuit OPEN: $service unavailable" >&2
                return 1
            fi
            ;;
    esac
    
    if call_service "$service" "$@"; then
        if [[ "$state" == "half-open" ]]; then
            _CIRCUIT_STATE[$service]="closed"
            _CIRCUIT_FAILURES[$service]=0
            echo "Circuit closed for $service"
        fi
        return 0
    else
        local failures=$(( ${_CIRCUIT_FAILURES[$service]:-0} + 1 ))
        _CIRCUIT_FAILURES[$service]=$failures
        _CIRCUIT_LAST_FAILURE[$service]=$(date +%s)
        
        if [[ $failures -ge $CIRCUIT_THRESHOLD ]]; then
            _CIRCUIT_STATE[$service]="open"
            echo "Circuit OPENED for $service after $failures failures"
        fi
        return 1
    fi
}

# ===== Demo =====
echo "=== Microservices Demo ==="

# Register services
echo ""
service_register "user-service" 3001 "/health"
service_register "product-service" 3002 "/health"
service_register "order-service" 3003 "/health"
service_register "payment-service" 3004 "/health"

echo ""
service_list

echo ""
echo "Service discovery:"
echo "user-service port: $(service_discover user-service)"
echo "product-service port: $(service_discover product-service)"

echo ""
echo "Service template:"
create_service "notification-service" "3005"

# Cleanup
rm -f "$SERVICE_REGISTRY" "$SERVICE_LOG"
```

---

## สรุป Module 2: Intermediate Level (Parts 11-25)

### ทั้งหมด 144 Steps (291-434+):

| Parts | หัวข้อ | Steps |
|-------|--------|-------|
| 11 | Regular Expressions | 291-312 |
| 12 | sed Stream Editor | 313-330 |
| 13 | awk | 331-346 |
| 14 | Process Management | 347-361 |
| 15 | Networking & APIs | 362-373 |
| 16 | Git Automation | 374-380 |
| 17 | Docker Integration | 381-387 |
| 18 | System Administration | 388-395 |
| 19 | Security Scripts | 396-403 |
| 20 | Performance Tuning | 404-415 |
| 21 | Testing & QA | 416-424 |
| 22 | Logging & Monitoring | 425-430 |
| 23 | Configuration Mgmt | 431-436 |
| 24 | Database Operations | 437-441 |
| 25 | Advanced Patterns | 442-455 |

### ก้าวหน้าสู่ Module 3 Advanced:
- Kubernetes & Container Orchestration
- Cloud Provider Integration (AWS, GCP, Azure)
- CI/CD Pipeline Automation
- Infrastructure as Code
- Advanced Security

**ขั้นตอนต่อไป**: Part 26 - Module 3 Advanced: Kubernetes Integration
