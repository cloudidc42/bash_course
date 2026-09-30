# Part 11: Regular Expressions และ grep

## Module 2: Intermediate Level (ระดับกลาง)

---

## ขั้นตอนที่ 291: Regular Expressions คืออะไร?

Regular Expressions (Regex หรือ RegExp) คือภาษาสำหรับกำหนด **pattern** ในการค้นหาและจับคู่ข้อความ

```bash
#!/usr/bin/env bash
# regex_intro.sh - แนะนำ Regular Expressions

echo "=== Regular Expressions เบื้องต้น ==="
echo ""

# ตัวอย่างง่ายๆ: ค้นหาคำในไฟล์
echo "apple
banana
cherry
apricot
avocado" > /tmp/fruits.txt

echo "1. ค้นหาผลไม้ที่ขึ้นต้นด้วย 'a':"
grep "^a" /tmp/fruits.txt

echo ""
echo "2. ค้นหาผลไม้ที่ลงท้ายด้วย 'y':"
grep "y$" /tmp/fruits.txt

echo ""
echo "3. ค้นหาผลไม้ที่มีตัวอักษร 'an':"
grep "an" /tmp/fruits.txt
```

### Regex Metacharacters หลัก

| Character | ความหมาย | ตัวอย่าง |
|-----------|----------|---------|
| `.` | ตัวอักษรใดๆ 1 ตัว | `a.c` → abc, aXc |
| `*` | 0 หรือมากกว่า | `ab*c` → ac, abc, abbc |
| `+` | 1 หรือมากกว่า | `ab+c` → abc, abbc |
| `?` | 0 หรือ 1 ครั้ง | `ab?c` → ac, abc |
| `^` | จุดเริ่มต้นของบรรทัด | `^abc` → บรรทัดที่เริ่มด้วย abc |
| `$` | จุดสิ้นสุดของบรรทัด | `abc$` → บรรทัดที่ลงท้ายด้วย abc |
| `[]` | Character class | `[abc]` → a หรือ b หรือ c |
| `[^]` | Negated class | `[^abc]` → ไม่ใช่ a, b, c |
| `\` | Escape character | `\.` → จุดจริงๆ |
| `\|` | OR (alternation) | `cat\|dog` → cat หรือ dog |

---

## ขั้นตอนที่ 292: Character Classes

```bash
#!/usr/bin/env bash
# char_classes.sh - Character Classes ใน Regex

echo "=== Character Classes ==="

# สร้างข้อมูลทดสอบ
cat > /tmp/test_data.txt << 'EOF'
Hello World 123
abc DEF 456
foo_bar-baz
email@example.com
phone: +66-81-234-5678
price: $99.99
date: 2024-01-15
EOF

echo "1. [a-z] - ตัวพิมพ์เล็ก:"
grep -o "[a-z]" /tmp/test_data.txt | tr -d '\n' && echo ""

echo ""
echo "2. [A-Z] - ตัวพิมพ์ใหญ่:"
grep -o "[A-Z]" /tmp/test_data.txt | tr -d '\n' && echo ""

echo ""
echo "3. [0-9] - ตัวเลข:"
grep -o "[0-9]" /tmp/test_data.txt | tr -d '\n' && echo ""

echo ""
echo "4. [a-zA-Z0-9] - alphanumeric:"
grep -o "[a-zA-Z0-9]" /tmp/test_data.txt | tr -d '\n' && echo ""

echo ""
echo "5. POSIX classes:"
echo "   [:alpha:] - ตัวอักษร"
grep -o "[[:alpha:]]" /tmp/test_data.txt | tr -d '\n' && echo ""

echo "   [:digit:] - ตัวเลข"
grep -o "[[:digit:]]" /tmp/test_data.txt | tr -d '\n' && echo ""

echo "   [:space:] - whitespace"
grep -c "[[:space:]]" /tmp/test_data.txt

echo "   [:upper:] - uppercase"
grep -o "[[:upper:]]" /tmp/test_data.txt | tr -d '\n' && echo ""

echo "   [:lower:] - lowercase"
grep -o "[[:lower:]]" /tmp/test_data.txt | tr -d '\n' && echo ""

echo "   [:alnum:] - alphanumeric"
echo "   [:punct:] - punctuation"
grep -o "[[:punct:]]" /tmp/test_data.txt | tr -d '\n' && echo ""
```

### POSIX Character Classes

```
[:alpha:]   - ตัวอักษร (a-z, A-Z)
[:digit:]   - ตัวเลข (0-9)
[:alnum:]   - ตัวอักษรและตัวเลข
[:space:]   - whitespace (\t, \n, \r, space)
[:upper:]   - uppercase
[:lower:]   - lowercase
[:punct:]   - เครื่องหมายวรรคตอน
[:print:]   - ตัวอักษรที่พิมพ์ได้
[:graph:]   - ตัวอักษรที่พิมพ์ได้ (ไม่รวม space)
```

---

## ขั้นตอนที่ 293: Quantifiers (ตัวกำหนดจำนวน)

```bash
#!/usr/bin/env bash
# quantifiers.sh - Regex Quantifiers

echo "=== Regex Quantifiers ==="

# สร้างข้อมูลทดสอบ
cat > /tmp/quant_test.txt << 'EOF'
color
colour
colur
colouur
colouuur
EOF

echo "1. ? (0 หรือ 1 ครั้ง):"
grep -E "colou?r" /tmp/quant_test.txt

echo ""
echo "2. * (0 หรือมากกว่า):"
grep -E "colou*r" /tmp/quant_test.txt

echo ""
echo "3. + (1 หรือมากกว่า):"
grep -E "colou+r" /tmp/quant_test.txt

# สร้างข้อมูลทดสอบ range
cat > /tmp/quant_range.txt << 'EOF'
a
ab
abc
abcd
abcde
abcdef
abcdefg
EOF

echo ""
echo "4. {n} - ตรงกันพอดี n ครั้ง:"
grep -E "^.{3}$" /tmp/quant_range.txt

echo ""
echo "5. {n,} - อย่างน้อย n ครั้ง:"
grep -E "^.{4,}$" /tmp/quant_range.txt

echo ""
echo "6. {n,m} - ระหว่าง n ถึง m ครั้ง:"
grep -E "^.{2,4}$" /tmp/quant_range.txt
```

---

## ขั้นตอนที่ 294: Anchors และ Word Boundaries

```bash
#!/usr/bin/env bash
# anchors.sh - Regex Anchors

echo "=== Anchors ==="

cat > /tmp/anchor_test.txt << 'EOF'
The cat sat on the mat
cats are cool
This is a cat
concatenate
cat
EOF

