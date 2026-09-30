# Part 03: ตัวแปรและประเภทข้อมูล
## หลักสูตร Bash Script - ขั้นตอนที่ 51-80

---

## ขั้นตอนที่ 51: ตัวแปรพื้นฐาน (Variables)

### 51.1 การประกาศตัวแปร

```bash
#!/bin/bash

# ประกาศตัวแปรพื้นฐาน
name="สมชาย"
age=25
height=175.5
is_student=true

# กฎการตั้งชื่อตัวแปร:
# ✅ ขึ้นต้นด้วยตัวอักษรหรือ _
# ✅ ประกอบด้วย a-z, A-Z, 0-9, _
# ✅ Case-sensitive (name ≠ Name ≠ NAME)
# ❌ ห้ามมี space รอบ = (name = "value" ผิด!)
# ❌ ห้ามขึ้นต้นด้วยตัวเลข (1name ผิด)

# ถูกต้อง
my_name="สมชาย"
MyName="สมชาย"
MY_NAME="สมชาย"
_name="สมชาย"

# ผิด
# my name="สมชาย"  # มี space
# 1name="สมชาย"   # ขึ้นต้นด้วยตัวเลข

# การเข้าถึงค่าตัวแปร
echo $name
echo "$name"        # แนะนำ: ใส่ " เสมอ
echo "${name}"      # แนะนำที่สุด: ชัดเจนกว่า

# ตัวอย่าง: ทำไมต้องใส่ " ?
greeting="Hello World"
echo $greeting      # Hello World (ถูก)
echo "$greeting"    # Hello World (ถูก)

# แต่ถ้ามี space และไม่ใส่ "
files="file 1.txt file 2.txt"
ls $files           # จะมองเป็น 4 arguments แยกกัน
ls "$files"         # มองเป็น string เดียว
```

### 51.2 ชนิดตัวแปรใน Bash

```bash
#!/bin/bash

# 1. String (ค่าเริ่มต้น)
name="Hello World"
number_as_string="42"

# 2. Integer (ใช้ declare -i)
declare -i count=0
declare -i age=25
count=count+5       # 5 (arithmetic auto-expanded)

# 3. Array (ใช้ declare -a หรือ ())
declare -a fruits=("apple" "banana" "cherry")
colors=("red" "green" "blue")

# 4. Associative Array (dictionary) (ใช้ declare -A)
declare -A person
person["name"]="สมชาย"
person["age"]="25"
person["city"]="Bangkok"

# 5. Readonly (constant)
declare -r PI=3.14159
readonly MAX_SIZE=100
# PI=3.14  # Error! cannot modify readonly variable

# 6. Export (environment variable)
declare -x MY_ENV="exported"
export MY_ENV2="also exported"

# 7. Lowercase conversion
declare -l lower_var="HELLO WORLD"
echo "$lower_var"   # hello world

# 8. Uppercase conversion
declare -u upper_var="hello world"
echo "$upper_var"   # HELLO WORLD
```

### 51.3 Special Variables

```bash
#!/bin/bash

echo "=== Special Variables ==="

# Script information
echo "Script name:      $0"         # ชื่อ script
echo "Script directory: $(dirname "$0")"

# Arguments
echo "Arg 1:    $1"                 # argument ที่ 1
echo "Arg 2:    $2"                 # argument ที่ 2
echo "All args: $@"                 # ทุก arguments (แยกแต่ละตัว)
echo "All args: $*"                 # ทุก arguments (รวมเป็น string)
echo "Num args: $#"                 # จำนวน arguments

# Process
echo "PID:     $$"                  # PID ของ script นี้
echo "PPID:    $PPID"               # PID ของ parent process
echo "BG PID:  $!"                  # PID ของ background process ล่าสุด

# Status
echo "Last exit: $?"                # exit code ของคำสั่งก่อนหน้า

# Shell
echo "Bash version: $BASH_VERSION"
echo "Bash PID:     $BASHPID"
echo "Shell opts:   $-"             # shell options ที่เปิดอยู่

# ตัวอย่างการใช้ $@ vs $*
example_args() {
    echo "=== ใช้ \$@ ==="
    for arg in "$@"; do
        echo "  - '$arg'"
    done
    
    echo "=== ใช้ \$* ==="
    for arg in "$*"; do
        echo "  - '$arg'"
    done
}

example_args "hello world" "foo" "bar baz"
```

