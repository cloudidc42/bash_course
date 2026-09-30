# Part 12: sed - Stream Editor

## Module 2: Intermediate Level (ระดับกลาง)

---

## ขั้นตอนที่ 313: sed คืออะไร?

**sed** (Stream EDitor) คือเครื่องมือสำหรับแก้ไขข้อความแบบ stream โดยอ่านทีละบรรทัดและประยุกต์ commands

```bash
#!/usr/bin/env bash
# sed_intro.sh - แนะนำ sed

echo "=== sed เบื้องต้น ==="
echo ""
echo "sed syntax:"
echo "  sed 'command' file"
echo "  sed -e 'command1' -e 'command2' file"
echo "  sed -n 'command' file   # ไม่พิมพ์ output โดยอัตโนมัติ"
echo "  sed -i 'command' file   # แก้ไขไฟล์ in-place"
echo ""

# ตัวอย่างง่ายๆ
cat > /tmp/sed_test.txt << 'EOF'
Hello World
This is a test
Hello again
Goodbye World
EOF

echo "1. s - Substitution (แทนที่):"
sed 's/Hello/Hi/' /tmp/sed_test.txt

echo ""
echo "2. d - Delete (ลบบรรทัด):"
sed '/Hello/d' /tmp/sed_test.txt

echo ""
echo "3. p - Print (พิมพ์):"
sed -n '/Hello/p' /tmp/sed_test.txt

echo ""
echo "4. q - Quit (หยุด):"
sed '2q' /tmp/sed_test.txt
```

---

## ขั้นตอนที่ 314: sed Substitution Command

```bash
#!/usr/bin/env bash
# sed_substitution.sh - sed s command

echo "=== sed Substitution ==="

cat > /tmp/sub_test.txt << 'EOF'
The cat sat on the mat
cats are cool animals
A cat is a cat
Hello Cat!
EOF

echo "1. แทนที่ครั้งแรก (default):"
sed 's/cat/dog/' /tmp/sub_test.txt

echo ""
echo "2. แทนที่ทั้งหมด (/g flag):"
sed 's/cat/dog/g' /tmp/sub_test.txt

echo ""
echo "3. Case insensitive (/I หรือ /i flag):"
sed 's/cat/dog/gi' /tmp/sub_test.txt

echo ""
echo "4. แทนที่ครั้งที่ n (/2 flag):"
echo "one two three one two three" | sed 's/one/ONE/2'

echo ""
echo "5. แทนที่ครั้งที่ n เป็นต้นไป (/2g):"
echo "one two three one two three" | sed 's/one/ONE/2g'

echo ""
echo "6. Delimiters อื่น (ใช้ | , # ฯลฯ):"
echo "path: /usr/local/bin" | sed 's|/usr/local|/opt|'
echo "url: http://example.com" | sed 's,http://,https://,'

echo ""
echo "7. Backreferences (\1, \2):"
echo "2024-01-15" | sed 's/\([0-9]*\)-\([0-9]*\)-\([0-9]*\)/\3\/\2\/\1/'

echo ""
echo "8. ERE backreferences (\1 กับ -E):"
echo "John Smith" | sed -E 's/([A-Za-z]+) ([A-Za-z]+)/\2, \1/'

echo ""
echo "9. & - ส่วนที่ match ทั้งหมด:"
echo "hello world" | sed 's/[a-z]*/[&]/'
echo "hello world" | sed 's/[a-z]*/[&]/g'
```

---

## ขั้นตอนที่ 315: sed Address Ranges

```bash
#!/usr/bin/env bash
# sed_addresses.sh - Address Ranges

echo "=== sed Addresses ==="

cat > /tmp/addr_test.txt << 'EOF'
Line 1: header
Line 2: data
Line 3: data
Line 4: data
Line 5: footer
EOF

echo "1. บรรทัดเฉพาะ:"
sed '2s/data/DATA/' /tmp/addr_test.txt

echo ""
echo "2. Range ของบรรทัด:"
sed '2,4s/data/DATA/' /tmp/addr_test.txt

echo ""
echo "3. ตั้งแต่บรรทัด n ถึงสุดท้าย:"
sed '3,$s/data/DATA/' /tmp/addr_test.txt

echo ""
echo "4. Pattern address:"
sed '/header/s/header/HEADER/' /tmp/addr_test.txt

echo ""
echo "5. Pattern range:"
sed '/header/,/footer/s/Line/LINE/' /tmp/addr_test.txt

echo ""
echo "6. Negation (!):"
sed '/header/!s/data/DATA/' /tmp/addr_test.txt

echo ""
echo "7. บรรทัดสุดท้าย (\$):"
sed '$s/footer/FOOTER/' /tmp/addr_test.txt

echo ""
echo "8. Step address (first~step):"
echo "Lines 1,3,5 (every 2nd starting from 1):"
sed -n '1~2p' /tmp/addr_test.txt

echo ""
echo "Lines 2,4 (every 2nd starting from 2):"
sed -n '2~2p' /tmp/addr_test.txt
```

