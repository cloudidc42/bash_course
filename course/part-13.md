# Part 13: awk - Text Processing Language

## Module 2: Intermediate Level (ระดับกลาง)

---

## ขั้นตอนที่ 331: awk คืออะไร?

**awk** คือภาษา scripting สำหรับประมวลผลข้อความที่มีโครงสร้าง โดยเฉพาะข้อมูลแบบ columnar

```bash
#!/usr/bin/env bash
# awk_intro.sh - แนะนำ awk

echo "=== awk เบื้องต้น ==="
echo ""
echo "awk syntax:"
echo "  awk 'pattern { action }' file"
echo "  awk -F delimiter 'pattern { action }' file"
echo "  awk -f script.awk file"
echo ""

# สร้างข้อมูลทดสอบ
cat > /tmp/employees.txt << 'EOF'
John    Manager    85000   Marketing
Jane    Engineer   75000   Engineering
Bob     Designer   65000   Design
Alice   Engineer   80000   Engineering
Charlie Manager    90000   HR
Diana   Designer   70000   Design
EOF

echo "ข้อมูล employees.txt:"
cat /tmp/employees.txt
echo ""

echo "1. พิมพ์ทุกบรรทัด:"
awk '{print}' /tmp/employees.txt

echo ""
echo "2. พิมพ์ column ที่ระบุ:"
echo "ชื่อและเงินเดือน:"
awk '{print $1, $3}' /tmp/employees.txt

echo ""
echo "3. Pattern matching:"
echo "เฉพาะ Engineers:"
awk '/Engineer/ {print}' /tmp/employees.txt

echo ""
echo "4. Field comparison:"
echo "เงินเดือนมากกว่า 75000:"
awk '$3 > 75000 {print $1, $3}' /tmp/employees.txt
```

---

## ขั้นตอนที่ 332: awk Built-in Variables

```bash
#!/usr/bin/env bash
# awk_variables.sh - Built-in Variables

echo "=== awk Built-in Variables ==="

cat > /tmp/data.txt << 'EOF'
apple,10,2.50
banana,5,1.20
cherry,20,3.00
grape,15,4.50
mango,8,2.80
EOF

echo "Variables สำคัญ:"
echo ""

echo "1. NR - Number of Records (เลขบรรทัด):"
awk -F, '{print NR, $1}' /tmp/data.txt

echo ""
echo "2. NF - Number of Fields (จำนวน columns):"
awk -F, '{print NF, $0}' /tmp/data.txt | head -3

echo ""
echo "3. \$0 - ทั้งบรรทัด:"
awk -F, 'NR==1 {print "First line:", $0}' /tmp/data.txt

echo ""
echo "4. \$1, \$2 - fields:"
awk -F, '{print "Item:", $1, "Qty:", $2, "Price:", $3}' /tmp/data.txt

echo ""
echo "5. FS - Field Separator:"
awk 'BEGIN {FS=","} {print $1}' /tmp/data.txt

echo ""
echo "6. OFS - Output Field Separator:"
awk -F, 'BEGIN {OFS="|"} {print $1, $2, $3}' /tmp/data.txt

echo ""
echo "7. RS - Record Separator (default \n):"
echo "ใช้ RS=',' เพื่อแยก token:"
echo "one,two,three,four" | awk 'BEGIN{RS=","} {print NR, $0}'

echo ""
echo "8. ORS - Output Record Separator:"
awk -F, 'BEGIN{ORS="|"} {print $1}' /tmp/data.txt
echo ""

echo ""
echo "9. FILENAME - ชื่อไฟล์:"
awk '{print FILENAME, NR, $0}' /tmp/data.txt | head -2

echo ""
echo "10. FNR - File Number of Records (เลขบรรทัดภายในไฟล์):"
awk '{print FNR, NR, $0}' /tmp/data.txt /tmp/data.txt | head -8
```

---

## ขั้นตอนที่ 333: BEGIN และ END Blocks

```bash
#!/usr/bin/env bash
# awk_begin_end.sh - BEGIN และ END

echo "=== BEGIN และ END ==="

cat > /tmp/sales.txt << 'EOF'
Jan  Electronics  15000
Feb  Electronics  18000
Mar  Clothing     12000
Apr  Electronics  20000
May  Clothing     9000
Jun  Food         8000
EOF

echo "1. BEGIN - ทำงานก่อนอ่าน input:"
awk 'BEGIN {
    print "=== Sales Report ==="
    print "Month  Category    Amount"
    print "-----  ----------  ------"
} {print}' /tmp/sales.txt

echo ""
echo "2. END - ทำงานหลัง input ทั้งหมด:"
awk 'BEGIN {total=0}
     {total += $3}
     END {print "Total:", total}' /tmp/sales.txt

echo ""
echo "3. BEGIN + main + END:"
awk 'BEGIN {
    total = 0
    count = 0
    print "Processing..."
}
{
    total += $3
    count++
}
END {
    avg = total / count
    printf "Records: %d\nTotal: %d\nAverage: %.2f\n", count, total, avg
}' /tmp/sales.txt

echo ""
echo "4. ใช้ BEGIN สำหรับ configuration:"
awk 'BEGIN {
    FS = " "
    OFS = " | "
    threshold = 10000
}
$3 > threshold {
    print $1, $2, $3
}' /tmp/sales.txt
```

