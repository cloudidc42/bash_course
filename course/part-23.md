# Part 23: Configuration Management

## Module 2: Intermediate Level
### ขั้นตอนที่ 431-440: การจัดการ Configuration

---

## ขั้นตอนที่ 431: Configuration File Formats

```bash
#!/usr/bin/env bash
# config_formats.sh

# ===== รองรับหลาย Config Formats =====

# 1. Key=Value Format (.env / .conf)
parse_env_file() {
    local file="$1"
    
    while IFS='=' read -r key value || [[ -n "$key" ]]; do
        # Skip comments and empty lines
        [[ "$key" =~ ^[[:space:]]*# ]] && continue
        [[ -z "${key// /}" ]] && continue
        
        # Trim whitespace
        key="${key#"${key%%[! ]*}"}"
        key="${key%"${key##*[! ]}"}"
        value="${value#"${value%%[! ]*}"}"
        value="${value%"${value##*[! ]}"}"
        
        # Remove quotes
        value="${value#\"}" ; value="${value%\"}"
        value="${value#\'}" ; value="${value%\'}"
        
        # Export
        export "$key=$value"
    done < "$file"
}

# 2. INI Format
declare -A INI_CONFIG=()

parse_ini_file() {
    local file="$1"
    local current_section="default"
    
    while IFS= read -r line || [[ -n "$line" ]]; do
        # Skip comments
        [[ "$line" =~ ^[[:space:]]*[;#] ]] && continue
        [[ -z "${line// /}" ]] && continue
        
        # Section header
        if [[ "$line" =~ ^\[([^\]]+)\] ]]; then
            current_section="${BASH_REMATCH[1]}"
            continue
        fi
        
        # Key = Value
        if [[ "$line" =~ ^[[:space:]]*([^=[:space:]]+)[[:space:]]*=[[:space:]]*(.*) ]]; then
            local key="${BASH_REMATCH[1]}"
            local value="${BASH_REMATCH[2]}"
            value="${value%"${value##*[! ]}"}"  # trim trailing
            INI_CONFIG["${current_section}.${key}"]="$value"
        fi
    done < "$file"
}

ini_get() {
    local section="$1"
    local key="$2"
    local default="${3:-}"
    echo "${INI_CONFIG["${section}.${key}"]:-$default}"
}

# 3. YAML (simple subset)
declare -A YAML_CONFIG=()

parse_simple_yaml() {
    local file="$1"
    local prefix="${2:-}"
    local current_key=""
    
    while IFS= read -r line; do
        # Skip comments
        [[ "$line" =~ ^[[:space:]]*# ]] && continue
        [[ -z "${line// /}" ]] && continue
        
        # Detect indentation level
        local stripped="${line#"${line%%[! ]*}"}"
        local indent=$(( ${#line} - ${#stripped} ))
        
        # Key: value
        if [[ "$line" =~ ^[[:space:]]*([^:]+):[[:space:]]*(.*) ]]; then
            local key="${BASH_REMATCH[1]}"
            local value="${BASH_REMATCH[2]}"
            key="${key// /}"
            value="${value%"${value##*[! ]}"}"
            
            if [[ -n "$value" ]]; then
                local full_key="${prefix:+${prefix}_}${key}"
                full_key="${full_key//[.-]/_}"
                YAML_CONFIG["$full_key"]="$value"
            fi
        fi
    done < "$file"
}

yaml_get() {
    local key="${1//./_}"
    local default="${2:-}"
    echo "${YAML_CONFIG[$key]:-$default}"
}

# 4. JSON (ด้วย jq)
parse_json_config() {
    local file="$1"
    
    if ! command -v jq &>/dev/null; then
        echo "jq not available" >&2
        return 1
    fi
    
    # อ่านค่าทั้งหมด
    jq -r 'to_entries | .[] | "\(.key)=\(.value)"' "$file" 2>/dev/null
}

json_get() {
    local file="$1"
    local path="$2"
    local default="${3:-}"
    
    if command -v jq &>/dev/null; then
        jq -r "$path // \"$default\"" "$file" 2>/dev/null
    else
        echo "$default"
    fi
}

# ===== Demo =====
echo "=== Config Format Demo ==="

# สร้าง config files
ENV_FILE="/tmp/test.env"
INI_FILE="/tmp/test.ini"
YAML_FILE="/tmp/test.yaml"

cat > "$ENV_FILE" << 'EOF'
# Application config
APP_NAME=MyApp
APP_PORT=8080
DB_HOST=localhost
DB_PORT=5432
DB_USER=admin
LOG_LEVEL=info
EOF

cat > "$INI_FILE" << 'EOF'
[app]
name = MyApp
port = 8080

[database]
host = localhost
port = 5432
name = mydb

[logging]
level = info
file = /var/log/app.log
EOF

cat > "$YAML_FILE" << 'EOF'
# App config
app_name: MyApp
app_port: 8080
database_host: localhost
database_port: 5432
log_level: info
EOF

# Parse and use
echo "--- .env format ---"
parse_env_file "$ENV_FILE"
echo "APP_NAME=$APP_NAME"
echo "DB_HOST=$DB_HOST"

echo ""
echo "--- INI format ---"
parse_ini_file "$INI_FILE"
echo "app.name=$(ini_get app name)"
echo "database.host=$(ini_get database host)"
echo "logging.level=$(ini_get logging level)"

echo ""
echo "--- YAML format ---"
parse_simple_yaml "$YAML_FILE"
echo "app_name=$(yaml_get app_name)"
echo "database_host=$(yaml_get database_host)"

rm -f "$ENV_FILE" "$INI_FILE" "$YAML_FILE"
```

