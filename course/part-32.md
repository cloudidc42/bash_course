# Part 32: Advanced Security และ Compliance

## Module 3: Advanced Level — การทำงานระดับสูง

---

## ขั้นตอนที่ 487: Security Scanning และ Vulnerability Assessment

```bash
#!/bin/bash
# security_scanner.sh - ตรวจสอบความปลอดภัยแบบครบวงจร

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
NC='\033[0m'

SCAN_REPORT="/tmp/security_scan_$$.json"
declare -A FINDINGS
SEVERITY_COUNTS=([critical]=0 [high]=0 [medium]=0 [low]=0 [info]=0)

# เพิ่ม finding
add_finding() {
    local severity="$1"
    local category="$2"
    local title="$3"
    local description="$4"
    local remediation="$5"
    
    ((SEVERITY_COUNTS[$severity]++))
    
    local key="${severity}_$((SEVERITY_COUNTS[$severity]))"
    FINDINGS[$key]=$(cat << EOF
{
    "severity": "${severity}",
    "category": "${category}",
    "title": "${title}",
    "description": "${description}",
    "remediation": "${remediation}"
}
EOF
)
    
    local icon
    case "$severity" in
        critical) icon="${RED}🔴 CRITICAL${NC}" ;;
        high)     icon="${RED}🔴 HIGH${NC}" ;;
        medium)   icon="${YELLOW}🟡 MEDIUM${NC}" ;;
        low)      icon="${BLUE}🔵 LOW${NC}" ;;
        info)     icon="${GREEN}ℹ️  INFO${NC}" ;;
    esac
    
    echo -e "  ${icon}: ${title}"
    echo "    ${description}"
}

# === OS Security Checks ===

scan_os_security() {
    echo -e "\n${BLUE}=== OS Security Scan ===${NC}"
    
    # ตรวจสอบ sudo configuration
    echo "Checking sudo configuration..."
    if grep -qE "^[^#].*NOPASSWD" /etc/sudoers 2>/dev/null; then
        add_finding "high" "authentication" \
            "NOPASSWD sudo entries found" \
            "Some users have passwordless sudo access" \
            "Review /etc/sudoers and require password for sudo"
    else
        echo -e "  ${GREEN}✓ No NOPASSWD sudo entries${NC}"
    fi
    
    # ตรวจสอบ empty passwords
    echo "Checking for empty passwords..."
    local empty_pass_users
    empty_pass_users=$(awk -F: '($2 == "" || $2 == "!") && NR>1 {print $1}' /etc/shadow 2>/dev/null || echo "")
    
    if [[ -n "$empty_pass_users" ]]; then
        add_finding "critical" "authentication" \
            "Accounts with empty passwords" \
            "Found accounts: ${empty_pass_users}" \
            "Set passwords or lock accounts: passwd -l <user>"
    fi
    
    # ตรวจสอบ world-writable files
    echo "Checking world-writable files..."
    local world_writable
    world_writable=$(find /etc /usr/local /opt -perm -o+w -type f 2>/dev/null | head -10)
    
    if [[ -n "$world_writable" ]]; then
        add_finding "high" "file_permissions" \
            "World-writable system files found" \
            "Files: $(echo "$world_writable" | head -3 | tr '\n' ', ')" \
            "Fix permissions: chmod o-w <file>"
    fi
    
    # ตรวจสอบ SUID/SGID
    echo "Checking SUID/SGID binaries..."
    local unusual_suid
    unusual_suid=$(find / -perm /4000 -type f 2>/dev/null | \
        grep -vE "^/(bin|sbin|usr/bin|usr/sbin|usr/local/bin)/" | \
        head -5)
    
    if [[ -n "$unusual_suid" ]]; then
        add_finding "medium" "file_permissions" \
            "Unusual SUID/SGID files found" \
            "Files: $(echo "$unusual_suid" | head -3 | tr '\n' ', ')" \
            "Review and remove unnecessary SUID bits"
    fi
    
    # ตรวจสอบ SSH configuration
    echo "Checking SSH configuration..."
    local sshd_config="/etc/ssh/sshd_config"
    
    if [[ -f "$sshd_config" ]]; then
        # Root login
        if grep -qiE "^PermitRootLogin\s+yes" "$sshd_config"; then
            add_finding "high" "ssh_security" \
                "SSH root login is enabled" \
                "PermitRootLogin yes in sshd_config" \
                "Set PermitRootLogin no in /etc/ssh/sshd_config"
        fi
        
        # Password authentication
        if grep -qiE "^PasswordAuthentication\s+yes" "$sshd_config" || \
           ! grep -qiE "^PasswordAuthentication" "$sshd_config"; then
            add_finding "medium" "ssh_security" \
                "SSH password authentication is enabled" \
                "PasswordAuthentication is not disabled" \
                "Set PasswordAuthentication no and use key-based auth"
        fi
        
        # Protocol version
        if grep -qiE "^Protocol\s+1" "$sshd_config"; then
            add_finding "critical" "ssh_security" \
                "SSH Protocol 1 is enabled" \
                "Insecure SSH protocol version" \
                "Remove or set Protocol 2 in sshd_config"
        fi
    fi
    
    # ตรวจสอบ firewall
    echo "Checking firewall status..."
    if ! command -v ufw &>/dev/null && ! command -v firewalld &>/dev/null; then
        add_finding "high" "network_security" \
            "No firewall detected" \
            "Neither ufw nor firewalld is installed" \
            "Install and configure a firewall: apt install ufw && ufw enable"
    else
        if command -v ufw &>/dev/null; then
            local ufw_status
            ufw_status=$(ufw status 2>/dev/null | grep -o "Status:.*")
            if ! echo "$ufw_status" | grep -qi "active"; then
                add_finding "medium" "network_security" \
                    "Firewall is not active" \
                    "UFW is installed but not enabled" \
                    "Enable firewall: ufw enable"
            fi
        fi
    fi
    
    # ตรวจสอบ open ports
    echo "Checking open ports..."
    local open_ports
    open_ports=$(ss -tlnp 2>/dev/null | awk 'NR>1 {print $4}' | \
        grep -oE ":[0-9]+" | tr -d ':' | sort -n | uniq)
    
    local risky_ports=(21 23 25 53 69 111 135 137 139 445 512 513 514 1433 3389)
    
    for port in "${risky_ports[@]}"; do
        if echo "$open_ports" | grep -q "^${port}$"; then
            add_finding "medium" "network_security" \
                "Potentially risky port ${port} is open" \
                "Port ${port} is listening" \
                "Close or restrict access to port ${port}"
        fi
    done
    
    # ตรวจสอบ kernel settings
    echo "Checking kernel security settings..."
    
    local kernel_params=(
        "net.ipv4.ip_forward:0:IP forwarding enabled"
        "net.ipv4.conf.all.accept_redirects:0:ICMP redirects accepted"
        "net.ipv4.conf.all.send_redirects:0:ICMP redirects sent"
        "kernel.randomize_va_space:2:ASLR not enabled"
    )
    
    for param_info in "${kernel_params[@]}"; do
        IFS=':' read -r param secure_value message <<< "$param_info"
        
        local current_value
        current_value=$(sysctl -n "$param" 2>/dev/null || echo "unknown")
        
        if [[ "$current_value" != "$secure_value" && "$current_value" != "unknown" ]]; then
            add_finding "medium" "kernel_security" \
                "Insecure kernel parameter: ${param}" \
                "${message} (current: ${current_value}, expected: ${secure_value})" \
                "Set ${param} = ${secure_value} in /etc/sysctl.conf"
        fi
    done
}

# === Application Security Checks ===

scan_application_security() {
    echo -e "\n${BLUE}=== Application Security Scan ===${NC}"
    
    # ตรวจสอบ hardcoded secrets
    echo "Scanning for hardcoded secrets..."
    
    local secret_patterns=(
        "password\s*=\s*['\"][^'\"]{8,}['\"]"
        "api_key\s*=\s*['\"][A-Za-z0-9+/]{20,}['\"]"
        "secret\s*=\s*['\"][^'\"]{8,}['\"]"
        "private_key"
        "BEGIN RSA PRIVATE"
        "BEGIN OPENSSH PRIVATE"
        "AKIA[0-9A-Z]{16}"  # AWS Access Key
        "ghp_[A-Za-z0-9]{36}"  # GitHub token
    )
    
    local files_checked=0
    local secrets_found=0
    
    while IFS= read -r file; do
        ((files_checked++))
        
        for pattern in "${secret_patterns[@]}"; do
            if grep -lqiE "$pattern" "$file" 2>/dev/null; then
                ((secrets_found++))
                add_finding "critical" "secret_management" \
                    "Potential hardcoded secret in ${file}" \
                    "Pattern '${pattern}' found" \
                    "Move secrets to environment variables or secret manager"
                break
            fi
        done
    done < <(find . -type f \( -name "*.sh" -o -name "*.py" -o -name "*.js" -o -name "*.env" -o -name "*.conf" \) \
        ! -path "*/.git/*" ! -path "*/node_modules/*" 2>/dev/null | head -100)
    
    echo "  Files scanned: ${files_checked} | Secrets found: ${secrets_found}"
    
    # ตรวจสอบ SQL injection vulnerabilities
    echo "Checking for SQL injection patterns..."
    
    local sql_patterns=(
        'query.*\$.*\+'  # String concatenation in SQL
        'execute.*f".*"'  # f-string in SQL
        'SELECT.*\${'    # Variable in SELECT
    )
    
    for pattern in "${sql_patterns[@]}"; do
        local count
        count=$(grep -rE "$pattern" . \
            --include="*.sh" --include="*.py" \
            ! -path "*/.git/*" 2>/dev/null | wc -l || echo "0")
        
        if [[ $count -gt 0 ]]; then
            add_finding "high" "sql_injection" \
                "Potential SQL injection vulnerability" \
                "Found ${count} instances of unsafe SQL construction" \
                "Use parameterized queries/prepared statements"
        fi
    done
    
    # ตรวจสอบ command injection
    echo "Checking for command injection patterns..."
    
    local cmd_inject_patterns=(
        'eval\s+"\$'      # eval with variable
        'exec\s+"\$'      # exec with variable
        '\$\(.*\$[^)]*\)' # command substitution with variable
    )
    
    for pattern in "${cmd_inject_patterns[@]}"; do
        local count
        count=$(grep -rE "$pattern" . \
            --include="*.sh" \
            ! -path "*/.git/*" 2>/dev/null | wc -l || echo "0")
        
        if [[ $count -gt 0 ]]; then
            add_finding "high" "command_injection" \
                "Potential command injection" \
                "Found ${count} potentially unsafe eval/exec patterns" \
                "Validate and sanitize all user inputs before execution"
        fi
    done
}

# === Container Security Checks ===

scan_container_security() {
    echo -e "\n${BLUE}=== Container Security Scan ===${NC}"
    
    if ! command -v docker &>/dev/null; then
        echo -e "  ${YELLOW}Docker not available, skipping${NC}"
        return
    fi
    
    # Scan Dockerfiles
    echo "Scanning Dockerfiles..."
    
    while IFS= read -r dockerfile; do
        # ตรวจสอบ root user
        if ! grep -qiE "^USER\s+[^r]|^USER\s+[0-9]" "$dockerfile" 2>/dev/null; then
            add_finding "high" "container_security" \
                "Container runs as root: ${dockerfile}" \
                "No USER instruction found in Dockerfile" \
                "Add 'USER nonroot' or 'USER 1000' to Dockerfile"
        fi
        
        # ตรวจสอบ latest tag
        if grep -qE "FROM\s+\S+:latest" "$dockerfile" 2>/dev/null; then
            add_finding "medium" "container_security" \
                "Using 'latest' tag in ${dockerfile}" \
                "Unpinned image tags can cause non-deterministic builds" \
                "Pin to specific version: FROM ubuntu:22.04"
        fi
        
        # ตรวจสอบ secrets in ENV
        if grep -qiE "ENV\s+(PASSWORD|SECRET|API_KEY|TOKEN)\s*=" "$dockerfile" 2>/dev/null; then
            add_finding "high" "container_security" \
                "Secret in ENV instruction: ${dockerfile}" \
                "Secrets in ENV are visible in image metadata" \
                "Use --build-arg or runtime secrets instead"
        fi
    done < <(find . -name "Dockerfile" -o -name "Dockerfile.*" 2>/dev/null | head -20)
    
    # ตรวจสอบ running containers
    echo "Scanning running containers..."
    
    local privileged_containers
    privileged_containers=$(docker inspect \
        $(docker ps -q 2>/dev/null) 2>/dev/null | \
        python3 -c "
import json, sys
containers = json.load(sys.stdin)
for c in containers:
    name = c['Name'].lstrip('/')
    priv = c.get('HostConfig', {}).get('Privileged', False)
    if priv:
        print(name)
" 2>/dev/null)
    
    if [[ -n "$privileged_containers" ]]; then
        add_finding "critical" "container_security" \
            "Privileged containers running" \
            "Containers: ${privileged_containers}" \
            "Remove --privileged flag from container run command"
    fi
}

# สร้าง security report
generate_security_report() {
    local output_format="${1:-text}"
    
    echo ""
    echo "======================================"
    echo "       SECURITY SCAN REPORT"
    echo "======================================"
    echo "Scan completed: $(date)"
    echo ""
    echo "Summary:"
    echo -e "  ${RED}Critical: ${SEVERITY_COUNTS[critical]}${NC}"
    echo -e "  ${RED}High:     ${SEVERITY_COUNTS[high]}${NC}"
    echo -e "  ${YELLOW}Medium:   ${SEVERITY_COUNTS[medium]}${NC}"
    echo -e "  ${BLUE}Low:      ${SEVERITY_COUNTS[low]}${NC}"
    echo -e "  ${GREEN}Info:     ${SEVERITY_COUNTS[info]}${NC}"
    echo ""
    
    local total_issues=$((
        SEVERITY_COUNTS[critical] + 
        SEVERITY_COUNTS[high] + 
        SEVERITY_COUNTS[medium]
    ))
    
    if [[ $total_issues -gt 0 ]]; then
        echo -e "${RED}⚠ ${total_issues} issues require immediate attention${NC}"
    else
        echo -e "${GREEN}✓ No critical/high/medium issues found${NC}"
    fi
    
    echo "======================================"
    
    # Save JSON report
    {
        echo "{"
        echo "  \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\","
        echo "  \"summary\": {"
        echo "    \"critical\": ${SEVERITY_COUNTS[critical]},"
        echo "    \"high\": ${SEVERITY_COUNTS[high]},"
        echo "    \"medium\": ${SEVERITY_COUNTS[medium]},"
        echo "    \"low\": ${SEVERITY_COUNTS[low]}"
        echo "  }"
        echo "}"
    } > "$SCAN_REPORT"
    
    echo "Report saved: ${SCAN_REPORT}"
}

# Main scanner
main() {
    echo -e "${PURPLE}${BOLD}"
    echo "╔══════════════════════════════════════╗"
    echo "║       SECURITY VULNERABILITY SCAN    ║"
    echo "╚══════════════════════════════════════╝"
    echo -e "${NC}"
    
    case "${1:-all}" in
        os)          scan_os_security ;;
        app)         scan_application_security ;;
        container)   scan_container_security ;;
        all)
            scan_os_security
            scan_application_security
            scan_container_security
            generate_security_report
            ;;
        report)      generate_security_report "${2:-text}" ;;
        *)
            echo "Usage: $0 {os|app|container|all|report} [format]"
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 488: Zero Trust Security Implementation

```bash
#!/bin/bash
# zero_trust.sh - Zero Trust Security Framework