---

## ขั้นตอนที่ 334: awk Arithmetic และ Math

```bash
#!/usr/bin/env bash
# awk_math.sh - Arithmetic ใน awk

echo "=== awk Arithmetic ==="

cat > /tmp/numbers.txt << 'EOF'
10 20 30
5  15 25
8  12 16
EOF

echo "1. Basic arithmetic:"
awk '{print $1 + $2 + $3}' /tmp/numbers.txt
awk '{print $1 * $2}' /tmp/numbers.txt
awk '{print $3 / $1}' /tmp/numbers.txt
awk '{print $3 % $1}' /tmp/numbers.txt

echo ""
echo "2. Math functions:"
awk 'BEGIN {
    print "sqrt(16) =", sqrt(16)
    print "sin(0) =", sin(0)
    print "cos(0) =", cos(0)
    print "exp(1) =", exp(1)
    print "log(1) =", log(1)
    print "int(3.7) =", int(3.7)
    print "atan2(1,1) =", atan2(1,1) * 180 / 3.14159
}'

echo ""
echo "3. Increment/Decrement:"
awk 'BEGIN {
    x = 5
    print x++  # post-increment
    print x    # now 6
    print ++x  # pre-increment
    print x--  # post-decrement
    print --x  # back to 6
}'

echo ""
echo "4. Running calculations:"
cat > /tmp/inventory.txt << 'EOF'
Item     Price  Qty  Discount
Apple    2.50   100  0.10
Banana   1.20   200  0.05
Cherry   3.00   50   0.15
Grape    4.50   75   0.20
EOF

awk 'NR>1 {
    subtotal = $2 * $3
    discount = subtotal * $4
    total = subtotal - discount
    printf "%-10s: %8.2f - %6.2f = %8.2f\n", $1, subtotal, discount, total
    grand_total += total
}
END {
    printf "\n%-10s: %25.2f\n", "Grand Total", grand_total
}' /tmp/inventory.txt
```

---

## ขั้นตอนที่ 335: awk String Functions

```bash
#!/usr/bin/env bash
# awk_strings.sh - String Functions

echo "=== awk String Functions ==="

cat > /tmp/str_test.txt << 'EOF'
Hello World
foo bar baz
2024-01-15
user@example.com
EOF

echo "1. length() - ความยาว:"
awk '{print length($0), $0}' /tmp/str_test.txt

echo ""
echo "2. substr() - substring:"
echo "Hello World" | awk '{
    print substr($0, 1, 5)    # Hello
    print substr($0, 7)       # World
    print substr($0, 7, 3)    # Wor
}'

echo ""
echo "3. index() - หา position:"
echo "Hello World" | awk '{
    pos = index($0, "World")
    print "World starts at:", pos
}'

echo ""
echo "4. split() - แบ่งเป็น array:"
echo "one:two:three:four" | awk '{
    n = split($0, parts, ":")
    for (i=1; i<=n; i++) print i, parts[i]
}'

echo ""
echo "5. sub() และ gsub() - แทนที่:"
echo "foo foo foo" | awk '{
    # sub แทนที่ครั้งแรก
    s = $0
    sub(/foo/, "bar", s)
    print "sub:", s
    
    # gsub แทนที่ทั้งหมด
    s = $0
    gsub(/foo/, "bar", s)
    print "gsub:", s
}'

echo ""
echo "6. match() - ค้นหา pattern:"
echo "The quick brown fox" | awk '{
    if (match($0, /[a-z]+ow[a-z]+/)) {
        print "Match:", substr($0, RSTART, RLENGTH)
        print "Start:", RSTART
        print "Length:", RLENGTH
    }
}'

echo ""
echo "7. sprintf() - format string:"
awk 'BEGIN {
    name = "John"
    age = 30
    salary = 75000.50
    result = sprintf("Name: %-10s Age: %3d Salary: %10.2f", name, age, salary)
    print result
}'

echo ""
echo "8. toupper() / tolower():"
echo "Hello World" | awk '{print toupper($0), "-", tolower($0)}'

echo ""
echo "9. printf - formatted output:"
cat > /tmp/scores.txt << 'EOF'
Alice 95 88 92
Bob   78 82 80
Charlie 91 87 94
EOF

awk '{
    avg = ($2 + $3 + $4) / 3
    printf "%-10s %5.1f\n", $1, avg
}' /tmp/scores.txt
```

