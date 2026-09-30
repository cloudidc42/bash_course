# Part 15: Networking และ APIs

## Module 2: Intermediate Level (ระดับกลาง)

---

## ขั้นตอนที่ 362: Network Commands พื้นฐาน

```bash
#!/usr/bin/env bash
# network_basics.sh - Network Commands

echo "=== Network Commands พื้นฐาน ==="

echo "1. ดู network interfaces:"
ip addr show 2>/dev/null | head -20 || ifconfig 2>/dev/null | head -20

echo ""
echo "2. ดู routing table:"
ip route show 2>/dev/null || route -n 2>/dev/null

echo ""
echo "3. ping - test connectivity:"
ping -c 3 -W 2 8.8.8.8 2>/dev/null | tail -4 || echo "ping ไม่สำเร็จ"

echo ""
echo "4. DNS lookup:"
host google.com 2>/dev/null | head -3 || \
nslookup google.com 2>/dev/null | head -10 || \
dig google.com +short 2>/dev/null

echo ""
echo "5. traceroute:"
echo "  traceroute google.com    # ดูเส้นทาง packet"
echo "  tracepath google.com     # alternative"

echo ""
echo "6. netstat - network statistics:"
netstat -tuln 2>/dev/null | head -15 || \
ss -tuln 2>/dev/null | head -15

echo ""
echo "7. ss - socket statistics (modern):"
ss -s 2>/dev/null || echo "ss ไม่พบ"

echo ""
echo "8. ดู active connections:"
ss -tnp 2>/dev/null | head -10 || netstat -tnp 2>/dev/null | head -10

echo ""
echo "9. hostname และ DNS:"
hostname
hostname -f 2>/dev/null || hostname --fqdn 2>/dev/null || true
cat /etc/resolv.conf 2>/dev/null | grep "nameserver"
```

---

## ขั้นตอนที่ 363: curl - HTTP Requests

```bash
#!/usr/bin/env bash
# curl_basics.sh - curl Command

echo "=== curl - HTTP Client ==="

echo "1. GET request:"
curl -s "https://httpbin.org/get" 2>/dev/null | head -20 || \
echo "(ต้องการ internet access)"

echo ""
echo "2. POST request:"
curl -s -X POST "https://httpbin.org/post" \
    -H "Content-Type: application/json" \
    -d '{"name":"John","age":30}' 2>/dev/null | head -20 || \
echo "(ต้องการ internet access)"

echo ""
echo "3. curl options สำคัญ:"
cat << 'EOF'
curl options:
  -s, --silent       ไม่แสดง progress
  -S, --show-error   แสดง error ถ้า -s
  -o file            save ไปยังไฟล์
  -O                 save ด้วยชื่อจาก URL
  -L                 ติดตาม redirects
  -I, --head         HEAD request เท่านั้น
  -i                 รวม response headers
  -v                 verbose (debug)
  -X METHOD          HTTP method
  -H "Header: Value" เพิ่ม header
  -d "data"          request body (POST)
  --data-urlencode   encode data
  -F "key=value"     form data (multipart)
  -u user:pass       basic auth
  --header "Authorization: Bearer TOKEN"
  --connect-timeout N  connection timeout
  --max-time N       total timeout
  -w "format"        output format
  --retry N          retry on failure
  --retry-delay N    delay between retries
EOF

echo ""
echo "4. Download file:"
echo "  curl -LO https://example.com/file.zip"
echo "  curl -L -o myfile.zip https://example.com/file.zip"

echo ""
echo "5. Check HTTP status:"
curl -s -o /dev/null -w "%{http_code}" "https://httpbin.org/status/200" 2>/dev/null && echo ""

echo ""
echo "6. Response time:"
curl -s -o /dev/null -w "
  time_connect: %{time_connect}
  time_starttransfer: %{time_starttransfer}
  time_total: %{time_total}
" "https://httpbin.org/get" 2>/dev/null || echo "(ต้องการ internet)"

echo ""
echo "7. Headers only:"
curl -sI "https://httpbin.org/get" 2>/dev/null | head -10 || echo "(ต้องการ internet)"
```

---

## ขั้นตอนที่ 364: curl กับ APIs

