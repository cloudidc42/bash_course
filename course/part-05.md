# Part 05: เงื่อนไข if-else และ Operators
## หลักสูตร Bash Script - ขั้นตอนที่ 111-140

---

## ขั้นตอนที่ 111: if-else พื้นฐาน

### 111.1 โครงสร้าง if

```bash
#!/bin/bash

# if พื้นฐาน
if [ condition ]; then
    # code
fi

# if-else
if [ condition ]; then
    # true branch
else
    # false branch
fi

# if-elif-else
if [ condition1 ]; then
    # branch 1
elif [ condition2 ]; then
    # branch 2
elif [ condition3 ]; then
    # branch 3
else
    # default branch
fi

# ตัวอย่างจริง
age=18

if [ "$age" -ge 18 ]; then
    echo "ผู้ใหญ่"
else
    echo "เยาวชน"
fi

# ตัวอย่าง if-elif-else
score=75

if [ "$score" -ge 90 ]; then
    echo "A"
elif [ "$score" -ge 80 ]; then
    echo "B"
elif [ "$score" -ge 70 ]; then
    echo "C"
elif [ "$score" -ge 60 ]; then
    echo "D"
else
    echo "F"
fi
```

### 111.2 [ ] vs [[ ]] vs (( ))

```bash
#!/bin/bash

# [ ] - POSIX test command
# [[ ]] - Bash extended test (แนะนำ)
# (( )) - Arithmetic evaluation

# === [ ] test ===
# เก่า แต่ portable
[ "$x" -eq 5 ]
[ -f "$file" ]

# === [[ ]] extended test (แนะนำ!) ===
# รองรับ regex, pattern matching, && || โดยตรง
[[ "$x" -eq 5 ]]
[[ "$str" == "hello" ]]
[[ "$str" =~ ^[0-9]+$ ]]    # Regex matching
[[ -f "$file" && -r "$file" ]]

# === (( )) arithmetic ===
# สำหรับการเปรียบเทียบตัวเลข
(( x > 5 ))
(( x >= 10 && x <= 20 ))

# ตัวอย่างเปรียบเทียบ
x=10
str="hello"
file="/etc/passwd"

# POSIX style
if [ "$x" -eq 10 ]; then echo "POSIX: x = 10"; fi

# Bash extended test
if [[ "$x" -eq 10 ]]; then echo "Extended: x = 10"; fi
if [[ "$str" == "hello" ]]; then echo "String match"; fi
if [[ "$str" =~ ^h ]]; then echo "Starts with h"; fi

# Arithmetic
if (( x > 5 )); then echo "Arithmetic: x > 5"; fi
if (( x >= 10 && x <= 20 )); then echo "In range 10-20"; fi
```

---

## ขั้นตอนที่ 112: Test Operators

### 112.1 File Test Operators

```bash
#!/bin/bash

file="/etc/passwd"
dir="/etc"
link="/bin/sh"

# File existence
[ -e "$file" ] && echo "มีอยู่ (exist)"
[ -f "$file" ] && echo "เป็นไฟล์ธรรมดา"
[ -d "$dir" ]  && echo "เป็น directory"
[ -L "$link" ] && echo "เป็น symbolic link"
[ -p /dev/stdin ] && echo "เป็น pipe"
[ -S /run/docker.sock ] && echo "เป็น socket"
[ -b /dev/sda ] && echo "เป็น block device"
[ -c /dev/null ] && echo "เป็น character device"

# File permissions
[ -r "$file" ] && echo "อ่านได้ (readable)"
[ -w "$file" ] && echo "เขียนได้ (writable)"
[ -x "$file" ] && echo "รันได้ (executable)"
[ -u "$file" ] && echo "มี setuid bit"
[ -g "$file" ] && echo "มี setgid bit"
[ -k "/tmp" ]  && echo "มี sticky bit"

# File size/content
[ -s "$file" ] && echo "ไม่ว่าง (non-empty)"
[ -z "$file" ] && echo "ว่าง (empty)"  # ใช้กับ string

# File comparison
[ "file1" -nt "file2" ] && echo "file1 ใหม่กว่า file2"  # newer than
[ "file1" -ot "file2" ] && echo "file1 เก่ากว่า file2"  # older than
[ "file1" -ef "file2" ] && echo "เป็นไฟล์เดียวกัน"       # equal file

# ตัวอย่างใช้งานจริง
check_file() {
    local file="$1"
    
    if [ ! -e "$file" ]; then
        echo "ไม่พบไฟล์: $file"
        return 1
    fi
    
    echo "ตรวจสอบ: $file"
    [ -f "$file" ] && echo "  ✓ เป็นไฟล์ธรรมดา"
    [ -d "$file" ] && echo "  ✓ เป็น directory"
    [ -r "$file" ] && echo "  ✓ อ่านได้"
    [ -w "$file" ] && echo "  ✓ เขียนได้"
    [ -x "$file" ] && echo "  ✓ รันได้"
    [ -s "$file" ] && echo "  ✓ มีเนื้อหา ($(du -sh "$file" | cut -f1))"
}

check_file "/etc/passwd"
```