---

## ขั้นตอนที่ 336: awk Arrays

```bash
#!/usr/bin/env bash
# awk_arrays.sh - Arrays ใน awk

echo "=== awk Arrays ==="

cat > /tmp/transactions.txt << 'EOF'
Jan  Electronics  1500
Feb  Clothing     800
Jan  Food         600
Mar  Electronics  2000
Feb  Electronics  1200
Mar  Clothing     900
Jan  Electronics  1800
Feb  Food         400
EOF

echo "1. Basic array:"
awk '{a[$1] += $3}
     END {for (k in a) print k, a[k]}' /tmp/transactions.txt

echo ""
echo "2. 2D array (month + category):"
awk '{key = $1 "-" $2; total[key] += $3}
     END {
         for (k in total) print k, total[k]
     }' /tmp/transactions.txt | sort

echo ""
echo "3. Count occurrences:"
echo "count by category:"
awk '{count[$2]++}
     END {for (k in count) print k, count[k]}' /tmp/transactions.txt

echo ""
echo "4. Check if key exists:"
awk '{total[$2] += $3}
     END {
         if ("Electronics" in total)
             print "Electronics total:", total["Electronics"]
         if ("Luxury" in total)
             print "Luxury:", total["Luxury"]
         else
             print "Luxury: ไม่มีข้อมูล"
     }' /tmp/transactions.txt

echo ""
echo "5. Delete array element:"
awk '{total[$2] += $3}
     END {
         delete total["Food"]
         for (k in total) print k, total[k]
     }' /tmp/transactions.txt

echo ""
echo "6. Array ของ arrays (simulate 2D):"
awk '{data[$1][$2] += $3}
     END {
         for (month in data)
             for (cat in data[month])
                 printf "%s %s %d\n", month, cat, data[month][cat]
     }' /tmp/transactions.txt 2>/dev/null || \
awk '{key = $1 SUBSEP $2; data[key] += $3}
     END {
         for (k in data) {
             split(k, parts, SUBSEP)
             printf "%s %s %d\n", parts[1], parts[2], data[k]
         }
     }' /tmp/transactions.txt
```

---

## ขั้นตอนที่ 337: awk Control Flow

```bash
#!/usr/bin/env bash
# awk_control.sh - Control Flow

echo "=== awk Control Flow ==="

cat > /tmp/grades.txt << 'EOF'
Alice 95
Bob   72
Charlie 88
Diana 55
Eve   99
Frank 61
Grace 78
EOF

echo "1. if-else:"
awk '{
    if ($2 >= 90)      grade = "A"
    else if ($2 >= 80) grade = "B"
    else if ($2 >= 70) grade = "C"
    else if ($2 >= 60) grade = "D"
    else               grade = "F"
    
    printf "%-10s %3d %s\n", $1, $2, grade
}' /tmp/grades.txt

echo ""
echo "2. for loop:"
awk 'BEGIN {
    for (i = 1; i <= 5; i++) {
        printf "%d ", i*i
    }
    print ""
}'

echo ""
echo "3. while loop:"
awk 'BEGIN {
    n = 1
    while (n <= 10) {
        printf "%d ", n
        n *= 2
    }
    print ""
}'

echo ""
echo "4. do-while loop:"
awk 'BEGIN {
    n = 1
    do {
        printf "%d ", n
        n++
    } while (n <= 5)
    print ""
}'

echo ""
echo "5. break และ continue:"
awk 'BEGIN {
    for (i = 1; i <= 10; i++) {
        if (i == 6) break
        if (i % 2 == 0) continue
        printf "%d ", i
    }
    print ""
}'

echo ""
echo "6. next - ข้ามไปบรรทัดถัดไป:"
awk '/^#/ {next} {print}' << 'EOF'
# Comment 1
line 1
# Comment 2
line 2
line 3
EOF

echo ""
echo "7. exit - หยุดประมวลผล:"
awk '{print; if (NR == 3) exit}' /tmp/grades.txt
```

---

## ขั้นตอนที่ 338: awk Functions