# Zero Trust principles:
# 1. Never trust, always verify
# 2. Least privilege access
# 3. Assume breach
# 4. Verify explicitly

# Identity verification
verify_identity() {
    local user="$1"
    local required_groups="${2:-}"
    local mfa_required="${3:-true}"
    
    echo "=== Identity Verification: ${user} ==="
    
    # ตรวจสอบ user มีอยู่จริง
    if ! id "$user" &>/dev/null; then
        echo "ERROR: User '${user}' not found"
        return 1
    fi
    
    # ตรวจสอบ group membership
    if [[ -n "$required_groups" ]]; then
        IFS=',' read -ra groups <<< "$required_groups"
        for group in "${groups[@]}"; do
            if ! id -nG "$user" | grep -qw "$group"; then
                echo "DENIED: ${user} not in group: ${group}"
                return 1
            fi
        done
    fi
    
    # ตรวจสอบ SSH key
    local ssh_key_file="/home/${user}/.ssh/authorized_keys"
    if [[ ! -f "$ssh_key_file" ]]; then
        echo "WARNING: No SSH authorized_keys for ${user}"
    fi
    
    # ตรวจสอบ account age
    local last_change
    last_change=$(chage -l "$user" 2>/dev/null | grep "Last password change" | awk -F: '{print $2}' | xargs)
    
    if [[ -n "$last_change" ]]; then
        echo "  Last password change: ${last_change}"
    fi
    
    echo "✓ Identity verified: ${user}"
    return 0
}

