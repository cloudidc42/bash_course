# Part 30: Advanced Networking และ Service Mesh

## Module 3: Advanced Level — การทำงานระดับสูง

---

## ขั้นตอนที่ 478: Advanced Networking ด้วย Bash

```bash
#!/bin/bash
# advanced_networking.sh - เครื่องมือ Networking ขั้นสูง

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# === Network Discovery ===

# Scan network range
scan_network() {
    local network="$1"    # e.g., 192.168.1.0/24
    local timeout="${2:-1}"
    
    echo "=== Network Scan: ${network} ==="
    
    # ใช้ nmap ถ้ามี
    if command -v nmap &>/dev/null; then
        nmap -sn "$network" --host-timeout "${timeout}s" 2>/dev/null | \
            grep -E "Nmap scan report|MAC Address" | \
            paste - - | \
            awk '{print $5, $NF}' | \
            sed 's/[()]//g'
        return
    fi
    
    # Fallback: ping sweep
    local base_ip="${network%.*}"
    local start=1
    local end=254
    
    local active_hosts=()
    
    for i in $(seq $start $end); do
        local ip="${base_ip}.${i}"
        if ping -c 1 -W "$timeout" "$ip" &>/dev/null; then
            active_hosts+=("$ip")
            echo "  ✓ ${ip}"
        fi
    done
    
    echo ""
    echo "Active hosts: ${#active_hosts[@]}"
}

# Port scanner
scan_ports() {
    local host="$1"
    local port_range="${2:-1-1024}"
    local timeout="${3:-1}"
    
    echo "=== Port Scan: ${host} (${port_range}) ==="
    
    # ใช้ nmap ถ้ามี
    if command -v nmap &>/dev/null; then
        nmap -sV -p "$port_range" --open "$host" 2>/dev/null
        return
    fi
    
    # Fallback: bash port scan
    local start_port="${port_range%-*}"
    local end_port="${port_range#*-}"
    
    local open_ports=()
    
    for port in $(seq "$start_port" "$end_port"); do
        (echo >/dev/tcp/"$host"/"$port") 2>/dev/null && {
            open_ports+=("$port")
            local service=""
            case "$port" in
                21)   service="FTP" ;;
                22)   service="SSH" ;;
                25)   service="SMTP" ;;
                53)   service="DNS" ;;
                80)   service="HTTP" ;;
                443)  service="HTTPS" ;;
                3306) service="MySQL" ;;
                5432) service="PostgreSQL" ;;
                6379) service="Redis" ;;
                8080) service="HTTP-Alt" ;;
                27017) service="MongoDB" ;;
            esac
            echo "  ${port}/tcp open  ${service}"
        }
    done
    
    echo ""
    echo "Open ports: ${#open_ports[@]}"
}

# Traceroute analysis
trace_route() {
    local host="$1"
    local max_hops="${2:-30}"
    
    echo "=== Traceroute: ${host} ==="
    
    if command -v traceroute &>/dev/null; then
        traceroute -m "$max_hops" "$host" 2>&1
    elif command -v tracepath &>/dev/null; then
        tracepath "$host" 2>&1
    else
        # Bash-based traceroute
        local ttl=1
        while [[ $ttl -le $max_hops ]]; do
            local reply
            reply=$(ping -c 1 -t "$ttl" -W 1 "$host" 2>&1 | grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" | head -1)
            
            if [[ -n "$reply" ]]; then
                local rtt
                rtt=$(ping -c 1 -W 1 "$reply" 2>/dev/null | grep -oE "time=[0-9.]+ ms" | head -1)
                echo "  ${ttl}: ${reply} ${rtt}"
                
                [[ "$reply" == "$host" || $(dig +short "$host" 2>/dev/null) == "$reply" ]] && break
            else
                echo "  ${ttl}: * (no response)"
            fi
            
            ((ttl++))
        done
    fi
}

# DNS analysis
analyze_dns() {
    local domain="$1"
    
    echo "=== DNS Analysis: ${domain} ==="
    
    # A records
    echo "A Records:"
    dig +short A "$domain" 2>/dev/null | while read -r ip; do
        local country=""
        # ดู geo location ถ้าทำได้
        echo "  ${ip}"
    done
    
    # MX records
    echo ""
    echo "MX Records:"
    dig +short MX "$domain" 2>/dev/null | while read -r record; do
        echo "  ${record}"
    done
    
    # NS records
    echo ""
    echo "NS Records:"
    dig +short NS "$domain" 2>/dev/null | while read -r ns; do
        echo "  ${ns}"
    done
    
    # TXT records (includes SPF, DKIM)
    echo ""
    echo "TXT Records:"
    dig +short TXT "$domain" 2>/dev/null | while read -r txt; do
        echo "  ${txt}"
    done
    
    # CNAME
    echo ""
    echo "CNAME Records:"
    dig +short CNAME "$domain" 2>/dev/null | while read -r cname; do
        echo "  ${cname}"
    done
    
    # TTL info
    echo ""
    echo "TTL Information:"
    dig A "$domain" 2>/dev/null | grep -E "^${domain}" | awk '{print "  " $1 " TTL:" $2 " " $4 " " $5}'
}

# Network bandwidth test
test_bandwidth() {
    local host="${1:-8.8.8.8}"
    
    echo "=== Bandwidth Test ==="
    
    # ใช้ iperf3 ถ้ามี
    if command -v iperf3 &>/dev/null; then
        echo "Running iperf3 to ${host}..."
        iperf3 -c "$host" -t 10 2>&1
        return
    fi
    
    # Fallback: curl speed test
    echo "Download speed test..."
    local speed
    speed=$(curl -o /dev/null -s \
        --max-time 10 \
        -w "%{speed_download}" \
        "http://speedtest.wdc01.softlayer.com/downloads/test10.zip" 2>/dev/null || echo "0")
    
    echo "Download speed: $(echo "scale=2; $speed / 1048576" | bc) MB/s"
    
    # Latency test
    echo ""
    echo "Latency test to ${host}..."
    ping -c 10 "$host" 2>/dev/null | tail -2
}

# SSL/TLS analysis
analyze_ssl() {
    local host="$1"
    local port="${2:-443}"
    
    echo "=== SSL/TLS Analysis: ${host}:${port} ==="
    
    # Certificate info
    echo "Certificate Information:"
    echo | openssl s_client \
        -servername "$host" \
        -connect "${host}:${port}" \
        2>/dev/null | \
    openssl x509 -noout \
        -subject \
        -issuer \
        -dates \
        -fingerprint \
        2>/dev/null
    
    # Supported protocols
    echo ""
    echo "Protocol Support:"
    for proto in ssl2 ssl3 tls1 tls1_1 tls1_2 tls1_3; do
        if echo | openssl s_client \
            -"$proto" \
            -servername "$host" \
            -connect "${host}:${port}" \
            2>/dev/null | grep -q "BEGIN CERTIFICATE"; then
            
            case "$proto" in
                ssl2|ssl3|tls1|tls1_1) echo -e "  ${RED}✗ ${proto} ENABLED (insecure)${NC}" ;;
                tls1_2|tls1_3)         echo -e "  ${GREEN}✓ ${proto} ENABLED${NC}" ;;
            esac
        else
            case "$proto" in
                ssl2|ssl3|tls1|tls1_1) echo -e "  ${GREEN}✓ ${proto} disabled${NC}" ;;
                tls1_2|tls1_3)         echo -e "  ${YELLOW}  ${proto} not available${NC}" ;;
            esac
        fi
    done
    
    # Cipher suites
    echo ""
    echo "Cipher Suites (top 5):"
    echo | openssl s_client \
        -servername "$host" \
        -connect "${host}:${port}" \
        2>/dev/null | \
    grep "Cipher is" | head -5
}

# Network monitoring
monitor_network() {
    local interface="${1:-$(ip route | grep default | awk '{print $5}' | head -1)}"
    local interval="${2:-5}"
    local duration="${3:-60}"
    
    echo "=== Network Monitor: ${interface} (${duration}s) ==="
    
    local start_time=$(date +%s)
    
    # รับ initial counters
    local prev_rx prev_tx
    prev_rx=$(cat /sys/class/net/${interface}/statistics/rx_bytes 2>/dev/null || echo "0")
    prev_tx=$(cat /sys/class/net/${interface}/statistics/tx_bytes 2>/dev/null || echo "0")
    
    while [[ $(($(date +%s) - start_time)) -lt $duration ]]; do
        sleep "$interval"
        
        local curr_rx curr_tx
        curr_rx=$(cat /sys/class/net/${interface}/statistics/rx_bytes 2>/dev/null || echo "0")
        curr_tx=$(cat /sys/class/net/${interface}/statistics/tx_bytes 2>/dev/null || echo "0")
        
        # คำนวณ rate
        local rx_rate=$(( (curr_rx - prev_rx) / interval ))
        local tx_rate=$(( (curr_tx - prev_tx) / interval ))
        
        # Format
        local rx_human tx_human
        rx_human=$(numfmt --to=iec --suffix=B/s "$rx_rate" 2>/dev/null || echo "${rx_rate}B/s")
        tx_human=$(numfmt --to=iec --suffix=B/s "$tx_rate" 2>/dev/null || echo "${tx_rate}B/s")
        
        printf "[%s] RX: %-12s TX: %-12s\n" \
            "$(date +%H:%M:%S)" "$rx_human" "$tx_human"
        
        prev_rx=$curr_rx
        prev_tx=$curr_tx
    done
}
```