```bash
#!/usr/bin/env bash
# awk_functions.sh - User-defined Functions

echo "=== awk Functions ==="

echo "1. Basic function:"
awk 'function square(n) { return n * n }
     BEGIN {
         for (i = 1; i <= 5; i++)
             print i, square(i)
     }'

echo ""
echo "2. Recursive function - Fibonacci:"
awk 'function fib(n) {
    if (n <= 1) return n
    return fib(n-1) + fib(n-2)
}
BEGIN {
    for (i = 0; i <= 10; i++)
        printf "%d ", fib(i)
    print ""
}'

echo ""
echo "3. Function ด้วย array parameter:"
awk 'function max_val(arr, n,    m, i) {
    # Local vars หลัง comma: m, i
    m = arr[1]
    for (i = 2; i <= n; i++)
        if (arr[i] > m) m = arr[i]
    return m
}
BEGIN {
    a[1] = 5; a[2] = 3; a[3] = 8; a[4] = 1; a[5] = 7
    print "Max:", max_val(a, 5)
}'

echo ""
echo "4. String utilities:"
awk 'function ltrim(s) { gsub(/^[[:space:]]+/, "", s); return s }
     function rtrim(s) { gsub(/[[:space:]]+$/, "", s); return s }
     function trim(s)  { return ltrim(rtrim(s)) }
     function repeat(s, n,    r, i) {
         r = ""
         for (i = 0; i < n; i++) r = r s
         return r
     }
     BEGIN {
         print trim("  hello  ")
         print repeat("ab", 4)
         print repeat("-", 20)
     }'

echo ""
echo "5. Practical example - format table:"
cat > /tmp/table_data.txt << 'EOF'
Name Age Department Salary
John 30 Engineering 75000
Jane 25 Marketing 65000
Bob 35 Engineering 85000
EOF

awk 'function pad_right(s, n,    r, i) {
    r = s
    for (i = length(s); i < n; i++) r = r " "
    return r
}
NR==1 {
    print pad_right($1,10) pad_right($2,5) pad_right($3,15) pad_right($4,10)
    print sprintf("%s", sprintf("%-s", sprintf("%-40s", "")) gsub(/ /, "-"))
}
NR>1 {
    printf "%-10s %-5s %-15s %-10s\n", $1, $2, $3, $4
}' /tmp/table_data.txt
```

---

## ขั้นตอนที่ 339: awk สำหรับ CSV Processing

```bash
#!/usr/bin/env bash
# awk_csv.sh - CSV Processing

echo "=== awk CSV Processing ==="

# CSV ง่ายๆ (ไม่มี quoted fields)
cat > /tmp/simple.csv << 'EOF'
name,age,city,salary
John,30,Bangkok,75000
Jane,25,Chiang Mai,65000
Bob,35,Phuket,85000
Alice,28,Bangkok,70000
Charlie,32,Bangkok,80000
EOF

echo "1. แสดง columns ที่เลือก:"
awk -F, 'NR>1 {print $1, $4}' /tmp/simple.csv

echo ""
echo "2. Filter ตาม condition:"
echo "พนักงานในกรุงเทพฯ:"
awk -F, 'NR>1 && $3=="Bangkok" {print $1}' /tmp/simple.csv

echo ""
echo "3. คำนวณ aggregate:"
awk -F, 'NR>1 {sum+=$4; count++}
         END {printf "Average salary: %.2f\n", sum/count}' /tmp/simple.csv

echo ""
echo "4. Group by:"
echo "จำนวนพนักงานต่อเมือง:"
awk -F, 'NR>1 {count[$3]++}
         END {for (city in count) print city, count[city]}' /tmp/simple.csv | sort

echo ""
echo "5. เพิ่ม computed column:"
awk -F, 'BEGIN {OFS=","; print "name,salary,bonus,total"}
         NR>1 {bonus=$4*0.1; print $1,$4,bonus,$4+bonus}' /tmp/simple.csv

echo ""
echo "6. Sort by column (ด้วย sort):"
echo "เรียงตาม salary:"
awk -F, 'NR>1 {print $4","$1}' /tmp/simple.csv | sort -n | awk -F, '{print $2, $1}'

echo ""
echo "7. Top N:"
echo "Top 3 เงินเดือน:"
awk -F, 'NR>1 {print $4, $1}' /tmp/simple.csv | sort -rn | head -3

echo ""
echo "8. Join สอง files:"
cat > /tmp/departments.csv << 'EOF'
name,dept
John,Engineering
Jane,Marketing
Bob,Engineering
Alice,HR
Charlie,Engineering
EOF

awk -F, 'FNR==NR {dept[$1]=$2; next}
         FNR>1 {print $1, $2, $3, $4, dept[$1]}' \
    /tmp/departments.csv /tmp/simple.csv
```

---

## ขั้นตอนที่ 340: awk สำหรับ Log Analysis

