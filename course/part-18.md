# Part 18: System Administration Scripts

## Module 2: Intermediate Level (ระดับกลาง)

---

## ขั้นตอนที่ 388: System Information

```bash
#!/usr/bin/env bash
# system_info.sh - System Information

set -euo pipefail

echo "=== System Administration Scripts ==="

# ==================== System Information ====================

get_os_info() {
    echo "OS Information:"
    
    if [[ -f /etc/os-release ]]; then
        source /etc/os-release
        echo "  OS: $PRETTY_NAME"
        echo "  Version: ${VERSION:-unknown}"
    fi
    
    echo "  Kernel: $(uname -r)"
    echo "  Architecture: $(uname -m)"
    echo "  Hostname: $(hostname)"
    echo "  Uptime: $(uptime -p 2>/dev/null || uptime)"
}

get_hardware_info() {
    echo "Hardware Information:"
    
    # CPU
    local cpu_model cores
    cpu_model=$(grep "model name" /proc/cpuinfo 2>/dev/null | head -1 | cut -d: -f2 | xargs)
    cores=$(nproc 2>/dev/null || grep -c "^processor" /proc/cpuinfo 2>/dev/null || echo "unknown")
    echo "  CPU: ${cpu_model:-unknown} (${cores} cores)"
    
    # RAM
    if [[ -f /proc/meminfo ]]; then
        local total_mem
        total_mem=$(awk '/MemTotal/ {printf "%.1f GB", $2/1024/1024}' /proc/meminfo)
        echo "  RAM: $total_mem"
    fi
    
    # Disk
    echo "  Disk:"
    df -h 2>/dev/null | awk 'NR>1 && /^\// {printf "    %-20s %5s used of %5s (%s)\n", $6, $3, $2, $5}'
}

get_network_info() {
    echo "Network Information:"
    
    # IP addresses
    ip addr show 2>/dev/null | awk '/inet / && !/127\.0\.0\.1/ {print "  IP:", $2}' | head -5
    
    # Default gateway
    local gateway
    gateway=$(ip route show default 2>/dev/null | awk '{print $3}' | head -1)
    echo "  Gateway: ${gateway:-unknown}"
    
    # DNS
    echo "  DNS servers:"
    grep "^nameserver" /etc/resolv.conf 2>/dev/null | awk '{print "    " $2}'
}

get_resource_usage() {
    echo "Resource Usage:"
    
    # CPU usage
    local cpu_usage
    cpu_usage=$(top -bn1 2>/dev/null | grep "Cpu(s)" | awk '{print $2}' | cut -d% -f1 || echo "N/A")
    echo "  CPU: ${cpu_usage}%"
    
    # Memory
    if [[ -f /proc/meminfo ]]; then
        awk '/MemTotal|MemAvailable/ {a[$1]=$2}
             END {
                 used = a["MemTotal:"] - a["MemAvailable:"]
                 pct = (used/a["MemTotal:"])*100
                 printf "  Memory: %.1f%% used (%.1f/%.1f GB)\n",
                        pct, used/1024/1024, a["MemTotal:"]/1024/1024
             }' /proc/meminfo
    fi
    
    # Load average
    echo "  Load: $(cat /proc/loadavg 2>/dev/null | awk '{print $1, $2, $3}')"
    
    # Disk I/O
    echo "  Open files: $(lsof 2>/dev/null | wc -l | tr -d ' ') (approx)"
}

# Main
echo "========================================"
get_os_info
echo ""
get_hardware_info
echo ""
get_network_info
echo ""
get_resource_usage
echo "========================================"
```

---

## ขั้นตอนที่ 389: User Management