```bash
#!/usr/bin/env bash
# curl_apis.sh - curl กับ REST APIs

echo "=== curl กับ REST APIs ==="

# ==================== HTTP Methods ====================
echo "1. HTTP Methods:"

# GET
echo "GET - ดึงข้อมูล:"
echo '  curl -s "https://api.example.com/users"'
echo '  curl -s "https://api.example.com/users/1"'

# POST
echo ""
echo "POST - สร้างข้อมูล:"
echo '  curl -s -X POST "https://api.example.com/users" \'
echo '    -H "Content-Type: application/json" \'
echo '    -d '"'"'{"name":"John","email":"john@example.com"}'"'"

# PUT
echo ""
echo "PUT - อัปเดตข้อมูล (replace):"
echo '  curl -s -X PUT "https://api.example.com/users/1" \'
echo '    -H "Content-Type: application/json" \'
echo '    -d '"'"'{"name":"John Updated"}'"'"

# PATCH
echo ""
echo "PATCH - อัปเดตบางส่วน:"
echo '  curl -s -X PATCH "https://api.example.com/users/1" \'
echo '    -d '"'"'{"email":"new@example.com"}'"'"

# DELETE
echo ""
echo "DELETE - ลบข้อมูล:"
echo '  curl -s -X DELETE "https://api.example.com/users/1"'

# ==================== Authentication ====================
echo ""
echo "2. Authentication:"

echo "Basic Auth:"
echo '  curl -s -u username:password "https://api.example.com/data"'

echo ""
echo "Bearer Token:"
echo '  curl -s -H "Authorization: Bearer YOUR_TOKEN" "https://api.example.com/data"'

echo ""
echo "API Key in header:"
echo '  curl -s -H "X-API-Key: YOUR_KEY" "https://api.example.com/data"'

echo ""
echo "API Key in URL:"
echo '  curl -s "https://api.example.com/data?api_key=YOUR_KEY"'

# ==================== Functions ====================
echo ""
echo "3. API Helper Functions:"
cat << 'FUNCTIONS'
#!/usr/bin/env bash

# Configuration
API_BASE="https://api.example.com"
API_TOKEN="your_token_here"

# HTTP GET
api_get() {
    local endpoint="$1"
    curl -s -f \
        -H "Authorization: Bearer $API_TOKEN" \
        -H "Accept: application/json" \
        "${API_BASE}${endpoint}"
}

# HTTP POST
api_post() {
    local endpoint="$1"
    local data="$2"
    curl -s -f -X POST \
        -H "Authorization: Bearer $API_TOKEN" \
        -H "Content-Type: application/json" \
        -H "Accept: application/json" \
        -d "$data" \
        "${API_BASE}${endpoint}"
}

# HTTP PUT
api_put() {
    local endpoint="$1"
    local data="$2"
    curl -s -f -X PUT \
        -H "Authorization: Bearer $API_TOKEN" \
        -H "Content-Type: application/json" \
        -d "$data" \
        "${API_BASE}${endpoint}"
}

# HTTP DELETE
api_delete() {
    local endpoint="$1"
    curl -s -f -X DELETE \
        -H "Authorization: Bearer $API_TOKEN" \
        "${API_BASE}${endpoint}"
}

# ตัวอย่างการใช้งาน:
# users=$(api_get "/users")
# new_user=$(api_post "/users" '{"name":"John"}')
FUNCTIONS
```

---

## ขั้นตอนที่ 365: JSON Processing ด้วย jq

```bash
#!/usr/bin/env bash
# jq_basics.sh - jq JSON Processor

echo "=== jq JSON Processor ==="

# ตรวจสอบว่ามี jq
if ! command -v jq &>/dev/null; then
    echo "jq ไม่ได้ติดตั้ง"
    echo "ติดตั้ง: sudo apt install jq"
    echo ""
    echo "jq คือ JSON processor command-line สำหรับ:"
    echo "  - parse JSON"
    echo "  - filter และ transform"
    echo "  - format output"
    exit 0
fi

# สร้าง sample JSON
cat > /tmp/users.json << 'EOF'
{
  "users": [
    {"id": 1, "name": "Alice", "age": 30, "dept": "Engineering", "salary": 75000},
    {"id": 2, "name": "Bob", "age": 25, "dept": "Marketing", "salary": 65000},
    {"id": 3, "name": "Charlie", "age": 35, "dept": "Engineering", "salary": 85000},
    {"id": 4, "name": "Diana", "age": 28, "dept": "HR", "salary": 70000},
    {"id": 5, "name": "Eve", "age": 32, "dept": "Engineering", "salary": 80000}
  ],
  "total": 5,
  "page": 1
}
EOF

echo "1. Pretty print:"
jq '.' /tmp/users.json | head -20

echo ""
echo "2. เข้าถึง field:"
jq '.total' /tmp/users.json
jq '.users[0]' /tmp/users.json
jq '.users[0].name' /tmp/users.json

echo ""
echo "3. Array iteration:"
jq '.users[]' /tmp/users.json | head -10

echo ""
echo "4. Filter fields:"
jq '.users[] | {name, salary}' /tmp/users.json

echo ""
echo "5. Filter by condition:"
jq '.users[] | select(.dept == "Engineering")' /tmp/users.json

echo ""
echo "6. Map transformation:"
jq '.users | map(select(.salary > 70000)) | map(.name)' /tmp/users.json

echo ""
echo "7. Aggregate:"
jq '.users | length' /tmp/users.json
jq '[.users[].salary] | add' /tmp/users.json
jq '[.users[].salary] | add / length' /tmp/users.json
jq '[.users[].salary] | max' /tmp/users.json

echo ""
echo "8. Sort:"
jq '.users | sort_by(.salary) | reverse | .[].name' /tmp/users.json

echo ""
echo "9. Group by:"
jq '.users | group_by(.dept) | map({dept: .[0].dept, count: length, avg_salary: ([.[].salary] | add / length)})' /tmp/users.json

echo ""
echo "10. Output as CSV:"
jq -r '.users[] | [.name, .age, .dept, .salary] | @csv' /tmp/users.json
```