echo "1. ^ - จุดเริ่มต้นบรรทัด:"
grep "^cat" /tmp/anchor_test.txt

echo ""
echo "2. \$ - จุดสิ้นสุดบรรทัด:"
grep "cat$" /tmp/anchor_test.txt

echo ""
echo "3. \b - word boundary (ต้องใช้ grep -w หรือ \b):"
grep -w "cat" /tmp/anchor_test.txt

echo ""
echo "4. \b ใน extended regex:"
grep -E "\bcat\b" /tmp/anchor_test.txt

echo ""
echo "5. ^ และ \$ พร้อมกัน (บรรทัดที่ตรงกันพอดี):"
grep "^cat$" /tmp/anchor_test.txt

echo ""
echo "6. .* (ทุกอย่าง):"
grep "^The.*mat$" /tmp/anchor_test.txt
```

---

## ขั้นตอนที่ 295: Groups และ Alternation

```bash
#!/usr/bin/env bash
# groups.sh - Groups และ Alternation

echo "=== Groups และ Alternation ==="

cat > /tmp/group_test.txt << 'EOF'
cat
dog
fish
cats
dogs
bird
catfish
dogfish
EOF

echo "1. Alternation (|):"
grep -E "cat|dog" /tmp/group_test.txt

echo ""
echo "2. Groups ด้วย ():"
grep -E "(cat|dog)fish" /tmp/group_test.txt

echo ""
echo "3. Repetition ของ group:"
grep -E "^(ca)+t$" /tmp/group_test.txt

# Backreferences
cat > /tmp/backref_test.txt << 'EOF'
aabbcc
abcabc
hello hello
world
the the problem
EOF

echo ""
echo "4. Backreferences - หาคำซ้ำ:"
grep -E "\b(\w+)\s+\1\b" /tmp/backref_test.txt

echo ""
echo "5. Non-capturing group (?:):"
grep -E "(?:cat|dog)s?" /tmp/group_test.txt 2>/dev/null || \
grep -P "(?:cat|dog)s?" /tmp/group_test.txt
```

---

## ขั้นตอนที่ 296: BRE vs ERE vs PCRE

```bash
#!/usr/bin/env bash
# regex_types.sh - ประเภทของ Regex

echo "=== ประเภทของ Regular Expressions ==="
echo ""
echo "1. BRE (Basic Regular Expressions) - grep ปกติ"
echo "   - ต้อง escape metacharacters: \+, \?, \{, \}, \(, \), \|"

cat > /tmp/bre_test.txt << 'EOF'
color
colour
colur
EOF

echo ""
echo "BRE - + ต้อง escape:"
grep "colou\+r" /tmp/bre_test.txt

echo ""
echo "2. ERE (Extended Regular Expressions) - grep -E หรือ egrep"
echo "   - ไม่ต้อง escape: +, ?, {, }, (, ), |"
grep -E "colou+r" /tmp/bre_test.txt

echo ""
echo "3. เปรียบเทียบ BRE และ ERE:"

cat > /tmp/compare.txt << 'EOF'
cat
cats
catfish
the cat
EOF

echo "BRE - group ต้อง escape ():"
grep "\(cat\)s\?" /tmp/compare.txt

echo ""
echo "ERE - group ไม่ต้อง escape:"
grep -E "(cat)s?" /tmp/compare.txt

echo ""
echo "4. PCRE - ใช้ grep -P:"
echo "   - รองรับ lookahead, lookbehind, (?:), \w, \d, \s ฯลฯ"

cat > /tmp/pcre_test.txt << 'EOF'
price: 100
price: 200
discount: 50
total: 300
EOF

echo ""
echo "PCRE lookahead - หาตัวเลขหลัง 'price: ':"
grep -P "(?<=price: )\d+" /tmp/pcre_test.txt
```

---

## ขั้นตอนที่ 297: grep - Global Regular Expression Print

```bash
#!/usr/bin/env bash
# grep_basics.sh - การใช้ grep

echo "=== grep เบื้องต้น ==="

# สร้างข้อมูลทดสอบ
cat > /tmp/grep_test.txt << 'EOF'
# Configuration file
server_host=localhost
server_port=8080
db_host=192.168.1.100
db_port=5432
db_name=myapp
debug=true
log_level=INFO
max_connections=100
timeout=30
# End of config
EOF

echo "1. grep พื้นฐาน - ค้นหา pattern:"
grep "host" /tmp/grep_test.txt

echo ""
echo "2. -i - case insensitive:"
grep -i "HOST" /tmp/grep_test.txt

echo ""
echo "3. -n - แสดงเลขบรรทัด:"
grep -n "port" /tmp/grep_test.txt

echo ""
echo "4. -c - นับจำนวนบรรทัดที่ match:"
grep -c "=" /tmp/grep_test.txt

echo ""
echo "5. -l - แสดงชื่อไฟล์ที่ match (useful กับหลายไฟล์):"
grep -l "host" /tmp/grep_test.txt

echo ""
echo "6. -v - invert match (บรรทัดที่ไม่ match):"
grep -v "^#" /tmp/grep_test.txt | grep -v "^$"

echo ""
echo "7. -w - word match:"
grep -w "host" /tmp/grep_test.txt

echo ""
echo "8. -x - line match:"
grep -x "debug=true" /tmp/grep_test.txt
```

---

## ขั้นตอนที่ 298: grep - Context Lines

```bash
#!/usr/bin/env bash
# grep_context.sh - grep พร้อม context

echo "=== grep Context Lines ==="

cat > /tmp/log.txt << 'EOF'
2024-01-15 10:00:01 INFO  Server starting
2024-01-15 10:00:02 INFO  Loading config
2024-01-15 10:00:03 INFO  Config loaded
2024-01-15 10:00:04 ERROR Failed to connect to database
2024-01-15 10:00:05 ERROR Connection timeout: 192.168.1.100:5432
2024-01-15 10:00:06 ERROR Retrying connection (1/3)
2024-01-15 10:00:07 INFO  Connection established
2024-01-15 10:00:08 INFO  Database ready
2024-01-15 10:00:09 ERROR Disk space low: 90% used
2024-01-15 10:00:10 WARN  Approaching limit
2024-01-15 10:00:11 INFO  Cleanup started
2024-01-15 10:00:12 INFO  Cleanup complete: freed 2GB
EOF

echo "1. -A (after) - แสดงบรรทัดหลัง:"
grep -A 2 "ERROR" /tmp/log.txt | head -20

echo ""
echo "2. -B (before) - แสดงบรรทัดก่อน:"
grep -B 1 "ERROR" /tmp/log.txt | head -15

