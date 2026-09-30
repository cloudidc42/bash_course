# Part 37: Advanced Container Patterns และ Optimization

## Module 3: Advanced Level - Container Optimization และ Advanced Patterns

---

## ขั้นตอนที่ 506: Multi-stage Docker Build Optimization

การ Optimize Docker Images เป็นสิ่งสำคัญสำหรับ Production Systems

```bash
#!/bin/bash
# docker-optimizer.sh - Docker Image Optimization Manager

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Analyze image layers
analyze_image() {
    local image="${1:-}"
    
    if [[ -z "$image" ]]; then
        error "Image name required"
        return 1
    fi
    
    log "Analyzing image: $image"
    
    if ! command -v dive &>/dev/null; then
        log "Installing dive..."
        curl -L https://github.com/wagoodman/dive/releases/latest/download/dive_linux_amd64.tar.gz | \
            tar xz -C /usr/local/bin dive
    fi
    
    echo "=== Image Layers ==="
    docker history "$image" --human --format "table {{.CreatedBy}}\t{{.Size}}"
    
    echo ""
    echo "=== Image Size ==="
    docker images "$image" --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
    
    echo ""
    echo "=== Dive Analysis ==="
    CI=true dive "$image" --ci-config /dev/stdin <<< "rules:
  lowestEfficiency: 0.9
  highestWastedBytes: 20MB
  highestUserWastedPercent: 0.20" 2>/dev/null || true
}

# Generate optimized Dockerfile for Node.js
generate_nodejs_dockerfile() {
    local app_name="${1:-my-app}"
    local node_version="${2:-20}"
    local port="${3:-3000}"
    local package_manager="${4:-npm}"
    
    log "Generating optimized Node.js Dockerfile for: $app_name"
    
    cat <<EOF > "Dockerfile.${app_name}"
# Stage 1: Dependencies
FROM node:${node_version}-alpine AS deps
WORKDIR /app

# Install system deps needed for native modules
RUN apk add --no-cache python3 make g++

COPY package*.json ./
RUN ${package_manager} ci --only=production --frozen-lockfile && \\
    ${package_manager} cache clean --force

# Stage 2: Build (if TypeScript or build step)
FROM node:${node_version}-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN ${package_manager} ci --frozen-lockfile

COPY . .
RUN ${package_manager} run build 2>/dev/null || echo "No build step"

# Stage 3: Production
FROM node:${node_version}-alpine AS production

# Security: Create non-root user
RUN addgroup --system --gid 1001 nodejs && \\
    adduser --system --uid 1001 nextjs

WORKDIR /app

# Copy only production files
COPY --from=deps --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist 2>/dev/null || true
COPY --from=builder --chown=nextjs:nodejs /app/src ./src 2>/dev/null || true
COPY --chown=nextjs:nodejs package.json .

# Security settings
USER nextjs

EXPOSE ${port}
ENV NODE_ENV=production \\
    PORT=${port}

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \\
    CMD node -e "require('http').get('http://localhost:${port}/health', r => r.statusCode === 200 ? process.exit(0) : process.exit(1))" || exit 1

CMD ["node", "src/index.js"]
EOF
    
    success "Dockerfile created: Dockerfile.${app_name}"
}

# Generate optimized Dockerfile for Python
generate_python_dockerfile() {
    local app_name="${1:-my-python-app}"
    local python_version="${2:-3.12}"
    local port="${3:-8000}"
    local framework="${4:-fastapi}"
    
    log "Generating optimized Python Dockerfile: $app_name"
    
    cat <<EOF > "Dockerfile.${app_name}"
# Stage 1: Base with deps
FROM python:${python_version}-slim AS base

ENV PYTHONDONTWRITEBYTECODE=1 \\
    PYTHONUNBUFFERED=1 \\
    PIP_NO_CACHE_DIR=1 \\
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# Install system dependencies
RUN apt-get update && \\
    apt-get install -y --no-install-recommends \\
    curl \\
    && rm -rf /var/lib/apt/lists/*

# Stage 2: Install dependencies
FROM base AS deps
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Stage 3: Production
FROM base AS production

# Create non-root user
RUN groupadd --gid 1000 appuser && \\
    useradd --uid 1000 --gid appuser --shell /bin/bash --create-home appuser

# Copy only installed packages
COPY --from=deps /usr/local/lib/python${python_version}/site-packages /usr/local/lib/python${python_version}/site-packages
COPY --from=deps /usr/local/bin /usr/local/bin

COPY --chown=appuser:appuser . .

USER appuser

EXPOSE ${port}

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \\
    CMD curl -f http://localhost:${port}/health || exit 1

$(case "$framework" in
    fastapi) echo 'CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "'$port'", "--workers", "4"]' ;;
    django) echo 'CMD ["gunicorn", "--bind", "0.0.0.0:'$port'", "--workers", "4", "myapp.wsgi"]' ;;
    flask) echo 'CMD ["gunicorn", "--bind", "0.0.0.0:'$port'", "--workers", "4", "app:app"]' ;;
    *) echo 'CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "'$port'"]' ;;
esac)
EOF
    
    success "Dockerfile created: Dockerfile.${app_name}"
}

# Generate optimized Dockerfile for Go
generate_go_dockerfile() {
    local app_name="${1:-my-go-app}"
    local go_version="${2:-1.22}"
    local port="${3:-8080}"
    local binary_name="${4:-app}"
    
    log "Generating optimized Go Dockerfile: $app_name"
    
    cat <<EOF > "Dockerfile.${app_name}"
# Stage 1: Build
FROM golang:${go_version}-alpine AS builder

# Install build deps
RUN apk add --no-cache ca-certificates tzdata git

WORKDIR /build

# Download dependencies first (cache layer)
COPY go.mod go.sum ./
RUN go mod download

# Build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \\
    go build -a -installsuffix cgo \\
    -ldflags='-w -s -extldflags "-static"' \\
    -o ${binary_name} ./cmd/...

# Stage 2: Minimal runtime
FROM scratch

# Copy required files
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /build/${binary_name} /${binary_name}

USER nobody

EXPOSE ${port}

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \\
    CMD ["/bin/sh", "-c", "wget -qO- http://localhost:${port}/health || exit 1"]

ENTRYPOINT ["/${binary_name}"]
EOF
    
    success "Dockerfile created: Dockerfile.${app_name}"
}

# Docker image security scan
scan_image_security() {
    local image="${1:-}"
    local severity="${2:-HIGH,CRITICAL}"
    local fail_on="${3:-CRITICAL}"
    
    if [[ -z "$image" ]]; then
        error "Image required"
        return 1
    fi
    
    log "Scanning image security: $image"
    
    if command -v trivy &>/dev/null; then
        trivy image \
            --severity "$severity" \
            --exit-code "$([ "$fail_on" == "none" ] && echo 0 || echo 1)" \
            --format table \
            "$image"
    elif command -v grype &>/dev/null; then
        grype "$image" --fail-on "$fail_on"
    elif command -v snyk &>/dev/null; then
        snyk container test "$image" --severity-threshold="$(echo "$fail_on" | tr '[:upper:]' '[:lower:]')"
    else
        log "No vulnerability scanner found (trivy, grype, snyk)"
        log "Install with: curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin"
        return 1
    fi
}

# Build image with SBOM
build_with_sbom() {
    local image="${1:-}"
    local dockerfile="${2:-Dockerfile}"
    local sbom_format="${3:-spdx-json}"
    local output_file="${4:-sbom.json}"
    
    if [[ -z "$image" ]]; then
        error "Image name required"
        return 1
    fi
    
    log "Building image with SBOM: $image"
    
    # Build image
    docker buildx build \
        --platform linux/amd64,linux/arm64 \
        --sbom=true \
        --provenance=true \
        --tag "$image" \
        --file "$dockerfile" \
        --push .
    
    # Generate SBOM
    if command -v syft &>/dev/null; then
        syft "$image" -o "$sbom_format" > "$output_file"
        log "SBOM generated: $output_file"
    fi
    
    success "Image built with provenance: $image"
}

# Container resource profiling
profile_container() {
    local container="${1:-}"
    local duration="${2:-60}"
    
    if [[ -z "$container" ]]; then
        error "Container name or ID required"
        return 1
    fi
    
    log "Profiling container: $container for ${duration}s"
    
    echo "=== Initial Stats ==="
    docker stats "$container" --no-stream \
        --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}"
    
    echo ""
    echo "=== Monitoring for ${duration}s ==="
    
    local end_time=$((SECONDS + duration))
    local cpu_samples=()
    local mem_samples=()
    
    while [[ $SECONDS -lt $end_time ]]; do
        local stats
        stats=$(docker stats "$container" --no-stream \
            --format "{{.CPUPerc}},{{.MemPerc}}" 2>/dev/null)
        
        if [[ -n "$stats" ]]; then
            local cpu mem
            cpu=$(echo "$stats" | cut -d, -f1 | tr -d '%')
            mem=$(echo "$stats" | cut -d, -f2 | tr -d '%')
            cpu_samples+=("$cpu")
            mem_samples+=("$mem")
        fi
        
        sleep 5
    done
    
    echo ""
    echo "=== Profiling Summary ==="
    echo "Duration: ${duration}s"
    echo "CPU samples: ${#cpu_samples[@]}"
    echo "Memory samples: ${#mem_samples[@]}"
    
    if [[ ${#cpu_samples[@]} -gt 0 ]]; then
        local avg_cpu=0
        for cpu in "${cpu_samples[@]}"; do
            avg_cpu=$(echo "$avg_cpu + $cpu" | bc)
        done
        avg_cpu=$(echo "scale=2; $avg_cpu / ${#cpu_samples[@]}" | bc)
        echo "Average CPU: ${avg_cpu}%"
    fi
}

# Main
case "${1:-help}" in
    analyze) analyze_image "${2:-}" ;;
    nodejs) generate_nodejs_dockerfile "${2:-my-app}" "${3:-20}" "${4:-3000}" "${5:-npm}" ;;
    python) generate_python_dockerfile "${2:-my-python-app}" "${3:-3.12}" "${4:-8000}" "${5:-fastapi}" ;;
    go) generate_go_dockerfile "${2:-my-go-app}" "${3:-1.22}" "${4:-8080}" "${5:-app}" ;;
    scan) scan_image_security "${2:-}" "${3:-HIGH,CRITICAL}" "${4:-CRITICAL}" ;;
    build-sbom) build_with_sbom "${2:-}" "${3:-Dockerfile}" "${4:-spdx-json}" "${5:-sbom.json}" ;;
    profile) profile_container "${2:-}" "${3:-60}" ;;
    *)
        echo "Usage: $0 {analyze|nodejs|python|go|scan|build-sbom|profile}"
        ;;
esac
```

