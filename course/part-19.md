# Part 19: Security Scripts

## Module 2: Intermediate Level (ระดับกลาง)

---

## ขั้นตอนที่ 396: Security Best Practices

```bash
#!/usr/bin/env bash
# security_basics.sh - Security Best Practices

echo "=== Security Best Practices ==="
echo ""
echo "Bash Security Rules:"
echo ""
echo "1. เสมอใช้ set -euo pipefail"
echo "2. Quote ตัวแปรเสมอ: \"\$var\" ไม่ใช่ \$var"
echo "3. ไม่ใช้ eval เว้นแต่จำเป็น"
echo "4. Validate input ทั้งหมด"
echo "5. ไม่เก็บ secrets ใน environment variables ถ้าหลีกเลี่ยงได้"
echo "6. ใช้ mktemp สำหรับไฟล์ชั่วคราว"
echo "7. Set umask ที่เหมาะสม"
echo "8. ตรวจสอบสิทธิ์ก่อนทำงาน"
echo ""

echo "=== Common Vulnerabilities ==="

echo ""
echo "1. Command Injection:"
cat << 'EXAMPLE'
# BAD - unsafe
user_input="$(read)"
eval "ls $user_input"  # อันตราย! อาจ inject "; rm -rf /"

# GOOD - safe
ls -- "$user_input"    # -- ป้องกัน option injection
EXAMPLE

echo ""
echo "2. Path Traversal:"
cat << 'EXAMPLE'
# BAD
cat "/var/data/$filename"  # filename อาจเป็น ../../etc/passwd

# GOOD
# ตรวจสอบ filename
if [[ "$filename" =~ \.\. ]] || [[ "$filename" =~ ^/ ]]; then
    echo "Invalid filename"
    exit 1
fi
cat "/var/data/$filename"
EXAMPLE

echo ""
echo "3. Symlink Attack:"
cat << 'EXAMPLE'
# BAD - race condition
echo "data" > /tmp/myfile

# GOOD - ใช้ O_NOFOLLOW หรือ mktemp
tmpfile=$(mktemp)
echo "data" > "$tmpfile"
EXAMPLE

echo ""
echo "4. Insecure umask:"
cat << 'EXAMPLE'
# Default umask อาจ 022 (world-readable)
# กำหนด umask ที่ strict กว่า
umask 077  # owner only: rw-------
tmpfile=$(mktemp)
EXAMPLE
```

---

## ขั้นตอนที่ 397: Input Validation และ Sanitization

