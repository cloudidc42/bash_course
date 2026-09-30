# Part 09: String Manipulation
## หลักสูตร Bash Script - ขั้นตอนที่ 231-260

---

## ขั้นตอนที่ 231: String Operations ครบวงจร

### 231.1 Parameter Expansion ขั้นสูง

```bash
#!/bin/bash

# =============================
# Basic String Operations
# =============================

str="Hello, World! Hello, Bash!"

# ความยาว
echo "Length: ${#str}"                  # 26

# Substring
echo "Substr: ${str:7:5}"              # World
echo "From end: ${str: -5}"            # Bash!

# ลบ prefix (shortest match #)
echo "${str#Hello, }"                  # World! Hello, Bash!

# ลบ prefix (longest match ##)
echo "${str##Hello, }"                 # Bash!

# ลบ suffix (shortest match %)
echo "${str%, *}"                      # Hello, World! Hello

# ลบ suffix (longest match %%)
echo "${str%%, *}"                     # Hello

# แทนที่ครั้งแรก
echo "${str/Hello/Hi}"                 # Hi, World! Hello, Bash!

# แทนที่ทั้งหมด
echo "${str//Hello/Hi}"                # Hi, World! Hi, Bash!

# แทนที่ที่ต้น
echo "${str/#Hello/Hi}"                # Hi, World! Hello, Bash!

# แทนที่ที่ท้าย
echo "${str/%Bash!/Python!}"           # Hello, World! Hello, Python!

# Case conversion (Bash 4+)
lower="${str,,}"
upper="${str^^}"
echo "Lower: $lower"
echo "Upper: $upper"

# Toggle case
toggled="${str~~}"
echo "Toggled: $toggled"
```

### 231.2 String Functions Library

```bash
#!/bin/bash

# =============================
# Comprehensive String Library
# =============================

# Trim functions
ltrim() { local s="$1"; s="${s#"${s%%[![:space:]]*}"}"; echo "$s"; }
rtrim() { local s="$1"; s="${s%"${s##*[![:space:]]}"}"; echo "$s"; }
trim()  { ltrim "$(rtrim "$1")"; }

# Pad functions
lpad() {
    local str="$1"
    local width="$2"
    local pad="${3:- }"
    local result="$str"
    while [ ${#result} -lt "$width" ]; do
        result="${pad}${result}"
    done
    echo "${result: -$width}"
}

rpad() {
    local str="$1"
    local width="$2"
    local pad="${3:- }"
    local result="$str"
    while [ ${#result} -lt "$width" ]; do
        result="${result}${pad}"
    done
    echo "${result:0:$width}"
}

center() {
    local str="$1"
    local width="$2"
    local pad="${3:- }"
    local len=${#str}
    local total_pad=$((width - len))
    local left_pad=$((total_pad / 2))
    local right_pad=$((total_pad - left_pad))
    
    printf "%*s%s%*s\n" "$left_pad" "" "$str" "$right_pad" ""
}

# Repeat
repeat() {
    local str="$1"
    local n="$2"
    for ((i=0; i<n; i++)); do printf "%s" "$str"; done
    echo ""
}

# Count occurrences
count_occurrences() {
    local str="$1"
    local sub="$2"
    local count=0
    local pos=0
    local sub_len=${#sub}
    
    while true; do
        local remaining="${str:$pos}"
        local idx="${remaining%%"$sub"*}"
        
        if [ "${#idx}" -eq "${#remaining}" ]; then
            break
        fi
        
        ((count++))
        ((pos += ${#idx} + sub_len))
    done
    
    echo "$count"
}

# Split string into array
split() {
    local str="$1"
    local delim="${2:-,}"
    local -n arr_ref="$3"
    
    IFS="$delim" read -ra arr_ref <<< "$str"
}

# Join array into string
join() {
    local delim="$1"
    shift
    local result=""
    local first=true
    
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

# Replace all occurrences
replace_all() {
    local str="$1"
    local from="$2"
    local to="$3"
    echo "${str//"$from"/"$to"}"
}

# Check starts/ends with
starts_with() { [[ "$1" == "$2"* ]]; }
ends_with()   { [[ "$1" == *"$2" ]]; }
contains()    { [[ "$1" == *"$2"* ]]; }

# Index of substring
index_of() {
    local str="$1"
    local sub="$2"
    local remaining="${str%%"$sub"*}"
    
    if [ "${#remaining}" -eq "${#str}" ]; then
        echo "-1"
    else
        echo "${#remaining}"
    fi
}

# Reverse string
reverse_str() {
    local str="$1"
    local result=""
    for ((i=${#str}-1; i>=0; i--)); do
        result+="${str:$i:1}"
    done
    echo "$result"
}

# Check palindrome
is_palindrome() {
    local str="${1,,}"
    str="${str//[^a-z0-9]/}"
    [ "$str" = "$(reverse_str "$str")" ]
}

# ทดสอบ
echo "=== String Library Tests ==="
echo "trim: '$(trim "  hello  ")'"
echo "lpad: '$(lpad "42" 8 "0")'"
echo "rpad: '$(rpad "hello" 10 ".")'"
echo "center: '$(center "hi" 20 "-")'"
echo "repeat: $(repeat "ab" 4)"
echo "count 'l': $(count_occurrences "hello world" "l")"
echo "index_of 'world': $(index_of "hello world" "world")"
echo "reverse: $(reverse_str "hello")"
echo "palindrome 'racecar': $(is_palindrome "racecar" && echo yes || echo no)"
echo "palindrome 'hello': $(is_palindrome "hello" && echo yes || echo no)"

declare -a parts
split "a,b,c,d" "," parts
echo "split: ${parts[@]}"
echo "join: $(join "-" "${parts[@]}")"
```