---

## ขั้นตอนที่ 52: String Operations

### 52.1 String Concatenation

```bash
#!/bin/bash

# ต่อ strings
first="Hello"
second="World"
combined="$first $second"
echo "$combined"    # Hello World

# ต่อแบบอื่น
prefix="pre"
suffix="fix"
word="${prefix}${suffix}"
echo "$word"        # prefix

# ต่อแบบ concatenation shorthand
str=""
str+="Hello"
str+=" "
str+="World"
echo "$str"         # Hello World

# สร้าง string จากตัวเลข
num=42
message="The answer is $num"
echo "$message"     # The answer is 42

# Multiline string
multi="บรรทัดที่ 1
บรรทัดที่ 2
บรรทัดที่ 3"
echo "$multi"

# String ที่มี quotes ข้างใน
with_quote="He said \"Hello\""
with_single='It'"'"'s fine'
echo "$with_quote"   # He said "Hello"
echo "$with_single"  # It's fine
```

### 52.2 String Length

```bash
#!/bin/bash

text="Hello, World!"

# ความยาว string
echo "${#text}"     # 13

# ตัวอย่าง
name="สวัสดี"
echo "ความยาวของ '$name' = ${#name} characters"

# ใช้ใน condition
password="secret123"
if [ ${#password} -lt 8 ]; then
    echo "Password สั้นเกินไป!"
else
    echo "Password ยาวพอแล้ว"
fi

# นับตัวอักษรใน array
fruits=("apple" "banana" "cherry")
echo "จำนวน fruits: ${#fruits[@]}"
```

### 52.3 Substring Extraction

```bash
#!/bin/bash

text="Hello, World!"

# ${string:start:length}
echo "${text:0:5}"    # Hello (ตั้งแต่ pos 0, ความยาว 5)
echo "${text:7:5}"    # World (ตั้งแต่ pos 7, ความยาว 5)
echo "${text:7}"      # World! (ตั้งแต่ pos 7 ถึงท้าย)
echo "${text: -6}"    # orld!  (6 ตัวสุดท้าย, ระวัง space!)
echo "${text: -6:5}"  # orld (6 ตัวสุดท้าย, ความยาว 5)

# Extract filename parts
filepath="/home/user/documents/report.txt"
echo "${filepath##*/}"    # report.txt (filename only)
echo "${filepath%/*}"     # /home/user/documents (directory)
echo "${filepath##*.}"    # txt (extension)
echo "${filepath%.*}"     # /home/user/documents/report (no extension)

# Examples with filenames
for file in file1.txt file2.jpg script.sh; do
    name="${file%.*}"
    ext="${file##*.}"
    echo "Name: $name | Extension: $ext"
done
```

### 52.4 String Replacement

```bash
#!/bin/bash

text="Hello World Hello"

# แทนที่ครั้งแรกที่พบ
echo "${text/Hello/Hi}"      # Hi World Hello

# แทนที่ทั้งหมด
echo "${text//Hello/Hi}"     # Hi World Hi

# แทนที่ที่ต้นสตริง (#)
echo "${text/#Hello/Hi}"     # Hi World Hello

# แทนที่ที่ท้ายสตริง (%)
echo "${text/%Hello/Hi}"     # Hello World Hi

# ลบ substring
echo "${text/Hello/}"        # " World Hello"
echo "${text//Hello/}"       # " World "

# Case conversion (Bash 4+)
lower="${text,,}"            # hello world hello
upper="${text^^}"            # HELLO WORLD HELLO
cap="${text~}"               # hELLO wORLD hELLO (toggle first char)
echo "$lower"
echo "$upper"
echo "$cap"

# Pattern matching
url="https://www.example.com/path?param=value"
# ลบ https://
path="${url#*://}"
echo "$path"    # www.example.com/path?param=value

# เอาเฉพาะ domain
domain="${path%%/*}"
echo "$domain"  # www.example.com
```

### 52.5 String Testing และ Comparison