---

## ขั้นตอนที่ 316: sed Delete, Print, Quit Commands

```bash
#!/usr/bin/env bash
# sed_dpa.sh - d, p, q commands

echo "=== sed d, p, q Commands ==="

cat > /tmp/dpq_test.txt << 'EOF'
# Comment line
key1=value1
# Another comment
key2=value2
key3=value3
# Final comment
key4=value4
EOF

echo "1. d - ลบบรรทัด:"
echo "ลบ comments:"
sed '/^#/d' /tmp/dpq_test.txt

echo ""
echo "ลบบรรทัดว่าง:"
printf "line1\n\nline2\n\nline3\n" | sed '/^$/d'

echo ""
echo "2. p - พิมพ์ (มักใช้กับ -n):"
echo "แสดงเฉพาะบรรทัดที่มี 'key':"
sed -n '/key/p' /tmp/dpq_test.txt

echo ""
echo "3. = - พิมพ์เลขบรรทัด:"
sed -n '/key/=' /tmp/dpq_test.txt

echo ""
echo "4. q - หยุดที่บรรทัด n:"
echo "แสดง 3 บรรทัดแรก:"
sed '3q' /tmp/dpq_test.txt

echo ""
echo "5. Q - หยุดโดยไม่พิมพ์บรรทัดสุดท้าย:"
sed '3Q' /tmp/dpq_test.txt

echo ""
echo "6. รวม commands ด้วย ; :"
sed '/^#/d; /^$/d' /tmp/dpq_test.txt

echo ""
echo "7. หรือใช้ -e หลายครั้ง:"
sed -e '/^#/d' -e '/^$/d' /tmp/dpq_test.txt
```

---

## ขั้นตอนที่ 317: sed Insert, Append, Change

```bash
#!/usr/bin/env bash
# sed_iac.sh - i, a, c commands

echo "=== sed i, a, c Commands ==="

cat > /tmp/iac_test.txt << 'EOF'
line1
line2
line3
EOF

echo "1. i - แทรกก่อนบรรทัด:"
sed '2i\INSERTED BEFORE LINE 2' /tmp/iac_test.txt

echo ""
echo "2. a - ต่อท้ายบรรทัด:"
sed '2a\APPENDED AFTER LINE 2' /tmp/iac_test.txt

echo ""
echo "3. c - เปลี่ยนบรรทัด:"
sed '2c\CHANGED LINE 2' /tmp/iac_test.txt

echo ""
echo "4. แทรกหลาย lines (ใช้ \\n):"
sed '1a\Added line 1\nAdded line 2' /tmp/iac_test.txt

echo ""
echo "5. แทรกก่อน pattern:"
sed '/line2/i\--- separator ---' /tmp/iac_test.txt

echo ""
echo "6. เพิ่มส่วนหัวและท้าย:"
cat /tmp/iac_test.txt | sed '1i\=== START ===' | sed '$a\=== END ==='

echo ""
echo "7. เพิ่ม line number ทุกบรรทัด:"
sed '=' /tmp/iac_test.txt | sed 'N;s/\n/\t/'
```

---

## ขั้นตอนที่ 318: sed Multiline Operations

```bash
#!/usr/bin/env bash
# sed_multiline.sh - Multiline operations

echo "=== sed Multiline ==="

echo "1. N - อ่านบรรทัดถัดไปเข้า pattern space:"
cat > /tmp/multi_test.txt << 'EOF'
Name: John
Age: 30
Name: Jane
Age: 25
EOF

# รวมสอง patterns ในบรรทัดเดียว
sed 'N;s/\nAge/,Age/' /tmp/multi_test.txt

echo ""
echo "2. แก้ line continuation:"
cat > /tmp/continuation.txt << 'EOF'
This is a very long \
line that continues \
on multiple lines
Another normal line
EOF

sed ':a;/\\$/N;s/\\\n//;ta' /tmp/continuation.txt

echo ""
echo "3. P - พิมพ์แค่ส่วนแรกของ multi-line:"
echo -e "line1\nline2\nline3" | sed 'N;P;D'

echo ""
echo "4. D - ลบส่วนแรกของ multi-line:"
echo "join lines that start with space:"
cat > /tmp/indent_test.txt << 'EOF'
function foo() {
    body line 1
    body line 2
}
function bar() {
    body
}
EOF
sed -n '/^[a-z]/p' /tmp/indent_test.txt
```

---

## ขั้นตอนที่ 319: sed Hold Space