---

## ขั้นตอนที่ 507: Container Runtime Security

```bash
#!/bin/bash
# container-runtime-security.sh - Container Runtime Security Manager

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Setup Falco runtime security
setup_falco() {
    local namespace="${1:-falco}"
    
    log "Setting up Falco runtime security..."
    
    helm repo add falcosecurity https://falcosecurity.github.io/charts
    helm repo update
    
    helm upgrade --install falco falcosecurity/falco \
        -n "$namespace" \
        --create-namespace \
        --set driver.kind=ebpf \
        --set falco.grpc.enabled=true \
        --set falco.grpcOutput.enabled=true \
        --set falcoctl.artifact.follow.enabled=true \
        --set falcosidekick.enabled=true \
        --set falcosidekick.config.slack.webhookurl="" \
        --set falcosidekick.webui.enabled=true \
        --wait
    
    success "Falco installed in namespace: $namespace"
}

# Create custom Falco rules
create_falco_rules() {
    local output_file="${1:-falco-custom-rules.yaml}"
    
    log "Creating custom Falco rules..."
    
    cat <<'EOF' > "$output_file"
# Custom Falco rules for application security
- rule: Detect Shell in Container
  desc: Alert when shell spawned in container
  condition: >
    spawned_process and
    container and
    shell_procs and
    not proc.pname in (shell_binaries) and
    not container.image.repository in (trusted_images)
  output: >
    Shell spawned in container
    (container=%container.name
     image=%container.image.repository:%container.image.tag
     shell=%proc.name
     parent=%proc.pname
     cmdline=%proc.cmdline
     user=%user.name)
  priority: WARNING
  tags: [container, shell, T1059]

- rule: Detect Crypto Mining
  desc: Detect crypto mining processes
  condition: >
    spawned_process and
    (proc.name in (crypto_miners) or
     proc.cmdline contains "--mining" or
     proc.cmdline contains "stratum+tcp")
  output: >
    Possible crypto mining detected
    (proc=%proc.name
     cmdline=%proc.cmdline
     container=%container.name)
  priority: CRITICAL
  tags: [process, mining, T1496]

- rule: Detect Sensitive File Access
  desc: Detect access to sensitive files
  condition: >
    open_read and
    fd.name in (sensitive_files) and
    not proc.name in (allowed_readers) and
    container
  output: >
    Sensitive file opened for reading
    (file=%fd.name
     proc=%proc.name
     container=%container.name
     image=%container.image.repository)
  priority: WARNING
  tags: [filesystem, sensitive, T1552]

- rule: Detect Outbound Connection to Suspicious Port
  desc: Detect outbound connections to suspicious ports
  condition: >
    outbound and
    fd.sport in (suspicious_ports) and
    not proc.name in (allowed_outbound) and
    container
  output: >
    Suspicious outbound connection
    (proc=%proc.name
     connection=%fd.name
     container=%container.name)
  priority: WARNING
  tags: [network, suspicious, T1071]

- macro: suspicious_ports
  condition: fd.sport in (4444, 1234, 6666, 7777, 8888, 9999, 31337)

- macro: sensitive_files
  condition: >
    fd.name in (/etc/passwd, /etc/shadow, /etc/sudoers,
                /root/.ssh/id_rsa, /root/.aws/credentials)

- macro: allowed_readers
  condition: proc.name in (cat, less, more, grep, awk)

- macro: allowed_outbound
  condition: proc.name in (curl, wget, node, python, java)

- macro: crypto_miners
  condition: proc.name in (xmrig, minerd, cpuminer, ethminer, cgminer, bfgminer)
EOF
    
    success "Custom Falco rules created: $output_file"
}

# Setup AppArmor profile
create_apparmor_profile() {
    local profile_name="${1:-my-container-profile}"
    local output_file="${2:-/etc/apparmor.d/$profile_name}"
    
    log "Creating AppArmor profile: $profile_name"
    
    cat <<EOF > "$output_file"
#include <tunables/global>

profile ${profile_name} flags=(attach_disconnected) {
  #include <abstractions/base>
  #include <abstractions/nameservice>
  
  # Allow process execution
  /usr/** rix,
  /bin/** rix,
  /sbin/** rix,
  
  # Network access
  network inet tcp,
  network inet udp,
  network inet6 tcp,
  network inet6 udp,
  
  # Allow reading system files
  /etc/ld.so.cache r,
  /etc/ld.so.conf r,
  /etc/ld.so.conf.d/ r,
  /etc/ld.so.conf.d/*.conf r,
  /etc/localtime r,
  /etc/timezone r,
  /etc/nsswitch.conf r,
  /etc/host.conf r,
  /etc/hosts r,
  /etc/resolv.conf r,
  
  # App-specific paths (adjust as needed)
  /app/** r,
  /app/** ix,
  /tmp/ rw,
  /tmp/** rw,
  /var/log/app/** rw,
  
  # Deny sensitive paths
  deny /etc/shadow r,
  deny /etc/passwd w,
  deny /root/** r,
  deny /proc/sysrq-trigger rw,
  deny /proc/mem rw,
  deny @{PROC}/@{pid}/mem rw,
  deny /dev/mem rw,
  deny /dev/kmem rw,
  deny /dev/port rw,
  
  # Allow specific capabilities
  capability dac_read_search,
  capability setuid,
  capability setgid,
  
  # Deny dangerous capabilities
  deny capability sys_admin,
  deny capability sys_ptrace,
  deny capability sys_module,
  deny capability sys_rawio,
}
EOF
    
    if [[ -d /etc/apparmor.d ]]; then
        apparmor_parser -r "$output_file" 2>/dev/null || \
            log "AppArmor profile created but not loaded (no AppArmor kernel support)"
    fi
    
    success "AppArmor profile created: $output_file"
}

# Seccomp profile
create_seccomp_profile() {
    local profile_name="${1:-my-seccomp}"
    local output_dir="${2:-/var/lib/kubelet/seccomp}"
    
    mkdir -p "$output_dir"
    
    log "Creating Seccomp profile: $profile_name"
    
    cat <<'EOF' > "${output_dir}/${profile_name}.json"
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_AARCH64"],
  "syscalls": [
    {
      "names": [
        "accept", "accept4", "access", "adjtimex", "alarm",
        "bind", "brk", "capget", "capset", "chdir",
        "chmod", "chown", "chroot", "clock_getres", "clock_gettime",
        "clock_nanosleep", "close", "connect", "copy_file_range",
        "creat", "dup", "dup2", "dup3", "epoll_create",
        "epoll_create1", "epoll_ctl", "epoll_pwait", "epoll_wait",
        "eventfd", "eventfd2", "execve", "execveat", "exit",
        "exit_group", "faccessat", "fadvise64", "fallocate",
        "fanotify_mark", "fchdir", "fchmod", "fchmodat",
        "fchown", "fchownat", "fcntl", "fdatasync", "fgetxattr",
        "flistxattr", "flock", "fork", "fremovexattr", "fsetxattr",
        "fstat", "fstatfs", "fsync", "ftruncate", "futex",
        "getcpu", "getcwd", "getdents", "getdents64", "getegid",
        "geteuid", "getgid", "getgroups", "getitimer", "getpeername",
        "getpgid", "getpgrp", "getpid", "getppid", "getpriority",
        "getrandom", "getresgid", "getresuid", "getrlimit",
        "getrusage", "getsid", "getsockname", "getsockopt",
        "gettid", "gettimeofday", "getuid", "getxattr",
        "inotify_add_watch", "inotify_init", "inotify_init1",
        "inotify_rm_watch", "io_cancel", "io_destroy", "io_getevents",
        "io_setup", "io_submit", "ioctl", "kill", "lchown",
        "lgetxattr", "link", "linkat", "listen", "listxattr",
        "llistxattr", "lremovexattr", "lseek", "lsetxattr",
        "lstat", "madvise", "memfd_create", "mkdir", "mkdirat",
        "mknod", "mknodat", "mlock", "mlock2", "mlockall",
        "mmap", "mount", "mprotect", "mremap", "msgctl",
        "msgget", "msgrcv", "msgsnd", "msync", "munlock",
        "munlockall", "munmap", "nanosleep", "newfstatat",
        "open", "openat", "pause", "pipe", "pipe2",
        "poll", "ppoll", "prctl", "pread64", "preadv",
        "preadv2", "prlimit64", "pselect6", "pwrite64",
        "pwritev", "pwritev2", "read", "readlink", "readlinkat",
        "readv", "recv", "recvfrom", "recvmmsg", "recvmsg",
        "rename", "renameat", "renameat2", "restart_syscall",
        "rmdir", "rt_sigaction", "rt_sigpending", "rt_sigprocmask",
        "rt_sigqueueinfo", "rt_sigreturn", "rt_sigsuspend",
        "rt_sigtimedwait", "rt_tgsigqueueinfo", "sched_getaffinity",
        "sched_getattr", "sched_getparam", "sched_get_priority_max",
        "sched_get_priority_min", "sched_getscheduler",
        "sched_setaffinity", "sched_yield", "select", "semctl",
        "semget", "semop", "semtimedop", "send", "sendfile",
        "sendmmsg", "sendmsg", "sendto", "set_robust_list",
        "set_tid_address", "setfsgid", "setfsuid", "setgid",
        "setgroups", "setitimer", "setpgid", "setpriority",
        "setregid", "setresgid", "setresuid", "setreuid",
        "setrlimit", "setsid", "setsockopt", "setuid",
        "setxattr", "shmat", "shmctl", "shmdt", "shmget",
        "shutdown", "sigaltstack", "signalfd", "signalfd4",
        "socket", "socketpair", "splice", "stat", "statfs",
        "statx", "symlink", "symlinkat", "sync", "sync_file_range",
        "syncfs", "sysinfo", "tgkill", "time", "timer_create",
        "timer_delete", "timer_getoverrun", "timer_gettime",
        "timer_settime", "timerfd_create", "timerfd_gettime",
        "timerfd_settime", "times", "tkill", "truncate",
        "umask", "uname", "unlink", "unlinkat", "utime",
        "utimensat", "utimes", "vfork", "vmsplice", "wait4",
        "waitid", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
EOF
    
    success "Seccomp profile created: ${output_dir}/${profile_name}.json"
}

# Runtime security monitoring
monitor_runtime_security() {
    local namespace="${1:-default}"
    local duration="${2:-60}"
    
    log "Monitoring runtime security for ${duration}s in namespace: $namespace"
    
    echo "=== Suspicious Activities Monitor ==="
    
    # Check for privileged containers
    echo ""
    echo "Privileged containers:"
    kubectl get pods -n "$namespace" -o json | \
        jq -r '.items[] | select(.spec.containers[].securityContext.privileged==true) | .metadata.name' \
        2>/dev/null || echo "None found"
    
    # Check for containers running as root
    echo ""
    echo "Containers running as root (uid=0):"
    kubectl get pods -n "$namespace" -o json | \
        jq -r '.items[] | select(.spec.securityContext.runAsUser==0 or .spec.containers[].securityContext.runAsUser==0) | .metadata.name' \
        2>/dev/null || echo "None found"
    
    # Check for containers with host namespaces
    echo ""
    echo "Containers with host namespaces:"
    kubectl get pods -n "$namespace" -o json | \
        jq -r '.items[] | select(.spec.hostNetwork==true or .spec.hostPID==true or .spec.hostIPC==true) | .metadata.name' \
        2>/dev/null || echo "None found"
    
    # Check Falco alerts
    echo ""
    echo "Recent Falco alerts:"
    kubectl logs -n falco -l app.kubernetes.io/name=falco --since="${duration}s" 2>/dev/null | \
        grep -i "WARNING\|CRITICAL\|NOTICE" | tail -20 || echo "No Falco alerts or Falco not installed"
}

# Main
case "${1:-help}" in
    falco) setup_falco "${2:-falco}" ;;
    falco-rules) create_falco_rules "${2:-falco-custom-rules.yaml}" ;;
    apparmor) create_apparmor_profile "${2:-my-container-profile}" "${3:-/tmp/apparmor-profile}" ;;
    seccomp) create_seccomp_profile "${2:-my-seccomp}" "${3:-/tmp/seccomp}" ;;
    monitor) monitor_runtime_security "${2:-default}" "${3:-60}" ;;
    *)
        echo "Usage: $0 {falco|falco-rules|apparmor|seccomp|monitor}"
        ;;
esac
```