```bash
#!/usr/bin/env bash
# input_validation.sh - Input Validation

echo "=== Input Validation and Sanitization ==="

# ==================== Validation Functions ====================

validate_alphanum() {
    local input="$1"
    local max_len="${2:-255}"
    
    if [[ ${#input} -gt $max_len ]]; then
        return 1
    fi
    [[ "$input" =~ ^[a-zA-Z0-9_-]+$ ]]
}

validate_email() {
    local email="$1"
    [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
}

validate_ip() {
    local ip="$1"
    local pattern='^([0-9]{1,3}\.){3}[0-9]{1,3}$'
    
    if [[ ! "$ip" =~ $pattern ]]; then
        return 1
    fi
    
    IFS='.' read -ra octets <<< "$ip"
    for octet in "${octets[@]}"; do
        if (( octet > 255 )); then
            return 1
        fi
    done
    return 0
}

validate_path() {
    local path="$1"
    local base_dir="${2:-/}"
    
    # Resolve แล้วตรวจสอบว่าอยู่ใน base_dir
    local real_path
    real_path=$(realpath -m "$path" 2>/dev/null || echo "$path")
    
    [[ "$real_path" == "${base_dir%/}"/* ]] || [[ "$real_path" == "$base_dir" ]]
}

validate_integer() {
    local input="$1"
    local min="${2:--2147483648}"
    local max="${3:-2147483647}"
    
    if [[ ! "$input" =~ ^-?[0-9]+$ ]]; then
        return 1
    fi
    
    (( input >= min && input <= max ))
}

validate_url() {
    local url="$1"
    [[ "$url" =~ ^https?://[a-zA-Z0-9._/-]+$ ]]
}

# ==================== Sanitization ====================

sanitize_shell() {
    # Escape shell special characters
    local input="$1"
    printf '%q' "$input"
}

sanitize_html() {
    local input="$1"
    echo "$input" | sed 's/&/\&amp;/g; s/</\&lt;/g; s/>/\&gt;/g; s/"/\&quot;/g'
}

sanitize_sql() {
    # Basic SQL escaping (ใช้ prepared statements ในกรณีจริง)
    local input="$1"
    echo "$input" | sed "s/'/''/g; s/\\\\/\\\\\\\\/g"
}

sanitize_filename() {
    local filename="$1"
    # ลบ special characters ที่อันตราย
    echo "$filename" | sed 's/[^a-zA-Z0-9._-]/_/g' | sed 's/^\./_/'
}

# ==================== Demo ====================
echo "1. Input validation:"

test_inputs=(
    "validUser123"
    "invalid user!"
    "user<script>"
    "../../../etc/passwd"
    "normal_file.txt"
)

for input in "${test_inputs[@]}"; do
    if validate_alphanum "$input"; then
        echo "  ✓ VALID:   '$input'"
    else
        echo "  ✗ INVALID: '$input'"
    fi
done

echo ""
echo "2. Email validation:"
for email in "user@example.com" "invalid" "a@b.c" "bad@.com"; do
    if validate_email "$email"; then
        echo "  ✓ $email"
    else
        echo "  ✗ $email"
    fi
done

echo ""
echo "3. Path validation:"
for path in "/var/data/file.txt" "/var/data/../etc/passwd" "/var/data" "/etc/passwd"; do
    if validate_path "$path" "/var/data"; then
        echo "  ✓ SAFE:   $path"
    else
        echo "  ✗ UNSAFE: $path"
    fi
done

echo ""
echo "4. Sanitization:"
echo "Shell: $(sanitize_shell "hello; rm -rf /")"
echo "HTML: $(sanitize_html '<script>alert("xss")</script>')"
echo "Filename: $(sanitize_filename "../../etc/passwd")"
```

---

## ขั้นตอนที่ 398: Password and Secret Management

```bash
#!/usr/bin/env bash
# secret_management.sh - Secret Management

echo "=== Secret Management ==="
echo ""

echo "1. Password Generation:"

generate_password() {
    local length="${1:-16}"
    local charset="${2:-'A-Za-z0-9!@#$%^&*'}"
    
    # Method 1: /dev/urandom
    tr -dc "$charset" < /dev/urandom 2>/dev/null | head -c "$length" && echo
}

generate_passphrase() {
    local words="${1:-4}"
    local wordlist="/usr/share/dict/words"
    
    if [[ ! -f "$wordlist" ]]; then
        # Fallback ถ้าไม่มี wordlist
        local fallback_words=("apple" "banana" "cherry" "dragon" "eagle" "forest" "garden" "harbor")
        local phrase=""
        for ((i=0; i<words; i++)); do
            phrase+="${fallback_words[$RANDOM % ${#fallback_words[@]}]}-"
        done
        echo "${phrase%-}"
        return
    fi
    
    grep -E '^[a-z]{4,8}$' "$wordlist" 2>/dev/null | shuf | head -"$words" | tr '\n' '-' | sed 's/-$//'
}

echo "Random password (16 chars):"
generate_password 16

echo "Alphanumeric (20 chars):"
generate_password 20 'A-Za-z0-9'

echo "Passphrase (4 words):"
generate_passphrase 4

echo ""
echo "2. Password hashing:"
hash_password() {
    local password="$1"
    local salt="${2:-$(head -c 16 /dev/urandom | base64 | tr -d '=+/' | head -c 16)}"
    
    echo "${salt}:$(echo -n "${salt}${password}" | sha256sum | cut -d' ' -f1)"
}

verify_password() {
    local password="$1"
    local stored_hash="$2"
    
    local salt="${stored_hash%%:*}"
    local expected_hash="${stored_hash##*:}"
    local actual_hash
    actual_hash=$(echo -n "${salt}${password}" | sha256sum | cut -d' ' -f1)
    
    [[ "$actual_hash" == "$expected_hash" ]]
}

echo "Password hashing:"
password="my_secure_password"
stored=$(hash_password "$password")
echo "Stored: $stored"

if verify_password "$password" "$stored"; then
    echo "✓ Password verified"
else
    echo "✗ Password mismatch"
fi

echo ""
echo "3. Environment secrets:"
cat << 'SECRETS'
# แนวทางจัดการ secrets:

# Method 1: .env file (ไม่ commit)
# .env:
DB_PASSWORD=secret123
API_KEY=abc456

# Load in script:
if [[ -f .env ]]; then
    set -a  # export ทุก variables
    source .env
    set +a
fi

# Method 2: Docker secrets
# docker secret create db_password - <<< "secret123"

# Method 3: Vault (HashiCorp)
# vault kv get -field=password secret/myapp/db

# Method 4: AWS Secrets Manager
# aws secretsmanager get-secret-value --secret-id myapp/db

# Method 5: environment file with permissions 600
chmod 600 .env
SECRETS

echo ""
echo "4. Secure file deletion:"
secure_delete() {
    local file="$1"
    
    if [[ ! -f "$file" ]]; then
        return 0
    fi
    
    # Overwrite ก่อนลบ
    local size
    size=$(stat -c%s "$file" 2>/dev/null || stat -f%z "$file" 2>/dev/null || echo 1024)
    
    # Overwrite with random data 3 times
    for pass in 1 2 3; do
        dd if=/dev/urandom of="$file" bs=1 count="$size" 2>/dev/null || true
    done
    
    rm -f "$file"
    echo "Securely deleted: $file"
}

echo "Secure delete demo:"
echo "test data" > /tmp/sensitive.txt
secure_delete /tmp/sensitive.txt
```