```bash
#!/usr/bin/env bash
# user_management.sh - User Management

echo "=== User Management ==="

# ==================== User Functions ====================

# ดูข้อมูล user
user_info() {
    local username="${1:-$USER}"
    
    if ! id "$username" &>/dev/null; then
        echo "User '$username' ไม่พบ"
        return 1
    fi
    
    echo "User: $username"
    echo "  UID:    $(id -u "$username")"
    echo "  GID:    $(id -g "$username")"
    echo "  Groups: $(id -Gn "$username" | tr ' ' ', ')"
    echo "  Home:   $(getent passwd "$username" | cut -d: -f6)"
    echo "  Shell:  $(getent passwd "$username" | cut -d: -f7)"
    
    # Last login
    if command -v last &>/dev/null; then
        local last_login
        last_login=$(last "$username" 2>/dev/null | head -1 | awk '{print $3, $4, $5, $6}')
        echo "  Last:   ${last_login:-never}"
    fi
}

# สร้าง user
create_user() {
    local username="$1"
    local groups="${2:-}"  # comma-separated
    local home_dir="${3:-/home/$username}"
    local shell="${4:-/bin/bash}"
    
    # ต้องเป็น root
    if [[ $EUID -ne 0 ]]; then
        echo "Error: ต้องเป็น root เพื่อสร้าง user"
        return 1
    fi
    
    if id "$username" &>/dev/null; then
        echo "User '$username' มีอยู่แล้ว"
        return 1
    fi
    
    # สร้าง user
    useradd \
        -m \
        -d "$home_dir" \
        -s "$shell" \
        "$username"
    
    # เพิ่ม groups ถ้ามี
    if [[ -n "$groups" ]]; then
        usermod -aG "$groups" "$username"
    fi
    
    echo "✓ User '$username' created"
}

# ตั้งค่า SSH key
setup_ssh_key() {
    local username="$1"
    local public_key="$2"
    local home_dir
    home_dir=$(getent passwd "$username" | cut -d: -f6)
    
    local ssh_dir="$home_dir/.ssh"
    local auth_keys="$ssh_dir/authorized_keys"
    
    mkdir -p "$ssh_dir"
    echo "$public_key" >> "$auth_keys"
    
    chmod 700 "$ssh_dir"
    chmod 600 "$auth_keys"
    chown -R "$username:$username" "$ssh_dir"
    
    echo "✓ SSH key added for $username"
}

echo "1. Current user info:"
user_info "$USER"

echo ""
echo "2. Active users:"
who 2>/dev/null || w 2>/dev/null | head -10

echo ""
echo "3. Last logged in users:"
last 2>/dev/null | head -10 || echo "(last ไม่พร้อม)"

echo ""
echo "4. Failed login attempts:"
grep "Failed password\|Invalid user" /var/log/auth.log 2>/dev/null | \
    tail -5 | awk '{print $1, $2, $3, $9, $11}' || \
    echo "(ไม่มีสิทธิ์อ่าน auth.log)"

echo ""
echo "5. Sudoers:"
grep -v "^#" /etc/sudoers 2>/dev/null | grep -v "^$" | head -5 || \
    echo "(ไม่มีสิทธิ์อ่าน sudoers)"
```

---

## ขั้นตอนที่ 390: Backup Script