---

## ขั้นตอนที่ 232: Text Processing Patterns

### 232.1 Parse และ Transform Text

```bash
#!/bin/bash

# ============================
# Text Parsing Functions
# ============================

# แยก key=value pairs
parse_key_value() {
    local input="$1"
    declare -A result
    
    while IFS='=' read -r key value; do
        # ลบ whitespace
        key="${key// /}"
        value="${value## }"
        value="${value%% }"
        
        # ข้าม comment และบรรทัดว่าง
        [[ "$key" =~ ^[[:space:]]*# ]] && continue
        [ -z "$key" ] && continue
        
        result["$key"]="$value"
    done <<< "$input"
    
    for k in "${!result[@]}"; do
        echo "$k = ${result[$k]}"
    done
}

# Parse CSV line
parse_csv_line() {
    local line="$1"
    local -n fields_ref="$2"
    
    # Handle quoted fields
    fields_ref=()
    local field=""
    local in_quotes=false
    
    for ((i=0; i<${#line}; i++)); do
        local char="${line:$i:1}"
        
        if [ "$char" = '"' ]; then
            if [ "$in_quotes" = true ]; then
                # Check for escaped quote ""
                local next="${line:$((i+1)):1}"
                if [ "$next" = '"' ]; then
                    field+='"'
                    ((i++))
                else
                    in_quotes=false
                fi
            else
                in_quotes=true
            fi
        elif [ "$char" = ',' ] && [ "$in_quotes" = false ]; then
            fields_ref+=("$field")
            field=""
        else
            field+="$char"
        fi
    done
    
    fields_ref+=("$field")
}

# ทดสอบ CSV parser
csv_line='"John Doe","john@example.com","25","Software Engineer, Senior"'
declare -a fields
parse_csv_line "$csv_line" fields

echo "CSV parsed fields:"
for i in "${!fields[@]}"; do
    echo "  [$i] '${fields[$i]}'"
done

# Parse URL
parse_url() {
    local url="$1"
    
    # Protocol
    local protocol="${url%%://*}"
    local rest="${url#*://}"
    
    # Credentials
    local credentials=""
    if [[ "$rest" == *@* ]]; then
        credentials="${rest%%@*}"
        rest="${rest#*@}"
    fi
    
    # Host:Port/Path
    local host_port="${rest%%/*}"
    local path="/${rest#*/}"
    [ "$path" = "/$rest" ] && path="/"
    
    # Query and fragment
    local query=""
    local fragment=""
    
    if [[ "$path" == *"#"* ]]; then
        fragment="${path#*#}"
        path="${path%%#*}"
    fi
    
    if [[ "$path" == *"?"* ]]; then
        query="${path#*?}"
        path="${path%%\?*}"
    fi
    
    # Host and Port
    local host="${host_port%%:*}"
    local port="${host_port#*:}"
    [ "$port" = "$host" ] && port=""
    
    echo "Protocol:    $protocol"
    echo "Credentials: ${credentials:-none}"
    echo "Host:        $host"
    echo "Port:        ${port:-default}"
    echo "Path:        $path"
    echo "Query:       ${query:-none}"
    echo "Fragment:    ${fragment:-none}"
}

echo ""
echo "=== URL Parser ==="
parse_url "https://user:pass@api.example.com:8080/v1/users?page=1&limit=10#top"
```