```bash
#!/bin/bash

str1="Hello"
str2="World"
empty=""

# ตรวจสอบว่าว่างหรือไม่
if [ -z "$empty" ]; then
    echo "ว่าง (zero length)"
fi

if [ -n "$str1" ]; then
    echo "ไม่ว่าง (non-zero length)"
fi

# เปรียบเทียบ strings
if [ "$str1" = "$str2" ]; then
    echo "เท่ากัน"
else
    echo "ไม่เท่ากัน"
fi

# ไม่เท่ากัน
if [ "$str1" != "$str2" ]; then
    echo "ไม่เท่ากัน"
fi

# เปรียบเทียบ lexicographic (ใน [[ ]])
if [[ "$str1" < "$str2" ]]; then
    echo "$str1 มาก่อน $str2 ตามลำดับ alphabet"
fi

# Pattern matching ใน [[ ]]
filename="report_2026.txt"
if [[ "$filename" == *.txt ]]; then
    echo "เป็นไฟล์ .txt"
fi

if [[ "$filename" == report_* ]]; then
    echo "ขึ้นต้นด้วย report_"
fi

# Regular expression matching
email="user@example.com"
if [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
    echo "Email format ถูกต้อง"
else
    echo "Email format ไม่ถูกต้อง"
fi
```

---

## ขั้นตอนที่ 53: Numbers และ Arithmetic

### 53.1 Arithmetic Operations

```bash
#!/bin/bash

# วิธีที่ 1: $(( )) - Arithmetic expansion (แนะนำ)
a=10
b=3

echo "Addition:       $((a + b))"    # 13
echo "Subtraction:    $((a - b))"    # 7
echo "Multiplication: $((a * b))"    # 30
echo "Division:       $((a / b))"    # 3 (integer division)
echo "Modulus:        $((a % b))"    # 1
echo "Exponentiation: $((a ** b))"   # 1000

# วิธีที่ 2: let
let result=a+b
let "result = a + b"
echo "Let result: $result"

# วิธีที่ 3: expr (เก่า ไม่แนะนำ)
result=$(expr $a + $b)
echo "Expr result: $result"

# วิธีที่ 4: bc - สำหรับ floating point
result=$(echo "10 / 3" | bc)
echo "Integer: $result"    # 3

result=$(echo "scale=2; 10 / 3" | bc)
echo "Float:   $result"    # 3.33

# Increment / Decrement
count=0
((count++))
((count++))
((count--))
echo "Count: $count"    # 1

# Augmented assignment
x=10
((x += 5))   # x = x + 5
((x -= 3))   # x = x - 3
((x *= 2))   # x = x * 2
((x /= 4))   # x = x / 4
((x %= 3))   # x = x % 3
echo "x = $x"
```

### 53.2 Floating Point Arithmetic

```bash
#!/bin/bash

# Bash ไม่รองรับ floating point โดยตรง
# ใช้ bc หรือ awk แทน

# bc - basic calculator
calculate() {
    local expression="$1"
    local precision="${2:-2}"
    echo "scale=$precision; $expression" | bc
}

echo "sqrt(2):    $(calculate 'sqrt(2)' 10)"
echo "pi:         $(calculate '4*a(1)' 10 )"  # bc atan function
echo "10/3:       $(calculate '10/3')"
echo "2^10:       $(calculate '2^10')"

# awk - ทำ arithmetic ได้ดี
echo "10/3" | awk '{printf "%.4f\n", $1/1}'
awk 'BEGIN { printf "%.2f\n", 355/113 }'
awk 'BEGIN { print sqrt(2) }'

# Python เป็นทางเลือกที่ดี
python3 -c "print(f'{10/3:.4f}')"
python3 -c "import math; print(math.pi)"

# ฟังก์ชัน math library
math() {
    local func="$1"
    local arg="$2"
    echo "$func($arg)" | bc -l
}

echo "sin(1): $(math 's' 1)"
echo "cos(1): $(math 'c' 1)"
echo "log(10): $(math 'l' 10)"
```

### 53.3 Number Validation

```bash
#!/bin/bash

# ตรวจสอบว่าเป็นตัวเลขหรือไม่
is_integer() {
    local value="$1"
    [[ "$value" =~ ^-?[0-9]+$ ]]
}

is_float() {
    local value="$1"
    [[ "$value" =~ ^-?[0-9]+(\.[0-9]+)?$ ]]
}

is_positive() {
    local value="$1"
    is_integer "$value" && [ "$value" -gt 0 ]
}

# ทดสอบ
test_values=("42" "-10" "3.14" "abc" "12.34.56" "")

for val in "${test_values[@]}"; do
    if is_integer "$val"; then
        echo "'$val' เป็น integer"
    elif is_float "$val"; then
        echo "'$val' เป็น float"
    else
        echo "'$val' ไม่ใช่ตัวเลข"
    fi
done

# แปลง base
decimal=255
echo "Decimal: $decimal"
echo "Binary:  $(echo "obase=2; $decimal" | bc)"
echo "Hex:     $(echo "obase=16; $decimal" | bc)"
echo "Octal:   $(echo "obase=8; $decimal" | bc)"

# แปลงกลับ
hex="FF"
echo "Hex $hex = $((16#$hex)) decimal"

binary="11111111"
echo "Binary $binary = $((2#$binary)) decimal"
```