echo ""
echo "3. -C (context) - แสดงบรรทัดก่อนและหลัง:"
grep -C 1 "ERROR.*database\|ERROR.*timeout" /tmp/log.txt
```

---

## ขั้นตอนที่ 299: grep - Recursive และ Multiple Files

```bash
#!/usr/bin/env bash
# grep_files.sh - grep กับหลายไฟล์

echo "=== grep กับหลายไฟล์ ==="

# สร้างโครงสร้างไฟล์ทดสอบ
mkdir -p /tmp/project/{src,config,tests}

cat > /tmp/project/src/app.py << 'EOF'
import os
import sys

def main():
    config = load_config()
    server = Server(config)
    server.start()

def load_config():
    # TODO: implement config loading
    return {}

class Server:
    def __init__(self, config):
        self.config = config
        self.port = config.get('port', 8080)
    
    def start(self):
        print(f"Starting server on port {self.port}")
        # TODO: implement server start
EOF

cat > /tmp/project/src/utils.py << 'EOF'
import re
import json

# TODO: add more utilities
def validate_email(email):
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

def parse_json(text):
    return json.loads(text)
EOF

cat > /tmp/project/config/settings.py << 'EOF'
DEBUG = True
DATABASE_URL = "postgresql://localhost/mydb"
SECRET_KEY = "change-me-in-production"
# TODO: move to environment variables
EOF

cat > /tmp/project/tests/test_app.py << 'EOF'
import pytest

def test_main():
    # TODO: write tests
    pass

def test_server():
    # TODO: test server initialization
    pass
EOF

echo "1. ค้นหาใน directory (-r หรือ --recursive):"
grep -r "TODO" /tmp/project/

echo ""
echo "2. -r พร้อม -n (เลขบรรทัด):"
grep -rn "TODO" /tmp/project/

echo ""
echo "3. -r พร้อม --include (กรองชนิดไฟล์):"
grep -r --include="*.py" "def " /tmp/project/

echo ""
echo "4. -r พร้อม --exclude:"
grep -r --exclude="*.pyc" "import" /tmp/project/

echo ""
echo "5. -l (แสดงแค่ชื่อไฟล์):"
grep -rl "TODO" /tmp/project/

echo ""
echo "6. -L (ไฟล์ที่ไม่มี pattern):"
grep -rL "TODO" /tmp/project/ 2>/dev/null

echo ""
echo "7. นับ TODO ต่อไฟล์:"
grep -rc "TODO" /tmp/project/ | grep -v ":0$"
```

---

## ขั้นตอนที่ 300: grep - Output Formatting

```bash
#!/usr/bin/env bash
# grep_output.sh - จัดรูปแบบ output ของ grep

echo "=== grep Output Formatting ==="

cat > /tmp/sample.txt << 'EOF'
apple is red
banana is yellow
cherry is red
grape is purple
strawberry is red
blueberry is blue
EOF

echo "1. -o - แสดงเฉพาะส่วนที่ match:"
grep -o "is [a-z]*" /tmp/sample.txt

echo ""
echo "2. แสดงเฉพาะสี (unique):"
grep -o "is [a-z]*" /tmp/sample.txt | sort -u

echo ""
echo "3. --color - เน้นสีส่วนที่ match:"
grep --color=always "red" /tmp/sample.txt

echo ""
echo "4. -h - ซ่อนชื่อไฟล์:"
grep -h "red" /tmp/sample.txt /tmp/sample.txt

echo ""
echo "5. -H - แสดงชื่อไฟล์เสมอ:"
grep -H "red" /tmp/sample.txt

echo ""
echo "6. -m - จำกัดจำนวนบรรทัดที่ match:"
grep -m 2 "red" /tmp/sample.txt

echo ""
echo "7. ใช้ regex กับ -o เพื่อ extract ข้อมูล:"
cat > /tmp/urls.txt << 'EOF'
Visit https://www.example.com for more info
Contact us at http://contact.example.org
Our API: https://api.example.io/v1/data
EOF

grep -oE "https?://[a-zA-Z0-9./-]+" /tmp/urls.txt
```

---

## ขั้นตอนที่ 301: grep - Multiple Patterns

```bash
#!/usr/bin/env bash
# grep_multi.sh - หลาย pattern ใน grep

echo "=== Multiple Patterns ==="

cat > /tmp/log2.txt << 'EOF'
2024-01-15 INFO  Server started
2024-01-15 DEBUG Loading module A
2024-01-15 DEBUG Loading module B
2024-01-15 WARN  Memory usage high
2024-01-15 ERROR Connection failed
2024-01-15 DEBUG Retrying connection
2024-01-15 INFO  Connection restored
2024-01-15 ERROR Disk full
2024-01-15 FATAL System shutdown
EOF

echo "1. -e - หลาย pattern:"
grep -e "ERROR" -e "FATAL" /tmp/log2.txt

echo ""
echo "2. ERE alternation:"
grep -E "ERROR|FATAL|WARN" /tmp/log2.txt

echo ""
echo "3. ใช้ไฟล์ pattern (-f):"
cat > /tmp/patterns.txt << 'EOF'
ERROR
FATAL
WARN
EOF

grep -f /tmp/patterns.txt /tmp/log2.txt

echo ""
echo "4. AND condition (grep ซ้อนกัน):"
echo "   บรรทัดที่มีทั้ง 'ERROR' และ 'Connection':"
grep "ERROR" /tmp/log2.txt | grep "Connection"

echo ""
echo "5. grep ด้วย lookahead (PCRE):"
grep -P "(?=.*ERROR)(?=.*failed)" /tmp/log2.txt 2>/dev/null || echo "ไม่รองรับ PCRE"
```

---

## ขั้นตอนที่ 302: egrep และ fgrep

```bash
#!/usr/bin/env bash
# egrep_fgrep.sh - egrep และ fgrep

echo "=== egrep และ fgrep ==="
echo ""
echo "egrep = grep -E (Extended Regular Expressions)"
echo "fgrep = grep -F (Fixed strings, ไม่ใช่ regex)"
echo ""

cat > /tmp/fixed_test.txt << 'EOF'
price: $10.00
tax: 10% of total
value: (1+2)*3=9
regex: [a-z]+
special: foo.bar
EOF

echo "1. fgrep - ค้นหา literal string (ไม่ treat . เป็น wildcard):"
echo "   fgrep 'foo.bar' :"
fgrep "foo.bar" /tmp/fixed_test.txt

echo ""
echo "   grep 'foo.bar' (. เป็น wildcard):"
grep "foo.bar" /tmp/fixed_test.txt

echo ""
echo "2. fgrep สำหรับ special characters:"
echo "   หา \$10.00 :"
fgrep '$10.00' /tmp/fixed_test.txt