```bash
#!/usr/bin/env bash
# backup.sh - Comprehensive Backup System

set -euo pipefail

# ==================== Configuration ====================
BACKUP_NAME="${BACKUP_NAME:-myapp}"
BACKUP_DIR="${BACKUP_DIR:-/tmp/backups}"
RETENTION_DAYS="${RETENTION_DAYS:-7}"
COMPRESS="${COMPRESS:-true}"
ENCRYPT="${ENCRYPT:-false}"

# Colors
GREEN='\033[0;32m'; RED='\033[0;31m'; YELLOW='\033[1;33m'; NC='\033[0m'

# ==================== Logging ====================
log_info()  { echo -e "${GREEN}[$(date '+%H:%M:%S')] INFO:${NC} $*"; }
log_warn()  { echo -e "${YELLOW}[$(date '+%H:%M:%S')] WARN:${NC} $*"; }
log_error() { echo -e "${RED}[$(date '+%H:%M:%S')] ERROR:${NC} $*" >&2; }

# ==================== Backup Functions ====================

backup_directory() {
    local source_dir="$1"
    local backup_name="${2:-$(basename "$source_dir")}"
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)
    local archive_name="${backup_name}_${timestamp}.tar"
    
    if [[ ! -d "$source_dir" ]]; then
        log_error "Source directory ไม่พบ: $source_dir"
        return 1
    fi
    
    mkdir -p "$BACKUP_DIR"
    
    log_info "Backing up: $source_dir"
    
    # Create archive
    local archive_path="$BACKUP_DIR/$archive_name"
    
    if [[ "$COMPRESS" == "true" ]]; then
        archive_path="${archive_path}.gz"
        tar -czf "$archive_path" -C "$(dirname "$source_dir")" "$(basename "$source_dir")" 2>/dev/null
    else
        tar -cf "$archive_path" -C "$(dirname "$source_dir")" "$(basename "$source_dir")" 2>/dev/null
    fi
    
    local size
    size=$(du -sh "$archive_path" | cut -f1)
    log_info "✓ Backup created: $archive_path ($size)"
    
    echo "$archive_path"
}

backup_database() {
    local db_type="${1:-postgres}"
    local db_name="$2"
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)
    local dump_file="$BACKUP_DIR/${db_name}_${timestamp}.sql"
    
    mkdir -p "$BACKUP_DIR"
    
    case "$db_type" in
        postgres|postgresql)
            log_info "Backing up PostgreSQL: $db_name"
            PGPASSWORD="${DB_PASS:-}" pg_dump \
                -h "${DB_HOST:-localhost}" \
                -U "${DB_USER:-postgres}" \
                "$db_name" > "$dump_file" 2>/dev/null && \
                log_info "✓ PostgreSQL dump: $dump_file" || \
                log_error "PostgreSQL dump failed"
            ;;
        mysql)
            log_info "Backing up MySQL: $db_name"
            mysqldump \
                -h "${DB_HOST:-localhost}" \
                -u "${DB_USER:-root}" \
                -p"${DB_PASS:-}" \
                "$db_name" > "$dump_file" 2>/dev/null && \
                log_info "✓ MySQL dump: $dump_file" || \
                log_error "MySQL dump failed"
            ;;
        sqlite)
            log_info "Backing up SQLite: $db_name"
            cp "$db_name" "${dump_file%.sql}.db" 2>/dev/null && \
                log_info "✓ SQLite backup" || \
                log_error "SQLite backup failed"
            ;;
        *)
            log_error "Unknown database type: $db_type"
            return 1
            ;;
    esac
}

cleanup_old_backups() {
    local dir="${1:-$BACKUP_DIR}"
    local days="${2:-$RETENTION_DAYS}"
    
    log_info "Cleaning backups older than $days days..."
    
    local count=0
    while IFS= read -r -d '' file; do
        rm -f "$file"
        (( count++ ))
    done < <(find "$dir" -name "*.tar*" -o -name "*.sql" -mtime +"$days" -print0 2>/dev/null)
    
    log_info "✓ Removed $count old backup(s)"
}

verify_backup() {
    local backup_file="$1"
    
    if [[ ! -f "$backup_file" ]]; then
        log_error "Backup file ไม่พบ: $backup_file"
        return 1
    fi
    
    case "$backup_file" in
        *.tar.gz)
            if tar -tzf "$backup_file" &>/dev/null; then
                log_info "✓ Backup verified: $backup_file"
                return 0
            fi
            ;;
        *.tar)
            if tar -tf "$backup_file" &>/dev/null; then
                log_info "✓ Backup verified: $backup_file"
                return 0
            fi
            ;;
        *.sql)
            if [[ -s "$backup_file" ]]; then
                log_info "✓ SQL backup exists: $backup_file"
                return 0
            fi
            ;;
    esac
    
    log_error "Backup verification failed: $backup_file"
    return 1
}

# ==================== Main ====================
echo "=== Backup System ==="
echo ""

echo "Configuration:"
echo "  Backup name: $BACKUP_NAME"
echo "  Backup dir:  $BACKUP_DIR"
echo "  Retention:   $RETENTION_DAYS days"
echo "  Compress:    $COMPRESS"
echo ""

# Demo backup
log_info "Starting backup..."

# สร้างข้อมูลทดสอบ
mkdir -p /tmp/test_app/{config,data,logs}
echo "config content" > /tmp/test_app/config/app.conf
echo "some data" > /tmp/test_app/data/records.csv
echo "2024-01-15 INFO App started" > /tmp/test_app/logs/app.log

# Backup directory
backup_file=$(backup_directory "/tmp/test_app" "test_app_backup")

echo ""
log_info "Verifying backup..."
verify_backup "$backup_file"

echo ""
log_info "Cleanup old backups..."
cleanup_old_backups "$BACKUP_DIR" 30

echo ""
log_info "Backup completed!"

# Cleanup demo
rm -rf /tmp/test_app /tmp/backups
```