### 232.2 Text Formatting

```bash
#!/bin/bash

# ============================
# Text Formatting Functions
# ============================

# Word wrap
word_wrap() {
    local text="$1"
    local width="${2:-80}"
    
    local line=""
    local word
    
    for word in $text; do
        if [ $((${#line} + ${#word} + 1)) -gt "$width" ]; then
            if [ -n "$line" ]; then
                echo "$line"
                line="$word"
            else
                echo "$word"
            fi
        else
            if [ -n "$line" ]; then
                line="$line $word"
            else
                line="$word"
            fi
        fi
    done
    
    [ -n "$line" ] && echo "$line"
}

# Format table
format_table() {
    local -a headers=()
    local -a rows=()
    
    # Read headers
    IFS='|' read -ra headers <<< "$1"
    shift
    
    # Read rows
    while [ $# -gt 0 ]; do
        rows+=("$1")
        shift
    done
    
    # Calculate column widths
    local -a col_widths=()
    for col in "${!headers[@]}"; do
        col_widths[$col]="${#headers[$col]}"
    done
    
    for row in "${rows[@]}"; do
        IFS='|' read -ra cells <<< "$row"
        for col in "${!cells[@]}"; do
            local cell_len="${#cells[$col]}"
            if [ "$cell_len" -gt "${col_widths[$col]:-0}" ]; then
                col_widths[$col]="$cell_len"
            fi
        done
    done
    
    # Print header
    local separator="+"
    for width in "${col_widths[@]}"; do
        separator+="$(printf '%*s' $((width+2)) | tr ' ' '-')+"
    done
    
    echo "$separator"
    printf "|"
    for col in "${!headers[@]}"; do
        printf " %-${col_widths[$col]}s |" "${headers[$col]}"
    done
    echo ""
    echo "$separator"
    
    # Print rows
    for row in "${rows[@]}"; do
        IFS='|' read -ra cells <<< "$row"
        printf "|"
        for col in "${!col_widths[@]}"; do
            printf " %-${col_widths[$col]}s |" "${cells[$col]:-}"
        done
        echo ""
    done
    
    echo "$separator"
}

# ทดสอบ
echo "=== Table Formatting ==="
format_table \
    "Name|Email|Age|Role" \
    "สมชาย|somchai@example.com|25|Developer" \
    "สมหญิง|somying@example.com|30|Designer" \
    "สมศักดิ์|somsak@example.com|35|Manager"

echo ""
echo "=== Word Wrap ==="
long_text="Lorem ipsum dolor sit amet consectetur adipiscing elit sed do eiusmod tempor incididunt ut labore et dolore magna aliqua"
word_wrap "$long_text" 40

# Number formatting
format_number() {
    local num="$1"
    local decimals="${2:-0}"
    
    # เพิ่ม commas
    printf "%'.${decimals}f" "$num" | \
        sed ':a;s/\([0-9]\)\([0-9]\{3\}\)\(\.\|$\)/\1,\2\3/;ta'
}

echo ""
echo "=== Number Formatting ==="
echo "$(format_number 1234567)"
echo "$(format_number 9876543.21 2)"

# Format file size
format_filesize() {
    local bytes="$1"
    local units=("B" "KB" "MB" "GB" "TB" "PB")
    local unit_idx=0
    local size="$bytes"
    
    while (( size >= 1024 && unit_idx < ${#units[@]}-1 )); do
        size=$((size / 1024))
        ((unit_idx++))
    done
    
    echo "${size} ${units[$unit_idx]}"
}

echo ""
echo "=== File Sizes ==="
for size in 512 1024 1048576 1073741824; do
    echo "$(format_filesize $size)"
done
```