---

## ขั้นตอนที่ 366: jq Advanced

```bash
#!/usr/bin/env bash
# jq_advanced.sh - Advanced jq

echo "=== jq Advanced ==="

echo "1. Conditional:"
echo '{"score": 85}' | jq 'if .score >= 90 then "A" elif .score >= 80 then "B" else "C" end'

echo ""
echo "2. String interpolation:"
echo '{"name": "Alice", "age": 30}' | jq '"Hello, \(.name)! You are \(.age) years old."'

echo ""
echo "3. Path operations:"
jq 'path(.users[].name)' /tmp/users.json 2>/dev/null | head -5

echo ""
echo "4. Update value:"
jq '.users[0].salary = 80000' /tmp/users.json | jq '.users[0]'

echo ""
echo "5. Add field:"
jq '.users[] | . + {"level": if .salary >= 80000 then "senior" else "junior" end}' \
    /tmp/users.json | head -20

echo ""
echo "6. Delete field:"
jq '.users[] | del(.age)' /tmp/users.json | head -15

echo ""
echo "7. Reduce:"
jq 'reduce .users[] as $u (0; . + $u.salary)' /tmp/users.json

echo ""
echo "8. String functions:"
echo '"  Hello World  "' | jq 'ltrimstr(" ") | rtrimstr(" ") | ascii_downcase'

echo ""
echo "9. Arrays:"
echo '[1,2,3,4,5]' | jq 'map(. * 2)'
echo '[1,2,3,4,5]' | jq '[.[] | select(. > 2)]'
echo '[1,2,3,4,5]' | jq 'flatten | unique | sort'

echo ""
echo "10. jq ใน pipeline:"
curl -s "https://httpbin.org/json" 2>/dev/null | jq '.slideshow.title' || \
echo '{"data":{"value":42}}' | jq '.data.value'

echo ""
echo "11. Format JSON output:"
echo '{"a":1,"b":2}' | jq -c '.'   # compact
echo '{"a":1,"b":2}' | jq -r '.a'  # raw string (ไม่มี quotes)
echo '{"a":1,"b":2}' | jq -e '.c' > /dev/null 2>&1 && echo "exists" || echo "null"
```

---

## ขั้นตอนที่ 367: wget - Download Files