# Network micro-segmentation rules
setup_micro_segmentation() {
    local service_name="$1"
    local allowed_sources="${2:-}"
    local allowed_destinations="${3:-}"
    
    echo "=== Setting up Micro-segmentation: ${service_name} ==="
    
    # ใช้ iptables rules
    local chain="ZT_${service_name^^}"
    
    # สร้าง custom chain
    iptables -N "$chain" 2>/dev/null || iptables -F "$chain"
    
    # Default deny
    iptables -A "$chain" -j DROP
    
    # Allow from specified sources
    if [[ -n "$allowed_sources" ]]; then
        IFS=',' read -ra sources <<< "$allowed_sources"
        for source in "${sources[@]}"; do
            iptables -I "$chain" 1 -s "$source" -j ACCEPT
            echo "  Allow from: ${source}"
        done
    fi
    
    echo "✓ Micro-segmentation configured: ${service_name}"
}

# Certificate-based authentication
setup_cert_auth() {
    local ca_name="${1:-myca}"
    local output_dir="${2:-/tmp/certs}"
    
    mkdir -p "$output_dir"
    
    echo "=== Setting up Certificate Authentication: ${ca_name} ==="
    
    # สร้าง CA
    echo "Creating CA certificate..."
    openssl req -x509 -newkey rsa:4096 \
        -keyout "${output_dir}/ca-key.pem" \
        -out "${output_dir}/ca-cert.pem" \
        -days 3650 \
        -nodes \
        -subj "/CN=${ca_name}/O=Zero Trust/C=TH" 2>/dev/null
    
    echo "✓ CA created: ${output_dir}/ca-cert.pem"
    
    # Function สร้าง client cert
    issue_client_cert() {
        local client_name="$1"
        local validity="${2:-365}"
        
        # Generate key
        openssl genrsa -out "${output_dir}/${client_name}-key.pem" 2048 2>/dev/null
        
        # Generate CSR
        openssl req -new \
            -key "${output_dir}/${client_name}-key.pem" \
            -out "${output_dir}/${client_name}-csr.pem" \
            -subj "/CN=${client_name}/O=Zero Trust Client/C=TH" 2>/dev/null
        
        # Sign with CA
        openssl x509 -req \
            -in "${output_dir}/${client_name}-csr.pem" \
            -CA "${output_dir}/ca-cert.pem" \
            -CAkey "${output_dir}/ca-key.pem" \
            -CAcreateserial \
            -out "${output_dir}/${client_name}-cert.pem" \
            -days "$validity" 2>/dev/null
        
        echo "✓ Client cert issued: ${client_name}"
        echo "  Cert: ${output_dir}/${client_name}-cert.pem"
        echo "  Key:  ${output_dir}/${client_name}-key.pem"
    }
    
    # ออก cert สำหรับ services
    for service in "api-gateway" "backend" "database"; do
        issue_client_cert "$service"
    done
}