---

## ขั้นตอนที่ 391: System Monitoring Dashboard

```bash
#!/usr/bin/env bash
# system_dashboard.sh - System Monitoring Dashboard

echo "=== System Monitoring Dashboard ==="

# ==================== Display Functions ====================

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'
BLUE='\033[0;34m'; CYAN='\033[0;36m'; BOLD='\033[1m'; NC='\033[0m'

draw_bar() {
    local pct="$1"
    local width="${2:-30}"
    local filled=$(( pct * width / 100 ))
    
    local bar="["
    for ((i=0; i<filled; i++)); do bar+="█"; done
    for ((i=filled; i<width; i++)); do bar+="░"; done
    bar+="]"
    
    local color="$GREEN"
    if (( pct > 80 )); then color="$RED"
    elif (( pct > 60 )); then color="$YELLOW"
    fi
    
    printf "${color}%s${NC} %3d%%" "$bar" "$pct"
}

get_cpu_pct() {
    top -bn1 2>/dev/null | grep "Cpu(s)" | \
        awk '{for(i=1;i<=NF;i++) if($i~/\%us/) {gsub(/[^0-9.]/,"",$i); print int($i+0.5); exit}}' || \
    awk '/^cpu / {idle=$5; total=0; for(i=2;i<=NF;i++) total+=$i; print int((1-idle/total)*100)}' /proc/stat 2>/dev/null || \
    echo 0
}

get_mem_pct() {
    awk '/MemTotal|MemAvailable/ {a[$1]=$2}
         END {printf "%.0f", (1-a["MemAvailable:"]/a["MemTotal:"])*100}' \
        /proc/meminfo 2>/dev/null || echo 0
}

get_disk_pct() {
    local mount="${1:-/}"
    df "$mount" 2>/dev/null | awk 'NR==2 {gsub(/%/,"",$5); print $5}' || echo 0
}

get_load_avg() {
    awk '{print $1}' /proc/loadavg 2>/dev/null || uptime | awk '{print $(NF-2)}' | tr -d ','
}

# ==================== Dashboard ====================

print_dashboard() {
    local cpu_pct mem_pct disk_pct load
    cpu_pct=$(get_cpu_pct)
    mem_pct=$(get_mem_pct)
    disk_pct=$(get_disk_pct "/")
    load=$(get_load_avg)
    
    echo ""
    echo -e "${BOLD}${BLUE}╔═══════════════════════════════════════════════╗${NC}"
    echo -e "${BOLD}${BLUE}║           System Monitor Dashboard            ║${NC}"
    echo -e "${BOLD}${BLUE}╠═══════════════════════════════════════════════╣${NC}"
    printf "${BLUE}║${NC} %-10s $(date '+%Y-%m-%d %H:%M:%S') Uptime: %-8s ${BLUE}║${NC}\n" \
        "Host: $(hostname)" "$(uptime -p 2>/dev/null | sed 's/up //' | cut -d, -f1)"
    echo -e "${BLUE}╠═══════════════════════════════════════════════╣${NC}"
    
    # CPU
    printf "${BLUE}║${NC} CPU:    $(draw_bar "$cpu_pct") Load: %-5s ${BLUE}║${NC}\n" "$load"
    
    # Memory
    printf "${BLUE}║${NC} Memory: $(draw_bar "$mem_pct")         ${BLUE}║${NC}\n"
    
    # Disk
    printf "${BLUE}║${NC} Disk /:  $(draw_bar "$disk_pct")         ${BLUE}║${NC}\n"
    
    echo -e "${BLUE}╠═══════════════════════════════════════════════╣${NC}"
    
    # Top processes
    echo -e "${BLUE}║${NC} Top Processes:                                ${BLUE}║${NC}"
    ps aux 2>/dev/null | sort -k3 -rn | awk 'NR>1 && NR<=4 {
        printf "'"${BLUE}║${NC}"' %-8s %-20s CPU: %5s%% MEM: %4s%% '"${BLUE}║${NC}"'\n",
               $2, substr($11,1,20), $3, $4
    }'
    
    echo -e "${BLUE}╠═══════════════════════════════════════════════╣${NC}"
    
    # Network
    if [[ -f /proc/net/dev ]]; then
        echo -e "${BLUE}║${NC} Network:                                      ${BLUE}║${NC}"
        awk 'NR>2 {
            gsub(/:/, "", $1)
            if ($1 != "lo" && $1 != "")
                printf "'"${BLUE}║${NC}"'  %-10s RX: %-8s TX: %-8s          '"${BLUE}║${NC}"'\n",
                       $1, 
                       ($2>1073741824? sprintf("%.1fGB",$2/1073741824) : \
                        $2>1048576? sprintf("%.1fMB",$2/1048576) : \
                        sprintf("%.0fKB",$2/1024)),
                       ($10>1073741824? sprintf("%.1fGB",$10/1073741824) : \
                        $10>1048576? sprintf("%.1fMB",$10/1048576) : \
                        sprintf("%.0fKB",$10/1024))
        }' /proc/net/dev | head -3
    fi
    
    echo -e "${BLUE}╚═══════════════════════════════════════════════╝${NC}"
}

print_dashboard
```