---

## ขั้นตอนที่ 54: Arrays

### 54.1 Indexed Arrays

```bash
#!/bin/bash

# การสร้าง array
fruits=("apple" "banana" "cherry" "date" "elderberry")

# หรือ
declare -a colors
colors[0]="red"
colors[1]="green"
colors[2]="blue"

# เข้าถึงข้อมูล
echo "${fruits[0]}"        # apple (index เริ่มที่ 0)
echo "${fruits[2]}"        # cherry
echo "${fruits[-1]}"       # elderberry (index สุดท้าย)

# เข้าถึงทุกตัว
echo "${fruits[@]}"        # apple banana cherry date elderberry
echo "${fruits[*]}"        # apple banana cherry date elderberry

# ความยาว array
echo "${#fruits[@]}"       # 5

# ดู indexes ทั้งหมด
echo "${!fruits[@]}"       # 0 1 2 3 4

# เพิ่มข้อมูล
fruits+=("fig")            # เพิ่มท้าย
fruits[6]="grape"          # เพิ่มที่ index ที่กำหนด

# ลบข้อมูล
unset fruits[2]            # ลบ cherry (index 2)
echo "${fruits[@]}"        # apple banana date elderberry fig grape
echo "${!fruits[@]}"       # 0 1 3 4 5 6 (index 2 หายไป!)

# Slicing
echo "${fruits[@]:1:3}"    # banana date elderberry (ตั้งแต่ index 1, เอา 3 ตัว)

# Loop ผ่าน array
for fruit in "${fruits[@]}"; do
    echo "  - $fruit"
done

# Loop พร้อม index
for i in "${!fruits[@]}"; do
    echo "  [$i] ${fruits[$i]}"
done
```

### 54.2 Associative Arrays (Dictionary)

```bash
#!/bin/bash

# สร้าง associative array
declare -A person=(
    ["name"]="สมชาย"
    ["age"]="25"
    ["city"]="Bangkok"
    ["occupation"]="Developer"
)

# เพิ่มข้อมูล
person["email"]="somchai@example.com"

# เข้าถึงข้อมูล
echo "${person["name"]}"    # สมชาย
echo "${person[age]}"       # 25

# ดู keys ทั้งหมด
echo "${!person[@]}"        # name age city occupation email

# ดู values ทั้งหมด
echo "${person[@]}"

# ความยาว
echo "${#person[@]}"        # 5

# ตรวจสอบว่า key มีอยู่หรือไม่
if [[ "${person[name]+exists}" ]]; then
    echo "key 'name' มีอยู่"
fi

# Loop ผ่าน associative array
for key in "${!person[@]}"; do
    echo "  $key: ${person[$key]}"
done

# ลบ key
unset person["city"]

# ตัวอย่าง: Count occurrences
declare -A word_count

text="the quick brown fox jumps over the lazy dog the"
for word in $text; do
    ((word_count[$word]++))
done

echo "นับคำ:"
for word in "${!word_count[@]}"; do
    echo "  '$word': ${word_count[$word]} ครั้ง"
done | sort
```

### 54.3 Array Operations