---

## ขั้นตอนที่ 508: Advanced Container Networking

```bash
#!/bin/bash
# container-networking-advanced.sh - Advanced Container Networking

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Setup Cilium CNI
setup_cilium() {
    local version="${1:-1.14.0}"
    local mode="${2:-vxlan}"  # vxlan, native, geneve
    
    log "Installing Cilium CNI..."
    
    helm repo add cilium https://helm.cilium.io/
    helm repo update
    
    helm upgrade --install cilium cilium/cilium \
        --version "$version" \
        -n kube-system \
        --set tunnel="$mode" \
        --set kubeProxyReplacement=strict \
        --set k8sServiceHost="$(kubectl get endpoints kubernetes -o jsonpath='{.subsets[0].addresses[0].ip}')" \
        --set k8sServicePort="$(kubectl get endpoints kubernetes -o jsonpath='{.subsets[0].ports[0].port}')" \
        --set hubble.relay.enabled=true \
        --set hubble.ui.enabled=true \
        --set prometheus.enabled=true \
        --set operator.prometheus.enabled=true \
        --wait
    
    success "Cilium installed"
}

# Create Cilium Network Policy
create_cilium_policy() {
    local name="${1:-my-policy}"
    local namespace="${2:-default}"
    local app_label="${3:-my-app}"
    
    log "Creating Cilium Network Policy: $name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: ${name}
  namespace: ${namespace}
spec:
  description: "${name} network policy"
  endpointSelector:
    matchLabels:
      app: ${app_label}
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v.*"
        - method: "POST"
          path: "/api/v.*/.*"
  - fromEntities:
    - "health"
    toPorts:
    - ports:
      - port: "9090"
        protocol: TCP
  egress:
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: kube-system
        k8s:k8s-app: kube-dns
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
      rules:
        dns:
        - matchPattern: "*"
  - toFQDNs:
    - matchPattern: "*.example.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
  - toServices:
    - k8sService:
        serviceName: my-backend
        namespace: ${namespace}
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP
EOF
    
    success "Cilium Network Policy created: $name"
}

# Service mesh traffic visualization with Hubble
view_hubble_traffic() {
    local namespace="${1:-default}"
    local duration="${2:-30}"
    
    log "Viewing Hubble traffic flows in namespace: $namespace"
    
    if ! command -v hubble &>/dev/null; then
        log "Installing Hubble CLI..."
        curl -L --remote-name-all https://github.com/cilium/hubble/releases/latest/download/hubble-linux-amd64.tar.gz
        tar xf hubble-linux-amd64.tar.gz -C /usr/local/bin hubble
        rm hubble-linux-amd64.tar.gz
    fi
    
    # Port-forward Hubble relay
    kubectl port-forward -n kube-system svc/hubble-relay 4245:80 &
    local pf_pid=$!
    
    sleep 2
    
    hubble observe \
        --namespace "$namespace" \
        --follow \
        --since "${duration}s" \
        --output json 2>/dev/null | \
    jq -r '[.time, .source.namespace, .source.pod_name, "->", .destination.namespace, .destination.pod_name, .l4.TCP.destination_port // .l4.UDP.destination_port, .verdict] | join(" ")' || true
    
    kill "$pf_pid" 2>/dev/null || true
}

# Multi-NIC pod setup
setup_multus_cni() {
    log "Setting up Multus CNI for multi-NIC pods..."
    
    kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml 2>/dev/null || true
    
    # Create NetworkAttachmentDefinition for secondary NIC
    cat <<EOF | kubectl apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
  namespace: default
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "mynet",
    "type": "macvlan",
    "master": "eth1",
    "mode": "bridge",
    "ipam": {
      "type": "host-local",
      "subnet": "192.168.1.0/24",
      "rangeStart": "192.168.1.200",
      "rangeEnd": "192.168.1.250",
      "routes": [
        { "dst": "0.0.0.0/0" }
      ],
      "gateway": "192.168.1.1"
    }
  }'
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-conf
  namespace: default
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "sriov-net",
    "type": "sriov",
    "ipam": {
      "type": "host-local",
      "subnet": "10.56.217.0/24",
      "routes": [{
        "dst": "0.0.0.0/0"
      }],
      "gateway": "10.56.217.1"
    }
  }'
EOF
    
    success "Multus CNI configured"
}

# Pod with multiple NICs
create_multi_nic_pod() {
    local name="${1:-multi-nic-pod}"
    local namespace="${2:-default}"
    local additional_networks="${3:-default/macvlan-conf}"
    
    log "Creating multi-NIC pod: $name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ${name}
  namespace: ${namespace}
  annotations:
    k8s.v1.cni.cncf.io/networks: ${additional_networks}
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sh", "-c", "ip addr show && sleep 3600"]
    securityContext:
      runAsNonRoot: false
EOF
    
    success "Multi-NIC pod created: $name"
    log "Verify with: kubectl exec $name -n $namespace -- ip addr show"
}

# Network performance testing
test_network_performance() {
    local server_pod="${1:-netperf-server}"
    local client_pod="${2:-netperf-client}"
    local namespace="${3:-default}"
    
    log "Testing network performance between pods..."
    
    # Deploy netperf server
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ${server_pod}
  namespace: ${namespace}
  labels:
    role: netperf-server
spec:
  containers:
  - name: netperf
    image: networkstatic/netperf:latest
    command: ["netserver", "-D"]
    ports:
    - containerPort: 12865
---
apiVersion: v1
kind: Service
metadata:
  name: ${server_pod}
  namespace: ${namespace}
spec:
  selector:
    role: netperf-server
  ports:
  - port: 12865
EOF

    # Wait for server
    kubectl wait pod "$server_pod" -n "$namespace" \
        --for=condition=Ready \
        --timeout=60s
    
    # Deploy client and run test
    kubectl run "$client_pod" \
        -n "$namespace" \
        --image=networkstatic/netperf:latest \
        --restart=Never \
        -- sh -c "netperf -H ${server_pod}.${namespace}.svc.cluster.local -t TCP_RR -l 30 -v 2"
    
    # Wait for test completion
    kubectl wait pod "$client_pod" -n "$namespace" \
        --for=condition=Completed \
        --timeout=120s 2>/dev/null || true
    
    # Get results
    kubectl logs "$client_pod" -n "$namespace" 2>/dev/null
    
    # Cleanup
    kubectl delete pod "$server_pod" "$client_pod" -n "$namespace" --ignore-not-found=true
    kubectl delete service "$server_pod" -n "$namespace" --ignore-not-found=true
    
    success "Network performance test complete"
}

# Main
case "${1:-help}" in
    cilium) setup_cilium "${2:-1.14.0}" "${3:-vxlan}" ;;
    cilium-policy) create_cilium_policy "${2:-my-policy}" "${3:-default}" "${4:-my-app}" ;;
    hubble) view_hubble_traffic "${2:-default}" "${3:-30}" ;;
    multus) setup_multus_cni ;;
    multi-nic-pod) create_multi_nic_pod "${2:-multi-nic-pod}" "${3:-default}" "${4:-default/macvlan-conf}" ;;
    perf-test) test_network_performance "${2:-netperf-server}" "${3:-netperf-client}" "${4:-default}" ;;
    *)
        echo "Usage: $0 {cilium|cilium-policy|hubble|multus|multi-nic-pod|perf-test}"
        ;;
esac
```