# Privilege management
manage_privileges() {
    local action="$1"
    local user="$2"
    local resource="$3"
    local permission="${4:-read}"
    
    echo "=== Privilege Management ==="
    
    # Audit log
    local audit_log="/var/log/zero_trust_audit.log"
    local timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    local request_id=$(cat /dev/urandom | tr -dc 'a-f0-9' | head -c 8)
    
    local log_entry
    log_entry=$(cat << EOF
{
    "timestamp": "${timestamp}",
    "request_id": "${request_id}",
    "action": "${action}",
    "user": "${user}",
    "resource": "${resource}",
    "permission": "${permission}",
    "ip": "$(who am i 2>/dev/null | awk '{print $NF}' | tr -d '()')",
    "result": "pending"
}
EOF
)
    
    case "$action" in
        grant)
            # บันทึกการให้สิทธิ์
            echo "${log_entry/pending/granted}" >> "$audit_log" 2>/dev/null
            
            # Just-in-time access (จำกัดเวลา)
            local duration="${5:-3600}"  # 1 hour default
            local expiry=$(date -d "+${duration} seconds" +%s 2>/dev/null || \
                          python3 -c "import time; print(int(time.time()) + ${duration})")
            
            echo "✓ Granted: ${user} → ${resource} (${permission}) expires in ${duration}s"
            echo "  Request ID: ${request_id}"
            echo "  Expires: $(date -d @${expiry} 2>/dev/null || date -r ${expiry} 2>/dev/null)"
            ;;
        
        revoke)
            echo "${log_entry/pending/revoked}" >> "$audit_log" 2>/dev/null
            echo "✓ Revoked: ${user} → ${resource} (${permission})"
            ;;
        
        check)
            echo "${log_entry/pending/checked}" >> "$audit_log" 2>/dev/null
            echo "Checking privilege: ${user} → ${resource} (${permission})"
            # ในระบบจริงจะตรวจสอบ database/policy engine
            ;;
    esac
}