---

## ขั้นตอนที่ 399: File Permissions และ ACLs

```bash
#!/usr/bin/env bash
# permissions.sh - File Permissions

echo "=== File Permissions ==="

echo "1. Permission basics:"
cat << 'EOF'
chmod format:
  Symbolic: chmod [ugoa][+-=][rwxXst] file
  Octal:    chmod 755 file

  u=user, g=group, o=other, a=all
  r=read(4), w=write(2), x=execute(1)
  
  755 = rwxr-xr-x (owner:rwx, group:rx, other:rx)
  644 = rw-r--r-- (owner:rw, group:r, other:r)
  700 = rwx------ (owner only)
  600 = rw------- (owner read/write only)
  
Special bits:
  SUID (4000) - run as owner: chmod u+s file
  SGID (2000) - run as group: chmod g+s dir
  Sticky (1000) - only owner can delete: chmod +t dir
EOF

echo ""
echo "2. Common permission scenarios:"
setup_permissions() {
    local base_dir="/tmp/perm_demo"
    mkdir -p "$base_dir"
    
    # Config files - owner read/write only
    touch "$base_dir/config.conf"
    chmod 600 "$base_dir/config.conf"
    echo "  config.conf: $(stat -c '%A' "$base_dir/config.conf" 2>/dev/null || stat -f '%Sp' "$base_dir/config.conf")"
    
    # Scripts - executable
    touch "$base_dir/script.sh"
    chmod 755 "$base_dir/script.sh"
    echo "  script.sh:   $(stat -c '%A' "$base_dir/script.sh" 2>/dev/null || stat -f '%Sp' "$base_dir/script.sh")"
    
    # Data files
    touch "$base_dir/data.csv"
    chmod 644 "$base_dir/data.csv"
    echo "  data.csv:    $(stat -c '%A' "$base_dir/data.csv" 2>/dev/null || stat -f '%Sp' "$base_dir/data.csv")"
    
    # Private directory
    mkdir "$base_dir/private"
    chmod 700 "$base_dir/private"
    echo "  private/:    $(stat -c '%A' "$base_dir/private" 2>/dev/null || stat -f '%Sp' "$base_dir/private")"
    
    # Log directory (group writable)
    mkdir "$base_dir/logs"
    chmod 775 "$base_dir/logs"
    echo "  logs/:       $(stat -c '%A' "$base_dir/logs" 2>/dev/null || stat -f '%Sp' "$base_dir/logs")"
    
    rm -rf "$base_dir"
}

setup_permissions

echo ""
echo "3. Find permission issues:"
cat << 'EOF'
# ไฟล์ที่ world-writable (อันตราย)
find /var/www -perm -002 -type f 2>/dev/null

# SUID files
find / -perm -4000 -type f 2>/dev/null | head -10

# Files with no owner
find / -nouser -o -nogroup 2>/dev/null | head -5

# Config files ที่ world-readable
find /etc -perm -004 -name "*.conf" 2>/dev/null | head -5
EOF

echo ""
echo "4. umask:"
echo "Current umask: $(umask)"
echo "umask 022 → files: 644, dirs: 755"
echo "umask 077 → files: 600, dirs: 700"
echo ""
cat << 'EOF'
# ตั้งค่า umask ที่ strict
umask 077    # ใน script
# หรือเพิ่มใน ~/.bashrc
EOF
```