```bash
#!/usr/bin/env bash
# wget_usage.sh - wget

echo "=== wget - File Downloader ==="

echo "wget options สำคัญ:"
cat << 'EOF'
wget [options] URL

Options:
  -O file          บันทึกเป็นชื่อไฟล์นี้
  -o log           เขียน output ไปยัง log file
  -q               quiet mode
  -v               verbose
  -c               resume download
  -r               recursive download
  -l N             recursive depth
  -A *.pdf         accept ชนิดไฟล์
  -R *.html        reject ชนิดไฟล์
  -np              no parent directory
  --limit-rate=1m  จำกัด speed
  -t N             retry N ครั้ง
  -T N             timeout N seconds
  --spider         check URL โดยไม่ download
  -p               download page assets
  -k               convert links for offline
  --auth-no-challenge  basic auth
  --user=USER --password=PASS
  --header="Header: Value"
  -U "User-Agent"
  --no-check-certificate   ข้าม SSL verify
EOF

echo ""
echo "1. Download ไฟล์:"
echo "  wget https://example.com/file.zip"
echo "  wget -O myfile.zip https://example.com/file.zip"

echo ""
echo "2. Download หลายไฟล์:"
echo "  wget -i urls.txt   # อ่าน URL จากไฟล์"
cat << 'EOF'
# urls.txt
https://example.com/file1.txt
https://example.com/file2.txt
https://example.com/file3.txt
EOF

echo ""
echo "3. Resume download:"
echo "  wget -c https://example.com/largefile.iso"

echo ""
echo "4. Mirror website:"
echo "  wget --mirror --convert-links --page-requisites --no-parent \\"
echo "       -P local_copy https://example.com"

echo ""
echo "5. Check URL availability:"
echo "  wget --spider https://example.com/file.txt"

echo ""
echo "6. Download ด้วย authentication:"
echo "  wget --user=username --password=pass https://example.com/private.zip"

echo ""
echo "7. Function: download with retry:"
download_with_retry() {
    local url="$1"
    local output="${2:-$(basename "$url")}"
    local max_retries="${3:-3}"
    local delay=5
    
    for ((i=1; i<=max_retries; i++)); do
        echo "Attempt $i/$max_retries: Downloading $url"
        if wget -q -O "$output" "$url" 2>/dev/null; then
            echo "✓ Download successful: $output"
            return 0
        fi
        
        if (( i < max_retries )); then
            echo "  Failed. Retrying in ${delay}s..."
            sleep "$delay"
            delay=$((delay * 2))
        fi
    done
    
    echo "✗ Download failed after $max_retries attempts"
    return 1
}
```

---

## ขั้นตอนที่ 368: SSH - Secure Shell

```bash
#!/usr/bin/env bash
# ssh_usage.sh - SSH Commands

echo "=== SSH - Secure Shell ==="

echo "1. เชื่อมต่อ SSH:"
cat << 'EOF'
ssh user@hostname
ssh -p 2222 user@hostname        # port อื่น
ssh -i ~/.ssh/mykey user@host    # ใช้ key เฉพาะ
ssh -v user@host                 # verbose (debug)
EOF

echo ""
echo "2. SSH Key Management:"
cat << 'EOF'
# สร้าง SSH key pair
ssh-keygen -t ed25519 -C "user@email.com"
ssh-keygen -t rsa -b 4096 -C "user@email.com"

# Copy key ไปยัง server
ssh-copy-id user@host
ssh-copy-id -i ~/.ssh/mykey.pub user@host

# ดู fingerprint
ssh-keygen -l -f ~/.ssh/id_ed25519.pub
EOF

echo ""
echo "3. รัน command ผ่าน SSH:"
cat << 'EOF'
ssh user@host "command"
ssh user@host "ls -la /var/log/"
ssh user@host "sudo systemctl status nginx"

# หลาย commands
ssh user@host << 'REMOTE'
echo "Running on remote host"
uptime
df -h
REMOTE
EOF

echo ""
echo "4. SCP - Secure Copy:"
cat << 'EOF'
# Copy file ไปยัง remote
scp file.txt user@host:/remote/path/

# Copy จาก remote
scp user@host:/remote/file.txt local/path/

# Copy directory
scp -r localdir/ user@host:/remote/path/

# Port อื่น
scp -P 2222 file.txt user@host:/path/
EOF

echo ""
echo "5. rsync ผ่าน SSH:"
cat << 'EOF'
rsync -avz localdir/ user@host:/remote/dir/
rsync -avz -e "ssh -p 2222" localdir/ user@host:/remote/dir/
rsync -avz --delete localdir/ user@host:/remote/dir/
EOF

echo ""
echo "6. SSH Config (~/.ssh/config):"
cat << 'EOF'
# ~/.ssh/config
Host myserver
    HostName server.example.com
    User john
    Port 2222
    IdentityFile ~/.ssh/myserver_key

Host prod
    HostName production.example.com
    User admin
    ForwardAgent yes
    
# แล้วใช้:
# ssh myserver
# ssh prod
EOF

echo ""
echo "7. SSH Tunneling:"
cat << 'EOF'
# Local port forwarding
ssh -L 8080:localhost:80 user@host
# เข้าถึง http://localhost:8080 เพื่อเชื่อมต่อ host:80

# Remote port forwarding
ssh -R 8080:localhost:80 user@host
# ให้ host เข้าถึง localhost:80 ผ่าน host:8080

# SOCKS proxy
ssh -D 1080 user@host
# ใช้เป็น SOCKS5 proxy ที่ localhost:1080
EOF
```