---

## ขั้นตอนที่ 392: Log Rotation

```bash
#!/usr/bin/env bash
# log_rotation.sh - Log Rotation

set -euo pipefail

echo "=== Log Rotation ==="

# ==================== Log Rotation ====================

rotate_log() {
    local log_file="$1"
    local max_size_mb="${2:-10}"
    local max_files="${3:-5}"
    
    if [[ ! -f "$log_file" ]]; then
        echo "Log file ไม่พบ: $log_file"
        return 0
    fi
    
    local size_mb
    size_mb=$(du -m "$log_file" 2>/dev/null | cut -f1)
    
    if (( size_mb < max_size_mb )); then
        echo "Log ยังเล็กอยู่ (${size_mb}MB < ${max_size_mb}MB), ยังไม่ต้อง rotate"
        return 0
    fi
    
    echo "Rotating $log_file (${size_mb}MB)..."
    
    # Rotate existing backups
    for ((i=max_files-1; i>=1; i--)); do
        local j=$(( i + 1 ))
        if [[ -f "${log_file}.${i}" ]]; then
            mv "${log_file}.${i}" "${log_file}.${j}"
        fi
        if [[ -f "${log_file}.${i}.gz" ]]; then
            mv "${log_file}.${i}.gz" "${log_file}.${j}.gz"
        fi
    done
    
    # Move current log to .1
    cp "$log_file" "${log_file}.1"
    
    # Compress
    gzip -f "${log_file}.1" 2>/dev/null && echo "  Compressed: ${log_file}.1.gz"
    
    # Clear current log (truncate, not delete - processes may still write to it)
    > "$log_file"
    
    echo "✓ Log rotated"
    
    # Remove oldest if exceeds max
    local oldest="${log_file}.${max_files}.gz"
    if [[ -f "$oldest" ]]; then
        rm -f "$oldest"
        echo "  Removed oldest: $oldest"
    fi
}

compress_logs() {
    local log_dir="$1"
    local days_old="${2:-1}"
    
    echo "Compressing logs older than $days_old days in $log_dir..."
    
    local count=0
    while IFS= read -r -d '' file; do
        gzip "$file" && (( count++ ))
    done < <(find "$log_dir" -name "*.log" -not -name "*.gz" -mtime +"$days_old" -print0 2>/dev/null)
    
    echo "✓ Compressed $count files"
}

cleanup_logs() {
    local log_dir="$1"
    local days_old="${2:-30}"
    
    echo "Removing logs older than $days_old days..."
    
    local count
    count=$(find "$log_dir" -name "*.log*" -mtime +"$days_old" -delete -print 2>/dev/null | wc -l)
    echo "✓ Removed $count old log files"
}

# Demo
mkdir -p /tmp/demo_logs

# สร้าง log file ขนาดเล็ก
for i in $(seq 1 20); do
    echo "2024-01-15 10:0$i:00 INFO Log entry $i" >> /tmp/demo_logs/app.log
done

echo "Log size before: $(du -sh /tmp/demo_logs/app.log)"
rotate_log "/tmp/demo_logs/app.log" 0  # 0 MB เพื่อ force rotation

echo ""
echo "Files after rotation:"
ls -la /tmp/demo_logs/

# Cleanup
rm -rf /tmp/demo_logs
```