---

## ขั้นตอนที่ 479: Service Mesh ด้วย Istio

```bash
#!/bin/bash
# istio_manager.sh - จัดการ Istio Service Mesh

KUBECTL="${KUBECTL:-kubectl}"
ISTIO_NAMESPACE="istio-system"

# ตรวจสอบ Istio installation
check_istio() {
    echo "=== Checking Istio Installation ==="
    
    # ตรวจสอบ control plane
    local components=("istiod" "istio-ingressgateway")
    
    for comp in "${components[@]}"; do
        if $KUBECTL get deployment "$comp" -n "$ISTIO_NAMESPACE" &>/dev/null; then
            local ready
            ready=$($KUBECTL get deployment "$comp" -n "$ISTIO_NAMESPACE" \
                -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo "0")
            echo "  ✓ ${comp}: ${ready} ready"
        else
            echo "  ✗ ${comp}: not found"
        fi
    done
    
    # ตรวจสอบ version
    if command -v istioctl &>/dev/null; then
        echo ""
        istioctl version 2>/dev/null
    fi
}

# Enable sidecar injection
enable_sidecar_injection() {
    local namespace="$1"
    
    echo "=== Enabling Sidecar Injection: ${namespace} ==="
    
    $KUBECTL label namespace "$namespace" \
        istio-injection=enabled \
        --overwrite
    
    echo "✓ Sidecar injection enabled for: ${namespace}"
    
    # Restart pods เพื่อ inject sidecar
    read -rp "Restart all pods in ${namespace}? (y/N): " answer
    if [[ "$answer" =~ ^[Yy]$ ]]; then
        $KUBECTL rollout restart deployment \
            -n "$namespace"
        echo "✓ Pods restarting..."
    fi
}

# สร้าง VirtualService
create_virtual_service() {
    local service_name="$1"
    local namespace="${2:-default}"
    local host="${3:-${service_name}}"
    
    cat << YAML | $KUBECTL apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ${service_name}
  namespace: ${namespace}
spec:
  hosts:
    - ${host}
  http:
    # Retry policy
    - match:
        - uri:
            prefix: /api/v1
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: gateway-error,connect-failure,retriable-4xx
      timeout: 10s
      route:
        - destination:
            host: ${service_name}
            port:
              number: 8080
            subset: v1
          weight: 90
        - destination:
            host: ${service_name}
            port:
              number: 8080
            subset: v2
          weight: 10
    
    # Default route
    - route:
        - destination:
            host: ${service_name}
            port:
              number: 8080
          weight: 100
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
YAML
    
    echo "✓ VirtualService created: ${service_name}"
}

# สร้าง DestinationRule
create_destination_rule() {
    local service_name="$1"
    local namespace="${2:-default}"
    
    cat << YAML | $KUBECTL apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ${service_name}
  namespace: ${namespace}
spec:
  host: ${service_name}
  trafficPolicy:
    # Connection pool settings
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
        tcpKeepalive:
          time: 7200s
          interval: 75s
      http:
        http1MaxPendingRequests: 1
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 3
        idleTimeout: 90s
    
    # Load balancing
    loadBalancer:
      simple: LEAST_CONN
    
    # Outlier detection (Circuit Breaker)
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
    
    # TLS settings
    tls:
      mode: ISTIO_MUTUAL
  
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        connectionPool:
          http:
            http2MaxRequests: 500
    
    - name: v2
      labels:
        version: v2
YAML
    
    echo "✓ DestinationRule created: ${service_name}"
}

# สร้าง Gateway
create_gateway() {
    local gateway_name="$1"
    local namespace="${2:-default}"
    local hosts="${3:-*}"
    
    cat << YAML | $KUBECTL apply -f -
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: ${gateway_name}
  namespace: ${namespace}
spec:
  selector:
    istio: ingressgateway
  servers:
    # HTTP - redirect to HTTPS
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - ${hosts}
      tls:
        httpsRedirect: true
    
    # HTTPS
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - ${hosts}
      tls:
        mode: SIMPLE
        credentialName: ${gateway_name}-tls
YAML
    
    echo "✓ Gateway created: ${gateway_name}"
}

# สร้าง PeerAuthentication (mTLS)
configure_mtls() {
    local namespace="$1"
    local mode="${2:-STRICT}"  # STRICT, PERMISSIVE, DISABLE
    
    echo "=== Configuring mTLS: ${namespace} (${mode}) ==="
    
    cat << YAML | $KUBECTL apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ${namespace}
spec:
  mtls:
    mode: ${mode}
YAML
    
    echo "✓ mTLS configured: ${mode}"
}

# สร้าง AuthorizationPolicy
create_auth_policy() {
    local service_name="$1"
    local namespace="${2:-default}"
    local allowed_services="${3:-}"
    
    cat << YAML | $KUBECTL apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ${service_name}-policy
  namespace: ${namespace}
spec:
  selector:
    matchLabels:
      app: ${service_name}
  rules:
    # Allow from monitoring
    - from:
        - source:
            namespaces:
              - monitoring
            principals:
              - "cluster.local/ns/monitoring/sa/prometheus"
      to:
        - operation:
            paths: ["/metrics"]
    
    # Allow internal traffic
    - from:
        - source:
            namespaces:
              - ${namespace}
      to:
        - operation:
            methods: ["GET", "POST", "PUT", "DELETE"]
            paths: ["/api/*"]
    
    # Deny all others
YAML
    
    echo "✓ AuthorizationPolicy created: ${service_name}"
}

# Traffic management - Circuit Breaker
test_circuit_breaker() {
    local service="$1"
    local namespace="${2:-default}"
    
    echo "=== Testing Circuit Breaker: ${service} ==="
    
    # Deploy fortio load tester
    cat << YAML | $KUBECTL apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: fortio
  namespace: ${namespace}
  labels:
    app: fortio
spec:
  containers:
  - name: fortio
    image: fortio/fortio:latest
    ports:
    - containerPort: 8080
YAML
    
    $KUBECTL wait pod/fortio -n "$namespace" \
        --for=condition=Ready --timeout=30s 2>/dev/null || true
    
    # รัน load test
    echo "Running load test..."
    $KUBECTL exec fortio -n "$namespace" -- \
        fortio load -c 3 -qps 0 -n 30 \
        "http://${service}:8080/api/health" 2>/dev/null | \
    grep -E "Code|Success|Duration"
    
    # Cleanup
    $KUBECTL delete pod fortio -n "$namespace" --ignore-not-found 2>/dev/null
}

# Traffic shifting (canary via Istio)
traffic_shift() {
    local service="$1"
    local namespace="${2:-default}"
    local v1_weight="${3:-90}"
    local v2_weight="${4:-10}"
    
    echo "=== Traffic Shift: ${service} (v1: ${v1_weight}%, v2: ${v2_weight}%) ==="
    
    $KUBECTL patch virtualservice "$service" -n "$namespace" \
        --type='json' \
        -p="[{
            \"op\": \"replace\",
            \"path\": \"/spec/http/0/route\",
            \"value\": [
                {\"destination\": {\"host\": \"${service}\", \"subset\": \"v1\"}, \"weight\": ${v1_weight}},
                {\"destination\": {\"host\": \"${service}\", \"subset\": \"v2\"}, \"weight\": ${v2_weight}}
            ]
        }]" 2>/dev/null
    
    echo "✓ Traffic shifted"
}

# Fault injection
inject_fault() {
    local service="$1"
    local namespace="${2:-default}"
    local fault_type="${3:-delay}"  # delay or abort
    
    echo "=== Injecting Fault: ${fault_type} into ${service} ==="
    
    if [[ "$fault_type" == "delay" ]]; then
        cat << YAML | $KUBECTL apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ${service}-fault
  namespace: ${namespace}
spec:
  hosts:
    - ${service}
  http:
    - fault:
        delay:
          percentage:
            value: 50
          fixedDelay: 5s
      route:
        - destination:
            host: ${service}
YAML
    else
        cat << YAML | $KUBECTL apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ${service}-fault
  namespace: ${namespace}
spec:
  hosts:
    - ${service}
  http:
    - fault:
        abort:
          percentage:
            value: 30
          httpStatus: 503
      route:
        - destination:
            host: ${service}
YAML
    fi
    
    echo "✓ Fault injection configured"
}

# แสดง service mesh status
show_mesh_status() {
    local namespace="${1:-default}"
    
    echo "=== Service Mesh Status: ${namespace} ==="
    
    # Pods with sidecar
    echo "Pods with Envoy sidecar:"
    $KUBECTL get pods -n "$namespace" \
        -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{","}{end}{"\n"}{end}' \
        2>/dev/null | \
    while IFS=$'\t' read -r pod containers; do
        if echo "$containers" | grep -q "istio-proxy"; then
            echo "  ✓ ${pod} (sidecar injected)"
        fi
    done
    
    # VirtualServices
    echo ""
    echo "VirtualServices:"
    $KUBECTL get virtualservices -n "$namespace" \
        --no-headers 2>/dev/null | \
    awk '{print "  " $1 " → " $2}'
    
    # DestinationRules
    echo ""
    echo "DestinationRules:"
    $KUBECTL get destinationrules -n "$namespace" \
        --no-headers 2>/dev/null | \
    awk '{print "  " $1 " → " $2}'
    
    # PeerAuthentication
    echo ""
    echo "PeerAuthentication:"
    $KUBECTL get peerauthentication -n "$namespace" \
        --no-headers 2>/dev/null | \
    awk '{print "  " $1 " → " $2}'
}

# Main
case "${1:-}" in
    check)      check_istio ;;
    inject)     enable_sidecar_injection "${2}" ;;
    vs)         create_virtual_service "${2}" "${3:-default}" "${4:-}" ;;
    dr)         create_destination_rule "${2}" "${3:-default}" ;;
    gateway)    create_gateway "${2}" "${3:-default}" "${4:-*}" ;;
    mtls)       configure_mtls "${2}" "${3:-STRICT}" ;;
    auth)       create_auth_policy "${2}" "${3:-default}" ;;
    cb-test)    test_circuit_breaker "${2}" "${3:-default}" ;;
    shift)      traffic_shift "${2}" "${3:-default}" "${4:-90}" "${5:-10}" ;;
    fault)      inject_fault "${2}" "${3:-default}" "${4:-delay}" ;;
    status)     show_mesh_status "${2:-default}" ;;
    *)
        echo "Usage: $0 {check|inject|vs|dr|gateway|mtls|auth|cb-test|shift|fault|status} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 480: Linkerd Service Mesh

```bash
#!/bin/bash
# linkerd_manager.sh - จัดการ Linkerd