---

## ขั้นตอนที่ 369: HTTP API Client

```bash
#!/usr/bin/env bash
# api_client.sh - HTTP API Client Library

echo "=== HTTP API Client Library ==="

# ==================== API Client ====================
cat << 'LIBRARY'
#!/usr/bin/env bash
# api_client.sh - Reusable API Client

# Configuration
declare -A API_CONFIG
API_CONFIG[base_url]="https://api.example.com"
API_CONFIG[token]=""
API_CONFIG[timeout]=30
API_CONFIG[retries]=3
API_CONFIG[verbose]="false"

# ==================== Core Functions ====================

http_request() {
    local method="$1"
    local endpoint="$2"
    local data="${3:-}"
    
    local url="${API_CONFIG[base_url]}${endpoint}"
    local args=()
    
    # Common options
    args+=(-s -S)
    args+=(-X "$method")
    args+=(-w "\n%{http_code}")
    args+=(--connect-timeout "${API_CONFIG[timeout]}")
    args+=(--max-time "$((API_CONFIG[timeout] * 3))")
    
    # Headers
    if [[ -n "${API_CONFIG[token]}" ]]; then
        args+=(-H "Authorization: Bearer ${API_CONFIG[token]}")
    fi
    args+=(-H "Content-Type: application/json")
    args+=(-H "Accept: application/json")
    
    # Body
    if [[ -n "$data" ]]; then
        args+=(-d "$data")
    fi
    
    # Verbose
    if [[ "${API_CONFIG[verbose]}" == "true" ]]; then
        args+=(-v)
    fi
    
    # Execute with retry
    local attempt=0
    while (( attempt < API_CONFIG[retries] )); do
        local response
        response=$(curl "${args[@]}" "$url" 2>/dev/null)
        local exit_code=$?
        
        if (( exit_code == 0 )); then
            local body status_code
            body="${response%$'\n'*}"
            status_code="${response##*$'\n'}"
            
            if (( status_code >= 200 && status_code < 300 )); then
                echo "$body"
                return 0
            elif (( status_code >= 500 )); then
                # Server error - retry
                (( attempt++ ))
                sleep "$((2 ** attempt))"
                continue
            else
                echo "HTTP Error $status_code: $body" >&2
                return 1
            fi
        fi
        
        (( attempt++ ))
        sleep "$((2 ** attempt))"
    done
    
    echo "Max retries exceeded" >&2
    return 1
}

# Convenience functions
api_get()    { http_request "GET"    "$1"; }
api_post()   { http_request "POST"   "$1" "$2"; }
api_put()    { http_request "PUT"    "$1" "$2"; }
api_patch()  { http_request "PATCH"  "$1" "$2"; }
api_delete() { http_request "DELETE" "$1"; }

# ==================== Usage Examples ====================
# API_CONFIG[base_url]="https://jsonplaceholder.typicode.com"
# 
# # GET list
# users=$(api_get "/users")
# echo "$users" | jq '.[].name'
# 
# # GET single
# user=$(api_get "/users/1")
# echo "$user" | jq '.name'
# 
# # POST create
# new_user=$(api_post "/users" '{"name":"John","email":"john@example.com"}')
# echo "Created: $(echo $new_user | jq '.id')"
# 
# # PUT update
# api_put "/users/1" '{"name":"Updated Name"}'
# 
# # DELETE
# api_delete "/users/1"
LIBRARY

echo ""
echo "ทดสอบกับ JSONPlaceholder API:"
if command -v jq &>/dev/null; then
    BASE_URL="https://jsonplaceholder.typicode.com"
    
    echo "GET /users:"
    result=$(curl -sf "${BASE_URL}/users" 2>/dev/null) || { echo "(ต้องการ internet)"; exit 0; }
    echo "$result" | jq '.[0:3] | .[].name'
    
    echo ""
    echo "GET /posts/1:"
    curl -sf "${BASE_URL}/posts/1" 2>/dev/null | jq '{title, body: (.body[0:50] + "...")}' || true
fi
```

---

## ขั้นตอนที่ 370: Webhook Server