---

## ขั้นตอนที่ 393: Package Management Automation

```bash
#!/usr/bin/env bash
# package_management.sh - Package Management

echo "=== Package Management Automation ==="

# ตรวจสอบ package manager
detect_package_manager() {
    if command -v apt &>/dev/null; then
        echo "apt"
    elif command -v yum &>/dev/null; then
        echo "yum"
    elif command -v dnf &>/dev/null; then
        echo "dnf"
    elif command -v pacman &>/dev/null; then
        echo "pacman"
    elif command -v brew &>/dev/null; then
        echo "brew"
    else
        echo "unknown"
    fi
}

PM=$(detect_package_manager)
echo "Package manager: $PM"
echo ""

# ==================== Universal Package Functions ====================

pkg_install() {
    local packages=("$@")
    
    echo "Installing: ${packages[*]}"
    
    case "$PM" in
        apt)
            sudo apt-get install -y "${packages[@]}"
            ;;
        yum)
            sudo yum install -y "${packages[@]}"
            ;;
        dnf)
            sudo dnf install -y "${packages[@]}"
            ;;
        pacman)
            sudo pacman -S --noconfirm "${packages[@]}"
            ;;
        brew)
            brew install "${packages[@]}"
            ;;
        *)
            echo "Unknown package manager"
            return 1
            ;;
    esac
}

pkg_update() {
    echo "Updating system..."
    
    case "$PM" in
        apt)
            sudo apt-get update -q
            sudo apt-get upgrade -y
            ;;
        yum)
            sudo yum update -y
            ;;
        dnf)
            sudo dnf update -y
            ;;
        pacman)
            sudo pacman -Syu --noconfirm
            ;;
        brew)
            brew update && brew upgrade
            ;;
    esac
}

pkg_is_installed() {
    local package="$1"
    
    case "$PM" in
        apt)
            dpkg -l "$package" 2>/dev/null | grep -q "^ii"
            ;;
        yum|dnf)
            rpm -q "$package" &>/dev/null
            ;;
        pacman)
            pacman -Qi "$package" &>/dev/null
            ;;
        brew)
            brew list "$package" &>/dev/null
            ;;
        *)
            command -v "$package" &>/dev/null
            ;;
    esac
}

ensure_installed() {
    local package="$1"
    local command_name="${2:-$package}"
    
    if ! command -v "$command_name" &>/dev/null; then
        if ! pkg_is_installed "$package" 2>/dev/null; then
            echo "Installing $package..."
            pkg_install "$package" 2>/dev/null || echo "Cannot install $package (no sudo)"
        fi
    else
        echo "✓ $package already installed"
    fi
}

echo "Checking required tools:"
for tool in curl jq git vim; do
    if command -v "$tool" &>/dev/null; then
        echo "  ✓ $tool: $(command -v $tool)"
    else
        echo "  ✗ $tool: not installed"
    fi
done

echo ""
echo "System packages (sample):"
case "$PM" in
    apt)
        dpkg -l 2>/dev/null | grep "^ii" | wc -l | xargs echo "  Total installed packages:"
        ;;
    brew)
        brew list 2>/dev/null | wc -l | xargs echo "  Total installed packages:"
        ;;
esac
```