### 112.2 String Operators

```bash
#!/bin/bash

str1="Hello"
str2="World"
empty=""

# ความยาว
[ -z "$empty" ] && echo "string ว่าง"
[ -n "$str1" ]  && echo "string ไม่ว่าง"

# เปรียบเทียบ
[ "$str1" = "$str2" ]  && echo "เท่ากัน"
[ "$str1" != "$str2" ] && echo "ไม่เท่ากัน"
[ "$str1" \< "$str2" ] && echo "$str1 < $str2 ตาม ASCII"  # ใน [ ]
[[ "$str1" < "$str2" ]] && echo "$str1 < $str2"           # ใน [[ ]]

# Pattern matching (เฉพาะใน [[ ]])
filename="report_2026.txt"
[[ "$filename" == *.txt ]]        && echo "เป็น .txt"
[[ "$filename" == report_* ]]     && echo "ขึ้นต้นด้วย report_"
[[ "$filename" == *2026* ]]       && echo "มี 2026"
[[ "$filename" != *.pdf ]]        && echo "ไม่ใช่ .pdf"

# Regex matching
email="user@example.com"
[[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]] \
    && echo "Email ถูกต้อง" || echo "Email ไม่ถูกต้อง"

# ดึงข้อมูลที่ match
text="Error: code 404 not found"
if [[ "$text" =~ code[[:space:]]([0-9]+) ]]; then
    echo "Error code: ${BASH_REMATCH[1]}"
fi
```

### 112.3 Numeric Operators

```bash
#!/bin/bash

a=10
b=5

# [ ] operators (ใช้ flag)
[ "$a" -eq "$b" ] && echo "เท่ากัน (-eq)"
[ "$a" -ne "$b" ] && echo "ไม่เท่ากัน (-ne)"
[ "$a" -gt "$b" ] && echo "มากกว่า (-gt)"
[ "$a" -ge "$b" ] && echo "มากกว่าหรือเท่ากับ (-ge)"
[ "$a" -lt "$b" ] && echo "น้อยกว่า (-lt)"
[ "$a" -le "$b" ] && echo "น้อยกว่าหรือเท่ากับ (-le)"

# [[ ]] operators (เหมือน [ ])
[[ "$a" -gt "$b" ]] && echo "[[ ]] greater than"

# (( )) operators (ใช้ C-style)
(( a == b )) && echo "เท่ากัน (==)"
(( a != b )) && echo "ไม่เท่ากัน (!=)"
(( a > b ))  && echo "มากกว่า (>)"
(( a >= b )) && echo "มากกว่าหรือเท่ากับ (>=)"
(( a < b ))  && echo "น้อยกว่า (<)"
(( a <= b )) && echo "น้อยกว่าหรือเท่ากับ (<=)"

# ตรวจสอบ range
value=15
if (( value >= 10 && value <= 20 )); then
    echo "$value อยู่ในช่วง 10-20"
fi

# แบบ [ ]
if [ "$value" -ge 10 ] && [ "$value" -le 20 ]; then
    echo "$value อยู่ในช่วง 10-20"
fi
```

### 112.4 Logical Operators

```bash
#!/bin/bash

x=10
y=20
name="สมชาย"

# AND: && หรือ -a
if [ "$x" -gt 5 ] && [ "$y" -gt 5 ]; then
    echo "ทั้งคู่มากกว่า 5"
fi

# ใน [[ ]]
if [[ "$x" -gt 5 && "$name" == "สมชาย" ]]; then
    echo "x > 5 และ name = สมชาย"
fi

# ใน [ ] แบบเก่า (-a = AND, -o = OR)
if [ "$x" -gt 5 -a "$name" = "สมชาย" ]; then
    echo "AND ใน [ ]"
fi

# OR: || หรือ -o
if [ "$x" -gt 15 ] || [ "$y" -gt 15 ]; then
    echo "อย่างน้อยหนึ่งตัวมากกว่า 15"
fi

if [[ "$x" -gt 15 || "$y" -gt 15 ]]; then
    echo "OR ใน [[ ]]"
fi

# NOT: !
if ! [ -f "/nonexistent" ]; then
    echo "ไม่มีไฟล์นี้"
fi

if [[ ! -d "/nonexistent" ]]; then
    echo "ไม่มี directory นี้"
fi

# Short-circuit evaluation
[ -f "/etc/passwd" ] && echo "มี /etc/passwd"
[ -f "/nonexistent" ] || echo "ไม่มีไฟล์"

# Ternary-like
result=$([[ "$x" -gt 5 ]] && echo "มาก" || echo "น้อย")
echo "x is $result"
```