# ติดตั้ง Linkerd CLI
install_linkerd_cli() {
    echo "=== Installing Linkerd CLI ==="
    
    # ดาวน์โหลดและติดตั้ง
    curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
    
    # เพิ่มใน PATH
    export PATH="$HOME/.linkerd2/bin:$PATH"
    
    echo "✓ Linkerd CLI installed: $(linkerd version --client)"
}

# Pre-flight check
linkerd_check() {
    echo "=== Linkerd Pre-flight Check ==="
    linkerd check --pre 2>&1
}

# ติดตั้ง Linkerd
install_linkerd() {
    echo "=== Installing Linkerd ==="
    
    # Generate certificates
    local ca_cert_file="/tmp/ca.crt"
    local ca_key_file="/tmp/ca.key"
    local issuer_cert_file="/tmp/issuer.crt"
    local issuer_key_file="/tmp/issuer.key"
    
    # สร้าง root CA
    step certificate create root.linkerd.cluster.local \
        "$ca_cert_file" "$ca_key_file" \
        --profile root-ca \
        --no-password \
        --insecure 2>/dev/null || {
        
        # Fallback: ใช้ openssl
        openssl genrsa -out "$ca_key_file" 4096 2>/dev/null
        openssl req -x509 -new -nodes \
            -key "$ca_key_file" \
            -sha256 -days 3650 \
            -out "$ca_cert_file" \
            -subj "/CN=root.linkerd.cluster.local" 2>/dev/null
    }
    
    # ติดตั้ง
    linkerd install \
        --identity-trust-anchors-file "$ca_cert_file" \
        --identity-issuer-certificate-file "$issuer_cert_file" \
        --identity-issuer-key-file "$issuer_key_file" | \
    kubectl apply -f -
    
    # ตรวจสอบ
    linkerd check 2>&1 | tail -10
    
    echo "✓ Linkerd installed"
}