```bash
#!/usr/bin/env bash
# sed_hold.sh - Hold space operations

echo "=== sed Hold Space ==="
echo ""
echo "sed มี 2 buffers:"
echo "  pattern space - บรรทัดที่กำลังประมวลผล"
echo "  hold space    - ที่เก็บข้อมูลชั่วคราว"
echo ""
echo "Commands:"
echo "  h - copy pattern → hold"
echo "  H - append pattern → hold"
echo "  g - copy hold → pattern"
echo "  G - append hold → pattern"
echo "  x - exchange pattern ↔ hold"

cat > /tmp/hold_test.txt << 'EOF'
apple
banana
cherry
EOF

echo ""
echo "1. ย้อนกลับบรรทัด:"
# เก็บทุกบรรทัดใน hold, แล้ว print สลับ
sed -n '1!G;h;$p' /tmp/hold_test.txt

echo ""
echo "2. ทำซ้ำบรรทัดสุดท้าย:"
sed -n '$p;$p' /tmp/hold_test.txt

echo ""
echo "3. Duplicate each line:"
sed 'p' /tmp/hold_test.txt

echo ""
echo "4. แสดงเฉพาะบรรทัดที่ไม่ซ้ำ (จาก sorted input):"
printf "a\na\nb\nc\nc\nd\n" | sed '$!N; /^\(.*\)\n\1$/!P; D'

echo ""
echo "5. สลับคู่ของบรรทัด:"
printf "1\n2\n3\n4\n5\n6\n" | sed -n 'h;n;p;g;p'
```

---

## ขั้นตอนที่ 320: sed Branching and Labels

```bash
#!/usr/bin/env bash
# sed_branch.sh - Branching และ Labels

echo "=== sed Branching ==="
echo ""
echo "Branching commands:"
echo "  :label  - กำหนด label"
echo "  b label - กระโดดไป label (unconditional)"
echo "  t label - กระโดดถ้า substitution สำเร็จ"
echo "  T label - กระโดดถ้า substitution ไม่สำเร็จ"

echo ""
echo "1. Loop ด้วย branch:"
# แทนที่ทุก _ ด้วย -
echo "hello_world_foo_bar" | sed ':loop; s/_/-/; t loop'

echo ""
echo "2. การ join lines ที่ลงท้ายด้วย backslash:"
cat > /tmp/backslash.txt << 'EOF'
first line \
second part \
end
next line
EOF
sed ':a;N;s/\\\n/ /;ta' /tmp/backslash.txt

echo ""
echo "3. ตัวอย่าง: แปลง CSV เป็น TSV:"
echo "name,age,city" | sed 's/,/\t/g'

echo ""
echo "4. ตัวอย่าง: เอา leading whitespace:"
printf "  hello\n    world\n  bye\n" | sed 's/^[[:space:]]*//'

echo ""
echo "5. เอา trailing whitespace:"
printf "hello  \nworld    \nbye \n" | sed 's/[[:space:]]*$//'

echo ""
echo "6. Trim both ends:"
printf "  hello world  \n" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//'
```

---

## ขั้นตอนที่ 321: sed In-Place Editing

```bash
#!/usr/bin/env bash
# sed_inplace.sh - In-place editing

echo "=== sed In-Place Editing ==="

# สร้าง backup ก่อน
cat > /tmp/inplace_test.txt << 'EOF'
# Configuration
server_host = localhost
server_port = 8080
debug_mode = true
database_url = mysql://localhost/db
EOF

echo "ไฟล์เดิม:"
cat /tmp/inplace_test.txt
echo ""

# -i แก้ไข in-place
# -i.bak สร้าง backup ด้วย extension .bak
cp /tmp/inplace_test.txt /tmp/inplace_backup.txt

echo "1. -i - แก้ไข in-place:"
sed -i 's/localhost/production-server/g' /tmp/inplace_test.txt
echo "หลังแก้ไข:"
cat /tmp/inplace_test.txt
echo ""

echo "2. -i.bak - พร้อม backup:"
cp /tmp/inplace_backup.txt /tmp/inplace_test2.txt
sed -i.bak 's/8080/443/' /tmp/inplace_test2.txt
echo "ไฟล์แก้ไข:"
cat /tmp/inplace_test2.txt
echo "ไฟล์ backup:"
cat /tmp/inplace_test2.txt.bak
echo ""

echo "3. แก้ไขหลายไฟล์ในคราวเดียว:"
mkdir -p /tmp/sed_multi
echo "version=1.0.0" > /tmp/sed_multi/app1.conf
echo "version=1.0.0" > /tmp/sed_multi/app2.conf
echo "version=1.0.0" > /tmp/sed_multi/app3.conf

sed -i 's/1.0.0/2.0.0/' /tmp/sed_multi/*.conf
echo "หลังอัปเดต version:"
grep "version" /tmp/sed_multi/*.conf

echo ""
echo "4. Safe in-place (backup + restore on error):"
safe_sed_inplace() {
    local pattern="$1"
    local file="$2"
    local backup="${file}.bak.$$"
    
    cp "$file" "$backup" || return 1
    
    if sed "$pattern" "$backup" > "$file"; then
        rm -f "$backup"
        echo "✓ แก้ไขสำเร็จ"
    else
        mv "$backup" "$file"
        echo "✗ แก้ไขล้มเหลว ไฟล์คืนสภาพเดิม"
        return 1
    fi
}

echo "ทดสอบ safe_sed_inplace:"
echo "test content" > /tmp/safe_test.txt
safe_sed_inplace 's/test/TEST/' /tmp/safe_test.txt
cat /tmp/safe_test.txt
```