---

## ขั้นตอนที่ 113: case Statement

### 113.1 case พื้นฐาน

```bash
#!/bin/bash

# case statement
day="Monday"

case "$day" in
    Monday)
        echo "วันจันทร์"
        ;;
    Tuesday)
        echo "วันอังคาร"
        ;;
    Wednesday)
        echo "วันพุธ"
        ;;
    Thursday)
        echo "วันพฤหัสบดี"
        ;;
    Friday)
        echo "วันศุกร์"
        ;;
    Saturday|Sunday)    # Multiple patterns
        echo "วันหยุดสุดสัปดาห์"
        ;;
    *)                  # Default
        echo "ไม่รู้จักวัน: $day"
        ;;
esac
```

### 113.2 case กับ Pattern Matching

```bash
#!/bin/bash

# Pattern matching ใน case
check_file_type() {
    local filename="$1"
    
    case "$filename" in
        *.jpg|*.jpeg|*.png|*.gif|*.webp)
            echo "$filename เป็นรูปภาพ"
            ;;
        *.mp4|*.avi|*.mkv|*.mov)
            echo "$filename เป็นวิดีโอ"
            ;;
        *.mp3|*.wav|*.flac|*.aac)
            echo "$filename เป็นเสียง"
            ;;
        *.sh|*.bash)
            echo "$filename เป็น shell script"
            ;;
        *.py)
            echo "$filename เป็น Python"
            ;;
        *.js|*.ts)
            echo "$filename เป็น JavaScript/TypeScript"
            ;;
        *.txt|*.md)
            echo "$filename เป็นเอกสาร"
            ;;
        *)
            echo "$filename ไม่รู้จักประเภท"
            ;;
    esac
}

# ทดสอบ
for f in photo.jpg video.mp4 script.sh app.py readme.md unknown.xyz; do
    check_file_type "$f"
done
```

### 113.3 case ;& และ ;;&

```bash
#!/bin/bash

# ;& = fall-through (ไม่ break, ทำต่อ pattern ถัดไป)
# ;;& = test next pattern

level=2

# ;& example (fall-through)
case "$level" in
    3)
        echo "Level 3 features"
        ;&    # fall-through ไป level 2
    2)
        echo "Level 2 features"
        ;&    # fall-through ไป level 1
    1)
        echo "Level 1 features (basic)"
        ;;
esac
# Output:
# Level 2 features
# Level 1 features (basic)

echo "---"

# ;;& example (test multiple patterns)
value="hello_world"
case "$value" in
    *hello*)
        echo "contains 'hello'"
        ;;&    # continue testing
    *world*)
        echo "contains 'world'"
        ;;&    # continue testing
    *_*)
        echo "contains underscore"
        ;;
esac
# Output:
# contains 'hello'
# contains 'world'
# contains underscore
```

---

## ขั้นตอนที่ 114: Conditional Expressions

### 114.1 Short-circuit Operators

```bash
#!/bin/bash

# && = AND: รัน right ถ้า left เป็น true
# || = OR: รัน right ถ้า left เป็น false

# ตัวอย่าง && 
[ -f "/etc/passwd" ] && echo "พบ /etc/passwd"
command -v git &>/dev/null && echo "Git installed"

# ตัวอย่าง ||
[ -d "/tmp/mydir" ] || mkdir -p "/tmp/mydir"
command -v curl &>/dev/null || { echo "ไม่พบ curl"; exit 1; }

# Chaining
check_and_process() {
    local file="$1"
    
    [ -f "$file" ] && \
    [ -r "$file" ] && \
    wc -l < "$file"
}

# หรือใช้ &&/|| แบบ ternary
is_root() { [ "$(id -u)" -eq 0 ]; }
is_root && echo "Running as root" || echo "Not root"

# Guard clause pattern
validate_input() {
    local value="$1"
    
    [ -n "$value" ] || { echo "ต้องระบุค่า"; return 1; }
    [[ "$value" =~ ^[0-9]+$ ]] || { echo "ต้องเป็นตัวเลข"; return 1; }
    [ "$value" -gt 0 ] || { echo "ต้องเป็นเลขบวก"; return 1; }
    
    echo "Valid: $value"
    return 0
}

validate_input ""
validate_input "abc"
validate_input "0"
validate_input "42"
```