# Inject sidecar
inject_linkerd() {
    local resource="$1"
    local namespace="${2:-default}"
    
    echo "=== Injecting Linkerd Proxy: ${resource} ==="
    
    kubectl get "$resource" -n "$namespace" -o yaml | \
        linkerd inject - | \
        kubectl apply -f -
    
    echo "✓ Linkerd proxy injected"
}

# Traffic split (canary)
linkerd_traffic_split() {
    local service="$1"
    local namespace="${2:-default}"
    local canary_weight="${3:-10}"
    local stable_weight=$((100 - canary_weight))
    
    cat << YAML | kubectl apply -f -
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: ${service}-split
  namespace: ${namespace}
spec:
  service: ${service}
  backends:
    - service: ${service}-stable
      weight: ${stable_weight}
    - service: ${service}-canary
      weight: ${canary_weight}
YAML
    
    echo "✓ Traffic split configured: stable=${stable_weight}%, canary=${canary_weight}%"
}

# ดู metrics
linkerd_metrics() {
    local namespace="${1:-default}"
    
    echo "=== Linkerd Metrics: ${namespace} ==="
    
    linkerd stat deployments -n "$namespace" 2>/dev/null
}

# Service profile
create_service_profile() {
    local service="$1"
    local namespace="${2:-default}"
    
    echo "=== Creating Service Profile: ${service} ==="
    
    # Auto-generate from swagger/openapi spec
    # linkerd profile --open-api swagger.json ${service}.${namespace}.svc.cluster.local
    
    cat << YAML | kubectl apply -f -
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: ${service}.${namespace}.svc.cluster.local
  namespace: ${namespace}
spec:
  routes:
    - condition:
        method: GET
        pathRegex: /api/v1/.*
      name: GET /api/v1
      timeout: 300ms
      retryBudget:
        retryRatio: 0.2
        minRetriesPerSecond: 10
        ttl: 10s
    
    - condition:
        method: POST
        pathRegex: /api/v1/.*
      name: POST /api/v1
      timeout: 5s
      isRetryable: false
    
    - condition:
        pathRegex: /health
      name: health check
      timeout: 100ms
YAML
    
    echo "✓ ServiceProfile created: ${service}"
}

# Main
case "${1:-}" in
    install-cli)  install_linkerd_cli ;;
    check)        linkerd_check ;;
    install)      install_linkerd ;;
    inject)       inject_linkerd "${2}" "${3:-default}" ;;
    split)        linkerd_traffic_split "${2}" "${3:-default}" "${4:-10}" ;;
    metrics)      linkerd_metrics "${2:-default}" ;;
    profile)      create_service_profile "${2}" "${3:-default}" ;;
    *)
        echo "Usage: $0 {install-cli|check|install|inject|split|metrics|profile} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 481: Load Balancing และ HAProxy