```bash
#!/usr/bin/env bash
# webhook_server.sh - Simple Webhook Server

echo "=== Webhook Server ด้วย netcat/socat ==="

echo "1. Simple HTTP server ด้วย netcat:"
cat << 'SCRIPT'
#!/usr/bin/env bash
# simple_http.sh

PORT=8080

while true; do
    # รับ connection
    request=$(echo -e "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<html><body>Hello!</body></html>" | \
        nc -l -p "$PORT" -q 1)
    
    echo "Request received:"
    echo "$request" | head -3
done
SCRIPT

echo ""
echo "2. Webhook receiver ด้วย Python (ง่ายกว่า):"
cat << 'SCRIPT'
#!/usr/bin/env python3
# webhook_receiver.py

from http.server import HTTPServer, BaseHTTPRequestHandler
import json

class WebhookHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers.get('Content-Length', 0))
        body = self.rfile.read(content_length)
        
        try:
            data = json.loads(body)
            print(f"Received webhook: {json.dumps(data, indent=2)}")
            
            # Process webhook
            event = data.get('event', 'unknown')
            print(f"Event: {event}")
            
        except json.JSONDecodeError:
            print(f"Raw body: {body}")
        
        # Respond
        self.send_response(200)
        self.send_header('Content-Type', 'application/json')
        self.end_headers()
        self.wfile.write(b'{"status":"ok"}')
    
    def log_message(self, format, *args):
        print(f"[{self.address_string()}] {format % args}")

if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 8080), WebhookHandler)
    print("Webhook server running on port 8080")
    server.serve_forever()
SCRIPT

echo ""
echo "3. ทดสอบ webhook:"
cat << 'EOF'
# ส่ง test webhook
curl -X POST http://localhost:8080/webhook \
    -H "Content-Type: application/json" \
    -H "X-Signature: sha256=abc123" \
    -d '{"event":"push","repo":"myproject","branch":"main"}'
EOF

echo ""
echo "4. Bash webhook processor:"
process_webhook() {
    local payload="$1"
    
    if ! command -v jq &>/dev/null; then
        echo "jq ไม่พบ, ประมวลผลแบบ raw"
        echo "Payload: $payload"
        return
    fi
    
    local event
    event=$(echo "$payload" | jq -r '.event // "unknown"')
    
    case "$event" in
        "push")
            local branch repo
            branch=$(echo "$payload" | jq -r '.branch // "unknown"')
            repo=$(echo "$payload" | jq -r '.repo // "unknown"')
            echo "Push event: $repo → $branch"
            ;;
        "pull_request")
            local action pr_number
            action=$(echo "$payload" | jq -r '.action // "unknown"')
            pr_number=$(echo "$payload" | jq -r '.number // 0')
            echo "PR #$pr_number: $action"
            ;;
        *)
            echo "Unknown event: $event"
            ;;
    esac
}

echo ""
echo "Demo webhook processing:"
process_webhook '{"event":"push","repo":"myapp","branch":"main","commit":"abc1234"}'
process_webhook '{"event":"pull_request","action":"opened","number":42}'
```

---

## ขั้นตอนที่ 371: Network Monitoring