echo "   หา (1+2)*3=9 :"
fgrep "(1+2)*3=9" /tmp/fixed_test.txt

echo "   หา [a-z]+ :"
fgrep "[a-z]+" /tmp/fixed_test.txt

echo ""
echo "3. egrep (= grep -E):"
cat > /tmp/egrep_test.txt << 'EOF'
color
colour
color!
colours
EOF

egrep "colou?rs?" /tmp/egrep_test.txt

echo ""
echo "4. ความเร็ว: fgrep เร็วที่สุด (ไม่ต้อง parse regex)"
echo "   สำหรับค้นหา literal strings ใน large files ใช้ fgrep"
```

---

## ขั้นตอนที่ 303: Regex ใน Bash [[ =~ ]]

```bash
#!/usr/bin/env bash
# bash_regex.sh - Regex ใน Bash

echo "=== Regex ใน Bash ==="

# [[ string =~ pattern ]]
# BASH_REMATCH array เก็บผลลัพธ์

echo "1. การตรวจสอบ pattern พื้นฐาน:"
check_pattern() {
    local str="$1"
    local pattern="$2"
    if [[ "$str" =~ $pattern ]]; then
        echo "  '$str' matches '$pattern' - YES"
    else
        echo "  '$str' matches '$pattern' - NO"
    fi
}

check_pattern "hello123" "[0-9]+"
check_pattern "hello" "[0-9]+"
check_pattern "HELLO" "[A-Z]+"
check_pattern "Hello123" "^[A-Z][a-z]+[0-9]+$"

echo ""
echo "2. BASH_REMATCH - เข้าถึงส่วนที่ match:"
str="John Smith, age 30, email: john@example.com"
pattern="([A-Za-z]+) ([A-Za-z]+), age ([0-9]+)"

if [[ "$str" =~ $pattern ]]; then
    echo "  Full match: ${BASH_REMATCH[0]}"
    echo "  First name: ${BASH_REMATCH[1]}"
    echo "  Last name:  ${BASH_REMATCH[2]}"
    echo "  Age:        ${BASH_REMATCH[3]}"
fi

echo ""
echo "3. การ validate ข้อมูล:"

validate_email() {
    local email="$1"
    local pattern='^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    [[ "$email" =~ $pattern ]]
}

validate_ip() {
    local ip="$1"
    local pattern='^([0-9]{1,3}\.){3}[0-9]{1,3}$'
    if [[ "$ip" =~ $pattern ]]; then
        # ตรวจสอบแต่ละ octet
        IFS='.' read -ra octets <<< "$ip"
        for octet in "${octets[@]}"; do
            if (( octet > 255 )); then
                return 1
            fi
        done
        return 0
    fi
    return 1
}

validate_date() {
    local date="$1"
    local pattern='^[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[01])$'
    [[ "$date" =~ $pattern ]]
}

echo "Email validation:"
for email in "user@example.com" "invalid.email" "a@b.c" "test+tag@domain.org"; do
    if validate_email "$email"; then
        echo "  ✓ $email"
    else
        echo "  ✗ $email"
    fi
done

echo ""
echo "IP validation:"
for ip in "192.168.1.1" "256.0.0.1" "10.0.0.1" "abc.def.ghi.jkl"; do
    if validate_ip "$ip"; then
        echo "  ✓ $ip"
    else
        echo "  ✗ $ip"
    fi
done

echo ""
echo "Date validation:"
for date in "2024-01-15" "2024-13-01" "2024-00-01" "20240115"; do
    if validate_date "$date"; then
        echo "  ✓ $date"
    else
        echo "  ✗ $date"
    fi
done
```

---

## ขั้นตอนที่ 304: Regex Patterns ที่ใช้บ่อย

```bash
#!/usr/bin/env bash
# common_patterns.sh - Pattern ที่ใช้บ่อย

echo "=== Common Regex Patterns ==="

# Collection of useful patterns
declare -A PATTERNS=(
    ["email"]='^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    ["ipv4"]='^([0-9]{1,3}\.){3}[0-9]{1,3}$'
    ["ipv6"]='(([0-9a-fA-F]{1,4}:){7,7}[0-9a-fA-F]{1,4}|([0-9a-fA-F]{1,4}:){1,7}:)'
    ["url"]='https?://[a-zA-Z0-9._/-]+'
    ["date_ymd"]='^[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[01])$'
    ["time_hms"]='^([01][0-9]|2[0-3]):[0-5][0-9]:[0-5][0-9]$'
    ["phone_th"]='^(\+66|0)[0-9]{8,9}$'
    ["credit_card"]='^[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}$'
    ["hex_color"]='^#([A-Fa-f0-9]{6}|[A-Fa-f0-9]{3})$'
    ["uuid"]='^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$'
    ["mac_addr"]='^([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}$'
    ["username"]='^[a-zA-Z][a-zA-Z0-9_-]{2,19}$'
    ["strong_pass"]='^(?=.*[a-z])(?=.*[A-Z])(?=.*[0-9])(?=.*[!@#\$%]).{8,}$'
    ["thai_id"]='^[0-9]{13}$'
    ["thai_phone"]='^(0[689][0-9]{8})$'
)

validate() {
    local pattern_name="$1"
    local value="$2"
    local pattern="${PATTERNS[$pattern_name]}"
    
    if [[ "$value" =~ $pattern ]]; then
        echo "  ✓ [$pattern_name] '$value'"
    else
        echo "  ✗ [$pattern_name] '$value'"
    fi
}

echo "1. Email addresses:"
validate "email" "user@example.com"
validate "email" "invalid@"
validate "email" "test.user+tag@domain.co.th"

echo ""
echo "2. IP addresses:"
validate "ipv4" "192.168.1.1"
validate "ipv4" "10.0.0.1"
validate "ipv4" "999.999.999.999"

echo ""
echo "3. URLs:"
validate "url" "https://www.example.com"
validate "url" "http://api.example.io/v1/data"
validate "url" "ftp://not-valid"

echo ""
echo "4. Dates:"
validate "date_ymd" "2024-01-15"
validate "date_ymd" "2024-13-01"
validate "date_ymd" "2024-12-31"

echo ""
echo "5. Times:"
validate "time_hms" "13:45:30"
validate "time_hms" "25:00:00"
validate "time_hms" "00:00:00"

echo ""
echo "6. Thai phone numbers:"
validate "thai_phone" "0812345678"
validate "thai_phone" "0912345678"
validate "thai_phone" "1234567890"

echo ""
echo "7. Hex colors:"
validate "hex_color" "#FF5733"
validate "hex_color" "#fff"
validate "hex_color" "FF5733"