---

## ขั้นตอนที่ 322: sed สำหรับ Config Files

```bash
#!/usr/bin/env bash
# sed_config.sh - จัดการ Config Files

echo "=== sed สำหรับ Config Files ==="

cat > /tmp/app.conf << 'EOF'
# Application Configuration
[server]
host = localhost
port = 8080
ssl = false
workers = 4

[database]
host = localhost
port = 5432
name = myapp
pool_size = 10

[logging]
level = DEBUG
file = /var/log/app.log
max_size = 100MB
EOF

echo "1. อ่าน value:"
read_config() {
    local file="$1"
    local key="$2"
    sed -n "s/^[[:space:]]*${key}[[:space:]]*=[[:space:]]*/\1/p" "$file" 2>/dev/null || \
    sed -n "/^${key}[[:space:]]*=/{s/.*=[[:space:]]*//;p}" "$file"
}

echo "server.port:"
sed -n '/^port[[:space:]]*=/{s/.*=[[:space:]]*//;p}' /tmp/app.conf | head -1

echo ""
echo "2. อัปเดต value:"
update_config() {
    local file="$1"
    local key="$2"
    local value="$3"
    sed -i "s/^\\(${key}[[:space:]]*=\\).*/\\1 ${value}/" "$file"
}

echo "ก่อน:"
grep "^port" /tmp/app.conf

cp /tmp/app.conf /tmp/app_backup.conf
update_config /tmp/app.conf "port" "9090"
echo "หลัง:"
grep "^port" /tmp/app.conf

# คืนค่า
cp /tmp/app_backup.conf /tmp/app.conf

echo ""
echo "3. ลบ comment lines:"
sed '/^[[:space:]]*#/d' /tmp/app.conf

echo ""
echo "4. ลบ section ทั้งหมด:"
echo "ลบ [logging] section:"
sed '/\[logging\]/,/^\[/{ /^\[logging\]/d; /^\[/!d }' /tmp/app.conf

echo ""
echo "5. แทนที่ทั้ง section:"
echo "เพิ่ม max_connections:"
sed '/\[database\]/a\max_connections = 50' /tmp/app.conf
```

---

## ขั้นตอนที่ 323: sed สำหรับ Text Transformation

```bash
#!/usr/bin/env bash
# sed_transform.sh - Text Transformation

echo "=== Text Transformation ด้วย sed ==="

echo "1. Capitalize first letter of each word:"
echo "hello world foo bar" | sed 's/\b\([a-z]\)/\u\1/g' 2>/dev/null || \
echo "hello world foo bar" | sed -E 's/(^| )([a-z])/\1\U\2/g' 2>/dev/null || \
echo "hello world foo bar" | awk '{for(i=1;i<=NF;i++) $i=toupper(substr($i,1,1)) substr($i,2); print}'

echo ""
echo "2. UPPERCASE:"
echo "hello world" | sed 'y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/'

echo ""
echo "3. lowercase:"
echo "HELLO WORLD" | sed 'y/ABCDEFGHIJKLMNOPQRSTUVWXYZ/abcdefghijklmnopqrstuvwxyz/'

echo ""
echo "4. ROT13:"
echo "Hello World" | sed 'y/ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz/NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm/'

echo ""
echo "5. Double-space (เพิ่มบรรทัดว่างระหว่างบรรทัด):"
printf "line1\nline2\nline3\n" | sed 'G'

echo ""
echo "6. เลขบรรทัด:"
printf "apple\nbanana\ncherry\n" | sed '=' | sed 'N;s/\n/\t/'

echo ""
echo "7. Trim whitespace:"
printf "  hello  \n  world  \n" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//'

echo ""
echo "8. แปลง Windows line endings (CRLF) เป็น Unix (LF):"
printf "line1\r\nline2\r\nline3\r\n" | sed 's/\r//' | cat -A

echo ""
echo "9. แปลง Unix เป็น Windows:"
printf "line1\nline2\nline3\n" | sed 's/$/\r/' | cat -A

echo ""
echo "10. Word wrap ที่ 40 ตัวอักษร:"
echo "This is a very long sentence that needs to be wrapped at a specific column width for display purposes" | \
    fold -s -w 40
```

---

## ขั้นตอนที่ 324: sed สำหรับ Code Refactoring