```bash
#!/bin/bash

# Merge arrays
arr1=("a" "b" "c")
arr2=("d" "e" "f")
merged=("${arr1[@]}" "${arr2[@]}")
echo "${merged[@]}"    # a b c d e f

# Copy array
original=("1" "2" "3")
copy=("${original[@]}")

# Sort array (ไม่มี built-in sort)
arr=("banana" "apple" "cherry" "date")
IFS=$'\n' sorted=($(sort <<< "${arr[*]}")); unset IFS
echo "${sorted[@]}"    # apple banana cherry date

# Filter array
numbers=(1 2 3 4 5 6 7 8 9 10)
even_numbers=()
for n in "${numbers[@]}"; do
    if (( n % 2 == 0 )); then
        even_numbers+=("$n")
    fi
done
echo "Even: ${even_numbers[@]}"

# Map (แปลง array)
words=("hello" "world" "foo")
upper_words=()
for word in "${words[@]}"; do
    upper_words+=("${word^^}")
done
echo "${upper_words[@]}"    # HELLO WORLD FOO

# ตรวจสอบว่า element อยู่ใน array
contains() {
    local element="$1"
    shift
    local array=("$@")
    for item in "${array[@]}"; do
        [[ "$item" == "$element" ]] && return 0
    done
    return 1
}

fruits=("apple" "banana" "cherry")
if contains "banana" "${fruits[@]}"; then
    echo "พบ banana"
fi

if ! contains "grape" "${fruits[@]}"; then
    echo "ไม่พบ grape"
fi
```

---

## ขั้นตอนที่ 55: Variable Scope

### 55.1 Global vs Local Variables

```bash
#!/bin/bash

# Global variable
global_var="ฉันเป็น global"

modify_global() {
    # เข้าถึง global ได้
    echo "ใน function: $global_var"
    
    # แก้ global
    global_var="ถูกแก้ใน function"
    
    # local variable
    local local_var="ฉันเป็น local"
    echo "Local: $local_var"
}

echo "ก่อนเรียก function: $global_var"
modify_global
echo "หลังเรียก function: $global_var"

# local_var ไม่มีใน global scope
echo "Local outside: ${local_var:-'ไม่มีค่า'}"

# ตัวอย่างปัญหาจาก scope
counter=0

increment() {
    local count="$1"
    # แก้ global
    counter=$((counter + count))
}

increment 5
increment 3
echo "Counter: $counter"    # 8
```

### 55.2 Parameter Expansion

```bash
#!/bin/bash

# Default values
name="${1:-สมชาย}"           # ถ้าไม่มี $1 ใช้ "สมชาย"
echo "Hello, $name!"

# Assign default
${name:="default"}           # ถ้าว่าง ใส่ค่า default ให้
echo "$name"

# Error if unset
# ${required:?"required is not set"}  # จะ exit ถ้าไม่มีค่า

# Use alternate
alt="${name:+"${name^^}"}"  # ถ้ามีค่า ใช้ uppercase แทน
echo "Alt: $alt"

# ตัวอย่างใช้งานจริง
host="${1:-localhost}"
port="${2:-8080}"
env="${3:-development}"

echo "Host: $host"
echo "Port: $port"
echo "Env:  $env"
```

---

## ขั้นตอนที่ 56: Advanced Variable Techniques

### 56.1 nameref (Reference Variables)

```bash
#!/bin/bash

# Bash 4.3+ feature
declare -n ref_var

# ใช้ nameref สำหรับ dynamic variable names
set_color() {
    local color_name="$1"
    local color_value="$2"
    
    # สร้าง reference ไปยัง variable ชื่อ color_name
    declare -n color_ref="$color_name"
    color_ref="$color_value"
}

set_color RED "#FF0000"
set_color GREEN "#00FF00"
set_color BLUE "#0000FF"

echo "RED:   $RED"
echo "GREEN: $GREEN"
echo "BLUE:  $BLUE"

# Pass array by reference
process_array() {
    declare -n arr_ref="$1"
    
    echo "Processing ${#arr_ref[@]} items:"
    for item in "${arr_ref[@]}"; do
        echo "  - $item"
    done
    
    # เพิ่มข้อมูลใน original array
    arr_ref+=("new_item")
}

my_data=("a" "b" "c")
process_array my_data
echo "After: ${my_data[@]}"    # a b c new_item
```

### 56.2 Dynamic Variable Names

```bash
#!/bin/bash

# สร้างตัวแปรแบบ dynamic
for i in 1 2 3; do
    declare "var_$i=value_$i"
done

echo "$var_1"    # value_1
echo "$var_2"    # value_2
echo "$var_3"    # value_3

# เข้าถึงแบบ indirect
var_name="var_2"
echo "${!var_name}"    # value_2

# ตัวอย่าง: สร้าง config จาก environment
declare -A config

# โหลด config จาก environment variables ที่ขึ้นต้นด้วย APP_
while IFS='=' read -r name value; do
    if [[ "$name" == APP_* ]]; then
        key="${name#APP_}"
        config["$key"]="$value"
    fi
done < <(env)

echo "App config:"
for key in "${!config[@]}"; do
    echo "  $key = ${config[$key]}"
done
```