echo ""
echo "8. MAC addresses:"
validate "mac_addr" "00:1A:2B:3C:4D:5E"
validate "mac_addr" "00-1A-2B-3C-4D-5E"
validate "mac_addr" "invalid"
```

---

## ขั้นตอนที่ 305: grep ขั้นสูง - Log Analysis

```bash
#!/usr/bin/env bash
# grep_log_analysis.sh - วิเคราะห์ Log ด้วย grep

echo "=== Log Analysis ด้วย grep ==="

# สร้าง sample log
cat > /tmp/access.log << 'EOF'
192.168.1.1 - - [15/Jan/2024:10:00:01 +0700] "GET /index.html HTTP/1.1" 200 1234
192.168.1.2 - - [15/Jan/2024:10:00:02 +0700] "POST /api/login HTTP/1.1" 200 567
192.168.1.3 - - [15/Jan/2024:10:00:03 +0700] "GET /secret.php HTTP/1.1" 404 890
192.168.1.1 - - [15/Jan/2024:10:00:04 +0700] "GET /admin HTTP/1.1" 403 123
10.0.0.1 - - [15/Jan/2024:10:00:05 +0700] "GET /index.html HTTP/1.1" 200 1234
192.168.1.4 - - [15/Jan/2024:10:00:06 +0700] "DELETE /api/user/1 HTTP/1.1" 200 456
192.168.1.2 - - [15/Jan/2024:10:00:07 +0700] "GET /api/data HTTP/1.1" 500 789
192.168.1.5 - - [15/Jan/2024:10:00:08 +0700] "POST /api/login HTTP/1.1" 401 234
192.168.1.1 - - [15/Jan/2024:10:00:09 +0700] "GET /robots.txt HTTP/1.1" 200 100
192.168.1.6 - - [15/Jan/2024:10:00:10 +0700] "GET /../../../etc/passwd HTTP/1.1" 400 0
EOF

echo "1. หา HTTP errors (4xx, 5xx):"
grep -E '" [45][0-9]{2} ' /tmp/access.log

echo ""
echo "2. หา 404 errors:"
grep '" 404 ' /tmp/access.log

echo ""
echo "3. หา 5xx server errors:"
grep -E '" 5[0-9]{2} ' /tmp/access.log

echo ""
echo "4. หา POST requests:"
grep '"POST ' /tmp/access.log

echo ""
echo "5. หา DELETE requests:"
grep '"DELETE ' /tmp/access.log

echo ""
echo "6. หา requests จาก IP เฉพาะ:"
grep "^192.168.1.1 " /tmp/access.log

echo ""
echo "7. หา suspicious patterns (path traversal):"
grep -E "\.\./|%2e%2e" /tmp/access.log

echo ""
echo "8. นับ request ต่อ status code:"
grep -oE '" [0-9]{3} ' /tmp/access.log | sort | uniq -c | sort -rn

echo ""
echo "9. Extract unique IPs:"
grep -oE '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' /tmp/access.log | sort -u

echo ""
echo "10. นับ requests ต่อ IP:"
grep -oE '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' /tmp/access.log | sort | uniq -c | sort -rn
```

---

## ขั้นตอนที่ 306: grep กับ Pipeline

```bash
#!/usr/bin/env bash
# grep_pipeline.sh - grep ใน pipeline

echo "=== grep ใน Pipeline ==="

echo "1. ค้นหาและประมวลผลต่อด้วย awk:"
cat /tmp/access.log | grep '" 200 ' | awk '{print $1}' | sort -u

echo ""
echo "2. ค้นหาและนับด้วย wc:"
echo "จำนวน success requests:"
grep -c '" 200 ' /tmp/access.log

echo ""
echo "3. Pipeline ซับซ้อน:"
echo "Top 3 most requested paths:"
grep -oE '"GET [^ ]+ ' /tmp/access.log | \
    sed 's/"GET //' | sed 's/ //' | \
    sort | uniq -c | sort -rn | head -3

echo ""
echo "4. ค้นหาแบบ interactive:"
# สำหรับ demo ใช้ echo แทน
echo "สมมติว่า user พิมพ์: error"
echo "error" | xargs -I{} grep -i {} /tmp/access.log 2>/dev/null || \
    echo "(ไม่พบ 'error' ใน access.log)"

echo ""
echo "5. ใช้ process substitution:"
diff <(grep "192.168.1" /tmp/access.log | wc -l) \
     <(grep "10.0.0" /tmp/access.log | wc -l) && \
     echo "เหมือนกัน" || echo "ต่างกัน"
```

---

## ขั้นตอนที่ 307: ripgrep (rg) - grep ที่เร็วกว่า

```bash
#!/usr/bin/env bash
# ripgrep_intro.sh - แนะนำ ripgrep

echo "=== ripgrep (rg) - Modern grep ==="
echo ""
echo "ripgrep (rg) คือ grep ที่:"
echo "- เร็วกว่า grep/ag ประมาณ 2-5x"
echo "- ข้าม .gitignore โดยอัตโนมัติ"
echo "- รองรับ Unicode"
echo "- มี syntax ที่ดีกว่า"
echo ""

if command -v rg &>/dev/null; then
    echo "rg ติดตั้งแล้ว!"
    
    echo "1. ค้นหาพื้นฐาน:"
    rg "TODO" /tmp/project/ 2>/dev/null
    
    echo ""
    echo "2. ค้นหาพร้อมชนิดไฟล์:"
    rg -t py "def " /tmp/project/ 2>/dev/null
    
    echo ""
    echo "3. แสดงเฉพาะชื่อไฟล์:"
    rg -l "TODO" /tmp/project/ 2>/dev/null
    
    echo ""
    echo "4. ค้นหาแบบ case insensitive:"
    rg -i "todo" /tmp/project/ 2>/dev/null
    
    echo ""
    echo "5. Context lines:"
    rg -C 2 "TODO" /tmp/project/ 2>/dev/null
    
    echo ""
    echo "6. Count matches:"
    rg -c "TODO" /tmp/project/ 2>/dev/null
    
else
    echo "rg ไม่ได้ติดตั้ง"
    echo "ติดตั้งด้วย:"
    echo "  Ubuntu/Debian: sudo apt install ripgrep"
    echo "  macOS: brew install ripgrep"
    echo "  Fedora: sudo dnf install ripgrep"
    echo ""
    echo "เปรียบเทียบ syntax:"
    echo ""
    echo "grep:  grep -r -n 'TODO' --include='*.py' ."
    echo "rg:    rg -n 'TODO' -t py ."
    echo ""
    echo "grep:  grep -r -l 'TODO' ."
    echo "rg:    rg -l 'TODO' ."
fi
```

---

## ขั้นตอนที่ 308: Regex สำหรับ Data Extraction

```bash
#!/usr/bin/env bash
# data_extraction.sh - Extract ข้อมูลด้วย Regex