```bash
#!/bin/bash
# haproxy_manager.sh - จัดการ HAProxy

HAPROXY_CONFIG="/etc/haproxy/haproxy.cfg"
HAPROXY_STATS_SOCKET="/run/haproxy/admin.sock"

# สร้าง HAProxy configuration
generate_haproxy_config() {
    local output_file="${1:-/tmp/haproxy.cfg}"
    
    cat > "$output_file" << 'HAPROXY_CFG'
# HAProxy Configuration
# Generated by haproxy_manager.sh

global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    
    # SSL/TLS defaults
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
    ssl-default-server-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-server-options ssl-min-ver TLSv1.2 no-tls-tickets
    
    # Tuning
    maxconn 50000
    nbthread 4

defaults
    log global
    mode http
    option httplog
    option dontlognull
    option forwardfor
    option http-server-close
    
    timeout connect 5s
    timeout client  50s
    timeout server  50s
    timeout http-request 10s
    timeout http-keep-alive 10s
    
    # Error pages
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

# === Stats ===
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST
    stats show-node
    stats show-legends
    stats show-desc "HAProxy Load Balancer"

# === HTTP Frontend ===
frontend http_front
    bind *:80
    
    # Redirect HTTP to HTTPS
    redirect scheme https code 301 if !{ ssl_fc }
    
    # Default backend
    default_backend http_back

# === HTTPS Frontend ===
frontend https_front
    bind *:443 ssl crt /etc/haproxy/ssl/combined.pem
    
    # Security headers
    http-response set-header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    http-response set-header X-Content-Type-Options "nosniff"
    http-response set-header X-Frame-Options "DENY"
    http-response set-header X-XSS-Protection "1; mode=block"
    
    # Rate limiting
    stick-table type ip size 100k expire 30s store http_req_rate(10s),conn_cur
    acl rate_limited src_http_req_rate(https_front) gt 100
    http-request track-sc0 src
    http-request deny if rate_limited
    
    # ACLs สำหรับ routing
    acl is_api path_beg /api/
    acl is_static path_beg /static/ /assets/
    acl host_api hdr(host) -i api.example.com
    acl host_www hdr(host) -i www.example.com example.com
    
    # Route ตาม host/path
    use_backend api_back if host_api
    use_backend api_back if is_api
    use_backend static_back if is_static
    default_backend web_back

# === API Backend ===
backend api_back
    balance leastconn
    option httpchk GET /health HTTP/1.1\r\nHost:\ api.example.com
    http-check expect status 200
    
    # Compression
    compression algo gzip
    compression type application/json text/plain
    
    # Session persistence (optional)
    cookie SERVERID insert indirect nocache
    
    server api-01 10.0.1.10:8080 check maxconn 200 weight 100 cookie s1
    server api-02 10.0.1.11:8080 check maxconn 200 weight 100 cookie s2
    server api-03 10.0.1.12:8080 check maxconn 200 weight 100 cookie s3
    
    # Backup server
    server api-backup 10.0.1.99:8080 check backup

# === Web Backend ===
backend web_back
    balance roundrobin
    option httpchk GET /health
    
    server web-01 10.0.2.10:3000 check inter 2s rise 2 fall 3
    server web-02 10.0.2.11:3000 check inter 2s rise 2 fall 3
    server web-03 10.0.2.12:3000 check inter 2s rise 2 fall 3

# === Static Files Backend ===
backend static_back
    balance uri
    option httpchk GET /health
    
    server static-01 10.0.3.10:80 check
    server static-02 10.0.3.11:80 check

# === TCP Frontend (for databases, etc.) ===
frontend mysql_front
    bind *:3306
    mode tcp
    option tcplog
    default_backend mysql_back

backend mysql_back
    mode tcp
    balance leastconn
    option mysql-check user haproxy_check
    
    server db-primary 10.0.4.10:3306 check
    server db-replica 10.0.4.11:3306 check
HAPROXY_CFG
    
    echo "✓ HAProxy configuration created: ${output_file}"
}

# ควบคุม HAProxy ผ่าน socket
haproxy_cmd() {
    local cmd="$1"
    echo "$cmd" | socat stdio "$HAPROXY_STATS_SOCKET" 2>/dev/null
}

# แสดง server status
show_server_status() {
    local backend="${1:-}"
    
    echo "=== HAProxy Server Status ==="
    
    haproxy_cmd "show stat" | \
    python3 -c "
import sys, csv

reader = csv.DictReader(sys.stdin, delimiter=',')
current_backend = None

for row in reader:
    backend = row.get('# pxname', '')
    server = row.get('svname', '')
    status = row.get('status', '')
    check = row.get('check_status', '')
    sessions = row.get('scur', '0')
    
    if backend != current_backend:
        print(f'\\n=== {backend} ===')
        current_backend = backend
    
    if server == 'FRONTEND' or server == 'BACKEND':
        print(f'  [{server}] Status: {status} | Sessions: {sessions}')
    else:
        status_icon = '✓' if status == 'UP' else '✗'
        print(f'  {status_icon} {server}: {status} (check: {check}) sessions: {sessions}')
" 2>/dev/null || haproxy_cmd "show stat" | head -30
}

# Enable/disable server
toggle_server() {
    local backend="$1"
    local server="$2"
    local action="$3"  # enable, disable, drain
    
    echo "=== ${action} server: ${backend}/${server} ==="
    
    haproxy_cmd "set server ${backend}/${server} state ${action}" && \
    echo "✓ Server ${action}d"
}

# Dynamic backend management
add_server() {
    local backend="$1"
    local server_name="$2"
    local address="$3"
    local port="$4"
    local weight="${5:-100}"
    
    echo "=== Adding server: ${backend}/${server_name} (${address}:${port}) ==="
    
    haproxy_cmd "add server ${backend}/${server_name} ${address}:${port} weight ${weight}" && \
    echo "✓ Server added"
}

# Health check monitoring
monitor_health() {
    local interval="${1:-5}"
    
    echo "=== HAProxy Health Monitor (interval: ${interval}s) ==="
    
    while true; do
        clear
        echo "=== HAProxy Status: $(date) ==="
        
        # แสดงสรุป
        local total_servers=0
        local up_servers=0
        local down_servers=0
        
        while IFS=',' read -r pxname svname status rest; do
            [[ "$svname" == "FRONTEND" || "$svname" == "BACKEND" || "$svname" == "svname" ]] && continue
            ((total_servers++))
            
            case "$status" in
                UP)   ((up_servers++)) ;;
                DOWN) ((down_servers++)) ;;
            esac
        done < <(haproxy_cmd "show stat" 2>/dev/null | tail -n +2)
        
        echo "Total: ${total_servers} | Up: ${up_servers} | Down: ${down_servers}"
        echo ""
        
        show_server_status
        
        sleep "$interval"
    done
}

# Graceful reload
reload_haproxy() {
    echo "=== Reloading HAProxy ==="
    
    # Validate config
    if haproxy -c -f "$HAPROXY_CONFIG" 2>&1; then
        echo "✓ Configuration valid"
        
        # Reload gracefully
        systemctl reload haproxy 2>/dev/null || \
        service haproxy reload 2>/dev/null || \
        kill -USR2 "$(cat /run/haproxy.pid)" 2>/dev/null
        
        echo "✓ HAProxy reloaded"
    else
        echo "✗ Configuration invalid"
        return 1
    fi
}

# Main
case "${1:-}" in
    generate)   generate_haproxy_config "${2:-}" ;;
    status)     show_server_status "${2:-}" ;;
    enable)     toggle_server "${2}" "${3}" "enable" ;;
    disable)    toggle_server "${2}" "${3}" "disable" ;;
    drain)      toggle_server "${2}" "${3}" "drain" ;;
    add-server) add_server "${2}" "${3}" "${4}" "${5}" "${6:-100}" ;;
    monitor)    monitor_health "${2:-5}" ;;
    reload)     reload_haproxy ;;
    *)
        echo "Usage: $0 {generate|status|enable|disable|drain|add-server|monitor|reload} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 482: Message Queue Integration — RabbitMQ และ Kafka

```bash
#!/bin/bash
# message_queue.sh - จัดการ Message Queues