---

## ขั้นตอนที่ 233: Regular Expressions ใน Bash

### 233.1 Regex Basics

```bash
#!/bin/bash

# =============================
# Regular Expressions ใน Bash [[ =~ ]]
# =============================

test_regex() {
    local str="$1"
    local pattern="$2"
    local description="$3"
    
    if [[ "$str" =~ $pattern ]]; then
        echo "✓ MATCH: $description"
        echo "  String:  '$str'"
        echo "  Pattern: '$pattern'"
        [ -n "${BASH_REMATCH[0]}" ] && echo "  Match:   '${BASH_REMATCH[0]}'"
        for i in "${!BASH_REMATCH[@]}"; do
            [ "$i" -gt 0 ] && echo "  Group $i: '${BASH_REMATCH[$i]}'"
        done
    else
        echo "✗ NO MATCH: $description"
    fi
    echo ""
}

# =============================
# Pattern Examples
# =============================

# Email validation
test_regex "user@example.com" \
    "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$" \
    "Valid email"

test_regex "not-an-email" \
    "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$" \
    "Invalid email"

# Phone number (Thai format)
test_regex "0812345678" \
    "^0[0-9]{9}$" \
    "Thai phone number"

test_regex "+66812345678" \
    "^\+66[0-9]{9}$" \
    "Thai international phone"

# Date validation
test_regex "2026-09-30" \
    "^([0-9]{4})-([0-9]{2})-([0-9]{2})$" \
    "ISO date with capture groups"

echo "Year: ${BASH_REMATCH[1]}"
echo "Month: ${BASH_REMATCH[2]}"
echo "Day: ${BASH_REMATCH[3]}"
echo ""

# IP address
test_regex "192.168.1.100" \
    "^([0-9]{1,3})\.([0-9]{1,3})\.([0-9]{1,3})\.([0-9]{1,3})$" \
    "IP address"

# URL
test_regex "https://www.example.com/path?query=value" \
    "^(https?)://([^/]+)(/.*)?\?(.+)$" \
    "URL with query"

# Credit card (basic)
test_regex "4532015112830366" \
    "^4[0-9]{15}$" \
    "Visa card format"

# Password strength
test_regex "SecureP@ss123" \
    "^(?=.*[A-Z])(?=.*[a-z])(?=.*[0-9])(?=.*[!@#$%]).{8,}$" \
    "Strong password (PCRE)"

# Bash regex ไม่รองรับ lookahead (?=...)
# ใช้หลาย patterns แทน
check_password_strength() {
    local pass="$1"
    local errors=()
    
    [ ${#pass} -lt 8 ] && errors+=("ต้องยาวอย่างน้อย 8 ตัวอักษร")
    [[ ! "$pass" =~ [A-Z] ]] && errors+=("ต้องมีตัวพิมพ์ใหญ่")
    [[ ! "$pass" =~ [a-z] ]] && errors+=("ต้องมีตัวพิมพ์เล็ก")
    [[ ! "$pass" =~ [0-9] ]] && errors+=("ต้องมีตัวเลข")
    [[ ! "$pass" =~ [!@#$%^&*] ]] && errors+=("ต้องมีอักขระพิเศษ")
    
    if [ ${#errors[@]} -eq 0 ]; then
        echo "✓ Password แข็งแกร่ง"
        return 0
    else
        echo "✗ Password ไม่แข็งแกร่ง:"
        printf "  - %s\n" "${errors[@]}"
        return 1
    fi
}

echo "=== Password Checker ==="
check_password_strength "weak"
echo ""
check_password_strength "SecureP@ss123"
```