---

## ขั้นตอนที่ 432: Configuration Hierarchy และ Overrides

```bash
#!/usr/bin/env bash
# config_hierarchy.sh

# ===== Configuration Hierarchy =====
# Priority (highest to lowest):
# 1. Command line arguments
# 2. Environment variables
# 3. User config file (~/.config/app/config)
# 4. Project config (./.apprc)
# 5. System config (/etc/app/config)
# 6. Default values

declare -A CONFIG=()

# Default values
set_defaults() {
    CONFIG[app_name]="MyApp"
    CONFIG[app_port]="8080"
    CONFIG[log_level]="info"
    CONFIG[db_host]="localhost"
    CONFIG[db_port]="5432"
    CONFIG[db_name]="app"
    CONFIG[max_connections]="10"
    CONFIG[timeout]="30"
}

# Load config from file
load_config_file() {
    local file="$1"
    local prefix="${2:-}"
    
    [[ -f "$file" ]] || return 0
    
    while IFS='=' read -r key value || [[ -n "$key" ]]; do
        [[ "$key" =~ ^[[:space:]]*# ]] && continue
        [[ -z "${key// /}" ]] && continue
        
        key="${key// /}"
        key="${key,,}"  # lowercase
        key="${key//-/_}"  # normalize
        
        value="${value#"${value%%[! ]*}"}"
        value="${value%"${value##*[! ]}"}"
        value="${value#\"}" ; value="${value%\"}"
        
        if [[ -n "$prefix" ]]; then
            CONFIG["${prefix}_${key}"]="$value"
        else
            CONFIG["$key"]="$value"
        fi
    done < "$file"
    
    echo "Loaded config: $file"
}

# Load from environment variables
load_env_vars() {
    local prefix="${1:-APP}"
    
    while IFS='=' read -r key value; do
        if [[ "$key" =~ ^${prefix}_ ]]; then
            local config_key="${key#${prefix}_}"
            config_key="${config_key,,}"
            CONFIG["$config_key"]="$value"
        fi
    done < <(env)
}

# Load from command line args
load_cli_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            --*=*)
                local key="${1%%=*}"
                local value="${1#*=}"
                key="${key#--}"
                key="${key//-/_}"
                CONFIG["$key"]="$value"
                shift
                ;;
            --*)
                local key="${1#--}"
                key="${key//-/_}"
                if [[ $# -gt 1 && "${2:-}" != --* ]]; then
                    CONFIG["$key"]="$2"
                    shift 2
                else
                    CONFIG["$key"]="true"
                    shift
                fi
                ;;
            *)
                shift
                ;;
        esac
    done
}

# Get config value
config_get() {
    local key="$1"
    local default="${2:-}"
    echo "${CONFIG[$key]:-$default}"
}

# Set config value
config_set() {
    local key="$1"
    local value="$2"
    CONFIG["$key"]="$value"
}

# Validate required config
config_require() {
    local missing=()
    for key in "$@"; do
        if [[ -z "${CONFIG[$key]+set}" ]]; then
            missing+=("$key")
        fi
    done
    
    if [[ ${#missing[@]} -gt 0 ]]; then
        echo "Error: Missing required config: ${missing[*]}" >&2
        return 1
    fi
}

# Print all config
config_dump() {
    echo "=== Configuration ==="
    for key in $(echo "${!CONFIG[@]}" | tr ' ' '\n' | sort); do
        local value="${CONFIG[$key]}"
        # Mask sensitive values
        if [[ "$key" =~ (password|secret|token|key|credentials) ]]; then
            value="***"
        fi
        printf "  %-30s = %s\n" "$key" "$value"
    done
}

# Config validation
config_validate() {
    local errors=0
    
    # Port range check
    local port
    port=$(config_get "app_port")
    if [[ -n "$port" ]] && ! [[ "$port" =~ ^[0-9]+$ ]] || \
       [[ -n "$port" ]] && (( port < 1 || port > 65535 )); then
        echo "Error: Invalid port: $port" >&2
        ((errors++))
    fi
    
    # Log level check
    local log_level
    log_level=$(config_get "log_level")
    if [[ -n "$log_level" ]] && ! [[ "$log_level" =~ ^(debug|info|warn|error|fatal)$ ]]; then
        echo "Error: Invalid log_level: $log_level" >&2
        ((errors++))
    fi
    
    return $errors
}

# ===== Demo =====

# สร้าง config files
mkdir -p /tmp/config_demo
cat > /tmp/config_demo/system.conf << 'EOF'
APP_PORT=8080
LOG_LEVEL=warn
DB_HOST=prod-db.example.com
EOF

cat > /tmp/config_demo/user.conf << 'EOF'
LOG_LEVEL=debug
DB_HOST=dev-db.local
APP_NAME=DevApp
EOF

echo "=== Configuration Hierarchy Demo ==="

# Load in order (lowest to highest priority)
set_defaults
load_config_file "/tmp/config_demo/system.conf"
load_config_file "/tmp/config_demo/user.conf"

# Simulate env vars
export APP_PORT=9090
export APP_DB_HOST=env-db.local
load_env_vars "APP"

# Simulate CLI args
load_cli_args --log-level=info --app-name=CLIApp

echo ""
config_dump

echo ""
echo "Validation:"
config_validate && echo "Config valid" || echo "Config has errors"

rm -rf /tmp/config_demo
```