---

## ขั้นตอนที่ 400: Security Audit Script

```bash
#!/usr/bin/env bash
# security_audit.sh - Security Audit

set -euo pipefail

echo "=== Security Audit Script ==="
echo ""

# ==================== Audit Functions ====================

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'

pass()  { echo -e "  ${GREEN}[PASS]${NC} $*"; }
fail()  { echo -e "  ${RED}[FAIL]${NC} $*"; }
warn()  { echo -e "  ${YELLOW}[WARN]${NC} $*"; }
info()  { echo -e "  [INFO] $*"; }

audit_users() {
    echo "=== User Security ==="
    
    # Root login check
    if grep -q "^PermitRootLogin yes" /etc/ssh/sshd_config 2>/dev/null; then
        fail "SSH root login enabled"
    else
        pass "SSH root login disabled"
    fi
    
    # Empty passwords
    local empty_pass
    empty_pass=$(awk -F: '($2 == "" && $1 != "") {print $1}' /etc/shadow 2>/dev/null | wc -l)
    if (( empty_pass > 0 )); then
        fail "Users with empty passwords: $empty_pass"
    else
        pass "No users with empty passwords"
    fi
    
    # UID 0 users (only root should have UID 0)
    local uid0_users
    uid0_users=$(awk -F: '$3==0 {print $1}' /etc/passwd 2>/dev/null | grep -v "^root$" | wc -l)
    if (( uid0_users > 0 )); then
        fail "Non-root users with UID 0: $(awk -F: '$3==0 {print $1}' /etc/passwd 2>/dev/null | grep -v root)"
    else
        pass "Only root has UID 0"
    fi
}

audit_ssh() {
    echo ""
    echo "=== SSH Security ==="
    
    local sshd_config="/etc/ssh/sshd_config"
    
    if [[ ! -f "$sshd_config" ]]; then
        info "SSH not installed"
        return
    fi
    
    # Password auth
    if grep -q "^PasswordAuthentication no" "$sshd_config" 2>/dev/null; then
        pass "SSH password auth disabled"
    else
        warn "SSH password auth may be enabled"
    fi
    
    # Protocol version
    if grep -q "^Protocol 2" "$sshd_config" 2>/dev/null; then
        pass "SSH Protocol 2 only"
    else
        info "SSH Protocol (not explicitly set, check version)"
    fi
    
    # Port
    local ssh_port
    ssh_port=$(grep "^Port " "$sshd_config" 2>/dev/null | awk '{print $2}' | head -1)
    if [[ "${ssh_port:-22}" == "22" ]]; then
        warn "SSH on default port 22"
    else
        pass "SSH on non-standard port: $ssh_port"
    fi
}

audit_filesystem() {
    echo ""
    echo "=== Filesystem Security ==="
    
    # World-writable files in /tmp
    local ww_tmp
    ww_tmp=$(find /tmp -perm -002 -not -type l 2>/dev/null | wc -l)
    if (( ww_tmp > 5 )); then
        warn "Many world-writable files in /tmp: $ww_tmp"
    else
        pass "World-writable files in /tmp: $ww_tmp"
    fi
    
    # SUID files (should be minimal)
    local suid_count
    suid_count=$(find /usr/bin /usr/sbin /bin /sbin -perm -4000 2>/dev/null | wc -l)
    info "SUID executables: $suid_count"
    
    # /etc/passwd permissions
    local passwd_perm
    passwd_perm=$(stat -c '%a' /etc/passwd 2>/dev/null || stat -f '%Lp' /etc/passwd 2>/dev/null || echo "unknown")
    if [[ "$passwd_perm" == "644" ]]; then
        pass "/etc/passwd permissions: $passwd_perm"
    else
        warn "/etc/passwd permissions: $passwd_perm (expected 644)"
    fi
    
    # /etc/shadow permissions
    local shadow_perm
    shadow_perm=$(stat -c '%a' /etc/shadow 2>/dev/null || echo "N/A")
    if [[ "$shadow_perm" == "640" ]] || [[ "$shadow_perm" == "000" ]]; then
        pass "/etc/shadow permissions: $shadow_perm"
    elif [[ "$shadow_perm" == "N/A" ]]; then
        info "/etc/shadow not found or not accessible"
    else
        warn "/etc/shadow permissions: $shadow_perm"
    fi
}

audit_services() {
    echo ""
    echo "=== Service Security ==="
    
    # Open ports
    local open_ports
    open_ports=$(ss -tlnp 2>/dev/null | awk 'NR>1 {print $4}' | grep -oE ':[0-9]+$' | tr -d ':' | sort -n | tr '\n' ' ')
    info "Open TCP ports: $open_ports"
    
    # Firewall
    if command -v ufw &>/dev/null; then
        local ufw_status
        ufw_status=$(ufw status 2>/dev/null | head -1)
        if echo "$ufw_status" | grep -q "active"; then
            pass "UFW firewall: active"
        else
            warn "UFW firewall: inactive"
        fi
    elif command -v firewall-cmd &>/dev/null; then
        if firewall-cmd --state 2>/dev/null | grep -q "running"; then
            pass "firewalld: running"
        else
            warn "firewalld: not running"
        fi
    else
        info "No standard firewall found (may use iptables directly)"
    fi
}

audit_updates() {
    echo ""
    echo "=== System Updates ==="
    
    if command -v apt &>/dev/null; then
        local pending
        pending=$(apt list --upgradable 2>/dev/null | grep -c "upgradable" || echo 0)
        if (( pending > 0 )); then
            warn "Pending updates: $pending"
        else
            pass "System up to date"
        fi
    elif command -v yum &>/dev/null; then
        local pending
        pending=$(yum check-update 2>/dev/null | grep -c "^[a-zA-Z]" || echo 0)
        info "Pending updates: $pending (yum)"
    fi
}

# ==================== Main ====================
echo "Running security audit on $(hostname)..."
echo "Date: $(date)"
echo ""

audit_users 2>/dev/null || true
audit_ssh 2>/dev/null || true
audit_filesystem 2>/dev/null || true
audit_services 2>/dev/null || true
audit_updates 2>/dev/null || true

echo ""
echo "Audit completed."
```