# === RabbitMQ Management ===

RABBITMQ_HOST="${RABBITMQ_HOST:-localhost}"
RABBITMQ_PORT="${RABBITMQ_PORT:-15672}"
RABBITMQ_USER="${RABBITMQ_USER:-guest}"
RABBITMQ_PASS="${RABBITMQ_PASS:-guest}"
RABBITMQ_API="http://${RABBITMQ_HOST}:${RABBITMQ_PORT}/api"

# เรียก RabbitMQ API
rabbitmq_api() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local curl_args=(
        -s
        -X "$method"
        -u "${RABBITMQ_USER}:${RABBITMQ_PASS}"
        -H "Content-Type: application/json"
    )
    
    [[ -n "$data" ]] && curl_args+=(-d "$data")
    
    curl "${curl_args[@]}" "${RABBITMQ_API}${endpoint}"
}

# แสดงรายการ queues
list_queues() {
    local vhost="${1:-%2F}"
    
    echo "=== RabbitMQ Queues ==="
    
    rabbitmq_api "GET" "/queues/${vhost}" | \
    python3 -c "
import json, sys
queues = json.load(sys.stdin)
print(f'Total: {len(queues)} queues')
print()
for q in queues:
    msgs = q.get('messages', 0)
    consumers = q.get('consumers', 0)
    state = q.get('state', 'unknown')
    ready = q.get('messages_ready', 0)
    unacked = q.get('messages_unacknowledged', 0)
    
    state_icon = '✓' if state == 'running' else '⚠'
    
    print(f'{state_icon} {q[\"name\"]}')
    print(f'   State: {state} | Consumers: {consumers}')
    print(f'   Messages: {msgs} (ready: {ready}, unacked: {unacked})')
    
    rate_details = q.get('message_stats', {})
    if rate_details:
        publish_rate = rate_details.get('publish_details', {}).get('rate', 0)
        deliver_rate = rate_details.get('deliver_details', {}).get('rate', 0)
        print(f'   Rates: publish={publish_rate:.1f}/s deliver={deliver_rate:.1f}/s')
    print()
" 2>/dev/null || echo "Could not connect to RabbitMQ"
}

# สร้าง queue
create_queue() {
    local queue_name="$1"
    local vhost="${2:-%2F}"
    local durable="${3:-true}"
    local ttl="${4:-}"
    
    local body
    body=$(cat << EOF
{
    "durable": ${durable},
    "auto_delete": false,
    "arguments": {
        "x-message-ttl": ${ttl:-3600000},
        "x-dead-letter-exchange": "dlx",
        "x-dead-letter-routing-key": "${queue_name}.dead"
    }
}
EOF
)
    
    rabbitmq_api "PUT" "/queues/${vhost}/${queue_name}" "$body"
    echo "✓ Queue created: ${queue_name}"
}

# ส่ง message
publish_message() {
    local exchange="$1"
    local routing_key="$2"
    local message="$3"
    local vhost="${4:-%2F}"
    
    local body
    body=$(cat << EOF
{
    "vhost": "/",
    "name": "${exchange}",
    "properties": {
        "delivery_mode": 2,
        "content_type": "application/json"
    },
    "routing_key": "${routing_key}",
    "payload": $(echo "$message" | python3 -c "import json,sys; print(json.dumps(sys.stdin.read().strip()))"),
    "payload_encoding": "string"
}
EOF
)
    
    rabbitmq_api "POST" "/exchanges/${vhost}/${exchange}/publish" "$body"
    echo "✓ Message published to ${exchange}/${routing_key}"
}

# รับ messages
consume_messages() {
    local queue_name="$1"
    local count="${2:-10}"
    local vhost="${3:-%2F}"
    
    local body
    body=$(cat << EOF
{
    "count": ${count},
    "ackmode": "ack_requeue_false",
    "encoding": "auto"
}
EOF
)
    
    echo "=== Consuming from ${queue_name} ==="
    
    rabbitmq_api "POST" "/queues/${vhost}/${queue_name}/get" "$body" | \
    python3 -c "
import json, sys
messages = json.load(sys.stdin)
print(f'Got {len(messages)} messages')
for i, msg in enumerate(messages, 1):
    print(f'--- Message {i} ---')
    print(f'Routing key: {msg.get(\"routing_key\", \"N/A\")}')
    print(f'Payload: {msg.get(\"payload\", \"\")}')
    print()
" 2>/dev/null
}