---

## ขั้นตอนที่ 433: Dynamic Configuration Reload

```bash
#!/usr/bin/env bash
# dynamic_config_reload.sh

# ===== Hot Reload Configuration =====
CONFIG_FILE="/tmp/dynamic_config_$$.conf"
CONFIG_HASH=""
CONFIG_RELOAD_INTERVAL=5

declare -A CURRENT_CONFIG=()

# Write initial config
cat > "$CONFIG_FILE" << 'EOF'
app_port=8080
log_level=info
feature_x=disabled
max_workers=4
EOF

# Calculate file hash
file_hash() {
    md5sum "$1" 2>/dev/null | cut -d' ' -f1 || \
    sha1sum "$1" 2>/dev/null | cut -d' ' -f1 || \
    stat -c%Y "$1" 2>/dev/null
}

# Load config
load_config() {
    local file="$1"
    
    while IFS='=' read -r key value || [[ -n "$key" ]]; do
        [[ "$key" =~ ^# ]] && continue
        [[ -z "${key// /}" ]] && continue
        key="${key// /}"
        value="${value// /}"
        CURRENT_CONFIG["$key"]="$value"
    done < "$file"
    
    CONFIG_HASH=$(file_hash "$file")
    echo "Config loaded: $file (${#CURRENT_CONFIG[@]} settings)"
}

# Check and reload if changed
check_reload() {
    local current_hash
    current_hash=$(file_hash "$CONFIG_FILE")
    
    if [[ "$current_hash" != "$CONFIG_HASH" ]]; then
        echo "Config changed, reloading..."
        local old_port="${CURRENT_CONFIG[app_port]}"
        load_config "$CONFIG_FILE"
        
        # Notify about changes
        for key in "${!CURRENT_CONFIG[@]}"; do
            # Would compare old vs new here
            :
        done
        
        echo "Config reloaded"
        return 0
    fi
    return 1
}

# Watch config with SIGHUP
setup_config_reload() {
    trap 'echo "SIGHUP received, reloading config..."; load_config "$CONFIG_FILE"' SIGHUP
    echo "Config reload on SIGHUP enabled (PID: $$)"
    echo "Reload with: kill -HUP $$"
}

# inotify-based watching (if available)
watch_config_inotify() {
    if ! command -v inotifywait &>/dev/null; then
        echo "inotifywait not available, using polling"
        return 1
    fi
    
    echo "Watching $CONFIG_FILE with inotify..."
    inotifywait -m -e modify "$CONFIG_FILE" 2>/dev/null | while read -r; do
        echo "File changed, reloading..."
        load_config "$CONFIG_FILE"
    done &
    
    echo "Watcher PID: $!"
}

# Polling fallback
poll_config_changes() {
    echo "Polling for config changes every ${CONFIG_RELOAD_INTERVAL}s..."
    
    while true; do
        sleep "$CONFIG_RELOAD_INTERVAL"
        check_reload
    done
}

# ===== Demo =====
echo "=== Dynamic Config Reload Demo ==="

load_config "$CONFIG_FILE"
echo ""
echo "Current config:"
for key in "${!CURRENT_CONFIG[@]}"; do
    echo "  $key=${CURRENT_CONFIG[$key]}"
done

# Modify config
echo ""
echo "Modifying config file..."
cat > "$CONFIG_FILE" << 'EOF'
app_port=9090
log_level=debug
feature_x=enabled
max_workers=8
new_setting=added
EOF

sleep 0.5
check_reload && {
    echo ""
    echo "Updated config:"
    for key in "${!CURRENT_CONFIG[@]}"; do
        echo "  $key=${CURRENT_CONFIG[$key]}"
    done
}

setup_config_reload

rm -f "$CONFIG_FILE"
```