---

## ขั้นตอนที่ 401: Intrusion Detection Basics

```bash
#!/usr/bin/env bash
# intrusion_detection.sh - Basic IDS

echo "=== Basic Intrusion Detection ==="

echo "1. ตรวจสอบ failed logins:"
detect_brute_force() {
    local log_file="${1:-/var/log/auth.log}"
    local threshold="${2:-5}"
    local window="${3:-60}"  # minutes
    
    if [[ ! -f "$log_file" ]] && [[ ! -r "$log_file" ]]; then
        echo "Cannot read $log_file"
        return 1
    fi
    
    echo "Checking for brute force attacks (threshold: $threshold failures in $window min):"
    
    # นับ failed attempts ต่อ IP
    grep "Failed password" "$log_file" 2>/dev/null | \
        grep -oE "from [0-9.]+" | \
        awk '{print $2}' | \
        sort | uniq -c | \
        sort -rn | \
        awk -v thresh="$threshold" '$1 >= thresh {
            printf "  ALERT: %d failed attempts from %s\n", $1, $2
        }'
}

detect_brute_force 2>/dev/null || echo "  (ไม่มีสิทธิ์อ่าน auth.log)"

echo ""
echo "2. ตรวจสอบ suspicious processes:"
detect_suspicious_processes() {
    # Processes ที่รันจาก /tmp หรือ /dev/shm (อาจเป็น malware)
    local suspicious
    suspicious=$(ps aux 2>/dev/null | awk '$11 ~ /^\/tmp|^\/dev\/shm|^\/var\/tmp/ {print $0}')
    
    if [[ -n "$suspicious" ]]; then
        echo "  ALERT: Suspicious processes:"
        echo "$suspicious"
    else
        echo "  OK: No suspicious processes in temp directories"
    fi
    
    # Processes ที่ใช้ CPU มาก
    echo ""
    echo "  High CPU processes:"
    ps aux 2>/dev/null | sort -k3 -rn | awk 'NR>1 && $3+0 > 50 {
        print "  CPU: " $3 "% PID: " $2 " CMD: " $11
    }' | head -5
}

detect_suspicious_processes 2>/dev/null

echo ""
echo "3. ตรวจสอบ open ports ที่ผิดปกติ:"
check_unexpected_ports() {
    local expected_ports="${1:-22 80 443}"
    
    echo "  Open ports:"
    while IFS= read -r port; do
        local found=false
        for expected in $expected_ports; do
            if [[ "$port" == "$expected" ]]; then
                found=true
                break
            fi
        done
        
        if [[ "$found" == "false" ]]; then
            echo "  UNEXPECTED: port $port"
        else
            echo "  OK: port $port"
        fi
    done < <(ss -tlnp 2>/dev/null | awk 'NR>1 {print $4}' | grep -oE ':[0-9]+$' | tr -d ':' | sort -un)
}

check_unexpected_ports "22 80 443 8080"

echo ""
echo "4. File integrity check:"
check_file_integrity() {
    local check_dirs=("/etc" "/usr/bin" "/usr/sbin")
    local db_file="/tmp/file_hashes.db"
    
    # Build hash database
    if [[ ! -f "$db_file" ]]; then
        echo "  Creating hash database..."
        find "${check_dirs[@]}" -type f 2>/dev/null | \
            head -20 | \
            while read -r file; do
                echo "$(md5sum "$file" 2>/dev/null)" >> "$db_file"
            done
        echo "  Database created: $db_file ($(wc -l < "$db_file") files)"
    else
        echo "  Checking for changes..."
        local changes=0
        while IFS=' ' read -r hash file; do
            local current_hash
            current_hash=$(md5sum "$file" 2>/dev/null | cut -d' ' -f1)
            if [[ "$hash" != "$current_hash" ]]; then
                echo "  CHANGED: $file"
                (( changes++ ))
            fi
        done < "$db_file"
        
        if (( changes == 0 )); then
            echo "  OK: No file changes detected"
        else
            echo "  ALERT: $changes file(s) changed"
        fi
        
        rm -f "$db_file"
    fi
}

check_file_integrity 2>/dev/null
```