```bash
#!/usr/bin/env bash
# awk_log.sh - Log Analysis

echo "=== awk Log Analysis ==="

cat > /tmp/web_access.log << 'EOF'
192.168.1.1 - - [15/Jan/2024:10:00:01 +0700] "GET /index.html HTTP/1.1" 200 1234 0.015
192.168.1.2 - - [15/Jan/2024:10:00:02 +0700] "POST /api/login HTTP/1.1" 200 567 0.120
192.168.1.3 - - [15/Jan/2024:10:00:03 +0700] "GET /secret.php HTTP/1.1" 404 890 0.008
192.168.1.1 - - [15/Jan/2024:10:00:04 +0700] "GET /admin HTTP/1.1" 403 123 0.005
10.0.0.1 - - [15/Jan/2024:10:00:05 +0700] "GET /index.html HTTP/1.1" 200 1234 0.014
192.168.1.4 - - [15/Jan/2024:10:00:06 +0700] "DELETE /api/user/1 HTTP/1.1" 200 456 0.250
192.168.1.2 - - [15/Jan/2024:10:00:07 +0700] "GET /api/data HTTP/1.1" 500 789 2.100
192.168.1.5 - - [15/Jan/2024:10:00:08 +0700] "POST /api/login HTTP/1.1" 401 234 0.050
192.168.1.1 - - [15/Jan/2024:10:00:09 +0700] "GET /robots.txt HTTP/1.1" 200 100 0.003
192.168.1.1 - - [15/Jan/2024:10:00:10 +0700] "GET /page1 HTTP/1.1" 200 2000 0.025
EOF

echo "1. จำนวน requests ต่อ status code:"
awk '{print $9}' /tmp/web_access.log | sort | uniq -c | sort -rn

echo ""
echo "2. Top IP addresses:"
awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' \
    /tmp/web_access.log | sort -rn | head -5

echo ""
echo "3. Average response time:"
awk '{sum+=$NF; count++} END {printf "Average: %.3f sec\n", sum/count}' \
    /tmp/web_access.log

echo ""
echo "4. Slow requests (> 1 sec):"
awk '$NF > 1.0 {print $1, $7, $9, $NF}' /tmp/web_access.log

echo ""
echo "5. Error analysis (4xx, 5xx):"
awk '$9 >= 400 {errors[$9]++; total_errors++}
     END {
         print "Error Summary:"
         for (code in errors)
             printf "  %s: %d (%.1f%%)\n", code, errors[code], (errors[code]/total_errors)*100
         print "  Total:", total_errors
     }' /tmp/web_access.log

echo ""
echo "6. Request method distribution:"
awk '{match($7, /"([A-Z]+)/, method); methods[method[1]]++}
     END {for (m in methods) print m, methods[m]}' /tmp/web_access.log 2>/dev/null || \
awk '{gsub(/"/, "", $7); methods[$7]++}
     END {for (m in methods) print m, methods[m]}' /tmp/web_access.log

echo ""
echo "7. Bandwidth usage:"
awk '{bytes+=$10}
     END {printf "Total bytes: %d (%.2f MB)\n", bytes, bytes/1024/1024}' \
    /tmp/web_access.log
```

---

## ขั้นตอนที่ 341: awk สำหรับ System Administration

```bash
#!/usr/bin/env bash
# awk_sysadmin.sh - System Administration

echo "=== awk สำหรับ System Admin ==="

echo "1. /etc/passwd analysis:"
awk -F: 'BEGIN {
    print "Username        UID    Shell"
    print "--------        ---    -----"
}
$3 >= 1000 {
    printf "%-15s %-6s %s\n", $1, $3, $7
}' /etc/passwd | head -10

echo ""
echo "2. Process analysis:"
ps aux 2>/dev/null | awk 'NR>1 {
    cpu[$1] += $3
    mem[$1] += $4
    procs[$1]++
}
END {
    print "User         CPU%   MEM%  Procs"
    for (u in procs)
        if (procs[u] > 0)
            printf "%-12s %5.1f  %5.1f  %5d\n", u, cpu[u], mem[u], procs[u]
}' | sort -k2 -rn | head -10

echo ""
echo "3. Disk usage summary:"
df -h 2>/dev/null | awk 'NR>1 && $5!="Use%" {
    gsub(/%/, "", $5)
    if ($5+0 > 80)
        print "WARNING:", $6, "is", $5"%", "full"
    else if ($5+0 > 60)
        print "NOTICE:", $6, "is", $5"%", "full"
}'

echo ""
echo "4. สรุปขนาดไฟล์ใน directory:"
ls -la /tmp/*.txt 2>/dev/null | awk '
NF > 8 {
    total += $5
    count++
    if ($5 > max) { max = $5; maxfile = $NF }
}
END {
    printf "Files: %d\n", count
    printf "Total: %s bytes\n", total
    printf "Largest: %s (%d bytes)\n", maxfile, max
}'

echo ""
echo "5. Network statistics:"
if [[ -f /proc/net/dev ]]; then
    awk 'NR>2 {
        gsub(/:/, "", $1)
        if ($1 != "lo")
            printf "Interface: %-10s RX: %10s bytes  TX: %10s bytes\n", $1, $2, $10
    }' /proc/net/dev
fi

echo ""
echo "6. /etc/hosts parsing:"
awk '!/^#/ && NF>1 {
    for (i=2; i<=NF; i++)
        print $i, "->", $1
}' /etc/hosts 2>/dev/null | head -10
```