---

## ขั้นตอนที่ 509: Container Storage Advanced Patterns

```bash
#!/bin/bash
# container-storage-advanced.sh - Advanced Container Storage Management

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; }
success() { echo "[SUCCESS] $*"; }

# Setup Rook-Ceph distributed storage
setup_rook_ceph() {
    local namespace="${1:-rook-ceph}"
    local replica_count="${2:-3}"
    
    log "Setting up Rook-Ceph distributed storage..."
    
    helm repo add rook-release https://charts.rook.io/release
    helm repo update
    
    # Install Rook Operator
    helm upgrade --install rook-ceph rook-release/rook-ceph \
        -n "$namespace" \
        --create-namespace \
        --wait
    
    # Create CephCluster
    cat <<EOF | kubectl apply -f -
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: ${namespace}
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.0
    allowUnsupported: false
  dataDirHostPath: /var/lib/rook
  skipUpgradeChecks: false
  continueUpgradeAfterChecksEvenIfNotHealthy: false
  mon:
    count: ${replica_count}
    allowMultiplePerNode: false
  mgr:
    count: 2
    modules:
    - name: pg_autoscaler
      enabled: true
  dashboard:
    enabled: true
    ssl: true
  monitoring:
    enabled: true
    rulesNamespace: rook-ceph
  network:
    provider: host
  crashCollector:
    disable: false
  cleanupPolicy:
    confirmation: ""
    sanitizeDisks:
      method: quick
      dataSource: zero
      iteration: 1
    allowUninstallWithVolumes: false
  placement:
    all:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: role
              operator: In
              values:
              - storage-node
      tolerations:
      - key: storage-node
        operator: Exists
  resources:
    mgr:
      limits:
        cpu: "1"
        memory: "1Gi"
      requests:
        cpu: "500m"
        memory: "512Mi"
    mon:
      limits:
        cpu: "2"
        memory: "2Gi"
      requests:
        cpu: "1"
        memory: "1Gi"
    osd:
      limits:
        cpu: "2"
        memory: "4Gi"
      requests:
        cpu: "1"
        memory: "4Gi"
  storage:
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^sdb"
    config:
      osdsPerDevice: "1"
EOF
    
    success "Rook-Ceph setup initiated"
    log "Monitor with: kubectl get cephcluster -n $namespace"
}

# Create Ceph Block Storage Class
create_ceph_storage_class() {
    local name="${1:-ceph-block}"
    local namespace="${2:-rook-ceph}"
    local replica_count="${3:-3}"
    
    log "Creating Ceph Block Storage Class: $name"
    
    # Create CephBlockPool
    cat <<EOF | kubectl apply -f -
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ${name}-pool
  namespace: ${namespace}
spec:
  failureDomain: host
  replicated:
    size: ${replica_count}
    requireSafeReplicaSize: true
  parameters:
    compression_mode: none
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ${name}
provisioner: ${namespace}.rbd.csi.ceph.com
parameters:
  clusterID: ${namespace}
  pool: ${name}-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: ${namespace}
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: ${namespace}
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: ${namespace}
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
    
    success "Ceph Block Storage Class created: $name"
}

# Setup Velero for backup
setup_velero_backup() {
    local provider="${1:-aws}"
    local bucket="${2:-my-backup-bucket}"
    local region="${3:-us-east-1}"
    local namespace="${4:-velero}"
    
    log "Setting up Velero backup with $provider..."
    
    if ! command -v velero &>/dev/null; then
        log "Installing Velero CLI..."
        curl -L https://github.com/vmware-tanzu/velero/releases/latest/download/velero-linux-amd64.tar.gz | \
            tar xz --wildcards '*/velero' -O > /usr/local/bin/velero
        chmod +x /usr/local/bin/velero
    fi
    
    case "$provider" in
        aws)
            velero install \
                --provider aws \
                --plugins velero/velero-plugin-for-aws:v1.8.0 \
                --bucket "$bucket" \
                --backup-location-config "region=$region" \
                --snapshot-location-config "region=$region" \
                --namespace "$namespace" \
                --use-node-agent
            ;;
        gcp)
            velero install \
                --provider gcp \
                --plugins velero/velero-plugin-for-gcp:v1.8.0 \
                --bucket "$bucket" \
                --namespace "$namespace"
            ;;
    esac
    
    success "Velero installed with $provider"
}

# Create backup schedule
create_backup_schedule() {
    local name="${1:-daily-backup}"
    local schedule="${2:-0 2 * * *}"  # Daily at 2am
    local namespace="${3:-}"
    local ttl="${4:-168h0m0s}"  # 7 days
    
    log "Creating backup schedule: $name"
    
    local ns_arg=""
    if [[ -n "$namespace" ]]; then
        ns_arg="--include-namespaces $namespace"
    fi
    
    velero schedule create "$name" \
        --schedule="$schedule" \
        --ttl "$ttl" \
        $ns_arg \
        --include-cluster-resources=true \
        --storage-location default
    
    success "Backup schedule created: $name (cron: $schedule)"
}

# Restore from backup
restore_backup() {
    local backup_name="${1:-}"
    local restore_name="${2:-restore-$(date +%Y%m%d%H%M%S)}"
    local namespace="${3:-}"
    
    if [[ -z "$backup_name" ]]; then
        log "Available backups:"
        velero backup get
        return 0
    fi
    
    log "Restoring from backup: $backup_name"
    
    local ns_arg=""
    if [[ -n "$namespace" ]]; then
        ns_arg="--include-namespaces $namespace"
    fi
    
    velero restore create "$restore_name" \
        --from-backup "$backup_name" \
        $ns_arg \
        --wait
    
    success "Restore complete: $restore_name"
}

# PVC snapshot management
create_pvc_snapshot() {
    local pvc_name="${1:-}"
    local snapshot_name="${2:-${pvc_name}-$(date +%Y%m%d%H%M%S)}"
    local namespace="${3:-default}"
    local snapshot_class="${4:-csi-aws-vss}"
    
    if [[ -z "$pvc_name" ]]; then
        error "PVC name required"
        return 1
    fi
    
    log "Creating snapshot of PVC: $pvc_name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: ${snapshot_name}
  namespace: ${namespace}
spec:
  volumeSnapshotClassName: ${snapshot_class}
  source:
    persistentVolumeClaimName: ${pvc_name}
EOF
    
    success "Snapshot created: $snapshot_name"
}

# Restore PVC from snapshot
restore_pvc_from_snapshot() {
    local snapshot_name="${1:-}"
    local new_pvc_name="${2:-${snapshot_name}-restored}"
    local namespace="${3:-default}"
    local storage_size="${4:-10Gi}"
    local storage_class="${5:-}"
    
    if [[ -z "$snapshot_name" ]]; then
        error "Snapshot name required"
        return 1
    fi
    
    log "Restoring PVC from snapshot: $snapshot_name"
    
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ${new_pvc_name}
  namespace: ${namespace}
spec:
  dataSource:
    name: ${snapshot_name}
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: ${storage_size}
  ${storage_class:+storageClassName: $storage_class}
EOF
    
    success "PVC restored from snapshot: $new_pvc_name"
}

# Main
case "${1:-help}" in
    setup-ceph) setup_rook_ceph "${2:-rook-ceph}" "${3:-3}" ;;
    ceph-sc) create_ceph_storage_class "${2:-ceph-block}" "${3:-rook-ceph}" "${4:-3}" ;;
    setup-velero) setup_velero_backup "${2:-aws}" "${3:-my-backup-bucket}" "${4:-us-east-1}" ;;
    backup-schedule) create_backup_schedule "${2:-daily-backup}" "${3:-0 2 * * *}" "${4:-}" "${5:-168h0m0s}" ;;
    restore) restore_backup "${2:-}" "${3:-}" "${4:-}" ;;
    snapshot) create_pvc_snapshot "${2:-}" "${3:-}" "${4:-default}" "${5:-csi-aws-vss}" ;;
    restore-pvc) restore_pvc_from_snapshot "${2:-}" "${3:-}" "${4:-default}" "${5:-10Gi}" ;;
    *)
        echo "Usage: $0 {setup-ceph|ceph-sc|setup-velero|backup-schedule|restore|snapshot|restore-pvc}"
        ;;
esac
```

---

## สรุป Part 37

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | เครื่องมือ | ขั้นตอน |
|--------|-----------|---------|
| Docker Image Optimization | Multi-stage builds, Dive, SBOM | 506 |
| Container Runtime Security | Falco, AppArmor, Seccomp | 507 |
| Advanced Container Networking | Cilium, Hubble, Multus | 508 |
| Container Storage | Rook-Ceph, Velero, PVC Snapshots | 509 |

**เทคโนโลยีที่ใช้:**
- Dive: Docker layer analyzer
- Trivy/Grype: Container security scanner
- Falco: Runtime security monitoring
- Cilium: eBPF-based networking
- Multus: Multi-NIC pod support
- Rook-Ceph: Distributed storage
- Velero: Kubernetes backup/restore

ขั้นตอนต่อไป: Part 38 - Advanced Monitoring และ Observability Patterns