echo "=== Data Extraction ด้วย Regex ==="

# สร้าง sample data
cat > /tmp/mixed_data.txt << 'EOF'
Name: John Smith, Age: 30, Email: john.smith@example.com
Server: web-01 (192.168.1.10:8080) Status: UP
Order #ORD-2024-001 - Total: THB 1,234.50
GitHub: https://github.com/user/repo commit: abc1234
Log: [2024-01-15 10:30:45] ERROR Something went wrong
IP ranges: 10.0.0.1 to 10.0.0.254 (subnet /24)
Color codes: #FF5733 and #C0C0C0
MAC: 00:1A:2B:3C:4D:5E on eth0
EOF

echo "1. Extract emails:"
grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' /tmp/mixed_data.txt

echo ""
echo "2. Extract IP addresses:"
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' /tmp/mixed_data.txt

echo ""
echo "3. Extract URLs:"
grep -oE 'https?://[a-zA-Z0-9./_-]+' /tmp/mixed_data.txt

echo ""
echo "4. Extract hex colors:"
grep -oE '#[0-9A-Fa-f]{6}' /tmp/mixed_data.txt

echo ""
echo "5. Extract numbers with currency:"
grep -oE 'THB [0-9,]+\.[0-9]+' /tmp/mixed_data.txt

echo ""
echo "6. Extract timestamps:"
grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}' /tmp/mixed_data.txt

echo ""
echo "7. Extract order numbers:"
grep -oE 'ORD-[0-9]{4}-[0-9]{3}' /tmp/mixed_data.txt

echo ""
echo "8. Extract MAC addresses:"
grep -oE '([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}' /tmp/mixed_data.txt

echo ""
echo "9. Extract git hashes (short):"
grep -oE '\b[0-9a-f]{7}\b' /tmp/mixed_data.txt
```

---

## ขั้นตอนที่ 309: Advanced Regex Techniques

```bash
#!/usr/bin/env bash
# advanced_regex.sh - Advanced Regex Techniques

echo "=== Advanced Regex ==="

echo "1. Non-greedy matching (PCRE):"
cat > /tmp/html_test.txt << 'EOF'
<b>Hello</b> and <b>World</b>
<a href="link1">text1</a> <a href="link2">text2</a>
EOF

echo "Greedy (matches ขยายสุด):"
grep -oE "<b>.*</b>" /tmp/html_test.txt

echo "Non-greedy ด้วย PCRE:"
grep -oP "<b>.*?</b>" /tmp/html_test.txt 2>/dev/null || echo "(ต้องใช้ -P flag)"

echo ""
echo "2. Lookahead และ Lookbehind (PCRE):"
cat > /tmp/look_test.txt << 'EOF'
price: 100
discount: 20
total: 80
shipping: 15
EOF

echo "Positive lookahead - ตัวเลขที่ตามหลัง 'price: ':"
grep -oP "(?<=price: )\d+" /tmp/look_test.txt 2>/dev/null || echo "(ต้องใช้ -P flag)"

echo ""
echo "Positive lookbehind - เหมือนกัน แต่อีกแบบ:"
grep -oP "\d+(?= \()" /tmp/look_test.txt 2>/dev/null

echo ""
echo "3. Named groups (PCRE):"
str="John Smith born 1990-05-15"
echo "$str" | grep -oP "(?P<name>[A-Za-z ]+) born (?P<date>[0-9-]+)" 2>/dev/null

echo ""
echo "4. การใช้ BASH_REMATCH กับ groups:"
data="2024-01-15T10:30:45+07:00"
pattern='^([0-9]{4})-([0-9]{2})-([0-9]{2})T([0-9]{2}):([0-9]{2}):([0-9]{2})'

if [[ "$data" =~ $pattern ]]; then
    echo "Parsed datetime:"
    echo "  Year:   ${BASH_REMATCH[1]}"
    echo "  Month:  ${BASH_REMATCH[2]}"
    echo "  Day:    ${BASH_REMATCH[3]}"
    echo "  Hour:   ${BASH_REMATCH[4]}"
    echo "  Minute: ${BASH_REMATCH[5]}"
    echo "  Second: ${BASH_REMATCH[6]}"
fi

echo ""
echo "5. Multiline matching:"
cat > /tmp/multi_test.txt << 'EOF'
function hello() {
    echo "hello"
}

function world() {
    echo "world"
}
EOF

echo "ค้นหา function declarations:"
grep -E "^function [a-z_]+" /tmp/multi_test.txt
```

---

## ขั้นตอนที่ 310: Regex Performance Tips

```bash
#!/usr/bin/env bash
# regex_performance.sh - Performance Tips

echo "=== Regex Performance Tips ==="
echo ""

echo "1. ใช้ fgrep (grep -F) สำหรับ literal strings:"
echo "   เร็วที่สุดเพราะไม่ต้อง compile regex"
time (grep -c "server" /tmp/access.log 2>/dev/null) 2>&1
time (fgrep -c "server" /tmp/access.log 2>/dev/null) 2>&1

echo ""
echo "2. Anchors ช่วยเพิ่มความเร็ว:"
echo "   ^ และ \$ บอก engine ว่าต้อง match ที่ไหน"
echo "   แทนที่: grep 'error' file"
echo "   ใช้:    grep '^error' file  (ถ้า error ต้องอยู่ต้นบรรทัด)"

echo ""
echo "3. Character classes แทน alternation:"
echo "   แทนที่: (a|e|i|o|u)"
echo "   ใช้:    [aeiou]"

echo ""
echo "4. เลี่ยง .* ที่ต้นและท้าย:"
echo "   แทนที่: '.*pattern.*'"
echo "   ใช้:    'pattern' (grep แสดงทั้งบรรทัดอยู่แล้ว)"

echo ""
echo "5. ใช้ -m เพื่อหยุดเร็ว:"
echo "   grep -m 1 'pattern' huge_file.log"
echo "   หยุดหลังพบ 1 match แรก"

echo ""
echo "6. ใช้ ripgrep สำหรับ large codebases:"
echo "   - ใช้ SIMD instructions"
echo "   - ข้าม binary files โดยอัตโนมัติ"
echo "   - ข้าม .gitignore"

echo ""
echo "7. Sort ก่อน grep ถ้าข้อมูลซ้ำมาก:"
echo "   sort file | uniq | grep 'pattern'"
echo "   (เร็วกว่าถ้า pattern match กับบรรทัดซ้ำๆ)"