# Security audit trail
audit_command() {
    local user="${SUDO_USER:-$(whoami)}"
    local command="$*"
    local timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    local session_id="${SSH_TTY:-local}"
    local source_ip
    source_ip=$(who am i 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | head -1 || echo "local")
    
    # บันทึก audit log
    local audit_log="/var/log/command_audit.log"
    
    cat >> "$audit_log" << EOF
{"timestamp":"${timestamp}","user":"${user}","session":"${session_id}","source":"${source_ip}","command":"${command}"}
EOF
    
    # รัน command จริง
    eval "$command"
}

# WAF (Web Application Firewall) rules
setup_waf_rules() {
    local output_file="${1:-/tmp/waf_rules.conf}"
    
    cat > "$output_file" << 'NGINX'
# nginx WAF rules
# ป้องกัน common web attacks

# Rate limiting
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;

# Block bad bots
map $http_user_agent $bad_bot {
    default 0;
    ~*(baiduspider|yandexbot|semrushbot|dotbot|ahrefs) 1;
    ~*(nikto|sqlmap|nessus|nmap|openvas|masscan) 1;
    "" 1;  # Empty user agent
}

server {
    # Block bad bots
    if ($bad_bot) {
        return 403;
    }
    
    # Block common attack patterns in URI
    location / {
        # SQL Injection patterns
        if ($request_uri ~* "(union|select|insert|drop|delete|update|exec)") {
            return 403;
        }
        
        # XSS patterns
        if ($request_uri ~* "(<script|javascript:|vbscript:|onload=|onerror=)") {
            return 403;
        }
        
        # Path traversal
        if ($request_uri ~* "\.\./") {
            return 403;
        }
        
        # Common exploit paths
        if ($request_uri ~* "(wp-login|wp-admin|phpmyadmin|phpinfo|\.env|\.git)") {
            return 403;
        }
        
        # Apply rate limiting
        limit_req zone=api burst=20 nodelay;
    }
    
    # Login endpoint - stricter rate limiting
    location /api/v1/auth/login {
        limit_req zone=login burst=5 nodelay;
        proxy_pass http://backend;
    }
    
    # Security headers
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Remove server info
    server_tokens off;
}
NGINX
    
    echo "✓ WAF rules created: ${output_file}"
}

# Main
case "${1:-}" in
    verify)       verify_identity "${2}" "${3:-}" "${4:-true}" ;;
    segment)      setup_micro_segmentation "${2}" "${3:-}" "${4:-}" ;;
    certs)        setup_cert_auth "${2:-myca}" "${3:-/tmp/certs}" ;;
    privilege)    manage_privileges "${2}" "${3}" "${4}" "${5:-read}" "${6:-3600}" ;;
    audit)        audit_command "${@:2}" ;;
    waf)          setup_waf_rules "${2:-/tmp/waf_rules.conf}" ;;
    *)
        echo "Usage: $0 {verify|segment|certs|privilege|audit|waf} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 489: Compliance Automation — CIS Benchmark