```bash
#!/usr/bin/env bash
# sed_refactor.sh - Code Refactoring ด้วย sed

echo "=== Code Refactoring ด้วย sed ==="

# สร้าง sample code
cat > /tmp/old_code.py << 'EOF'
import sys
import os

def calculate_sum(x, y):
    # old function name
    result = x + y
    print("Result: " + str(result))
    return result

def calculate_product(x, y):
    # old style
    result = x * y
    print("Result: " + str(result))
    return result

def main():
    a = 10
    b = 20
    sum_result = calculate_sum(a, b)
    prod_result = calculate_product(a, b)
    
if __name__ == "__main__":
    main()
EOF

echo "Code เดิม:"
cat /tmp/old_code.py
echo ""

echo "1. เปลี่ยน function names:"
sed 's/calculate_sum/add_numbers/g; s/calculate_product/multiply_numbers/g' /tmp/old_code.py

echo ""
echo "2. อัปเดต print syntax (Python 2 → 3):"
sed 's/print "\(.*\)"/print(\1)/g' /tmp/old_code.py 2>/dev/null || \
sed 's/print "\(.*\)"/print("\1")/g' /tmp/old_code.py

echo ""
echo "3. เพิ่ม type hints:"
sed 's/def \([a-z_]*\)(x, y):/def \1(x: int, y: int) -> int:/' /tmp/old_code.py

echo ""
echo "4. เพิ่ม docstring:"
sed '/^def /a\    """Function documentation."""' /tmp/old_code.py

echo ""
echo "5. ลบ comment lines:"
sed '/^[[:space:]]*#/d' /tmp/old_code.py
```

---

## ขั้นตอนที่ 325: sed Script Files