---

## ขั้นตอนที่ 342: awk Report Generation

```bash
#!/usr/bin/env bash
# awk_reports.sh - Report Generation

echo "=== awk Report Generation ==="

cat > /tmp/sales_data.txt << 'EOF'
2024-01-01 Electronics iPhone    1  59900
2024-01-01 Electronics MacBook   1 119900
2024-01-02 Clothing    TShirt    5   799
2024-01-02 Clothing    Jeans     2  1299
2024-01-03 Electronics iPad      2  25900
2024-01-03 Food        Coffee    10   150
2024-01-04 Electronics iPhone    2  59900
2024-01-04 Food        Lunch     5   200
2024-01-05 Clothing    Dress     3  2500
2024-01-05 Electronics Laptop    1  45000
EOF

echo "=== Sales Report ===" 
awk 'BEGIN {
    OFS = ""
    print "=" SPRINTF("%-50s", "") "="
    print "| " sprintf("%-48s", "SALES REPORT") " |"
    print "=" sprintf("%-50s", "=========================") "="
    printf "| %-10s %-15s %-12s %4s %8s %10s |\n", 
           "Date", "Category", "Product", "Qty", "Price", "Total"
    print "| " sprintf("%-48s", "-------------------------------------------") " |"
}

function separator() {
    print "+" sprintf("%-50s", "==================================================") "+"
}

{
    total = $4 * $5
    category_total[$2] += total
    grand_total += total
    printf "| %-10s %-15s %-12s %4d %8d %10d |\n",
           $1, $2, $3, $4, $5, total
}

END {
    print "+" sprintf("%-50s", "--------------------------------------------------") "+"
    print "| Category Summary:"
    for (cat in category_total)
        printf "| %-20s %30s |\n", cat, sprintf("%d", category_total[cat])
    print "+" sprintf("%-50s", "==================================================") "+"
    printf "| %-30s %20d |\n", "GRAND TOTAL", grand_total
    print "+" sprintf("%-50s", "==================================================") "+"
}' /tmp/sales_data.txt

echo ""
echo "Monthly breakdown:"
awk '{
    month = substr($1, 1, 7)
    monthly[$2 "-" month] += $4 * $5
}
END {
    for (k in monthly)
        print k, monthly[k]
}' /tmp/sales_data.txt | sort
```

---

## ขั้นตอนที่ 343: awk Multi-file Processing

```bash
#!/usr/bin/env bash
# awk_multifile.sh - Multi-file Processing

echo "=== awk Multi-file Processing ==="

# สร้างไฟล์ทดสอบ
cat > /tmp/file_a.txt << 'EOF'
ID Name
1  Alice
2  Bob
3  Charlie
EOF

cat > /tmp/file_b.txt << 'EOF'
ID Score
1  95
2  82
3  78
4  90
EOF

echo "1. ประมวลผลแต่ละไฟล์แตกต่างกัน:"
awk 'FNR==1 {print "Processing:", FILENAME}
     FNR>1  {print "  Record", FNR-1, ":", $0}' /tmp/file_a.txt /tmp/file_b.txt

echo ""
echo "2. Join สองไฟล์ (เหมือน SQL JOIN):"
awk 'FNR==NR && FNR>1 {name[$1]=$2; next}
     FNR>1 {
         if ($1 in name)
             printf "%-3s %-10s %s\n", $1, name[$1], $2
     }' /tmp/file_a.txt /tmp/file_b.txt

echo ""
echo "3. สร้าง lookup table:"
cat > /tmp/dept_lookup.txt << 'EOF'
1 Engineering
2 Marketing
3 HR
4 Design
EOF

cat > /tmp/employees.txt << 'EOF'
Alice 1 75000
Bob 2 65000
Charlie 1 85000
Diana 3 70000
Eve 4 68000
EOF

awk 'FNR==NR {dept[$1]=$2; next}
     {printf "%-10s %-15s %d\n", $1, dept[$2], $3}' \
    /tmp/dept_lookup.txt /tmp/employees.txt

echo ""
echo "4. Merge files by common key:"
awk 'FNR==NR {a[$1]=$2; next} $1 in a {print $1, a[$1], $2}' \
    /tmp/file_a.txt /tmp/file_b.txt | sed '1d'
```

---

## ขั้นตอนที่ 344: awk Scripts