# === Kafka Management ===

KAFKA_BROKER="${KAFKA_BROKER:-localhost:9092}"
KAFKA_SCRIPTS="${KAFKA_SCRIPTS:-/opt/kafka/bin}"

# แสดงรายการ topics
list_kafka_topics() {
    echo "=== Kafka Topics ==="
    
    "${KAFKA_SCRIPTS}/kafka-topics.sh" \
        --bootstrap-server "$KAFKA_BROKER" \
        --list 2>/dev/null | \
    while read -r topic; do
        [[ "$topic" == __* ]] && continue  # skip internal topics
        
        local info
        info=$("${KAFKA_SCRIPTS}/kafka-topics.sh" \
            --bootstrap-server "$KAFKA_BROKER" \
            --describe \
            --topic "$topic" 2>/dev/null | head -3)
        
        echo "Topic: ${topic}"
        echo "$info" | grep -E "PartitionCount|ReplicationFactor" | awk '{print "  " $0}'
        echo ""
    done
}

# สร้าง topic
create_kafka_topic() {
    local topic="$1"
    local partitions="${2:-3}"
    local replication="${3:-3}"
    local retention_ms="${4:-604800000}"  # 7 days
    
    echo "=== Creating Kafka Topic: ${topic} ==="
    
    "${KAFKA_SCRIPTS}/kafka-topics.sh" \
        --bootstrap-server "$KAFKA_BROKER" \
        --create \
        --topic "$topic" \
        --partitions "$partitions" \
        --replication-factor "$replication" \
        --config "retention.ms=${retention_ms}" \
        --config "compression.type=lz4" \
        --if-not-exists 2>&1
    
    echo "✓ Topic created: ${topic}"
}

# ส่ง messages ไป Kafka
kafka_produce() {
    local topic="$1"
    local message="$2"
    local key="${3:-}"
    
    if [[ -n "$key" ]]; then
        echo "${key}:${message}" | \
        "${KAFKA_SCRIPTS}/kafka-console-producer.sh" \
            --bootstrap-server "$KAFKA_BROKER" \
            --topic "$topic" \
            --property "key.separator=:" \
            --property "parse.key=true" 2>/dev/null
    else
        echo "$message" | \
        "${KAFKA_SCRIPTS}/kafka-console-producer.sh" \
            --bootstrap-server "$KAFKA_BROKER" \
            --topic "$topic" 2>/dev/null
    fi
    
    echo "✓ Message sent to ${topic}"
}

# รับ messages จาก Kafka
kafka_consume() {
    local topic="$1"
    local group="${2:-bash-consumer}"
    local from_beginning="${3:-false}"
    local count="${4:-10}"
    
    echo "=== Consuming from Kafka: ${topic} ==="
    
    local args=(
        "--bootstrap-server" "$KAFKA_BROKER"
        "--topic" "$topic"
        "--group" "$group"
        "--max-messages" "$count"
    )
    
    [[ "$from_beginning" == "true" ]] && args+=("--from-beginning")
    
    "${KAFKA_SCRIPTS}/kafka-console-consumer.sh" "${args[@]}" 2>/dev/null
}

# แสดง consumer group lag
show_consumer_lag() {
    local group="${1:-}"
    
    echo "=== Kafka Consumer Group Lag ==="
    
    local groups_args=("--bootstrap-server" "$KAFKA_BROKER" "--describe")
    
    if [[ -n "$group" ]]; then
        groups_args+=("--group" "$group")
    else
        groups_args+=("--all-groups")
    fi
    
    "${KAFKA_SCRIPTS}/kafka-consumer-groups.sh" "${groups_args[@]}" 2>/dev/null | \
    awk 'NR>1 && $0!="" {
        lag = $5 + 0
        if (lag > 100) printf "\033[31m%s\033[0m\n", $0
        else if (lag > 0) printf "\033[33m%s\033[0m\n", $0
        else printf "\033[32m%s\033[0m\n", $0
    }'
}

# Kafka health check
kafka_health() {
    echo "=== Kafka Health Check ==="
    
    # ตรวจสอบ broker
    local brokers
    brokers=$("${KAFKA_SCRIPTS}/kafka-broker-api-versions.sh" \
        --bootstrap-server "$KAFKA_BROKER" \
        2>/dev/null | grep -c "^" || echo "0")
    
    if [[ $brokers -gt 0 ]]; then
        echo "✓ Kafka broker accessible"
    else
        echo "✗ Cannot reach Kafka broker"
        return 1
    fi
    
    # ตรวจสอบ topic count
    local topic_count
    topic_count=$("${KAFKA_SCRIPTS}/kafka-topics.sh" \
        --bootstrap-server "$KAFKA_BROKER" \
        --list 2>/dev/null | wc -l)
    
    echo "  Topics: ${topic_count}"
    
    # Consumer groups
    local group_count
    group_count=$("${KAFKA_SCRIPTS}/kafka-consumer-groups.sh" \
        --bootstrap-server "$KAFKA_BROKER" \
        --list 2>/dev/null | wc -l)
    
    echo "  Consumer groups: ${group_count}"
}

# Main dispatcher
case "${1:-}" in
    # RabbitMQ
    rmq-list)    list_queues "${2:-%2F}" ;;
    rmq-create)  create_queue "${2}" "${3:-%2F}" "${4:-true}" ;;
    rmq-publish) publish_message "${2}" "${3}" "${4}" "${5:-%2F}" ;;
    rmq-consume) consume_messages "${2}" "${3:-10}" "${4:-%2F}" ;;
    
    # Kafka
    kafka-topics)   list_kafka_topics ;;
    kafka-create)   create_kafka_topic "${2}" "${3:-3}" "${4:-3}" ;;
    kafka-produce)  kafka_produce "${2}" "${3}" "${4:-}" ;;
    kafka-consume)  kafka_consume "${2}" "${3:-bash-consumer}" "${4:-false}" "${5:-10}" ;;
    kafka-lag)      show_consumer_lag "${2:-}" ;;
    kafka-health)   kafka_health ;;
    
    *)
        echo "Usage: $0 {rmq-list|rmq-create|rmq-publish|rmq-consume|kafka-topics|kafka-create|kafka-produce|kafka-consume|kafka-lag|kafka-health} [args]"
        ;;