```bash
#!/usr/bin/env bash
# sed_scripts.sh - sed Script Files

echo "=== sed Script Files ==="
echo ""
echo "สำหรับ transformations ที่ซับซ้อน เราสามารถใส่ sed commands ในไฟล์"
echo "แล้วใช้ sed -f script_file input_file"
echo ""

# สร้าง sed script สำหรับ HTML escaping
cat > /tmp/html_escape.sed << 'EOF'
# HTML Escape Script
s/&/\&amp;/g
s/</\&lt;/g
s/>/\&gt;/g
s/"/\&quot;/g
s/'/\&#39;/g
EOF

echo "1. HTML escaping:"
echo '<h1 class="title">Hello & "World"</h1>' | sed -f /tmp/html_escape.sed

# สร้าง sed script สำหรับ clean log
cat > /tmp/clean_log.sed << 'EOF'
# Remove empty lines
/^$/d

# Remove comment lines
/^#/d

# Remove leading whitespace
s/^[[:space:]]*//

# Remove trailing whitespace
s/[[:space:]]*$//

# Normalize multiple spaces to one
s/[[:space:]]\+/ /g
EOF

echo ""
echo "2. Clean log file:"
cat > /tmp/messy_log.txt << 'EOF'
# Log file

  2024-01-15  INFO   Server started
  
  2024-01-15  DEBUG  Loading modules
# Debug info
  2024-01-15  ERROR  Connection failed
EOF

sed -f /tmp/clean_log.sed /tmp/messy_log.txt

# สร้าง sed script สำหรับ markdown to HTML
cat > /tmp/md_to_html.sed << 'EOF'
# Simple Markdown to HTML converter

# Headers
s/^### \(.*\)/<h3>\1<\/h3>/
s/^## \(.*\)/<h2>\1<\/h2>/
s/^# \(.*\)/<h1>\1<\/h1>/

# Bold
s/\*\*\([^*]*\)\*\*/<strong>\1<\/strong>/g

# Italic
s/\*\([^*]*\)\*/<em>\1<\/em>/g

# Code inline
s/`\([^`]*\)`/<code>\1<\/code>/g

# Horizontal rule
s/^---$/<hr>/

# Empty line → paragraph break
s/^$/<p>/
EOF

echo ""
echo "3. Markdown to HTML:"
cat > /tmp/sample.md << 'EOF'
# Hello World

This is **bold** and *italic* text.

## Section 2

Use `code` for inline code.

---

End of document.
EOF

sed -f /tmp/md_to_html.sed /tmp/sample.md
```

---

## ขั้นตอนที่ 326: sed Advanced Examples

```bash
#!/usr/bin/env bash
# sed_advanced.sh - ตัวอย่างขั้นสูง

echo "=== sed Advanced Examples ==="

echo "1. Extract ข้อมูลจาก XML/HTML:"
cat > /tmp/data.xml << 'EOF'
<config>
  <server>
    <host>production.example.com</host>
    <port>443</port>
    <ssl>true</ssl>
  </server>
  <database>
    <host>db.example.com</host>
    <port>5432</port>
  </database>
</config>
EOF

echo "Extract host values:"
sed -n 's/.*<host>\(.*\)<\/host>.*/\1/p' /tmp/data.xml

echo "Extract port values:"
sed -n 's/.*<port>\(.*\)<\/port>.*/\1/p' /tmp/data.xml

echo ""
echo "2. Parse CSV:"
cat > /tmp/data.csv << 'EOF'
name,age,city,email
John,30,Bangkok,john@example.com
Jane,25,Chiang Mai,jane@example.com
Bob,35,Phuket,bob@example.com
EOF

echo "แสดงเฉพาะ name และ email (column 1 และ 4):"
sed '1d' /tmp/data.csv | sed 's/\([^,]*\),[^,]*,[^,]*,\([^,]*\)/\1: \2/'

echo ""
echo "3. Multi-line block deletion:"
cat > /tmp/code_with_debug.sh << 'EOF'
#!/usr/bin/env bash
echo "start"

# DEBUG START
echo "debug: x=$x"
echo "debug: y=$y"
# DEBUG END

echo "middle"

# DEBUG START
echo "debug info"
# DEBUG END

echo "end"
EOF

echo "ลบ debug blocks:"
sed '/# DEBUG START/,/# DEBUG END/d' /tmp/code_with_debug.sh

echo ""
echo "4. JSON pretty formatting (basic):"
echo '{"name":"John","age":30,"city":"Bangkok"}' | \
    sed 's/{/{\n/;s/}/\n}/;s/,/,\n/g' | \
    sed 's/"/"/g'
    
echo ""
echo "5. Escape shell special characters:"
escape_shell() {
    echo "$1" | sed 's/[()&|;<>{}!$]/\\&/g'
}

echo "Original: hello (world) && foo | bar"
echo "Escaped:  $(escape_shell 'hello (world) && foo | bar')"
```

---

## ขั้นตอนที่ 327: sed Performance

```bash
#!/usr/bin/env bash
# sed_performance.sh - Performance Tips

echo "=== sed Performance Tips ==="
echo ""

echo "1. ใช้ address ranges ให้ถูกต้อง:"
echo "   ถ้าต้องการแก้ไขเฉพาะส่วน ระบุ address range"
echo "   sed '/START/,/END/s/old/new/' vs sed 's/old/new/'"
echo ""

echo "2. เรียง operations จาก common ไป rare:"
echo "   operations ที่ match บ่อยควรอยู่ก่อน"
echo ""

echo "3. ใช้ -n และ p แทน filter:"
echo "   เร็วกว่า: sed -n '/pattern/p'"
echo "   ช้ากว่า:  sed '/pattern/!d'"
echo ""

echo "4. สร้าง large test file:"
python3 -c "
for i in range(50000):
    print(f'Line {i}: key=value_{i} status=active timestamp=2024-01-{(i%30)+1:02d}')
" > /tmp/large_sed.txt 2>/dev/null
echo "ไฟล์ทดสอบ: $(wc -l < /tmp/large_sed.txt) บรรทัด"

echo ""
echo "5. Benchmark:"
if [[ -f /tmp/large_sed.txt ]]; then
    echo -n "sed substitution: "
    time sed 's/active/ACTIVE/' /tmp/large_sed.txt > /dev/null
    
    echo -n "sed with address: "
    time sed '1,10000s/active/ACTIVE/' /tmp/large_sed.txt > /dev/null
    
    echo -n "grep + sed: "
    time grep "active" /tmp/large_sed.txt | sed 's/active/ACTIVE/' > /dev/null
fi

echo ""
echo "6. GNU sed vs BSD sed:"
echo "   GNU sed (Linux): -i ไม่ต้องมี extension"
echo "   BSD sed (macOS): -i '' (ต้องมี empty string)"
echo ""
echo "   Portable syntax:"
echo "   sed -i.bak 's/old/new/' file"
echo "   rm file.bak"
```

---

## ขั้นตอนที่ 328: Workshop - sed Text Processor

```bash
#!/usr/bin/env bash
# text_processor_sed.sh - Workshop: sed Text Processor

set -euo pipefail

# ==================== sed-based Text Processing Library ====================

# HTML encode
html_encode() {
    sed 's/&/\&amp;/g; s/</\&lt;/g; s/>/\&gt;/g; s/"/\&quot;/g'
}

# HTML decode
html_decode() {
    sed 's/\&amp;/\&/g; s/\&lt;/</g; s/\&gt;/>/g; s/\&quot;/"/g; s/\&#39;/'"'"'/g'
}

# Trim whitespace
trim() {
    sed 's/^[[:space:]]*//; s/[[:space:]]*$//'
}

# Squeeze multiple spaces
squeeze_spaces() {
    sed 's/[[:space:]]\+/ /g'
}

# Remove blank lines
remove_blank_lines() {
    sed '/^[[:space:]]*$/d'
}

# Remove comments
remove_comments() {
    local comment_char="${1:-#}"
    sed "/^[[:space:]]*${comment_char}/d"
}

# Add line numbers
add_line_numbers() {
    sed '=' | sed 'N;s/\n/\t/'
}

# Wrap text at N chars
wrap_text() {
    local width="${1:-80}"
    fold -s -w "$width"
}

# Convert tabs to spaces
tabs_to_spaces() {
    local spaces="${1:-4}"
    local replacement
    replacement=$(printf "%${spaces}s" "")
    sed "s/\t/${replacement}/g"
}

# Slugify (URL-friendly)
slugify() {
    sed 's/[^a-zA-Z0-9]/-/g; s/-\+/-/g; s/^-\|-$//g' | \
    tr '[:upper:]' '[:lower:]'
}

# CSV to TSV
csv_to_tsv() {
    sed 's/,/\t/g'
}

# Strip HTML tags
strip_html() {
    sed 's/<[^>]*>//g'
}

# Extract between delimiters
extract_between() {
    local start="$1"
    local end="$2"
    sed -n "/${start}/,/${end}/p"
}

# Comment out lines matching pattern
comment_out() {
    local pattern="$1"
    local comment="${2:-#}"
    sed "s/^\(${pattern}.*\)/${comment} \1/"
}

# Uncomment lines
uncomment() {
    local comment="${1:-#}"
    sed "s/^[[:space:]]*${comment}[[:space:]]*//"
}

# ==================== Demo ====================
echo "=== sed Text Processor Workshop ==="
echo ""

echo "1. HTML encoding:"
echo '<h1 class="title">Hello & "World"</h1>' | html_encode

echo ""
echo "2. HTML decoding:"
echo '&lt;h1&gt;Hello &amp; World&lt;/h1&gt;' | html_decode

echo ""
echo "3. Trim + squeeze:"
echo "   hello   world   " | trim | squeeze_spaces

echo ""
echo "4. Remove blank lines:"
printf "line1\n\nline2\n\n\nline3\n" | remove_blank_lines

echo ""
echo "5. Slugify:"
echo "Hello World! This is a Title" | slugify

echo ""
echo "6. Strip HTML:"
echo "<p>This is <b>bold</b> and <i>italic</i> text.</p>" | strip_html

echo ""
echo "7. Extract block:"
cat > /tmp/extract_test.txt << 'EOF'
Before block
START
This is the content
inside the block
END
After block
EOF
extract_between "START" "END" < /tmp/extract_test.txt

echo ""
echo "8. Comment out matching lines:"
echo "server_port=8080" | comment_out "server"

echo ""
echo "9. แปลง config file:"
cat > /tmp/old_config.txt << 'EOF'
# Old format
HOST=localhost
PORT=8080
DEBUG=true
SECRET=old_secret_key
EOF

# แปลงเป็น format ใหม่
cat /tmp/old_config.txt | \
    remove_comments | \
    sed 's/HOST=/SERVER_HOST=/' | \
    sed 's/PORT=/SERVER_PORT=/' | \
    sed 's/SECRET=.*/SECRET=REDACTED/' | \
    remove_blank_lines

echo ""
echo "10. Pipeline demo:"
cat << 'EOF' | trim | squeeze_spaces | remove_blank_lines
  Line 1 with   extra   spaces
  
  Line 2 normal
    Line 3 indented  
  
EOF
```

---

## ขั้นตอนที่ 329: sed One-liners ที่มีประโยชน์

```bash
#!/usr/bin/env bash
# sed_oneliners.sh - One-liners

echo "=== sed One-liners ที่มีประโยชน์ ==="
echo ""

cat > /tmp/sample_file.txt << 'EOF'
  First line with leading spaces
Normal line
  Another indented line
Line with trailing spaces   
UPPERCASE LINE
lowercase line
Mixed Case Line

Empty line above
Last line
EOF

echo "File operations:"
echo "---"

echo "1. พิมพ์บรรทัดที่ n:"
sed -n '3p' /tmp/sample_file.txt

echo ""
echo "2. พิมพ์บรรทัดสุดท้าย:"
sed -n '$p' /tmp/sample_file.txt

echo ""
echo "3. ลบบรรทัดแรก (head -n +2):"
sed '1d' /tmp/sample_file.txt | head -5

echo ""
echo "4. ลบบรรทัดสุดท้าย:"
sed '$d' /tmp/sample_file.txt | tail -5

echo ""
echo "5. ลบบรรทัดที่ n ถึง m:"
sed '3,5d' /tmp/sample_file.txt | head -7

echo ""
echo "6. พิมพ์บรรทัดที่ match:"
sed -n '/[A-Z]/p' /tmp/sample_file.txt

echo ""
echo "7. ลบ leading whitespace:"
sed 's/^[[:space:]]*//' /tmp/sample_file.txt | head -5

echo ""
echo "8. ลบ trailing whitespace:"
sed 's/[[:space:]]*$//' /tmp/sample_file.txt | cat -A | head -5

echo ""
echo "9. ลบบรรทัดว่าง:"
sed '/^[[:space:]]*$/d' /tmp/sample_file.txt

echo ""
echo "10. เลขบรรทัด:"
sed '=' /tmp/sample_file.txt | sed 'N;s/\n/ /' | head -5

echo ""
echo "11. ย้อนกลับ order (tac):"
sed -n '1!G;h;$p' /tmp/sample_file.txt

echo ""
echo "Substitution one-liners:"
echo "---"

echo "12. เปลี่ยนทุก occurrence:"
echo "foo foo foo" | sed 's/foo/bar/g'

echo ""
echo "13. เปลี่ยนเฉพาะบรรทัดที่ match pattern อื่น:"
echo -e "keep this\nchange this: old value\nkeep this too" | \
    sed '/change/s/old/new/'

echo ""
echo "14. Double-space file:"
printf "a\nb\nc\n" | sed 'G'

echo ""
echo "15. เพิ่ม prefix/suffix:"
echo -e "apple\nbanana\ncherry" | sed 's/.*/fruit: &/'

echo ""
echo "16. Extract ระหว่าง delimiters:"
echo "start[EXTRACT THIS]end" | sed 's/.*\[\(.*\)\].*/\1/'

echo ""
echo "17. Swap words:"
echo "hello world" | sed 's/\(\w*\) \(\w*\)/\2 \1/'
```

---

## ขั้นตอนที่ 330: สรุป Part 12 - sed

```bash
#!/usr/bin/env bash
# summary_sed.sh

echo "=== สรุป sed Stream Editor ==="
echo ""
echo "Commands หลัก:"
echo "  s/pattern/replacement/flags  - substitute"
echo "  d                             - delete"
echo "  p                             - print"
echo "  i\\text                        - insert before"
echo "  a\\text                        - append after"
echo "  c\\text                        - change line"
echo "  q/Q                           - quit"
echo "  =                             - print line number"
echo "  y/chars1/chars2/              - transliterate"
echo ""
echo "Addresses:"
echo "  n          - บรรทัดที่ n"
echo "  n,m        - range n ถึง m"
echo "  n,\$        - n ถึงสุดท้าย"
echo "  /pattern/  - บรรทัดที่ match"
echo "  first~step - every step บรรทัด"
echo "  addr!      - negation"
echo ""
echo "Hold Space:"
echo "  h/H  - copy/append pattern → hold"
echo "  g/G  - copy/append hold → pattern"
echo "  x    - exchange"
echo ""
echo "Flags ของ sed:"
echo "  -n  - suppress auto-print"
echo "  -e  - expression"
echo "  -f  - script file"
echo "  -i  - in-place edit"
echo "  -E  - extended regex"
echo ""
echo "Substitution flags:"
echo "  g  - global"
echo "  I  - case insensitive"
echo "  n  - nth occurrence"
echo "  p  - print if substituted"
echo "  w  - write to file"
echo ""
echo "Next: Part 13 - awk Text Processing"
```

---

## แบบฝึกหัด Part 12

### แบบฝึกหัดที่ 1: Config File Manager
สร้างสคริปต์ที่ใช้ sed เพื่อ:
- อ่าน key=value จาก config file
- อัปเดต value ของ key ที่ระบุ
- เพิ่ม key ใหม่ถ้ายังไม่มี
- ลบ key ที่ระบุ
- Comment/uncomment settings

### แบบฝึกหัดที่ 2: Log Sanitizer
เขียน sed script ที่:
- ลบข้อมูล sensitive (passwords, tokens, credit cards)
- แทนที่ IP addresses ด้วย REDACTED
- แทนที่ email ด้วย user@redacted.com
- ลบ debug messages

### แบบฝึกหัดที่ 3: Code Formatter
ใช้ sed เพื่อ:
- Normalize indentation (tabs → spaces)
- Remove trailing whitespace
- Ensure file ends with newline
- Remove Windows line endings (CRLF → LF)

### แบบฝึกหัดที่ 4: Template Engine
สร้าง simple template engine ด้วย sed:
- Template มี placeholders เช่น `{{name}}`, `{{version}}`
- อ่านค่าจาก environment variables
- แทนที่ placeholders ด้วยค่าจริง

---

## สรุป

Part 12 ครอบคลุม sed ทั้งหมด:

| หัวข้อ | Steps |
|--------|-------|
| sed introduction | 313 |
| Substitution command | 314 |
| Address ranges | 315 |
| Delete, Print, Quit | 316 |
| Insert, Append, Change | 317 |
| Multiline operations | 318 |
| Hold space | 319 |
| Branching & labels | 320 |
| In-place editing | 321 |
| Config files | 322 |
| Text transformation | 323 |
| Code refactoring | 324 |
| Script files | 325 |
| Advanced examples | 326 |
| Performance | 327 |
| Workshop | 328 |
| One-liners | 329 |

**ขั้นตอนต่อไป**: Part 13 - awk: Text Processing Language