---

## ขั้นตอนที่ 434: Secret Management

```bash
#!/usr/bin/env bash
# secret_management.sh

# ===== Secret Management =====
# ห้าม hardcode secrets ใน scripts

# 1. Environment variables (recommended)
get_secret_from_env() {
    local key="$1"
    local value="${!key}"
    
    if [[ -z "$value" ]]; then
        echo "Error: Secret '$key' not set in environment" >&2
        return 1
    fi
    
    echo "$value"
}

# 2. Secret files ด้วย permissions
SECRET_DIR="/run/secrets"  # Docker secrets location

get_secret_from_file() {
    local secret_name="$1"
    local secret_file="${SECRET_DIR}/${secret_name}"
    
    # Try common locations
    for dir in "$SECRET_DIR" "/etc/secrets" "/tmp/secrets_$$"; do
        if [[ -f "${dir}/${secret_name}" ]]; then
            # ตรวจสอบ permissions (ควรเป็น 400 หรือ 600)
            local perms
            perms=$(stat -c "%a" "${dir}/${secret_name}" 2>/dev/null)
            if [[ "$perms" == "400" || "$perms" == "600" ]]; then
                cat "${dir}/${secret_name}"
                return 0
            else
                echo "Warning: Secret file has insecure permissions: $perms" >&2
                cat "${dir}/${secret_name}"
                return 0
            fi
        fi
    done
    
    echo "Secret not found: $secret_name" >&2
    return 1
}

# 3. GPG encrypted secrets
get_secret_from_gpg() {
    local secret_file="$1"
    
    if ! command -v gpg &>/dev/null; then
        echo "GPG not available" >&2
        return 1
    fi
    
    gpg --quiet --batch --decrypt "$secret_file" 2>/dev/null
}

# 4. Keyring (desktop)
get_secret_from_keyring() {
    local service="$1"
    local account="$2"
    
    if command -v secret-tool &>/dev/null; then
        secret-tool lookup service "$service" account "$account" 2>/dev/null
    elif command -v keychain &>/dev/null; then
        keychain --eval "$service" 2>/dev/null
    else
        echo "No keyring available" >&2
        return 1
    fi
}

# 5. HashiCorp Vault
get_secret_from_vault() {
    local path="$1"
    local key="${2:-value}"
    
    if ! command -v vault &>/dev/null; then
        echo "Vault CLI not available" >&2
        return 1
    fi
    
    vault kv get -field="$key" "$path" 2>/dev/null
}

# Secure credential prompting
prompt_secret() {
    local prompt="$1"
    local var_name="$2"
    
    # Read without echo
    local secret
    read -r -s -p "$prompt" secret
    echo ""  # newline after silent input
    
    # Assign to variable
    eval "${var_name}='${secret}'"
    
    # Wipe from history
    history -d $(history 1 | awk '{print $1}') 2>/dev/null || true
}

# ===== Secure Credential Store =====
CRED_STORE="/tmp/.secure_store_$$"
CRED_KEY=""

init_cred_store() {
    local password="$1"
    CRED_KEY="$password"
    mkdir -p "$CRED_STORE"
    chmod 700 "$CRED_STORE"
}

store_credential() {
    local name="$1"
    local value="$2"
    
    if [[ -z "$CRED_KEY" ]]; then
        echo "Credential store not initialized" >&2
        return 1
    fi
    
    if command -v openssl &>/dev/null; then
        echo "$value" | openssl enc -aes-256-cbc -pbkdf2 -pass "pass:${CRED_KEY}" \
            -out "${CRED_STORE}/${name}.enc" 2>/dev/null
        echo "Stored: $name"
    else
        # Fallback: base64 (not secure, just for demo)
        echo "$value" | base64 > "${CRED_STORE}/${name}.b64"
        echo "Stored: $name (base64 only - not secure!)"
    fi
}

retrieve_credential() {
    local name="$1"
    
    if [[ -f "${CRED_STORE}/${name}.enc" ]]; then
        openssl enc -d -aes-256-cbc -pbkdf2 -pass "pass:${CRED_KEY}" \
            -in "${CRED_STORE}/${name}.enc" 2>/dev/null
    elif [[ -f "${CRED_STORE}/${name}.b64" ]]; then
        base64 -d "${CRED_STORE}/${name}.b64"
    else
        echo "Credential not found: $name" >&2
        return 1
    fi
}

cleanup_cred_store() {
    if [[ -d "$CRED_STORE" ]]; then
        # Securely wipe
        find "$CRED_STORE" -type f -exec shred -u {} \; 2>/dev/null || \
        find "$CRED_STORE" -type f -exec rm -f {} \;
        rmdir "$CRED_STORE"
    fi
    CRED_KEY=""
}

# Redact secrets from output
redact_secrets() {
    local output="$1"
    local -a secrets=("${@:2}")
    
    for secret in "${secrets[@]}"; do
        [[ -z "$secret" ]] && continue
        output="${output//$secret/***REDACTED***}"
    done
    
    echo "$output"
}

# ===== Demo =====
echo "=== Secret Management Demo ==="

# Simulate env var secret
export DB_PASSWORD="super_secret_pass"
echo "From env: $(get_secret_from_env DB_PASSWORD)"

# Credential store
init_cred_store "master_password_123"
store_credential "api_key" "abc123xyz789"
store_credential "db_pass" "mydbpassword"

echo ""
echo "Retrieved api_key: $(retrieve_credential api_key)"

# Redaction
output="Connecting to db with password=mydbpassword and api_key=abc123xyz789"
echo ""
echo "Original: $output"
echo "Redacted: $(redact_secrets "$output" "mydbpassword" "abc123xyz789")"

cleanup_cred_store
```