### 114.2 Ternary Expressions

```bash
#!/bin/bash

# Bash ไม่มี ternary operator โดยตรง
# แต่ทำได้หลายวิธี

x=10

# วิธีที่ 1: && ||
result=$([[ "$x" -gt 5 ]] && echo "big" || echo "small")

# วิธีที่ 2: if ใน subshell
result=$(if [ "$x" -gt 5 ]; then echo "big"; else echo "small"; fi)

# วิธีที่ 3: Parameter expansion
is_big=true
result="${is_big:+มาก}"         # ถ้า is_big ไม่ว่าง ใช้ "มาก"
result="${is_big:-น้อย}"        # ถ้า is_big ว่าง ใช้ "น้อย"

# วิธีที่ 4: Array lookup
declare -A BOOL_NAMES=([true]="มาก" [false]="น้อย")
is_big=true
result="${BOOL_NAMES[$is_big]}"
echo "$result"

# ตัวอย่างใช้งานจริง
get_grade() {
    local score="$1"
    if (( score >= 90 )); then echo "A"
    elif (( score >= 80 )); then echo "B"
    elif (( score >= 70 )); then echo "C"
    elif (( score >= 60 )); then echo "D"
    else echo "F"
    fi
}

for score in 95 85 75 65 55; do
    echo "Score $score = Grade $(get_grade "$score")"
done
```

---

## ขั้นตอนที่ 115: Complex Conditions

### 115.1 Nested Conditions

```bash
#!/bin/bash

# ตรวจสอบ user permissions
check_access() {
    local user="$1"
    local resource="$2"
    
    if [ "$user" = "admin" ]; then
        echo "Admin มีสิทธิ์เต็ม"
        return 0
    elif [ "$user" = "moderator" ]; then
        if [ "$resource" = "articles" ] || [ "$resource" = "comments" ]; then
            echo "Moderator สามารถจัดการ $resource ได้"
            return 0
        else
            echo "Moderator ไม่มีสิทธิ์จัดการ $resource"
            return 1
        fi
    else
        echo "User ทั่วไป สามารถอ่านเท่านั้น"
        return 1
    fi
}

check_access "admin" "users"
check_access "moderator" "articles"
check_access "moderator" "users"
check_access "user1" "articles"
```

### 115.2 Condition Functions

```bash
#!/bin/bash

# แยก condition เป็น function เพื่อความชัดเจน

is_directory()  { [ -d "$1" ]; }
is_file()       { [ -f "$1" ]; }
is_readable()   { [ -r "$1" ]; }
is_writable()   { [ -w "$1" ]; }
is_executable() { [ -x "$1" ]; }
is_empty()      { [ -z "$1" ]; }
is_not_empty()  { [ -n "$1" ]; }
is_integer()    { [[ "$1" =~ ^-?[0-9]+$ ]]; }
is_root()       { [ "$(id -u)" -eq 0 ]; }
command_exists() { command -v "$1" &>/dev/null; }
port_in_use()   { ss -tuln 2>/dev/null | grep -q ":$1 "; }

# ใช้งาน
file="/etc/passwd"

if is_file "$file" && is_readable "$file"; then
    echo "สามารถอ่าน $file ได้"
fi

if is_root; then
    echo "กำลังรันเป็น root"
else
    echo "ไม่ใช่ root"
fi

if command_exists "docker"; then
    echo "Docker พร้อมใช้งาน"
fi

if port_in_use 80; then
    echo "Port 80 กำลังใช้งาน"
else
    echo "Port 80 ว่าง"
fi
```

---

## ขั้นตอนที่ 116: Error Handling Patterns

### 116.1 Exit on Error