---

## ขั้นตอนที่ 394: Workshop - System Setup Script

```bash
#!/usr/bin/env bash
# system_setup.sh - Workshop: Complete System Setup

set -euo pipefail

# ==================== System Setup Script ====================

echo "=== System Setup Script ==="
echo ""

# Colors
GREEN='\033[0;32m'; RED='\033[0;31m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; NC='\033[0m'
BOLD='\033[1m'

log_step()    { echo -e "\n${BOLD}${BLUE}▶ $*${NC}"; }
log_success() { echo -e "  ${GREEN}✓ $*${NC}"; }
log_warn()    { echo -e "  ${YELLOW}⚠ $*${NC}"; }
log_error()   { echo -e "  ${RED}✗ $*${NC}" >&2; }
log_info()    { echo -e "  ℹ $*"; }

# ==================== Configuration ====================
REQUIRED_TOOLS=(git curl jq vim)
OPTIONAL_TOOLS=(docker tmux htop tree)
DIRECTORIES=("/opt/apps" "/var/log/myapp" "/etc/myapp")
MIN_DISK_GB=10
MIN_RAM_MB=512

# ==================== Checks ====================

check_os() {
    log_step "Checking OS compatibility"
    
    local os_id
    os_id=$(. /etc/os-release 2>/dev/null && echo "$ID" || uname -s)
    
    case "$os_id" in
        ubuntu|debian)
            log_success "Ubuntu/Debian detected"
            ;;
        centos|rhel|fedora)
            log_success "RHEL family detected"
            ;;
        darwin)
            log_success "macOS detected"
            ;;
        *)
            log_warn "Unknown OS: $os_id (may work)"
            ;;
    esac
    
    echo "  OS: $(. /etc/os-release 2>/dev/null && echo "$PRETTY_NAME" || uname -s)"
    echo "  Kernel: $(uname -r)"
}

check_resources() {
    log_step "Checking system resources"
    
    # Disk space
    local avail_gb
    avail_gb=$(df -BG / 2>/dev/null | awk 'NR==2 {gsub(/G/,""); print $4}' || echo 0)
    if (( avail_gb >= MIN_DISK_GB )); then
        log_success "Disk space: ${avail_gb}GB available"
    else
        log_error "Insufficient disk space: ${avail_gb}GB (minimum ${MIN_DISK_GB}GB)"
        return 1
    fi
    
    # RAM
    if [[ -f /proc/meminfo ]]; then
        local ram_mb
        ram_mb=$(awk '/MemTotal/ {print int($2/1024)}' /proc/meminfo)
        if (( ram_mb >= MIN_RAM_MB )); then
            log_success "RAM: ${ram_mb}MB available"
        else
            log_warn "Low RAM: ${ram_mb}MB (recommended ${MIN_RAM_MB}MB)"
        fi
    fi
}

check_tools() {
    log_step "Checking required tools"
    
    local missing=()
    for tool in "${REQUIRED_TOOLS[@]}"; do
        if command -v "$tool" &>/dev/null; then
            log_success "$tool: $(command -v $tool)"
        else
            log_error "Missing: $tool"
            missing+=("$tool")
        fi
    done
    
    log_info "Optional tools:"
    for tool in "${OPTIONAL_TOOLS[@]}"; do
        if command -v "$tool" &>/dev/null; then
            log_success "$tool: installed"
        else
            log_warn "$tool: not installed (optional)"
        fi
    done
    
    if (( ${#missing[@]} > 0 )); then
        log_error "Missing required tools: ${missing[*]}"
        return 1
    fi
}

create_directories() {
    log_step "Creating directories"
    
    for dir in "${DIRECTORIES[@]}"; do
        if mkdir -p "$dir" 2>/dev/null; then
            log_success "Created: $dir"
        else
            log_warn "Cannot create: $dir (may need sudo)"
        fi
    done
}

setup_config() {
    log_step "Setting up configuration"
    
    local config_dir
    config_dir="/tmp/myapp_config_$(date +%s)"
    mkdir -p "$config_dir"
    
    # สร้าง default config
    cat > "$config_dir/app.conf" << 'EOF'
# Application Configuration
APP_NAME=myapp
APP_VERSION=1.0.0
APP_PORT=8080
LOG_LEVEL=INFO
DEBUG=false
MAX_CONNECTIONS=100
TIMEOUT=30
EOF
    
    log_success "Config created: $config_dir/app.conf"
    cat "$config_dir/app.conf"
    
    # Cleanup
    rm -rf "$config_dir"
}

run_checks() {
    log_step "Running system checks"
    
    local all_ok=true
    
    check_os || true
    check_resources || all_ok=false
    check_tools || all_ok=false
    
    if [[ "$all_ok" == "true" ]]; then
        echo ""
        log_success "All checks passed!"
        return 0
    else
        echo ""
        log_error "Some checks failed. Review above."
        return 1
    fi
}

# ==================== Main ====================
run_checks || true
echo ""
create_directories || true
echo ""
setup_config

echo ""
echo -e "${GREEN}${BOLD}Setup completed!${NC}"
```