```bash
#!/usr/bin/env bash
# awk_scripts.sh - awk Script Files

echo "=== awk Script Files ==="

# สร้าง awk script สำหรับ CSV report
cat > /tmp/csv_report.awk << 'AWKEOF'
#!/usr/bin/awk -f
# CSV Report Generator

BEGIN {
    FS = ","
    OFS = "|"
    total = 0
    count = 0
    
    # Print header
    print "=" repeat("=", 50)
    printf "| %-10s %-10s %-10s %-12s |\n", "Name", "Age", "City", "Salary"
    print "=" repeat("=", 50)
}

function repeat(s, n,    r, i) {
    r = ""
    for (i = 0; i < n; i++) r = r s
    return r
}

NR > 1 {
    printf "| %-10s %-10s %-10s %12d |\n", $1, $2, $3, $4
    total += $4
    count++
    
    # Track max/min
    if (count == 1 || $4 > max_sal) { max_sal = $4; max_name = $1 }
    if (count == 1 || $4 < min_sal) { min_sal = $4; min_name = $1 }
    
    # Group by city
    city_total[$3] += $4
    city_count[$3]++
}

END {
    print "=" repeat("=", 50)
    printf "Summary:\n"
    printf "  Records:  %d\n", count
    printf "  Total:    %d\n", total
    printf "  Average:  %.2f\n", total/count
    printf "  Highest:  %s (%d)\n", max_name, max_sal
    printf "  Lowest:   %s (%d)\n", min_name, min_sal
    
    print "\nBy City:"
    for (city in city_count)
        printf "  %-15s Avg: %.0f\n", city":", city_total[city]/city_count[city]
}
AWKEOF

echo "Running awk script:"
awk -f /tmp/csv_report.awk /tmp/simple.csv

echo ""
echo "หรือรันด้วย: awk -f script.awk data.csv"
```

---

## ขั้นตอนที่ 345: Workshop - Data Processor ด้วย awk

```bash
#!/usr/bin/env bash
# data_processor_awk.sh - Workshop

set -euo pipefail

# ==================== awk Data Processor ====================

echo "=== Data Processor Workshop ==="

# สร้างข้อมูลจำลอง
cat > /tmp/orders.csv << 'EOF'
order_id,date,customer,product,category,quantity,unit_price,status
1001,2024-01-15,John,iPhone 15,Electronics,2,59900,completed
1002,2024-01-15,Jane,MacBook Pro,Electronics,1,119900,completed
1003,2024-01-16,Bob,Nike Shoes,Clothing,3,3500,completed
1004,2024-01-16,Alice,Python Book,Books,5,890,completed
1005,2024-01-17,Charlie,Samsung TV,Electronics,1,45000,pending
1006,2024-01-17,Diana,Coffee Maker,Kitchen,2,8900,completed
1007,2024-01-18,Eve,Jeans,Clothing,4,1500,cancelled
1008,2024-01-18,Frank,iPad Air,Electronics,2,29900,completed
1009,2024-01-19,Grace,Cookbook,Books,2,750,completed
1010,2024-01-19,Hank,Blender,Kitchen,1,4500,completed
EOF

echo "=== ORDER ANALYSIS REPORT ==="
echo ""

# Report 1: Overall statistics
echo "1. Overall Statistics:"
awk -F, 'NR>1 && $8=="completed" {
    total += $6 * $7
    orders++
}
END {
    printf "  Completed Orders: %d\n", orders
    printf "  Total Revenue:    THB %,.0f\n", total
    printf "  Average Order:    THB %,.0f\n", total/orders
}' /tmp/orders.csv

echo ""
echo "2. Revenue by Category:"
awk -F, 'NR>1 && $8=="completed" {
    cat_rev[$5] += $6 * $7
    cat_orders[$5]++
}
END {
    printf "  %-15s %12s %8s\n", "Category", "Revenue", "Orders"
    printf "  %s\n", sprintf("%-40s", "----------------------------------------")
    for (cat in cat_rev)
        printf "  %-15s %12d %8d\n", cat, cat_rev[cat], cat_orders[cat]
}' /tmp/orders.csv | sort -k2 -rn

echo ""
echo "3. Top Customers:"
awk -F, 'NR>1 && $8=="completed" {
    cust_rev[$3] += $6 * $7
    cust_orders[$3]++
}
END {
    n = 0
    print "  Customer    Revenue    Orders"
    for (c in cust_rev) {
        customers[++n] = c
    }
    # Sort by revenue (simple bubble sort)
    for (i=1; i<n; i++)
        for (j=1; j<=n-i; j++)
            if (cust_rev[customers[j]] < cust_rev[customers[j+1]]) {
                tmp = customers[j]
                customers[j] = customers[j+1]
                customers[j+1] = tmp
            }
    for (i=1; i<=n && i<=3; i++)
        printf "  %-10s %9d %8d\n", customers[i], cust_rev[customers[i]], cust_orders[customers[i]]
}' /tmp/orders.csv

echo ""
echo "4. Status Distribution:"
awk -F, 'NR>1 {status[$8]++; total++}
END {
    for (s in status)
        printf "  %-12s %4d (%5.1f%%)\n", s":", status[s], (status[s]/total)*100
}' /tmp/orders.csv

echo ""
echo "5. Daily Revenue:"
awk -F, 'NR>1 && $8=="completed" {
    daily[$2] += $6 * $7
    daily_orders[$2]++
}
END {
    print "  Date         Revenue  Orders"
    n = asorti(daily, sorted_dates)
    for (i=1; i<=n; i++) {
        d = sorted_dates[i]
        printf "  %-12s %8d %7d\n", d, daily[d], daily_orders[d]
    }
}' /tmp/orders.csv 2>/dev/null || \
awk -F, 'NR>1 && $8=="completed" {
    daily[$2] += $6 * $7
    daily_orders[$2]++
}
END {
    for (d in daily)
        printf "  %-12s %8d %7d\n", d, daily[d], daily_orders[d]
}' /tmp/orders.csv | sort

echo ""
echo "6. Generate CSV Summary:"
awk -F, 'NR>1 {
    cat_rev[$5] += $6 * $7
    cat_count[$5]++
}
BEGIN {print "category,total_revenue,order_count,avg_revenue"}
END {
    for (cat in cat_rev)
        printf "%s,%d,%d,%.2f\n", cat, cat_rev[cat], cat_count[cat], cat_rev[cat]/cat_count[cat]
}' /tmp/orders.csv | sort
```