echo ""
echo "8. Benchmark:"
generate_large_file() {
    python3 -c "
for i in range(10000):
    print(f'Line {i}: some data with error in it sometimes')
    print(f'Line {i}: normal line without the keyword')
" > /tmp/large_test.txt 2>/dev/null
}

if command -v python3 &>/dev/null; then
    generate_large_file
    echo "Benchmark ใน /tmp/large_test.txt:"
    echo -n "grep:  "; time grep -c "error" /tmp/large_test.txt
    echo -n "fgrep: "; time fgrep -c "error" /tmp/large_test.txt
fi
```

---

## ขั้นตอนที่ 311: Workshop - Log Analyzer ขั้นสูง

```bash
#!/usr/bin/env bash
# log_analyzer_pro.sh - Log Analyzer ขั้นสูง

set -euo pipefail

readonly SCRIPT_NAME="log_analyzer_pro"
readonly VERSION="2.0.0"

# ==================== Configuration ====================
LOG_FILE=""
OUTPUT_FORMAT="text"  # text, json, csv
SEVERITY_FILTER=""
TIME_FROM=""
TIME_TO=""
SHOW_STATS=false
SHOW_TRENDS=false
TOP_N=10

# ==================== Colors ====================
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
NC='\033[0m'

# ==================== Usage ====================
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS] <log_file>

Options:
  -s, --severity LEVEL   กรองตาม severity (ERROR, WARN, INFO, DEBUG)
  -f, --from TIME        เวลาเริ่มต้น (YYYY-MM-DD HH:MM:SS)
  -t, --to TIME          เวลาสิ้นสุด
  -n, --top N            แสดง top N results (default: 10)
  --stats                แสดง statistics
  --trends               แสดง trends
  --format FORMAT        output format: text, json, csv
  -h, --help             แสดง help

Examples:
  $SCRIPT_NAME -s ERROR app.log
  $SCRIPT_NAME --stats --trends app.log
  $SCRIPT_NAME -f "2024-01-15 10:00:00" -t "2024-01-15 11:00:00" app.log
EOF
}

# ==================== Parser ====================
parse_log_line() {
    local line="$1"
    
    # Format: 2024-01-15 10:00:01 INFO  Message here
    local pattern='^([0-9]{4}-[0-9]{2}-[0-9]{2}) ([0-9]{2}:[0-9]{2}:[0-9]{2}) (ERROR|WARN|INFO|DEBUG|FATAL) +(.*)'
    
    if [[ "$line" =~ $pattern ]]; then
        echo "date=${BASH_REMATCH[1]}"
        echo "time=${BASH_REMATCH[2]}"
        echo "level=${BASH_REMATCH[3]}"
        echo "message=${BASH_REMATCH[4]}"
        return 0
    fi
    return 1
}

# ==================== Analysis Functions ====================
count_by_severity() {
    local file="$1"
    echo "Severity Distribution:"
    echo "----------------------"
    grep -oE '(ERROR|WARN|INFO|DEBUG|FATAL)' "$file" 2>/dev/null | \
        sort | uniq -c | sort -rn | \
        while read -r count level; do
            local color="$NC"
            case "$level" in
                ERROR|FATAL) color="$RED" ;;
                WARN)        color="$YELLOW" ;;
                INFO)        color="$GREEN" ;;
                DEBUG)       color="$CYAN" ;;
            esac
            printf "  ${color}%-8s${NC} : %5d\n" "$level" "$count"
        done
}

extract_errors() {
    local file="$1"
    local limit="${2:-$TOP_N}"
    
    echo "Top $limit Error Messages:"
    echo "--------------------------"
    grep -E '(ERROR|FATAL)' "$file" 2>/dev/null | \
        grep -oE '(ERROR|FATAL) +.*' | \
        sed 's/ERROR  *//; s/FATAL  *//' | \
        sort | uniq -c | sort -rn | \
        head -"$limit" | \
        while read -r count message; do
            printf "  %5d x %s\n" "$count" "$message"
        done
}

analyze_time_distribution() {
    local file="$1"
    
    echo "Hourly Distribution:"
    echo "--------------------"
    grep -oE '[0-9]{2}:[0-9]{2}:[0-9]{2}' "$file" 2>/dev/null | \
        cut -d: -f1 | \
        sort | uniq -c | \
        while read -r count hour; do
            local bar=""
            local bar_len=$(( count / 5 ))
            for ((i=0; i<bar_len && i<50; i++)); do
                bar+="█"
            done
            printf "  %02d:00 : %4d %s\n" "$((10#$hour))" "$count" "$bar"
        done
}

find_patterns() {
    local file="$1"
    
    echo "Pattern Analysis:"
    echo "-----------------"
    
    echo -n "  IP Addresses found: "
    grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' "$file" 2>/dev/null | \
        sort -u | wc -l
    
    echo -n "  URLs found: "
    grep -oE 'https?://[a-zA-Z0-9./_-]+' "$file" 2>/dev/null | sort -u | wc -l
    
    echo -n "  Unique error messages: "
    grep -E 'ERROR|FATAL' "$file" 2>/dev/null | sort -u | wc -l
    
    # ค้นหา patterns ที่น่าสนใจ
    local timeout_count
    timeout_count=$(grep -c -i "timeout\|timed out" "$file" 2>/dev/null || echo 0)
    echo "  Timeout occurrences: $timeout_count"
    
    local oom_count
    oom_count=$(grep -c -i "out of memory\|oom\|memory" "$file" 2>/dev/null || echo 0)
    echo "  Memory issues: $oom_count"
    
    local auth_fail
    auth_fail=$(grep -c -i "auth.*fail\|invalid.*pass\|wrong.*pass\|unauthorized" "$file" 2>/dev/null || echo 0)
    echo "  Auth failures: $auth_fail"
}

generate_summary() {
    local file="$1"
    local total lines errors warnings
    
    total=$(wc -l < "$file" 2>/dev/null || echo 0)
    errors=$(grep -c -E 'ERROR|FATAL' "$file" 2>/dev/null || echo 0)
    warnings=$(grep -c 'WARN' "$file" 2>/dev/null || echo 0)
    
    echo "Summary:"
    echo "--------"
    echo "  File: $file"
    echo "  Total lines: $total"
    printf "  Errors/Fatal: ${RED}%d${NC}\n" "$errors"
    printf "  Warnings: ${YELLOW}%d${NC}\n" "$warnings"
    
    if (( errors > 0 )); then
        local error_rate
        error_rate=$(awk "BEGIN {printf \"%.1f\", ($errors/$total)*100}")
        echo "  Error rate: $error_rate%"
    fi
}