---

## ขั้นตอนที่ 402: Workshop - Security Hardening Script

```bash
#!/usr/bin/env bash
# security_hardening.sh - Workshop: Security Hardening

set -euo pipefail

echo "=== Security Hardening Script ==="
echo ""
echo "NOTE: Scripts ต่อไปนี้ใช้สำหรับ demo/reference เท่านั้น"
echo "การ apply จริงต้องทำกับ root และ test ในสภาพแวดล้อมที่ปลอดภัย"
echo ""

# ==================== Hardening Functions ====================

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'

harden_sshd() {
    echo "=== SSH Hardening ==="
    
    cat << 'SSH_CONFIG'
# Recommended /etc/ssh/sshd_config settings:

# Disable root login
PermitRootLogin no

# Only allow SSH2
Protocol 2

# Disable password auth (use keys only)
PasswordAuthentication no
PubkeyAuthentication yes

# Limit login attempts
MaxAuthTries 3

# Disconnect idle sessions after 15 min
ClientAliveInterval 300
ClientAliveCountMax 3

# Disable X11 forwarding if not needed
X11Forwarding no

# Disable TCP forwarding if not needed
AllowTcpForwarding no

# Only allow specific users/groups
AllowGroups sshusers

# Use strong ciphers
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
SSH_CONFIG
}

harden_kernel() {
    echo ""
    echo "=== Kernel Hardening ==="
    
    cat << 'SYSCTL'
# Recommended /etc/sysctl.conf settings:

# Network security
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1

# IPv6
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_source_route = 0

# Memory protection
kernel.randomize_va_space = 2
kernel.exec-shield = 1
vm.mmap_min_addr = 65536

# Restrict core dumps
fs.suid_dumpable = 0

# Apply with: sysctl -p /etc/sysctl.conf
SYSCTL
}

harden_filesystem() {
    echo ""
    echo "=== Filesystem Hardening ==="
    
    cat << 'FSTAB'
# Secure /etc/fstab mount options:

# /tmp - noexec, nosuid, nodev
tmpfs /tmp tmpfs defaults,noexec,nosuid,nodev 0 0

# /var/tmp - same restrictions
tmpfs /var/tmp tmpfs defaults,noexec,nosuid,nodev 0 0

# /home - noexec
/dev/sda3 /home ext4 defaults,noexec,nosuid,nodev 0 2
FSTAB

    echo ""
    echo "Remove world-writable permissions:"
    cat << 'SCRIPT'
# หา world-writable files
find / -path /proc -prune -o -perm -002 -not -type l -print 2>/dev/null | while read file; do
    chmod o-w "$file"
    echo "Fixed: $file"
done
SCRIPT
}

create_security_policy() {
    echo ""
    echo "=== Security Policy Document ==="
    
    cat << 'POLICY'
# Security Policy v1.0

## Password Policy
- Minimum length: 12 characters
- Must contain: uppercase, lowercase, numbers, special chars
- Max age: 90 days
- No password reuse: last 12 passwords
- Lock after: 5 failed attempts

## SSH Policy
- Root login: DISABLED
- Password auth: DISABLED (keys only)
- Idle timeout: 15 minutes
- Max auth tries: 3
- Allowed groups: sshusers

## Firewall Rules (minimal)
- Incoming: allow 22, 80, 443 only
- Outgoing: allow established, DNS, HTTP/HTTPS
- Default: DROP

## Monitoring
- Failed logins: alert after 5 attempts
- Root commands: log all
- File integrity: daily check
- Security updates: auto-apply
POLICY
}

# ==================== Demo ====================
harden_sshd
harden_kernel
harden_filesystem
create_security_policy

echo ""
echo "Security hardening reference completed"
echo "Apply settings one by one in test environment first!"
```