---

## ขั้นตอนที่ 435: Template Engine

```bash
#!/usr/bin/env bash
# template_engine.sh

# ===== Template Engine =====

# Simple variable substitution
render_template_simple() {
    local template="$1"
    shift
    
    # รับ variable=value pairs
    local content="$template"
    while [[ $# -gt 0 ]]; do
        local key="${1%%=*}"
        local value="${1#*=}"
        content="${content//\{\{$key\}\}/$value}"
        shift
    done
    
    echo "$content"
}

# Template with environment variables
render_template_env() {
    local template_file="$1"
    
    # Replace {{VAR_NAME}} with env value
    local content
    content=$(< "$template_file")
    
    # Find all {{...}} placeholders
    while [[ "$content" =~ \{\{([A-Z_]+)\}\} ]]; do
        local var="${BASH_REMATCH[1]}"
        local value="${!var:-}"
        content="${content/\{\{$var\}\}/$value}"
    done
    
    echo "$content"
}

# Full template engine with logic
render_template() {
    local template="$1"
    declare -A vars
    
    # Parse vars from remaining args
    shift
    while [[ $# -gt 0 ]]; do
        local kv="$1"
        vars["${kv%%=*}"]="${kv#*=}"
        shift
    done
    
    local output=""
    local in_if=false
    local if_condition=false
    local in_for=false
    local for_var=""
    local for_items=()
    local for_body=""
    
    while IFS= read -r line; do
        # Process directives
        if [[ "$line" =~ ^\{\%[[:space:]]*if[[:space:]]+(.+)[[:space:]]*%\} ]]; then
            local condition="${BASH_REMATCH[1]}"
            # Evaluate condition
            local cond_var="${condition// /}"
            if [[ "${vars[$cond_var]:-false}" == "true" || "${vars[$cond_var]:-}" == "1" ]]; then
                if_condition=true
            else
                if_condition=false
            fi
            in_if=true
            continue
        fi
        
        if [[ "$line" =~ ^\{\%[[:space:]]*endif[[:space:]]*%\} ]]; then
            in_if=false
            if_condition=false
            continue
        fi
        
        if [[ "$line" =~ ^\{\%[[:space:]]*for[[:space:]]+([a-z_]+)[[:space:]]+in[[:space:]]+(.+)[[:space:]]*%\} ]]; then
            for_var="${BASH_REMATCH[1]}"
            IFS=',' read -ra for_items <<< "${BASH_REMATCH[2]}"
            in_for=true
            for_body=""
            continue
        fi
        
        if [[ "$line" =~ ^\{\%[[:space:]]*endfor[[:space:]]*%\} ]]; then
            in_for=false
            # Render loop
            for item in "${for_items[@]}"; do
                item="${item// /}"
                local rendered_line="${for_body//\{\{$for_var\}\}/$item}"
                output+="$rendered_line"$'\n'
            done
            for_body=""
            continue
        fi
        
        # Collect for body
        if $in_for; then
            for_body+="$line"$'\n'
            continue
        fi
        
        # Skip if block content when condition is false
        if $in_if && ! $if_condition; then
            continue
        fi
        
        # Variable substitution
        local processed_line="$line"
        for var in "${!vars[@]}"; do
            processed_line="${processed_line//\{\{$var\}\}/${vars[$var]}}"
        done
        
        output+="$processed_line"$'\n'
    done <<< "$template"
    
    echo "$output"
}

# Generate files from templates
template_to_file() {
    local template_file="$1"
    local output_file="$2"
    shift 2
    
    if [[ ! -f "$template_file" ]]; then
        echo "Template not found: $template_file" >&2
        return 1
    fi
    
    local content
    content=$(< "$template_file")
    render_template "$content" "$@" > "$output_file"
    echo "Generated: $output_file"
}

# ===== Demo =====
echo "=== Template Engine Demo ==="

# Simple substitution
template="Hello, {{name}}! Welcome to {{app_name}}."
rendered=$(render_template_simple "$template" "name=Alice" "app_name=MyApp")
echo "Simple: $rendered"

# Nginx config template
nginx_template='/tmp/nginx_template_$$.conf'
cat > "/tmp/nginx_template_$$.conf" << 'TMPL'
server {
    listen {{PORT}};
    server_name {{SERVER_NAME}};
    
    location / {
        proxy_pass http://{{BACKEND_HOST}}:{{BACKEND_PORT}};
    }
    
    access_log {{LOG_DIR}}/access.log;
    error_log {{LOG_DIR}}/error.log;
}
TMPL

export PORT=80
export SERVER_NAME=myapp.example.com
export BACKEND_HOST=127.0.0.1
export BACKEND_PORT=3000
export LOG_DIR=/var/log/nginx

echo ""
echo "Nginx Config (from template):"
render_template_env "/tmp/nginx_template_$$.conf"

# Template with conditionals and loops
complex_template='
App: {{app_name}} v{{version}}
Environment: {{env}}

{% if debug %}
Debug mode is ON
{% endif %}

Features:
{% for feature in auth,logging,cache,monitoring %}
  - {{feature}}
{% endfor %}
'

echo ""
echo "Complex template:"
render_template "$complex_template" \
    "app_name=MyApp" \
    "version=1.2.3" \
    "env=production" \
    "debug=false"

rm -f "/tmp/nginx_template_$$.conf"
```