```bash
#!/bin/bash
# cis_benchmark.sh - CIS Benchmark Compliance Checking

REPORT_FILE="/tmp/cis_report_$$.json"
declare -A CIS_RESULTS
PASS_COUNT=0
FAIL_COUNT=0
WARN_COUNT=0

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

cis_check() {
    local id="$1"
    local title="$2"
    local level="${3:-1}"  # Level 1 or 2
    local check_cmd="$4"
    local expected="$5"
    local remediation="$6"
    
    local result
    result=$(eval "$check_cmd" 2>/dev/null || echo "ERROR")
    
    local status
    if eval "[[ \"$result\" $expected ]]" 2>/dev/null; then
        status="PASS"
        ((PASS_COUNT++))
        echo -e "  ${GREEN}✓ [L${level}] ${id}: ${title}${NC}"
    else
        status="FAIL"
        ((FAIL_COUNT++))
        echo -e "  ${RED}✗ [L${level}] ${id}: ${title}${NC}"
        echo -e "    ${RED}Current: ${result}${NC}"
        echo -e "    ${YELLOW}Fix: ${remediation}${NC}"
    fi
    
    CIS_RESULTS[$id]="$status"
}

cis_warn() {
    local id="$1"
    local title="$2"
    local message="$3"
    
    ((WARN_COUNT++))
    echo -e "  ${YELLOW}⚠ ${id}: ${title} (manual check required)${NC}"
    echo -e "    ${message}"
    CIS_RESULTS[$id]="WARN"
}

# CIS Section 1: Initial Setup
check_section1() {
    echo -e "\n${GREEN}=== Section 1: Initial Setup ===${NC}"
    
    # 1.1.1 Disable unused filesystems
    cis_check "1.1.1.1" "Ensure mounting of cramfs filesystems is disabled" 1 \
        "modprobe -n -v cramfs 2>&1 | head -1" \
        '== "install /bin/true"' \
        "echo 'install cramfs /bin/true' >> /etc/modprobe.d/cramfs.conf"
    
    cis_check "1.1.1.2" "Ensure mounting of freevxfs filesystems is disabled" 1 \
        "modprobe -n -v freevxfs 2>&1 | head -1" \
        '== "install /bin/true"' \
        "echo 'install freevxfs /bin/true' >> /etc/modprobe.d/freevxfs.conf"
    
    # 1.1.2 Configure /tmp
    cis_check "1.1.2" "/tmp is configured as separate partition" 1 \
        "mount | grep ' /tmp '" \
        '!= ""' \
        "Configure /tmp as separate partition in /etc/fstab"
    
    # 1.3 Sudo
    cis_check "1.3.1" "Ensure sudo is installed" 1 \
        "which sudo 2>/dev/null" \
        '!= ""' \
        "apt install sudo"
    
    cis_check "1.3.2" "Ensure sudo commands use pty" 1 \
        "grep -r 'use_pty' /etc/sudoers /etc/sudoers.d/ 2>/dev/null | head -1" \
        '!= ""' \
        "Add 'Defaults use_pty' to /etc/sudoers"
    
    # 1.4 Filesystem integrity
    cis_check "1.4.1" "Ensure AIDE is installed" 1 \
        "which aide 2>/dev/null" \
        '!= ""' \
        "apt install aide && aide --init"
    
    # 1.5 Boot settings
    cis_check "1.5.1" "Ensure permissions on bootloader config are configured" 1 \
        "stat /boot/grub/grub.cfg 2>/dev/null | grep -o 'Uid:.*Gid:' | head -1" \
        '== "Uid: (    0/    root)   Gid: (    0/    root)"' \
        "chown root:root /boot/grub/grub.cfg && chmod og-rwx /boot/grub/grub.cfg"
}

# CIS Section 2: Services
check_section2() {
    echo -e "\n${GREEN}=== Section 2: Services ===${NC}"
    
    local services_to_disable=(
        "avahi-daemon:2.1.1:Ensure Avahi Server is not installed"
        "cups:2.1.2:Ensure CUPS is not installed"
        "dhcpd:2.2.2:Ensure DHCP Server is not installed"
        "slapd:2.3.1:Ensure LDAP server is not installed"
        "nfs-server:2.3.4:Ensure NFS Server is not installed"
        "rpcbind:2.3.7:Ensure rpcbind is not installed"
        "bind9:2.3.9:Ensure DNS Server is not installed"
        "vsftpd:2.3.10:Ensure FTP Server is not installed"
        "apache2:2.3.11:Ensure HTTP server is not installed"
        "dovecot:2.3.12:Ensure IMAP/POP3 server is not installed"
        "samba:2.3.13:Ensure Samba is not installed"
    )
    
    for service_info in "${services_to_disable[@]}"; do
        IFS=':' read -r service_name cis_id cis_title <<< "$service_info"
        
        cis_check "$cis_id" "$cis_title" 1 \
            "systemctl is-enabled ${service_name} 2>/dev/null" \
            '== "disabled" || $result == "not-found" || $result == ""' \
            "systemctl disable --now ${service_name}"
    done
    
    # NTP
    cis_check "2.1.1" "Ensure time synchronization is in use" 1 \
        "systemctl is-enabled systemd-timesyncd 2>/dev/null || systemctl is-enabled chronyd 2>/dev/null || systemctl is-enabled ntp 2>/dev/null" \
        '== "enabled"' \
        "systemctl enable systemd-timesyncd"
}

# CIS Section 3: Network Configuration
check_section3() {
    echo -e "\n${GREEN}=== Section 3: Network Configuration ===${NC}"
    
    # 3.1 Network parameters
    local sysctl_checks=(
        "3.1.1:net.ipv4.ip_forward:Ensure IP forwarding is disabled:0"
        "3.1.2:net.ipv4.conf.all.send_redirects:Ensure packet redirect sending is disabled:0"
        "3.2.1:net.ipv4.conf.all.accept_source_route:Ensure source routed packets are not accepted:0"
        "3.2.2:net.ipv4.conf.all.accept_redirects:Ensure ICMP redirects are not accepted:0"
        "3.2.3:net.ipv4.conf.all.secure_redirects:Ensure secure ICMP redirects are not accepted:0"
        "3.2.4:net.ipv4.conf.all.log_martians:Ensure suspicious packets are logged:1"
        "3.2.7:net.ipv4.conf.all.rp_filter:Ensure Reverse Path Filtering is enabled:1"
        "3.2.8:net.ipv4.tcp_syncookies:Ensure TCP SYN Cookies is enabled:1"
        "3.3.1:net.ipv6.conf.all.accept_ra:Ensure IPv6 router advertisements are not accepted:0"
    )
    
    for check_info in "${sysctl_checks[@]}"; do
        IFS=':' read -r cis_id param cis_title expected_val <<< "$check_info"
        
        cis_check "$cis_id" "$cis_title" 1 \
            "sysctl -n ${param} 2>/dev/null" \
            "== \"${expected_val}\"" \
            "echo '${param} = ${expected_val}' >> /etc/sysctl.d/99-cis.conf && sysctl -p"
    done
}

# CIS Section 4: Logging and Auditing
check_section4() {
    echo -e "\n${GREEN}=== Section 4: Logging and Auditing ===${NC}"
    
    cis_check "4.1.1.1" "Ensure auditd is installed" 2 \
        "which auditctl 2>/dev/null" \
        '!= ""' \
        "apt install auditd"
    
    cis_check "4.1.1.2" "Ensure auditd service is enabled" 2 \
        "systemctl is-enabled auditd 2>/dev/null" \
        '== "enabled"' \
        "systemctl enable auditd"
    
    cis_check "4.2.1.1" "Ensure rsyslog is installed" 1 \
        "which rsyslogd 2>/dev/null" \
        '!= ""' \
        "apt install rsyslog"
    
    cis_check "4.2.1.2" "Ensure rsyslog Service is enabled" 1 \
        "systemctl is-enabled rsyslog 2>/dev/null" \
        '== "enabled"' \
        "systemctl enable rsyslog"
    
    # Log file permissions
    cis_check "4.2.3" "Ensure permissions on log files are configured" 1 \
        "find /var/log -type f -perm /137 2>/dev/null | wc -l" \
        '== "0"' \
        "find /var/log -type f -exec chmod g-wx,o-rwx {} +"
}

# CIS Section 5: Access Control
check_section5() {
    echo -e "\n${GREEN}=== Section 5: Access Control ===${NC}"
    
    # SSH settings
    local ssh_config="/etc/ssh/sshd_config"
    
    if [[ -f "$ssh_config" ]]; then
        cis_check "5.2.1" "Ensure permissions on /etc/ssh/sshd_config are configured" 1 \
            "stat -c '%U %G %a' ${ssh_config} 2>/dev/null" \
            '== "root root 600"' \
            "chown root:root ${ssh_config} && chmod 600 ${ssh_config}"
        
        cis_check "5.2.4" "Ensure SSH Protocol is set to 2" 1 \
            "sshd -T 2>/dev/null | grep -i '^protocol' | awk '{print \$2}'" \
            '== "2" || $result == ""' \
            "echo 'Protocol 2' >> ${ssh_config}"
        
        cis_check "5.2.5" "Ensure SSH LogLevel is appropriate" 1 \
            "sshd -T 2>/dev/null | grep -i loglevel | awk '{print \$2}'" \
            '== "INFO" || $result == "VERBOSE"' \
            "echo 'LogLevel INFO' >> ${ssh_config}"
        
        cis_check "5.2.7" "Ensure SSH MaxAuthTries is set to 4 or less" 1 \
            "sshd -T 2>/dev/null | grep -i maxauthtries | awk '{print \$2}'" \
            '<= 4' \
            "echo 'MaxAuthTries 4' >> ${ssh_config}"
        
        cis_check "5.2.8" "Ensure SSH IgnoreRhosts is enabled" 1 \
            "sshd -T 2>/dev/null | grep -i ignorerhosts | awk '{print \$2}'" \
            '== "yes"' \
            "echo 'IgnoreRhosts yes' >> ${ssh_config}"
        
        cis_check "5.2.9" "Ensure SSH HostbasedAuthentication is disabled" 1 \
            "sshd -T 2>/dev/null | grep -i hostbasedauthentication | awk '{print \$2}'" \
            '== "no"' \
            "echo 'HostbasedAuthentication no' >> ${ssh_config}"
        
        cis_check "5.2.10" "Ensure SSH root login is disabled" 1 \
            "sshd -T 2>/dev/null | grep -i permitrootlogin | awk '{print \$2}'" \
            '== "no"' \
            "echo 'PermitRootLogin no' >> ${ssh_config}"
        
        cis_check "5.2.11" "Ensure SSH PermitEmptyPasswords is disabled" 1 \
            "sshd -T 2>/dev/null | grep -i permitemptypasswords | awk '{print \$2}'" \
            '== "no"' \
            "echo 'PermitEmptyPasswords no' >> ${ssh_config}"
    else
        cis_warn "5.2" "SSH Configuration" "sshd_config not found"
    fi
    
    # Password policy
    cis_check "5.4.1.1" "Ensure password expiration is 365 days or less" 1 \
        "grep -E '^PASS_MAX_DAYS' /etc/login.defs 2>/dev/null | awk '{print \$2}'" \
        '<= 365 && $result != ""' \
        "Set PASS_MAX_DAYS 90 in /etc/login.defs"
    
    cis_check "5.4.1.2" "Ensure minimum days between password changes is 7 or more" 1 \
        "grep -E '^PASS_MIN_DAYS' /etc/login.defs 2>/dev/null | awk '{print \$2}'" \
        '>= 7' \
        "Set PASS_MIN_DAYS 7 in /etc/login.defs"
}

# สร้าง compliance report
generate_compliance_report() {
    local total=$((PASS_COUNT + FAIL_COUNT + WARN_COUNT))
    local pass_rate=0
    
    [[ $total -gt 0 ]] && pass_rate=$((PASS_COUNT * 100 / total))
    
    echo ""
    echo "========================================"
    echo "       CIS BENCHMARK COMPLIANCE REPORT"
    echo "========================================"
    echo "Date:     $(date)"
    echo "Hostname: $(hostname)"
    echo "OS:       $(cat /etc/os-release | grep PRETTY_NAME | cut -d'"' -f2 2>/dev/null || uname -s)"
    echo ""
    echo "Results:"
    echo -e "  ${GREEN}PASS: ${PASS_COUNT}${NC}"
    echo -e "  ${RED}FAIL: ${FAIL_COUNT}${NC}"
    echo -e "  ${YELLOW}WARN: ${WARN_COUNT}${NC}"
    echo ""
    echo "Compliance Rate: ${pass_rate}%"
    
    if [[ $pass_rate -ge 90 ]]; then
        echo -e "Status: ${GREEN}COMPLIANT${NC}"
    elif [[ $pass_rate -ge 70 ]]; then
        echo -e "Status: ${YELLOW}PARTIALLY COMPLIANT${NC}"
    else
        echo -e "Status: ${RED}NON-COMPLIANT${NC}"
    fi
    
    echo "========================================"
    
    # Save detailed report
    echo "Saving report to: ${REPORT_FILE}"
    python3 -c "
import json

results = {}
$(for id in "${!CIS_RESULTS[@]}"; do echo "results['${id}'] = '${CIS_RESULTS[$id]}'"; done)

report = {
    'timestamp': '$(date -u +%Y-%m-%dT%H:%M:%SZ)',
    'hostname': '$(hostname)',
    'pass_count': ${PASS_COUNT},
    'fail_count': ${FAIL_COUNT},
    'warn_count': ${WARN_COUNT},
    'compliance_rate': ${pass_rate},
    'results': results
}
print(json.dumps(report, indent=2))
" > "$REPORT_FILE" 2>/dev/null
}

# Main
echo -e "${GREEN}${BOLD}"
echo "╔══════════════════════════════════════╗"
echo "║    CIS BENCHMARK COMPLIANCE CHECK    ║"
echo "╚══════════════════════════════════════╝"
echo -e "${NC}"

case "${1:-all}" in
    all)
        check_section1
        check_section2
        check_section3
        check_section4
        check_section5
        generate_compliance_report
        ;;
    section1) check_section1 ;;
    section2) check_section2 ;;
    section3) check_section3 ;;
    section4) check_section4 ;;
    section5) check_section5 ;;
    report)   generate_compliance_report ;;
    *)
        echo "Usage: $0 {all|section1|section2|section3|section4|section5|report}"
        ;;
esac
```

---

## สรุป Part 32

| หัวข้อ | เนื้อหา |
|--------|---------|
| **Security Scanning** | OS security, app secrets, SQL/CMD injection, container security |
| **Zero Trust** | Identity verification, micro-segmentation, cert auth, privilege mgmt |
| **WAF Rules** | Nginx WAF, rate limiting, attack pattern blocking |
| **CIS Benchmark** | Automated compliance checking: sections 1-5 |
| **Compliance Reporting** | JSON reports, compliance rate calculation |

**ขั้นตอนต่อไป**: Part 33 - Site Reliability Engineering (SRE) Practices