```bash
#!/usr/bin/env bash
# network_monitor.sh - Network Monitoring

echo "=== Network Monitoring ==="

# ==================== Monitor Functions ====================

check_host() {
    local host="$1"
    local timeout="${2:-5}"
    
    if ping -c 1 -W "$timeout" "$host" &>/dev/null; then
        echo "  [UP] $host"
        return 0
    else
        echo "  [DOWN] $host"
        return 1
    fi
}

check_port() {
    local host="$1"
    local port="$2"
    local timeout="${3:-5}"
    
    if timeout "$timeout" bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null; then
        echo "  [OPEN] $host:$port"
        return 0
    else
        echo "  [CLOSED] $host:$port"
        return 1
    fi
}

check_http() {
    local url="$1"
    local expected_code="${2:-200}"
    
    local code
    code=$(curl -s -o /dev/null -w "%{http_code}" \
        --connect-timeout 5 --max-time 10 "$url" 2>/dev/null)
    
    if [[ "$code" == "$expected_code" ]]; then
        echo "  [OK] $url (HTTP $code)"
        return 0
    else
        echo "  [FAIL] $url (HTTP $code, expected $expected_code)"
        return 1
    fi
}

check_ssl_expiry() {
    local host="$1"
    local days_warning="${2:-30}"
    
    local expiry
    expiry=$(echo | openssl s_client -connect "${host}:443" 2>/dev/null | \
        openssl x509 -noout -enddate 2>/dev/null | \
        cut -d= -f2)
    
    if [[ -z "$expiry" ]]; then
        echo "  [ERROR] Cannot check SSL for $host"
        return 1
    fi
    
    local expiry_ts
    expiry_ts=$(date -d "$expiry" +%s 2>/dev/null || date -j -f "%b %d %T %Y %Z" "$expiry" +%s 2>/dev/null)
    local now_ts
    now_ts=$(date +%s)
    local days_left
    days_left=$(( (expiry_ts - now_ts) / 86400 ))
    
    if (( days_left < 0 )); then
        echo "  [EXPIRED] $host SSL expired $((days_left * -1)) days ago"
        return 1
    elif (( days_left < days_warning )); then
        echo "  [WARNING] $host SSL expires in $days_left days"
        return 1
    else
        echo "  [OK] $host SSL valid for $days_left days"
        return 0
    fi
}

# ==================== Main Monitor ====================
echo "Network Health Check:"
echo "===================="
echo ""
echo "Hosts:"
for host in "8.8.8.8" "1.1.1.1" "localhost"; do
    check_host "$host" 2 || true
done

echo ""
echo "Ports:"
check_port "localhost" 22 2>/dev/null || true
check_port "localhost" 80 2>/dev/null || true
check_port "localhost" 443 2>/dev/null || true

echo ""
echo "HTTP Endpoints:"
check_http "https://httpbin.org/status/200" 200 2>/dev/null || echo "  (ต้องการ internet)"
check_http "https://httpbin.org/status/404" 404 2>/dev/null || echo "  (ต้องการ internet)"

echo ""
echo "Bandwidth test:"
if command -v curl &>/dev/null; then
    echo -n "  Download speed: "
    speed=$(curl -s -o /dev/null -w "%{speed_download}" \
        "https://httpbin.org/bytes/102400" 2>/dev/null)
    if [[ -n "$speed" && "$speed" != "0" ]]; then
        printf "%.2f KB/s\n" "$(awk "BEGIN {print $speed/1024}")"
    else
        echo "(ต้องการ internet)"
    fi
fi
```

---

## ขั้นตอนที่ 372: Workshop - API Integration Script

```bash
#!/usr/bin/env bash
# api_integration.sh - Workshop: API Integration

set -euo pipefail

# ==================== GitHub API Integration Demo ====================
echo "=== API Integration Workshop ==="
echo ""
echo "ตัวอย่าง: GitHub API Client"
echo ""

# GitHub API Base
GITHUB_API="https://api.github.com"

# Function: ดู user info
get_github_user() {
    local username="$1"
    curl -sf \
        -H "Accept: application/vnd.github.v3+json" \
        "${GITHUB_API}/users/${username}" 2>/dev/null
}

# Function: list repos
get_github_repos() {
    local username="$1"
    curl -sf \
        -H "Accept: application/vnd.github.v3+json" \
        "${GITHUB_API}/users/${username}/repos?per_page=10&sort=updated" 2>/dev/null
}

# Function: search repos
search_github_repos() {
    local query="$1"
    local language="${2:-}"
    local url="${GITHUB_API}/search/repositories?q=${query}+language:${language}&sort=stars&order=desc&per_page=5"
    curl -sf \
        -H "Accept: application/vnd.github.v3+json" \
        "$url" 2>/dev/null
}

# Demo
if command -v jq &>/dev/null; then
    echo "1. ดูข้อมูล GitHub user:"
    user_data=$(get_github_user "torvalds" 2>/dev/null) || { echo "  (ต้องการ internet)"; user_data="{}"; }
    if [[ "$user_data" != "{}" ]]; then
        echo "$user_data" | jq '{
            name: .name,
            bio: .bio,
            public_repos: .public_repos,
            followers: .followers
        }'
    fi
    
    echo ""
    echo "2. ค้นหา repos:"
    search_data=$(search_github_repos "bash scripting" "Shell" 2>/dev/null) || search_data="{}"
    if [[ "$search_data" != "{}" ]]; then
        echo "$search_data" | jq '.items[0:3] | .[] | {name: .full_name, stars: .stargazers_count, desc: .description}' 2>/dev/null || true
    fi
else
    echo "(ต้องการ jq สำหรับ JSON processing)"
fi

echo ""
echo "=== Weather API Example ==="
cat << 'WEATHER'
#!/usr/bin/env bash
# weather.sh - Weather API Client

WEATHER_API="https://wttr.in"

get_weather() {
    local city="${1:-Bangkok}"
    local format="${2:-3}"
    
    curl -s "${WEATHER_API}/${city}?format=${format}" 2>/dev/null
}

get_weather_json() {
    local city="${1:-Bangkok}"
    curl -s "${WEATHER_API}/${city}?format=j1" 2>/dev/null | \
        jq '{
            city: .nearest_area[0].areaName[0].value,
            temp_c: .current_condition[0].temp_C,
            feels_like: .current_condition[0].FeelsLikeC,
            humidity: .current_condition[0].humidity,
            description: .current_condition[0].weatherDesc[0].value
        }' 2>/dev/null
}

echo "Weather in Bangkok:"
get_weather "Bangkok" "%l:+%c+%t+%h"
WEATHER

echo ""
echo "Demo weather (ถ้ามี internet):"
curl -s "https://wttr.in/Bangkok?format=%l:+%c+%t" 2>/dev/null || echo "  (ต้องการ internet)"

echo ""
echo "=== Slack Notification ==="
cat << 'SLACK'
#!/usr/bin/env bash
# send_slack.sh

SLACK_WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

send_slack() {
    local message="$1"
    local channel="${2:-#general}"
    local username="${3:-Bot}"
    local emoji="${4:-:robot_face:}"
    
    local payload
    payload=$(jq -n \
        --arg channel "$channel" \
        --arg username "$username" \
        --arg emoji "$emoji" \
        --arg text "$message" \
        '{
            channel: $channel,
            username: $username,
            icon_emoji: $emoji,
            text: $text
        }')
    
    curl -s -X POST \
        -H "Content-Type: application/json" \
        -d "$payload" \
        "$SLACK_WEBHOOK_URL"
}

# ตัวอย่าง:
# send_slack "Build #123 succeeded!" "#deployments" "CI Bot" ":white_check_mark:"
# send_slack "Server CPU > 90%" "#alerts" "Monitor" ":warning:"
SLACK
```