esac
```

---

## Workshop: Complete Service Mesh Setup

```bash
#!/bin/bash
# service_mesh_workshop.sh - Workshop: Complete Service Mesh

set -euo pipefail

NAMESPACE="mesh-demo"
KUBECTL="${KUBECTL:-kubectl}"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
NC='\033[0m'

log() { echo -e "${BLUE}[$(date +%H:%M:%S)] $*${NC}"; }
ok()  { echo -e "${GREEN}[$(date +%H:%M:%S)] ✓ $*${NC}"; }
err() { echo -e "${RED}[$(date +%H:%M:%S)] ✗ $*${NC}"; }

# สร้าง demo namespace
setup_namespace() {
    log "Setting up namespace: ${NAMESPACE}"
    
    $KUBECTL create namespace "$NAMESPACE" --dry-run=client -o yaml | \
    $KUBECTL apply -f -
    
    # Enable Istio injection
    $KUBECTL label namespace "$NAMESPACE" \
        istio-injection=enabled \
        --overwrite 2>/dev/null || true
    
    ok "Namespace ready: ${NAMESPACE}"
}

# Deploy microservices
deploy_microservices() {
    log "Deploying microservices..."
    
    # Frontend service
    cat << YAML | $KUBECTL apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ${NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      version: v1
  template:
    metadata:
      labels:
        app: frontend
        version: v1
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
        env:
        - name: API_URL
          value: "http://api:8080"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ${NAMESPACE}
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
YAML
    
    # API service (v1 and v2 for canary)
    cat << YAML | $KUBECTL apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v1
  namespace: ${NAMESPACE}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
      version: v1
  template:
    metadata:
      labels:
        app: api
        version: v1
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:alpine
        args:
        - "-text=API v1 response"
        ports:
        - containerPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v2
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
      version: v2
  template:
    metadata:
      labels:
        app: api
        version: v2
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:alpine
        args:
        - "-text=API v2 response"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: ${NAMESPACE}
spec:
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 5678
YAML
    
    ok "Microservices deployed"
}

# Configure service mesh
configure_mesh() {
    log "Configuring service mesh..."
    
    # DestinationRule for API
    cat << YAML | $KUBECTL apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api
  namespace: ${NAMESPACE}
spec:
  host: api
  trafficPolicy:
    connectionPool:
      http:
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
YAML
    
    # VirtualService - 90/10 split
    cat << YAML | $KUBECTL apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
  namespace: ${NAMESPACE}
spec:
  hosts:
  - api
  http:
  - route:
    - destination:
        host: api
        subset: v1
      weight: 90
    - destination:
        host: api
        subset: v2
      weight: 10
    timeout: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
YAML
    
    # mTLS Policy
    cat << YAML | $KUBECTL apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ${NAMESPACE}
spec:
  mtls:
    mode: STRICT
YAML
    
    ok "Service mesh configured"
}

# Test traffic distribution
test_traffic() {
    local iterations="${1:-20}"
    
    log "Testing traffic distribution (${iterations} requests)..."
    
    local v1_count=0
    local v2_count=0
    
    # สร้าง test pod
    $KUBECTL run curl-test \
        --image=curlimages/curl:latest \
        --namespace="$NAMESPACE" \
        --restart=Never \
        --command -- sleep 3600 2>/dev/null || true
    
    $KUBECTL wait pod/curl-test \
        -n "$NAMESPACE" \
        --for=condition=Ready \
        --timeout=30s 2>/dev/null || true
    
    for i in $(seq 1 "$iterations"); do
        local response
        response=$($KUBECTL exec curl-test \
            -n "$NAMESPACE" \
            -- curl -sf http://api:8080/ 2>/dev/null || echo "error")
        
        if echo "$response" | grep -q "v2"; then
            ((v2_count++))
        else
            ((v1_count++))
        fi
    done
    
    # Cleanup test pod
    $KUBECTL delete pod curl-test -n "$NAMESPACE" --ignore-not-found 2>/dev/null || true
    
    echo ""
    echo "Traffic Distribution Results:"
    echo "  v1: ${v1_count}/${iterations} ($(( v1_count * 100 / iterations ))%)"
    echo "  v2: ${v2_count}/${iterations} ($(( v2_count * 100 / iterations ))%)"
    echo ""
    echo "Expected: v1=~90%, v2=~10%"
}

# Cleanup
cleanup() {
    log "Cleaning up..."
    $KUBECTL delete namespace "$NAMESPACE" --ignore-not-found 2>/dev/null || true
    ok "Cleanup complete"
}

# Main
echo -e "${BOLD}${CYAN}"
echo "╔══════════════════════════════════════╗"
echo "║   SERVICE MESH WORKSHOP              ║"
echo "╚══════════════════════════════════════╝"
echo -e "${NC}"

case "${1:-setup}" in
    setup)
        setup_namespace
        deploy_microservices
        configure_mesh
        log "Workshop setup complete!"
        log "Test with: $0 test"
        ;;
    test)
        test_traffic "${2:-20}"
        ;;
    cleanup)
        cleanup
        ;;
    *)
        echo "Usage: $0 {setup|test|cleanup}"
        ;;
esac
```

---

## สรุป Part 30

| หัวข้อ | เนื้อหา |
|--------|---------|
| **Advanced Networking** | Network scan, port scan, DNS analysis, SSL analysis, bandwidth test |
| **Istio** | VirtualService, DestinationRule, Gateway, mTLS, AuthPolicy, fault injection |
| **Linkerd** | Installation, sidecar injection, traffic split, service profiles |
| **HAProxy** | Configuration generation, server management, health monitoring |
| **RabbitMQ** | Queue management, publish/consume messages |
| **Kafka** | Topic management, produce/consume, consumer lag |
| **Service Mesh Workshop** | Complete microservices setup with traffic splitting |

**ขั้นตอนต่อไป**: Part 31 - Observability: Metrics, Tracing, และ Logging