---

## ขั้นตอนที่ 403: สรุป Part 19 - Security

```bash
#!/usr/bin/env bash
# summary_security.sh

echo "=== สรุป Security Scripts ==="
echo ""
echo "หัวข้อที่ครอบคลุม:"
echo ""
echo "1. Security Best Practices:"
echo "   - set -euo pipefail"
echo "   - Quote ตัวแปรเสมอ"
echo "   - ไม่ใช้ eval"
echo "   - Validate ทุก input"
echo ""
echo "2. Input Validation:"
echo "   - validate_alphanum, validate_email"
echo "   - validate_ip, validate_path"
echo "   - sanitize_shell, sanitize_html"
echo ""
echo "3. Password/Secret Management:"
echo "   - generate_password"
echo "   - hash_password, verify_password"
echo "   - secure_delete"
echo ""
echo "4. File Permissions:"
echo "   - chmod 600/644/700/755"
echo "   - umask 077"
echo "   - SUID/SGID/Sticky"
echo ""
echo "5. Security Audit:"
echo "   - User audit"
echo "   - SSH audit"
echo "   - Filesystem audit"
echo "   - Service audit"
echo ""
echo "6. IDS Basics:"
echo "   - Brute force detection"
echo "   - Suspicious processes"
echo "   - File integrity"
echo ""
echo "Next: Part 20 - Performance Tuning"
```

---

## สรุป

| หัวข้อ | Steps |
|--------|-------|
| Security best practices | 396 |
| Input validation | 397 |
| Password management | 398 |
| File permissions | 399 |
| Security audit | 400 |
| Intrusion detection | 401 |
| Workshop: Hardening | 402 |

**ขั้นตอนต่อไป**: Part 20 - Performance Tuning