### 233.2 grep, sed, awk Integration

```bash
#!/bin/bash

# =============================
# grep - ค้นหาด้วย regex
# =============================

# สร้างไฟล์ test
cat > /tmp/test.log << 'EOF'
2026-01-01 10:00:00 INFO Application started
2026-01-01 10:01:00 DEBUG Loading config
2026-01-01 10:02:00 ERROR Connection failed: timeout
2026-01-01 10:03:00 INFO Retrying...
2026-01-01 10:04:00 WARNING Disk space low: 15%
2026-01-01 10:05:00 ERROR Database error: connection refused
2026-01-01 10:06:00 INFO Recovery complete
EOF

echo "=== grep examples ==="

# Basic grep
grep "ERROR" /tmp/test.log

echo ""
# Extended regex
grep -E "ERROR|WARNING" /tmp/test.log

echo ""
# Extract specific patterns
grep -oE "[0-9]{2}:[0-9]{2}:[0-9]{2}" /tmp/test.log

echo ""
# Count matches
echo "Error count: $(grep -c "ERROR" /tmp/test.log)"

echo ""
# Context (before/after)
grep -A 1 "ERROR" /tmp/test.log

# =============================
# sed - แก้ไขด้วย regex
# =============================

echo ""
echo "=== sed examples ==="

# Replace
echo "Hello World" | sed 's/World/Bash/'

# Replace all
echo "aababab" | sed 's/ab/X/g'

# Delete lines matching pattern
sed '/DEBUG/d' /tmp/test.log

echo ""
# Extract timestamps
sed -n 's/\([0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\} [0-9]\{2\}:[0-9]\{2\}:[0-9]\{2\}\).*/\1/p' /tmp/test.log

echo ""
# Transform log level
sed 's/ERROR/\033[31mERROR\033[0m/g' /tmp/test.log | head -5

# =============================
# awk - ประมวลผลด้วย pattern
# =============================

echo ""
echo "=== awk examples ==="

# Print specific fields
awk '{print $2, $3}' /tmp/test.log

echo ""
# Filter by pattern
awk '/ERROR/' /tmp/test.log

echo ""
# Conditional processing
awk '$3 == "ERROR" {print "ERROR at", $2, ":", substr($0, index($0,$4))}' /tmp/test.log

echo ""
# Count by log level
awk '{count[$3]++} END {for (level in count) printf "%-10s: %d\n", level, count[level]}' \
    /tmp/test.log | sort

echo ""
# Calculate statistics
awk '
    /Disk space low/ {
        match($0, /([0-9]+)%/, arr)
        total += arr[1]
        count++
    }
    END {
        if (count > 0)
            printf "Average disk usage: %.1f%%\n", total/count
    }
' /tmp/test.log
```

---

## ขั้นตอนที่ 234: String Security

### 234.1 Input Sanitization

```bash
#!/bin/bash

# =============================
# Input Sanitization
# =============================

# ลบ dangerous characters
sanitize_input() {
    local input="$1"
    local sanitized
    
    # ลบ null bytes
    sanitized=$(echo "$input" | tr -d '\000')
    
    # ลบ control characters ยกเว้น tab และ newline
    sanitized=$(echo "$sanitized" | tr -d '[:cntrl:]' | tr -s ' ')
    
    echo "$sanitized"
}

# SQL injection prevention
escape_sql() {
    local str="$1"
    # Escape single quotes
    echo "${str//\'/\'\'}"
}

# HTML escaping
html_escape() {
    local str="$1"
    str="${str//&/&amp;}"
    str="${str//</&lt;}"
    str="${str//>/&gt;}"
    str="${str//\"/&quot;}"
    str="${str//\'/&#39;}"
    echo "$str"
}

# Shell escaping
shell_escape() {
    printf '%q' "$1"
}

# Path traversal prevention
sanitize_path() {
    local path="$1"
    local base_dir="$2"
    
    # Resolve canonical path
    local resolved
    resolved=$(realpath -m "$path" 2>/dev/null || echo "$path")
    
    # Check if within base_dir
    if [[ "$resolved" == "$base_dir"* ]]; then
        echo "$resolved"
        return 0
    else
        echo "Invalid path: path traversal detected" >&2
        return 1
    fi
}

# ทดสอบ
echo "=== Security Tests ==="

echo "HTML escape: $(html_escape '<script>alert("XSS")</script>')"
echo "SQL escape: $(escape_sql "O'Brien")"
echo "Shell escape: $(shell_escape "hello world & more")"

# Path sanitization
echo ""
echo "Path tests:"
sanitize_path "/var/www/uploads/file.txt" "/var/www" && echo "Safe path"
sanitize_path "/var/www/../../../etc/passwd" "/var/www" || echo "Blocked!"
```