---

## ขั้นตอนที่ 57: Workshop 03 - Variable Mastery

### Workshop: Calculator Script

```bash
#!/usr/bin/env bash
# =============================================================================
# Workshop 03: calculator.sh
# เครื่องคิดเลขอย่างง่าย
# =============================================================================

set -euo pipefail

# Colors
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly CYAN='\033[0;36m'
readonly NC='\033[0m'

# History
declare -a HISTORY=()
HISTORY_MAX=10

# แสดงผลลัพธ์
show_result() {
    local expr="$1"
    local result="$2"
    echo -e "${GREEN}= $result${NC}"
    
    # เพิ่มใน history
    HISTORY+=("$expr = $result")
    if [ ${#HISTORY[@]} -gt $HISTORY_MAX ]; then
        HISTORY=("${HISTORY[@]:1}")
    fi
}

# แสดง history
show_history() {
    if [ ${#HISTORY[@]} -eq 0 ]; then
        echo "ยังไม่มี history"
        return
    fi
    
    echo -e "${CYAN}=== History ===${NC}"
    local i=1
    for item in "${HISTORY[@]}"; do
        echo "  $i. $item"
        ((i++))
    done
}

# คำนวณด้วย bc
calc() {
    local expression="$1"
    local precision="${2:-4}"
    
    # ตรวจสอบ division by zero
    if [[ "$expression" =~ /[[:space:]]*0([^.0-9]|$) ]]; then
        echo -e "${RED}Error: Division by zero!${NC}"
        return 1
    fi
    
    local result
    result=$(echo "scale=$precision; $expression" | bc -l 2>/dev/null)
    
    if [ $? -ne 0 ] || [ -z "$result" ]; then
        echo -e "${RED}Error: Invalid expression${NC}"
        return 1
    fi
    
    # ลบ trailing zeros
    result=$(echo "$result" | sed 's/\.0*$//' | sed 's/\(\.[0-9]*[1-9]\)0*/\1/')
    
    echo "$result"
}

# คำนวณ expressions พิเศษ
special_calc() {
    local op="$1"
    local num="${2:-0}"
    
    case "$op" in
        sqrt)   echo "scale=4; sqrt($num)" | bc -l ;;
        abs)    echo "scale=4; if ($num < 0) -1*$num else $num" | bc -l ;;
        log)    python3 -c "import math; print(f'{math.log($num):.4f}')" ;;
        log10)  python3 -c "import math; print(f'{math.log10($num):.4f}')" ;;
        sin)    python3 -c "import math; print(f'{math.sin(math.radians($num)):.4f}')" ;;
        cos)    python3 -c "import math; print(f'{math.cos(math.radians($num)):.4f}')" ;;
        tan)    python3 -c "import math; print(f'{math.tan(math.radians($num)):.4f}')" ;;
        *)      echo "Unknown operation"; return 1 ;;
    esac
}

# แปลง base
convert_base() {
    local num="$1"
    local from_base="$2"
    local to_base="$3"
    
    # แปลงเป็น decimal ก่อน
    local decimal
    decimal=$(echo "obase=10; ibase=$from_base; $num" | bc 2>/dev/null)
    
    if [ -z "$decimal" ]; then
        echo "Invalid number for base $from_base"
        return 1
    fi
    
    # แปลงเป็น target base
    echo "obase=$to_base; ibase=10; $decimal" | bc
}

# Main calculator
calculator() {
    echo -e "${CYAN}"
    echo "╔══════════════════════════════╗"
    echo "║     Bash Calculator v1.0     ║"
    echo "╚══════════════════════════════╝"
    echo -e "${NC}"
    echo "คำสั่ง: quit=ออก, hist=ประวัติ, clear=ล้างหน้าจอ"
    echo "ฟังก์ชัน: sqrt(n), sin/cos/tan(n), log(n)"
    echo "แปลง base: hex(n), bin(n), oct(n)"
    echo ""
    
    while true; do
        echo -ne "${YELLOW}calc> ${NC}"
        read -r input
        
        # แปลง input เป็น lowercase
        input="${input,,}"
        
        case "$input" in
            quit|exit|q)
                echo "ลาก่อน!"
                break
                ;;
            hist|history)
                show_history
                ;;
            clear)
                clear
                ;;
            sqrt*)
                # sqrt(number)
                num=$(echo "$input" | grep -oE '[0-9.]+')
                result=$(special_calc "sqrt" "$num")
                show_result "sqrt($num)" "$result"
                ;;
            sin*|cos*|tan*)
                # sin(degrees)
                op=$(echo "$input" | grep -oE '^[a-z]+')
                num=$(echo "$input" | grep -oE '[0-9.]+')
                result=$(special_calc "$op" "$num")
                show_result "$op($num°)" "$result"
                ;;
            log*)
                num=$(echo "$input" | grep -oE '[0-9.]+')
                result=$(special_calc "log" "$num")
                show_result "ln($num)" "$result"
                ;;
            hex*)
                # แปลงเป็น hex
                num=$(echo "$input" | grep -oE '[0-9]+')
                result=$(convert_base "$num" "10" "16")
                show_result "decimal $num" "0x$result (hex)"
                ;;
            bin*)
                # แปลงเป็น binary
                num=$(echo "$input" | grep -oE '[0-9]+')
                result=$(convert_base "$num" "10" "2")
                show_result "decimal $num" "0b$result (binary)"
                ;;
            "")
                continue
                ;;
            *)
                # คำนวณ expression ปกติ
                result=$(calc "$input" 2>/dev/null)
                if [ $? -eq 0 ]; then
                    show_result "$input" "$result"
                fi
                ;;
        esac
    done
}

# Run
calculator
```