---

## ขั้นตอนที่ 373: สรุป Part 15 - Networking

```bash
#!/usr/bin/env bash
# summary_networking.sh

echo "=== สรุป Networking และ APIs ==="
echo ""
echo "Network Tools:"
echo "  ip/ifconfig    - network interfaces"
echo "  ping           - connectivity test"
echo "  traceroute     - path tracing"
echo "  netstat/ss     - connections"
echo "  nslookup/dig   - DNS"
echo "  nmap           - port scanning"
echo ""
echo "HTTP Tools:"
echo "  curl     - HTTP client"
echo "  wget     - file downloader"
echo "  httpie   - user-friendly HTTP"
echo ""
echo "curl options:"
echo "  -s     silent"
echo "  -X     HTTP method"
echo "  -H     header"
echo "  -d     data (POST)"
echo "  -o     output file"
echo "  -L     follow redirects"
echo "  -w     output format"
echo "  --retry retry count"
echo ""
echo "jq filters:"
echo "  .field      - access field"
echo "  .[]         - iterate array"
echo "  select()    - filter"
echo "  map()       - transform"
echo "  sort_by()   - sort"
echo "  group_by()  - group"
echo "  add, length, max, min"
echo ""
echo "SSH:"
echo "  ssh user@host"
echo "  ssh-keygen"
echo "  ssh-copy-id"
echo "  scp, rsync"
echo "  SSH tunneling"
echo ""
echo "Next: Part 16 - Git Automation"
```

---

## แบบฝึกหัด Part 15

### แบบฝึกหัดที่ 1: Health Check Script
สร้าง health check script ที่:
- ตรวจสอบ hosts, ports, HTTP endpoints
- SSL certificate expiry
- บันทึก history
- แจ้งเตือนเมื่อมีปัญหา
- รายงาน uptime percentage

### แบบฝึกหัดที่ 2: API Data Fetcher
ดึงข้อมูลจาก public API:
- JSONPlaceholder API
- Parse และ format ข้อมูล
- Save ลงฐานข้อมูล (CSV)
- Incremental sync (ดึงเฉพาะข้อมูลใหม่)

### แบบฝึกหัดที่ 3: Deployment Script
สร้าง deployment script ที่:
- Build application
- Run tests
- Deploy ผ่าน SSH/rsync
- Verify deployment สำเร็จ
- Rollback หากล้มเหลว
- Notify ผ่าน Slack

---

## สรุป

| หัวข้อ | Steps |
|--------|-------|
| Network basics | 362 |
| curl basics | 363 |
| curl with APIs | 364 |
| jq JSON processor | 365 |
| jq advanced | 366 |
| wget | 367 |
| SSH | 368 |
| HTTP API client | 369 |
| Webhook server | 370 |
| Network monitoring | 371 |
| Workshop | 372 |

**ขั้นตอนต่อไป**: Part 16 - Git Automation