```bash
#!/bin/bash

# set -e: ออกทันทีเมื่อมี error
set -e

# set -u: error เมื่อใช้ตัวแปรที่ไม่ได้กำหนด
set -u

# set -o pipefail: error เมื่อ pipe ล้มเหลว
set -o pipefail

# รวมกัน (แนะนำ)
set -euo pipefail

# ตัวอย่าง
create_directory() {
    local dir="$1"
    
    if ! mkdir -p "$dir"; then
        echo "ERROR: ไม่สามารถสร้าง directory: $dir" >&2
        return 1
    fi
    
    echo "สร้าง directory: $dir"
    return 0
}

# ใช้กับ || เพื่อ handle error
create_directory "/tmp/myapp" || exit 1

# die function
die() {
    echo "ERROR: $*" >&2
    exit 1
}

# ใช้ die
[ -f "/etc/config" ] || die "ไม่พบ config file"
```

### 116.2 Trap for Cleanup

```bash
#!/bin/bash

# Trap สำหรับ cleanup เมื่อ script จบหรือ error

# สร้าง temp files
TEMP_FILE=$(mktemp)
TEMP_DIR=$(mktemp -d)

# Cleanup function
cleanup() {
    echo "กำลัง cleanup..."
    rm -f "$TEMP_FILE"
    rm -rf "$TEMP_DIR"
    echo "Cleanup เสร็จ"
}

# Register cleanup
trap cleanup EXIT     # ทำงานเมื่อ script จบ
trap cleanup INT      # ทำงานเมื่อ Ctrl+C
trap cleanup TERM     # ทำงานเมื่อถูก kill

# Main work
echo "Working with temp file: $TEMP_FILE"
echo "data" > "$TEMP_FILE"

echo "Working with temp dir: $TEMP_DIR"
touch "$TEMP_DIR/file1" "$TEMP_DIR/file2"

# cleanup จะถูกเรียกอัตโนมัติ
echo "Script เสร็จ"
```

---

## ขั้นตอนที่ 117: Workshop 05 - Validation System

### Workshop: Form Validation System