---

## ขั้นตอนที่ 346: สรุป Part 13 - awk

```bash
#!/usr/bin/env bash
# summary_awk.sh

echo "=== สรุป awk ==="
echo ""
echo "awk เหมาะสำหรับ:"
echo "  - ประมวลผลข้อมูลแบบ columnar"
echo "  - สร้าง reports"
echo "  - คำนวณ aggregate (sum, avg, max)"
echo "  - Join และ merge files"
echo "  - Text transformation"
echo ""
echo "Built-in Variables:"
echo "  NR    - record number (บรรทัดรวม)"
echo "  NF    - field count"
echo "  \$0   - ทั้งบรรทัด"
echo "  \$1..\$n - fields"
echo "  FS    - field separator"
echo "  OFS   - output field separator"
echo "  RS    - record separator"
echo "  ORS   - output record separator"
echo "  FILENAME - ชื่อไฟล์"
echo "  FNR   - record number ต่อไฟล์"
echo ""
echo "String Functions:"
echo "  length(), substr(), index()"
echo "  split(), sub(), gsub(), match()"
echo "  sprintf(), toupper(), tolower()"
echo ""
echo "Patterns:"
echo "  /regex/     - match pattern"
echo "  \$2 > 100   - comparison"
echo "  NR > 1      - record number"
echo "  BEGIN / END - special patterns"
echo ""
echo "Next: Part 14 - Process Management"
```

---

## แบบฝึกหัด Part 13

### แบบฝึกหัดที่ 1: Payroll Calculator
ใช้ awk สร้าง payroll report:
- รับ CSV: name, hours, hourly_rate, department
- คำนวณ gross pay (hours * rate)
- คำนวณ overtime (> 40 hrs = 1.5x rate)
- หัก tax 20%
- สรุปต่อ department

### แบบฝึกหัดที่ 2: Log Statistics
วิเคราะห์ nginx access log:
- จำนวน requests ต่อ hour
- Top 10 requested URLs
- Error rate per endpoint
- Average response time

### แบบฝึกหัดที่ 3: Inventory Management
ประมวลผล inventory CSV:
- หา items ที่ stock ต่ำกว่า minimum
- คำนวณ total value ต่อ category
- สร้าง reorder report

### แบบฝึกหัดที่ 4: Student Grade Report
ประมวลผล grade data:
- คำนวณ GPA ต่อนักเรียน
- หา top students ต่อ subject
- สร้าง grade distribution histogram

---

## สรุป

| หัวข้อ | Steps |
|--------|-------|
| awk introduction | 331 |
| Built-in variables | 332 |
| BEGIN/END blocks | 333 |
| Arithmetic | 334 |
| String functions | 335 |
| Arrays | 336 |
| Control flow | 337 |
| User functions | 338 |
| CSV processing | 339 |
| Log analysis | 340 |
| System admin | 341 |
| Report generation | 342 |
| Multi-file processing | 343 |
| awk scripts | 344 |
| Workshop | 345 |

**ขั้นตอนต่อไป**: Part 14 - Process Management