---

## ขั้นตอนที่ 58: สรุปและแบบฝึกหัด

### สิ่งที่เรียนรู้ใน Part 03

1. ✅ การประกาศและใช้งานตัวแปร
2. ✅ ชนิดตัวแปร: string, integer, array, associative array
3. ✅ Special variables ($0, $1, $@, $$, $?)
4. ✅ String operations: length, substring, replacement
5. ✅ Number arithmetic และ floating point
6. ✅ Indexed arrays และ associative arrays
7. ✅ Variable scope: global vs local
8. ✅ Parameter expansion
9. ✅ Dynamic variable names

### แบบฝึกหัด Part 03

**Easy:**
1. สร้าง script ที่รับ firstName และ lastName แล้วแสดง Full Name
2. สร้าง array ของ 5 ผลไม้ แล้ว loop แสดงแต่ละตัว
3. เขียน script ตรวจสอบว่า string ว่างหรือไม่

**Medium:**
4. สร้าง associative array เป็น address book
5. เขียนฟังก์ชัน trim ที่ลบ whitespace หัวท้าย string

**Hard:**
6. สร้าง stack data structure ด้วย arrays (push, pop, peek)

### เฉลย

```bash
# ข้อ 1: Full Name
#!/bin/bash
read -rp "First name: " first
read -rp "Last name: " last
full_name="$first $last"
echo "Full name: $full_name"

# ข้อ 2: Array loop
#!/bin/bash
fruits=("Apple" "Banana" "Cherry" "Date" "Elderberry")
echo "รายการผลไม้:"
for i in "${!fruits[@]}"; do
    echo "  $((i+1)). ${fruits[$i]}"
done

# ข้อ 5: Trim function
trim() {
    local str="$1"
    # ลบ leading whitespace
    str="${str#"${str%%[![:space:]]*}"}"
    # ลบ trailing whitespace
    str="${str%"${str##*[![:space:]]}"}"
    echo "$str"
}

test_str="   Hello World   "
trimmed=$(trim "$test_str")
echo "Before: '$test_str'"
echo "After:  '$trimmed'"

# ข้อ 6: Stack
declare -a STACK=()

push() { STACK+=("$1"); echo "Pushed: $1"; }
pop()  {
    if [ ${#STACK[@]} -eq 0 ]; then
        echo "Stack empty!"
        return 1
    fi
    local top="${STACK[-1]}"
    unset 'STACK[-1]'
    echo "Popped: $top"
}
peek() {
    [ ${#STACK[@]} -eq 0 ] && echo "Stack empty!" || echo "Top: ${STACK[-1]}"
}
size() { echo "Size: ${#STACK[@]}"; }

push "first"
push "second"
push "third"
peek
pop
peek
size
```

---

**ต่อไป:** [Part 04 - Input/Output และ Redirection](part-04.md)

*Part 03 จบแล้ว! พร้อมเรียน Part 04 →*