---

## ขั้นตอนที่ 235: Workshop 09 - Text Processing Tool

### Workshop: Log Parser และ Reporter

```bash
#!/usr/bin/env bash
# =============================================================================
# Workshop 09: text_processor.sh
# Text Processing Tool
# =============================================================================

set -euo pipefail

# Colors
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly BLUE='\033[0;34m'
readonly CYAN='\033[0;36m'
readonly NC='\033[0m'

# ============================
# String Template Engine
# ============================

# Simple template rendering
# Template syntax: {{variable_name}}
render_template() {
    local template="$1"
    shift
    
    local rendered="$template"
    
    # Process variables (key=value pairs)
    while [ $# -gt 0 ]; do
        local pair="$1"
        local key="${pair%%=*}"
        local value="${pair#*=}"
        rendered="${rendered//\{\{$key\}\}/$value}"
        shift
    done
    
    echo "$rendered"
}

# ============================
# Text Statistics
# ============================

text_stats() {
    local text="$1"
    local words chars lines unique_words
    
    # Count
    chars="${#text}"
    words=$(echo "$text" | wc -w)
    lines=$(echo "$text" | wc -l)
    unique_words=$(echo "$text" | tr '[:upper:]' '[:lower:]' | \
        tr -cs '[:alpha:]' '\n' | sort -u | wc -l)
    
    # Average word length
    local total_word_chars=0
    local word_count=0
    for word in $text; do
        total_word_chars=$((total_word_chars + ${#word}))
        ((word_count++))
    done
    
    local avg_word_len=0
    [ "$word_count" -gt 0 ] && \
        avg_word_len=$(echo "scale=1; $total_word_chars / $word_count" | bc)
    
    echo "Characters: $chars"
    echo "Words: $words"
    echo "Lines: $lines"
    echo "Unique words: $unique_words"
    echo "Avg word length: $avg_word_len"
}

# ============================
# Markdown Processor
# ============================

# Simple Markdown to text converter
md_to_text() {
    local input
    input=$(cat "${1:-/dev/stdin}")
    
    # Headers
    echo "$input" | sed \
        -e 's/^# \(.*\)$/\n=== \1 ===/g' \
        -e 's/^## \(.*\)$/\n--- \1 ---/g' \
        -e 's/^### \(.*\)$/\n  \1:/g' \
        -e 's/\*\*\(.*\)\*\*/\1/g' \
        -e 's/\*\(.*\)\*/\1/g' \
        -e 's/`\([^`]*\)`/[\1]/g' \
        -e 's/^\s*[-*] /  • /g' \
        -e 's/^\s*[0-9]\+\. /  → /g' \
        -e 's/\[\([^]]*\)\]([^)]*)/\1/g'
}

# ============================
# Data Extraction
# ============================

# Extract emails from text
extract_emails() {
    local text="$1"
    echo "$text" | grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' | sort -u
}

# Extract URLs from text
extract_urls() {
    local text="$1"
    echo "$text" | grep -oE 'https?://[^[:space:]]+' | sort -u
}

# Extract IPs from text
extract_ips() {
    local text="$1"
    echo "$text" | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' | sort -u
}