# ==================== Main Analysis ====================
analyze_log() {
    local file="$1"
    
    if [[ ! -f "$file" ]]; then
        echo "Error: ไม่พบไฟล์ $file" >&2
        exit 1
    fi
    
    echo -e "${BLUE}╔══════════════════════════════════════╗${NC}"
    echo -e "${BLUE}║    Log Analysis Report               ║${NC}"
    echo -e "${BLUE}╚══════════════════════════════════════╝${NC}"
    echo ""
    
    generate_summary "$file"
    echo ""
    
    count_by_severity "$file"
    echo ""
    
    extract_errors "$file"
    echo ""
    
    if [[ "$SHOW_STATS" == "true" ]]; then
        find_patterns "$file"
        echo ""
    fi
    
    if [[ "$SHOW_TRENDS" == "true" ]]; then
        analyze_time_distribution "$file"
        echo ""
    fi
}

# ==================== Argument Parsing ====================
main() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -s|--severity)
                SEVERITY_FILTER="$2"; shift 2 ;;
            -f|--from)
                TIME_FROM="$2"; shift 2 ;;
            -t|--to)
                TIME_TO="$2"; shift 2 ;;
            -n|--top)
                TOP_N="$2"; shift 2 ;;
            --stats)
                SHOW_STATS=true; shift ;;
            --trends)
                SHOW_TRENDS=true; shift ;;
            --format)
                OUTPUT_FORMAT="$2"; shift 2 ;;
            -h|--help)
                usage; exit 0 ;;
            -*)
                echo "Unknown option: $1" >&2; usage; exit 1 ;;
            *)
                LOG_FILE="$1"; shift ;;
        esac
    done
    
    if [[ -z "$LOG_FILE" ]]; then
        # สร้าง demo log ถ้าไม่ระบุไฟล์
        cat > /tmp/demo_app.log << 'EOF'
2024-01-15 10:00:01 INFO  Application started
2024-01-15 10:00:02 INFO  Loading configuration
2024-01-15 10:00:03 DEBUG Config: server.port=8080
2024-01-15 10:00:04 INFO  Connecting to database
2024-01-15 10:00:05 ERROR Failed to connect: timeout after 30s
2024-01-15 10:00:06 WARN  Retrying connection (1/3)
2024-01-15 10:00:07 ERROR Failed to connect: connection refused
2024-01-15 10:00:08 WARN  Retrying connection (2/3)
2024-01-15 10:00:09 INFO  Connection established
2024-01-15 10:01:15 INFO  Processing request from 192.168.1.1
2024-01-15 10:01:16 DEBUG Query: SELECT * FROM users WHERE id=1
2024-01-15 10:01:17 INFO  Request processed in 150ms
2024-01-15 10:02:30 ERROR Out of memory: cannot allocate 512MB
2024-01-15 10:02:31 FATAL System shutdown initiated
2024-01-15 10:02:32 INFO  Cleanup completed
EOF
        LOG_FILE="/tmp/demo_app.log"
        echo "(ใช้ demo log: $LOG_FILE)"
        echo ""
    fi
    
    analyze_log "$LOG_FILE"
}

main "$@"
```

---

## ขั้นตอนที่ 312: สรุป Module 2 Part 11

```bash
#!/usr/bin/env bash
# summary_regex.sh

echo "=== สรุป Regular Expressions และ grep ==="
echo ""
echo "Key Concepts ที่เรียนใน Part 11:"
echo ""
echo "1. Regex Fundamentals:"
echo "   . * + ? [] [^] ^ \$ \\ |"
echo "   Quantifiers: {n} {n,} {n,m}"
echo "   Groups: () และ backreferences \1"
echo ""
echo "2. BRE vs ERE vs PCRE:"
echo "   BRE:  grep 'pattern'"
echo "   ERE:  grep -E 'pattern'"
echo "   PCRE: grep -P 'pattern' (lookahead/lookbehind)"
echo ""
echo "3. grep options สำคัญ:"
echo "   -i  case insensitive"
echo "   -n  show line numbers"
echo "   -r  recursive"
echo "   -l  show filenames only"
echo "   -c  count matches"
echo "   -v  invert match"
echo "   -w  word match"
echo "   -o  show only matching part"
echo "   -A/-B/-C  context lines"
echo "   -E  extended regex"
echo "   -F  fixed string (fgrep)"
echo "   -P  PCRE"
echo ""
echo "4. Bash regex:"
echo "   [[ \"\$str\" =~ \$pattern ]]"
echo "   BASH_REMATCH[0..n]"
echo ""
echo "5. Common Patterns:"
echo "   Email, IP, URL, Date, Phone, Color, UUID"
echo ""
echo "Next: Part 12 - sed (Stream Editor)"
```

---

## แบบฝึกหัด Part 11

### แบบฝึกหัดที่ 1: Validate ข้อมูล
สร้างสคริปต์ที่รับ input จากผู้ใช้และ validate:
- ชื่อผู้ใช้: a-z, A-Z, 0-9, _ (5-20 ตัวอักษร)
- รหัสผ่าน: อย่างน้อย 8 ตัว มีตัวเลขและตัวอักษร
- เบอร์โทรไทย: 0[689][0-9]{8}
- วันเกิด: YYYY-MM-DD

### แบบฝึกหัดที่ 2: Log Parser
สร้าง log parser ที่:
- แยก timestamp, level, message จากแต่ละบรรทัด
- รวมกลุ่ม errors ที่คล้ายกัน
- แสดง timeline ของ errors

### แบบฝึกหัดที่ 3: Config File Searcher
เขียนสคริปต์ที่:
- ค้นหา config files ใน system
- Extract key-value pairs
- รายงาน settings ที่ใช้ default values (เช่น password=default)

### แบบฝึกหัดที่ 4: Web Scraper
ใช้ grep + regex เพื่อ:
- Extract ทุก link จาก HTML file
- หา email addresses ทั้งหมด
- List ทุก phone number ที่พบ

---

## สรุป

Part 11 ครอบคลุม Regular Expressions ทั้งหมด:

| หัวข้อ | Steps |
|--------|-------|
| Regex fundamentals | 291-292 |
| Quantifiers | 293 |
| Anchors & boundaries | 294 |
| Groups & alternation | 295 |
| BRE/ERE/PCRE | 296 |
| grep basics | 297 |
| Context lines | 298 |
| Recursive & multi-file | 299 |
| Output formatting | 300 |
| Multiple patterns | 301 |
| egrep & fgrep | 302 |
| Bash [[ =~ ]] | 303 |
| Common patterns | 304 |
| Log analysis | 305 |
| Pipeline | 306 |
| ripgrep | 307 |
| Data extraction | 308 |
| Advanced techniques | 309 |
| Performance tips | 310 |
| Workshop | 311 |

**ขั้นตอนต่อไป**: Part 12 - sed: Stream Editor