```bash
#!/usr/bin/env bash
# =============================================================================
# Workshop 05: validate_form.sh
# ระบบตรวจสอบข้อมูล Form
# =============================================================================

set -euo pipefail

# Error collection
declare -a ERRORS=()
declare -A FIELD_VALUES=()

# Color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Validators
validate_required() {
    local field="$1"
    local value="$2"
    
    if [ -z "$value" ]; then
        ERRORS+=("$field: จำเป็นต้องกรอก")
        return 1
    fi
    return 0
}

validate_min_length() {
    local field="$1"
    local value="$2"
    local min="$3"
    
    if [ "${#value}" -lt "$min" ]; then
        ERRORS+=("$field: ต้องมีความยาวอย่างน้อย $min ตัวอักษร")
        return 1
    fi
    return 0
}

validate_max_length() {
    local field="$1"
    local value="$2"
    local max="$3"
    
    if [ "${#value}" -gt "$max" ]; then
        ERRORS+=("$field: ต้องมีความยาวไม่เกิน $max ตัวอักษร")
        return 1
    fi
    return 0
}

validate_email_format() {
    local field="$1"
    local value="$2"
    
    if ! [[ "$value" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
        ERRORS+=("$field: รูปแบบ email ไม่ถูกต้อง")
        return 1
    fi
    return 0
}

validate_numeric() {
    local field="$1"
    local value="$2"
    
    if ! [[ "$value" =~ ^[0-9]+$ ]]; then
        ERRORS+=("$field: ต้องเป็นตัวเลขเท่านั้น")
        return 1
    fi
    return 0
}

validate_in_range() {
    local field="$1"
    local value="$2"
    local min="$3"
    local max="$4"
    
    if ! validate_numeric "$field" "$value" 2>/dev/null; then
        return 1
    fi
    
    if ! (( value >= min && value <= max )); then
        ERRORS+=("$field: ต้องอยู่ในช่วง $min - $max")
        return 1
    fi
    return 0
}

validate_pattern() {
    local field="$1"
    local value="$2"
    local pattern="$3"
    local message="$4"
    
    if ! [[ "$value" =~ $pattern ]]; then
        ERRORS+=("$field: $message")
        return 1
    fi
    return 0
}

validate_match() {
    local field="$1"
    local value="$2"
    local match_value="$3"
    
    if [ "$value" != "$match_value" ]; then
        ERRORS+=("$field: ค่าไม่ตรงกัน")
        return 1
    fi
    return 0
}

validate_unique_in_file() {
    local field="$1"
    local value="$2"
    local file="$3"
    
    if [ -f "$file" ] && grep -qx "$value" "$file" 2>/dev/null; then
        ERRORS+=("$field: '$value' มีอยู่แล้ว")
        return 1
    fi
    return 0
}

# Display errors
show_errors() {
    if [ ${#ERRORS[@]} -gt 0 ]; then
        echo -e "${RED}พบข้อผิดพลาด:${NC}"
        for err in "${ERRORS[@]}"; do
            echo -e "  ${RED}✗${NC} $err"
        done
        return 1
    fi
    return 0
}

# Clear errors
clear_errors() {
    ERRORS=()
}

# Validate user registration form
validate_registration() {
    local username="$1"
    local email="$2"
    local password="$3"
    local confirm_password="$4"
    local age="$5"
    
    clear_errors
    
    # Username validation
    validate_required "Username" "$username"
    if [ -n "$username" ]; then
        validate_min_length "Username" "$username" 3
        validate_max_length "Username" "$username" 20
        validate_pattern "Username" "$username" "^[a-zA-Z0-9_]+$" \
            "ต้องประกอบด้วยตัวอักษร ตัวเลข และ _ เท่านั้น"
    fi
    
    # Email validation
    validate_required "Email" "$email"
    if [ -n "$email" ]; then
        validate_email_format "Email" "$email"
    fi
    
    # Password validation
    validate_required "Password" "$password"
    if [ -n "$password" ]; then
        validate_min_length "Password" "$password" 8
        validate_pattern "Password" "$password" "[A-Z]" "ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว"
        validate_pattern "Password" "$password" "[a-z]" "ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว"
        validate_pattern "Password" "$password" "[0-9]" "ต้องมีตัวเลขอย่างน้อย 1 ตัว"
    fi
    
    # Confirm password
    if [ -n "$password" ] && [ -n "$confirm_password" ]; then
        validate_match "Confirm Password" "$confirm_password" "$password"
    fi
    
    # Age validation
    validate_required "Age" "$age"
    if [ -n "$age" ]; then
        validate_in_range "Age" "$age" 13 120
    fi
    
    # ตรวจสอบผล
    if show_errors; then
        echo -e "${GREEN}✓ ข้อมูลถูกต้องทั้งหมด!${NC}"
        return 0
    else
        return 1
    fi
}

# Interactive registration form
registration_form() {
    echo ""
    echo -e "${YELLOW}=== แบบฟอร์มสมัครสมาชิก ===${NC}"
    echo ""
    
    read -rp "Username: " username
    read -rp "Email: " email
    read -rsp "Password: " password; echo ""
    read -rsp "Confirm Password: " confirm_password; echo ""
    read -rp "Age: " age
    
    echo ""
    
    if validate_registration "$username" "$email" "$password" "$confirm_password" "$age"; then
        echo ""
        echo -e "${GREEN}สมัครสมาชิกสำเร็จ!${NC}"
        echo "Username: $username"
        echo "Email: $email"
        echo "Age: $age"
    fi
}

# Test with predefined data
echo "=== ทดสอบ Validation ==="

echo ""
echo "Test 1: ข้อมูลไม่ถูกต้อง"
validate_registration "ab" "notanemail" "weak" "mismatch" "abc" || true

echo ""
echo "Test 2: ข้อมูลถูกต้อง"
validate_registration "john_doe" "john@example.com" "Secure123!" "Secure123!" "25"

# หรือรัน interactive form
# registration_form
```

---

## สรุป Part 05

### สิ่งที่เรียนรู้

1. ✅ if-else ทุกรูปแบบ
2. ✅ [ ] vs [[ ]] vs (( ))
3. ✅ File test operators
4. ✅ String operators
5. ✅ Numeric operators
6. ✅ Logical operators
7. ✅ case statement กับ patterns
8. ✅ Short-circuit operators
9. ✅ Condition functions
10. ✅ Error handling patterns

### แบบฝึกหัด

**Easy:**
1. สร้าง script ตรวจสอบว่าปีที่กำหนดเป็นปีอธิกสุรทินหรือไม่
2. สร้าง script ที่เปรียบเทียบตัวเลข 3 ตัว และแสดงมากสุด/น้อยสุด

**Medium:**
3. สร้าง script ตรวจสอบ password strength ด้วยเกณฑ์หลายข้อ
4. สร้าง script ที่ตรวจสอบ disk usage และแสดง warning เมื่อเกิน threshold

**Hard:**
5. สร้าง validation library ที่ใช้งานได้หลายโปรเจกต์

---

**ต่อไป:** [Part 06 - Loop: for, while, until](part-06.md)

*Part 05 จบแล้ว! พร้อมเรียน Part 06 →*