---

## ขั้นตอนที่ 436: Environment Management

```bash
#!/usr/bin/env bash
# environment_management.sh

# ===== Multi-Environment Support =====
ENVIRONMENTS=(development staging production)
CURRENT_ENV="${APP_ENV:-development}"

# Validate environment
validate_env() {
    local env="$1"
    for valid in "${ENVIRONMENTS[@]}"; do
        [[ "$env" == "$valid" ]] && return 0
    done
    echo "Invalid environment: $env (valid: ${ENVIRONMENTS[*]})" >&2
    return 1
}

# สร้าง environment configs
create_env_configs() {
    mkdir -p /tmp/env_demo/{development,staging,production}
    
    cat > /tmp/env_demo/development/config.conf << 'EOF'
APP_DEBUG=true
DB_HOST=localhost
DB_PORT=5432
DB_NAME=app_dev
REDIS_HOST=localhost
LOG_LEVEL=debug
API_URL=http://localhost:3000
FEATURE_BETA=true
EOF
    
    cat > /tmp/env_demo/staging/config.conf << 'EOF'
APP_DEBUG=false
DB_HOST=staging-db.internal
DB_PORT=5432
DB_NAME=app_staging
REDIS_HOST=staging-redis.internal
LOG_LEVEL=info
API_URL=https://staging.example.com
FEATURE_BETA=true
EOF
    
    cat > /tmp/env_demo/production/config.conf << 'EOF'
APP_DEBUG=false
DB_HOST=prod-db.internal
DB_PORT=5432
DB_NAME=app_prod
REDIS_HOST=prod-redis.internal
LOG_LEVEL=warn
API_URL=https://www.example.com
FEATURE_BETA=false
EOF
}

# Load environment
load_environment() {
    local env="${1:-$CURRENT_ENV}"
    local env_dir="/tmp/env_demo/${env}"
    
    validate_env "$env" || return 1
    
    if [[ ! -d "$env_dir" ]]; then
        echo "Environment config not found: $env_dir" >&2
        return 1
    fi
    
    # Load base config first
    if [[ -f "$env_dir/config.conf" ]]; then
        while IFS='=' read -r key value || [[ -n "$key" ]]; do
            [[ "$key" =~ ^[[:space:]]*# ]] && continue
            [[ -z "${key// /}" ]] && continue
            key="${key// /}"
            value="${value// /}"
            export "${key}=${value}"
        done < "$env_dir/config.conf"
    fi
    
    export APP_ENV="$env"
    echo "Environment loaded: $env"
}

# Environment-specific behavior
env_run() {
    local env="${APP_ENV:-development}"
    
    case "$env" in
        development)
            echo "DEV: Verbose output enabled"
            echo "DEV: Using local services"
            ;;
        staging)
            echo "STAGING: Using staging services"
            echo "STAGING: Performance monitoring enabled"
            ;;
        production)
            echo "PROD: Using production services"
            echo "PROD: All safeguards active"
            ;;
    esac
}

# Environment comparison
compare_envs() {
    local env1="$1"
    local env2="$2"
    
    echo "=== Comparing $env1 vs $env2 ==="
    
    declare -A config1 config2
    
    while IFS='=' read -r k v; do
        [[ "$k" =~ ^# ]] && continue
        [[ -z "${k// /}" ]] && continue
        config1["${k// /}"]="${v// /}"
    done < "/tmp/env_demo/$env1/config.conf"
    
    while IFS='=' read -r k v; do
        [[ "$k" =~ ^# ]] && continue
        [[ -z "${k// /}" ]] && continue
        config2["${k// /}"]="${v// /}"
    done < "/tmp/env_demo/$env2/config.conf"
    
    # Keys in both
    local all_keys=()
    for k in "${!config1[@]}" "${!config2[@]}"; do
        all_keys+=("$k")
    done
    
    while IFS= read -r key; do
        local v1="${config1[$key]:-<not set>}"
        local v2="${config2[$key]:-<not set>}"
        
        if [[ "$v1" != "$v2" ]]; then
            printf "  %-25s  %-20s vs %-20s\n" "$key" "$v1" "$v2"
        fi
    done < <(printf '%s\n' "${all_keys[@]}" | sort -u)
}

# ===== Demo =====
echo "=== Environment Management Demo ==="

create_env_configs

echo ""
echo "Loading development environment:"
load_environment development
echo "DB_HOST=$DB_HOST, LOG_LEVEL=$LOG_LEVEL"

echo ""
echo "Loading production environment:"
load_environment production
echo "DB_HOST=$DB_HOST, LOG_LEVEL=$LOG_LEVEL"

echo ""
compare_envs development production

echo ""
load_environment development
env_run

rm -rf /tmp/env_demo
```

---

## สรุป Part 23

### สิ่งที่เรียนรู้ (Steps 431-436):

| Step | หัวข้อ | เทคนิค |
|------|--------|--------|
| 431 | Config Formats | .env, INI, YAML, JSON parsing |
| 432 | Config Hierarchy | Priority layers, CLI > ENV > File > Default |
| 433 | Dynamic Reload | SIGHUP, inotify, polling |
| 434 | Secret Management | env vars, GPG, keyring, Vault |
| 435 | Template Engine | variable substitution, if/for |
| 436 | Environment Mgmt | multi-env, comparison, loading |

### Best Practices:
- ไม่ hardcode sensitive values ใน script
- ใช้ hierarchy: CLI > ENV > File > Default
- Validate config เมื่อ startup
- Support hot reload สำหรับ long-running scripts
- ใช้ templates แทนการ hardcode configs

**ขั้นตอนต่อไป**: Part 24 - Database Operations