---

## ขั้นตอนที่ 395: สรุป Part 18 - System Administration

```bash
#!/usr/bin/env bash
# summary_sysadmin.sh

echo "=== สรุป System Administration ==="
echo ""
echo "Topics covered:"
echo ""
echo "1. System Information:"
echo "   - OS, hardware, network info"
echo "   - Resource usage (CPU, memory, disk)"
echo ""
echo "2. User Management:"
echo "   - user_info, create_user"
echo "   - SSH key setup"
echo "   - Login history"
echo ""
echo "3. Backup System:"
echo "   - Directory backups"
echo "   - Database dumps"
echo "   - Verification and retention"
echo ""
echo "4. Monitoring Dashboard:"
echo "   - Visual resource bars"
echo "   - Top processes"
echo "   - Network stats"
echo ""
echo "5. Log Rotation:"
echo "   - rotate_log"
echo "   - compress_logs"
echo "   - cleanup_logs"
echo ""
echo "6. Package Management:"
echo "   - Cross-platform (apt/yum/brew)"
echo "   - ensure_installed"
echo ""
echo "Next: Part 19 - Security Scripts"
```

---

## สรุป

| หัวข้อ | Steps |
|--------|-------|
| System information | 388 |
| User management | 389 |
| Backup system | 390 |
| Monitoring dashboard | 391 |
| Log rotation | 392 |
| Package management | 393 |
| Workshop | 394 |

**ขั้นตอนต่อไป**: Part 19 - Security Scripts