# Extract numbers from text
extract_numbers() {
    local text="$1"
    echo "$text" | grep -oE '-?[0-9]+\.?[0-9]*' | sort -n
}

# Extract phone numbers (Thai format)
extract_phones() {
    local text="$1"
    echo "$text" | grep -oE '(0[0-9]{9}|\+66[0-9]{9})' | sort -u
}

# ============================
# Text Analysis
# ============================

word_frequency() {
    local text="$1"
    local top_n="${2:-10}"
    
    echo "$text" | \
        tr '[:upper:]' '[:lower:]' | \
        tr -cs '[:alpha:]' '\n' | \
        grep -v '^$' | \
        sort | \
        uniq -c | \
        sort -rn | \
        head -"$top_n" | \
        awk '{printf "%5d %s\n", $1, $2}'
}

sentence_analysis() {
    local text="$1"
    
    # แยกเป็นประโยค (แยกที่ . ! ?)
    local sentence_count
    sentence_count=$(echo "$text" | grep -o '[.!?]' | wc -l)
    
    local word_count
    word_count=$(echo "$text" | wc -w)
    
    local avg_words_per_sentence=0
    [ "$sentence_count" -gt 0 ] && \
        avg_words_per_sentence=$(echo "scale=1; $word_count / $sentence_count" | bc)
    
    echo "Sentences: $sentence_count"
    echo "Words: $word_count"
    echo "Avg words/sentence: $avg_words_per_sentence"
}

# ============================
# Main Program
# ============================

# Sample text for testing
SAMPLE_TEXT="Hello World! Visit us at https://example.com or email info@example.com.
Call us: 0812345678 or 0898765432.
Server IPs: 192.168.1.1 and 10.0.0.1.
Price: 1234.56 THB. Discount: 10%.
This is a sample text for testing. It contains multiple sentences.
The quick brown fox jumps over the lazy dog."

echo -e "${CYAN}=== Text Processing Tool ===${NC}"
echo ""

echo -e "${YELLOW}Input Text:${NC}"
echo "$SAMPLE_TEXT"
echo ""

echo -e "${YELLOW}1. Text Statistics:${NC}"
text_stats "$SAMPLE_TEXT"
echo ""

echo -e "${YELLOW}2. Extracted Data:${NC}"
echo "Emails:"
extract_emails "$SAMPLE_TEXT" | while read -r e; do echo "  - $e"; done

echo "URLs:"
extract_urls "$SAMPLE_TEXT" | while read -r u; do echo "  - $u"; done

echo "IPs:"
extract_ips "$SAMPLE_TEXT" | while read -r ip; do echo "  - $ip"; done

echo "Phones:"
extract_phones "$SAMPLE_TEXT" | while read -r p; do echo "  - $p"; done

echo "Numbers:"
extract_numbers "$SAMPLE_TEXT" | tr '\n' ' '
echo ""

echo ""
echo -e "${YELLOW}3. Word Frequency (Top 10):${NC}"
word_frequency "$SAMPLE_TEXT"

echo ""
echo -e "${YELLOW}4. Sentence Analysis:${NC}"
sentence_analysis "$SAMPLE_TEXT"

echo ""
echo -e "${YELLOW}5. Template Rendering:${NC}"
template="Hello, {{name}}! Your order #{{order_id}} is {{status}}."
render_template "$template" \
    "name=สมชาย" \
    "order_id=12345" \
    "status=shipped"
```

---

## สรุป Part 09

### สิ่งที่เรียนรู้

1. ✅ Parameter expansion ครบทุกรูปแบบ
2. ✅ String library functions
3. ✅ Text parsing (CSV, key-value, URL)
4. ✅ Text formatting (table, word wrap)
5. ✅ Regular expressions ใน Bash
6. ✅ grep, sed, awk integration
7. ✅ Input sanitization
8. ✅ Template engine

---

**ต่อไป:** [Part 10 - File Operations](part-10.md)

*Part 09 จบแล้ว! พร้อมเรียน Part 10 →*
