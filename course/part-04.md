# Part 04: Input/Output และ Redirection
## หลักสูตร Bash Script - ขั้นตอนที่ 81-110

---

## ขั้นตอนที่ 81: Output Methods

### 81.1 echo vs printf

```bash
#!/bin/bash

# echo - แสดงข้อความพื้นฐาน
echo "Hello World"
echo "Line 1"
echo "Line 2"

# echo flags
echo -n "No newline at end"    # -n ไม่ขึ้นบรรทัดใหม่
echo -e "Line1\nLine2"         # -e enable escape sequences
echo -e "Tab:\there"
echo -e "Color:\033[32mGreen\033[0m"

# Escape sequences ใน echo -e:
# \n  = newline
# \t  = tab
# \r  = carriage return
# \a  = alert (bell)
# \b  = backspace
# \e  = escape character
# \\  = backslash
# \0NNN = octal value

# printf - แสดงผลแบบ formatted (แนะนำกว่า echo)
printf "Hello %s!\n" "World"
printf "Name: %-20s Age: %d\n" "สมชาย" 25
printf "Price: %.2f THB\n" 1234.5
printf "Hex: %x\n" 255          # ff
printf "Oct: %o\n" 255          # 377
printf "Pad: %05d\n" 42         # 00042
printf "Left: %-10s|\n" "abc"   # abc       |
printf "Right: %10s|\n" "abc"   #        abc|

# printf format specifiers:
# %s  = string
# %d  = integer
# %f  = float
# %e  = scientific notation
# %x  = hex lowercase
# %X  = hex uppercase
# %o  = octal
# %b  = binary (bash only)
# %-  = left-align
# %0N = zero-pad to N digits
# %.N = N decimal places

# ตัวอย่าง: สร้างตาราง
printf "%-20s %10s %10s\n" "Product" "Price" "Qty"
printf "%s\n" "$(printf '%.0s-' {1..42})"
printf "%-20s %10.2f %10d\n" "Apple" 15.00 100
printf "%-20s %10.2f %10d\n" "Banana" 8.50 250
printf "%-20s %10.2f %10d\n" "Cherry" 45.00 50
```

### 81.2 Colored Output

```bash
#!/bin/bash

# ANSI Color codes
declare -A COLORS=(
    [RED]='\033[0;31m'
    [GREEN]='\033[0;32m'
    [YELLOW]='\033[1;33m'
    [BLUE]='\033[0;34m'
    [PURPLE]='\033[0;35m'
    [CYAN]='\033[0;36m'
    [WHITE]='\033[1;37m'
    [GRAY]='\033[0;37m'
    [BOLD]='\033[1m'
    [DIM]='\033[2m'
    [UNDERLINE]='\033[4m'
    [BLINK]='\033[5m'
    [REVERSE]='\033[7m'
    [NC]='\033[0m'
)

# ฟังก์ชัน colored output
color_echo() {
    local color="${COLORS[$1]}"
    shift
    echo -e "${color}$*${COLORS[NC]}"
}

# ใช้งาน
color_echo RED "Error message!"
color_echo GREEN "Success!"
color_echo YELLOW "Warning!"
color_echo BLUE "Info"
color_echo CYAN "Processing..."
color_echo BOLD "Bold text"
color_echo UNDERLINE "Underlined text"

# Log functions มาตรฐาน
log_info()    { echo -e "\033[0;34m[INFO]\033[0m  $*"; }
log_success() { echo -e "\033[0;32m[OK]\033[0m    $*"; }
log_warning() { echo -e "\033[1;33m[WARN]\033[0m  $*"; }
log_error()   { echo -e "\033[0;31m[ERROR]\033[0m $*" >&2; }
log_debug()   { echo -e "\033[0;37m[DEBUG]\033[0m $*" >&2; }

log_info    "Starting process..."
log_success "File created successfully"
log_warning "Disk space is low"
log_error   "Connection failed"

# ตรวจสอบว่า terminal รองรับ colors หรือไม่
setup_colors() {
    if [ -t 1 ] && command -v tput &>/dev/null; then
        RED=$(tput setaf 1)
        GREEN=$(tput setaf 2)
        YELLOW=$(tput setaf 3)
        BLUE=$(tput setaf 4)
        NC=$(tput sgr0)
    else
        RED="" GREEN="" YELLOW="" BLUE="" NC=""
    fi
}

setup_colors
echo "${GREEN}Terminal supports colors!${NC}"
```

### 81.3 Progress Indicators

```bash
#!/bin/bash

# Simple progress bar
show_progress() {
    local current="$1"
    local total="$2"
    local width="${3:-40}"
    
    local percent=$((current * 100 / total))
    local filled=$((current * width / total))
    local empty=$((width - filled))
    
    printf "\r["
    printf "%${filled}s" | tr ' ' '█'
    printf "%${empty}s" | tr ' ' '░'
    printf "] %3d%% (%d/%d)" "$percent" "$current" "$total"
    
    if [ "$current" -eq "$total" ]; then
        echo ""  # newline เมื่อเสร็จ
    fi
}

# ตัวอย่างการใช้
echo "Downloading..."
total=100
for ((i=0; i<=total; i++)); do
    show_progress "$i" "$total"
    sleep 0.02
done
echo "Done!"

# Spinner animation
spinner() {
    local pid="$1"
    local message="${2:-Loading}"
    local delay=0.1
    local spinstr='⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏'
    
    while kill -0 "$pid" 2>/dev/null; do
        for ((i=0; i<${#spinstr}; i++)); do
            printf "\r${spinstr:$i:1} $message..."
            sleep "$delay"
        done
    done
    printf "\r✓ $message done!\n"
}

# ใช้งาน spinner
(sleep 3) &
bg_pid=$!
spinner "$bg_pid" "Processing"
wait "$bg_pid"
```

---

## ขั้นตอนที่ 82: Input Methods

### 82.1 read - อ่าน Input จากผู้ใช้

```bash
#!/bin/bash

# อ่าน input พื้นฐาน
read -r name
echo "Hello, $name!"

# อ่านพร้อม prompt (-p)
read -rp "Enter your name: " name
echo "Hello, $name!"

# อ่านแบบ silent (-s) สำหรับ password
read -rsp "Enter password: " password
echo ""  # เพิ่ม newline
echo "Password length: ${#password}"

# Timeout (-t)
if read -rt 5 -p "Enter input (5 sec timeout): " input; then
    echo "You entered: $input"
else
    echo ""
    echo "Timeout!"
fi

# จำกัดจำนวน characters (-n)
read -rn1 -p "Press any key to continue..."
echo ""

# อ่านหลายค่า
read -r first last age <<< "สมชาย นามสกุล 25"
echo "First: $first, Last: $last, Age: $age"

# อ่านทีละบรรทัด
while IFS= read -r line; do
    echo "Line: $line"
done <<< "line 1
line 2
line 3"

# อ่านจากไฟล์
while IFS=: read -r username _ uid gid _ home shell; do
    printf "%-20s UID:%-6s Home: %s\n" "$username" "$uid" "$home"
done < /etc/passwd | head -5

# อ่าน array
IFS=' ' read -ra words <<< "hello world foo bar"
echo "Word count: ${#words[@]}"
for word in "${words[@]}"; do
    echo "  - $word"
done
```

### 82.2 Input Validation

```bash
#!/bin/bash

# ฟังก์ชัน validation
validate_integer() {
    local value="$1"
    local min="${2:-}"
    local max="${3:-}"
    
    if ! [[ "$value" =~ ^-?[0-9]+$ ]]; then
        echo "ต้องเป็นตัวเลขจำนวนเต็ม"
        return 1
    fi
    
    if [ -n "$min" ] && [ "$value" -lt "$min" ]; then
        echo "ต้องมากกว่าหรือเท่ากับ $min"
        return 1
    fi
    
    if [ -n "$max" ] && [ "$value" -gt "$max" ]; then
        echo "ต้องน้อยกว่าหรือเท่ากับ $max"
        return 1
    fi
    
    return 0
}

validate_email() {
    local email="$1"
    if [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
        return 0
    else
        echo "Email format ไม่ถูกต้อง"
        return 1
    fi
}

validate_date() {
    local date="$1"
    if [[ "$date" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]]; then
        if date -d "$date" &>/dev/null; then
            return 0
        fi
    fi
    echo "วันที่ต้องอยู่ในรูปแบบ YYYY-MM-DD"
    return 1
}

# อ่าน input แบบมี validation
read_with_validation() {
    local prompt="$1"
    local validator="$2"
    local result
    
    while true; do
        read -rp "$prompt" result
        if $validator "$result"; then
            echo "$result"
            return 0
        fi
    done
}

# ตัวอย่างใช้งาน
echo "=== Form Input Example ==="

# อ่าน age ระหว่าง 1-120
read -rp "อายุ (1-120): " age
while ! validate_integer "$age" 1 120 &>/dev/null; do
    echo -e "\033[31m$(validate_integer "$age" 1 120 2>&1)\033[0m"
    read -rp "อายุ (1-120): " age
done
echo "Age: $age"

# อ่าน email
read -rp "Email: " email
while ! validate_email "$email" &>/dev/null; do
    echo -e "\033[31mEmail ไม่ถูกต้อง\033[0m"
    read -rp "Email: " email
done
echo "Email: $email"
```

### 82.3 Command Line Arguments

```bash
#!/bin/bash

# การรับ arguments พื้นฐาน
echo "Script: $0"
echo "Args:   $@"
echo "Count:  $#"
echo "Arg1:   $1"
echo "Arg2:   $2"

# ตรวจสอบว่ามี arguments
if [ $# -eq 0 ]; then
    echo "ไม่มี arguments"
    exit 1
fi

# การ parse arguments ด้วย getopts
usage() {
    echo "Usage: $0 [-h] [-v] [-o output] [-n name] input_file"
    echo ""
    echo "Options:"
    echo "  -h         แสดง help"
    echo "  -v         verbose mode"
    echo "  -o FILE    output file"
    echo "  -n NAME    ชื่อ"
    echo ""
    echo "Arguments:"
    echo "  input_file  ไฟล์ input"
}

# getopts สำหรับ short options
VERBOSE=false
OUTPUT=""
NAME=""

while getopts "hvo:n:" opt; do
    case "$opt" in
        h)  usage; exit 0 ;;
        v)  VERBOSE=true ;;
        o)  OUTPUT="$OPTARG" ;;
        n)  NAME="$OPTARG" ;;
        \?) echo "Invalid option: -$OPTARG" >&2; usage; exit 1 ;;
        :)  echo "Option -$OPTARG requires an argument" >&2; exit 1 ;;
    esac
done

# ข้ามไปยัง non-option arguments
shift $((OPTIND - 1))
INPUT_FILE="${1:-}"

echo "Verbose: $VERBOSE"
echo "Output:  ${OUTPUT:-'(none)'}"
echo "Name:    ${NAME:-'(none)'}"
echo "Input:   ${INPUT_FILE:-'(none)'}"

# Long options ด้วย manual parsing
parse_long_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            --help|-h)
                usage
                exit 0
                ;;
            --verbose|-v)
                VERBOSE=true
                shift
                ;;
            --output=*)
                OUTPUT="${1#*=}"
                shift
                ;;
            --output|-o)
                OUTPUT="$2"
                shift 2
                ;;
            --name=*)
                NAME="${1#*=}"
                shift
                ;;
            --name|-n)
                NAME="$2"
                shift 2
                ;;
            -*)
                echo "Unknown option: $1" >&2
                exit 1
                ;;
            *)
                INPUT_FILE="$1"
                shift
                ;;
        esac
    done
}
```

---

## ขั้นตอนที่ 83: File Descriptors

### 83.1 Standard File Descriptors

```bash
#!/bin/bash

# File descriptors:
# 0 = stdin  (standard input)
# 1 = stdout (standard output)
# 2 = stderr (standard error)

# เขียนไป stdout (FD 1)
echo "This goes to stdout"
echo "This too" >&1

# เขียนไป stderr (FD 2)
echo "Error message" >&2
echo "Warning!" 1>&2

# เปลี่ยน stdout ไปไฟล์
exec 1>output.txt
echo "This goes to file"
exec 1>/dev/tty  # กลับมา terminal

# ปิด file descriptor
exec 3>&-  # ปิด FD 3

# ตรวจสอบว่า output ถูก redirect หรือไม่
if [ -t 1 ]; then
    echo "stdout is a terminal"
else
    echo "stdout is redirected"
fi

# ใช้ tee สำหรับ split output
command | tee output.txt          # แสดงและบันทึก
command | tee -a output.txt       # append
command | tee file1 file2 file3   # บันทึกหลายไฟล์
```

### 83.2 Custom File Descriptors

```bash
#!/bin/bash

# เปิด custom file descriptor
exec 3>output.txt      # FD 3 เขียนไปไฟล์
exec 4<input.txt       # FD 4 อ่านจากไฟล์
exec 5<>bidirectional  # FD 5 อ่านและเขียน

# เขียนผ่าน FD 3
echo "Line 1" >&3
echo "Line 2" >&3

# อ่านผ่าน FD 4
read -r line <&4
echo "Read: $line"

# ปิด file descriptors
exec 3>&-
exec 4<&-
exec 5>&-

# ตัวอย่าง: Logging ไปหลายที่พร้อมกัน
setup_logging() {
    local logfile="$1"
    
    # เปิด FD 3 สำหรับ log file
    exec 3>>"$logfile"
    
    # Function log ที่เขียนทั้ง terminal และ file
    log() {
        local message="[$( date '+%Y-%m-%d %H:%M:%S')] $*"
        echo "$message"        # terminal
        echo "$message" >&3    # log file
    }
}

setup_logging "/tmp/myapp.log"
log "Application started"
log "Processing data..."
log "Done!"
exec 3>&-  # ปิด log file
```

---

## ขั้นตอนที่ 84: Redirection Techniques

### 84.1 Advanced Redirection

```bash
#!/bin/bash

# ==================
# Output Redirection
# ==================

# > ส่งไปไฟล์ (overwrite)
echo "content" > file.txt

# >> ต่อท้ายไฟล์ (append)
echo "more content" >> file.txt

# 2> ส่ง stderr ไปไฟล์
command 2> errors.txt

# &> ส่งทั้ง stdout และ stderr
command &> output.txt

# ส่ง stderr ไปหา stdout
command 2>&1

# ส่งไปหลายที่ด้วย tee
command | tee stdout.txt 2>/dev/null

# ==================
# Input Redirection
# ==================

# < อ่านจากไฟล์
sort < unsorted.txt

# << Here Document
cat << 'EOF'
บรรทัดที่ 1
บรรทัดที่ 2
EOF

# <<- Here Document แบบ strip leading tabs
cat <<- EOF
	บรรทัดที่ 1 (tab นำหน้าถูกลบ)
	บรรทัดที่ 2
	EOF

# <<< Here String
grep "pattern" <<< "some text with pattern"

# ==================
# Process Substitution
# ==================

# <() ทำให้ command output ดูเหมือนไฟล์
diff <(ls dir1) <(ls dir2)

# >() ส่ง output ไปยัง command
command > >(tee output.txt)

# ตัวอย่าง: เปรียบเทียบ output 2 คำสั่ง
diff <(sort file1.txt) <(sort file2.txt)

# ตัวอย่าง: อ่านจาก 2 sources พร้อมกัน
while read -r -u 3 line1 && read -r -u 4 line2; do
    echo "$line1 | $line2"
done 3< file1.txt 4< file2.txt
```

### 84.2 Here Documents

```bash
#!/bin/bash

# Basic Here Document
cat << EOF
Hello World
Current dir: $PWD
Date: $(date)
EOF

# Prevent variable expansion (single-quoted)
cat << 'EOF'
This will NOT expand: $HOME
This is literal: $(date)
EOF

# เขียนหลายบรรทัดลงไฟล์
cat > /tmp/config.conf << 'EOF'
# Application config
HOST=localhost
PORT=8080
DEBUG=false
EOF

# ใช้กับ heredoc ใน function
create_script() {
    local filename="$1"
    local name="$2"
    
    cat > "$filename" << SCRIPT
#!/bin/bash
# Auto-generated script
NAME="$name"
echo "Hello, \$NAME!"
SCRIPT
    
    chmod +x "$filename"
}

create_script "/tmp/test_script.sh" "World"
/tmp/test_script.sh

# MySQL/psql via heredoc
# mysql -u root << 'SQL'
# USE mydb;
# SELECT * FROM users;
# SQL

# SSH via heredoc
# ssh user@server << 'REMOTE'
# echo "Running on remote server"
# uptime
# REMOTE
```

---

## ขั้นตอนที่ 85: Logging System

### 85.1 Logging Framework

```bash
#!/bin/bash

# =============================================================================
# Logging Framework
# =============================================================================

# Log levels
declare -A LOG_LEVELS=(
    [TRACE]=0
    [DEBUG]=1
    [INFO]=2
    [WARNING]=3
    [ERROR]=4
    [CRITICAL]=5
)

# Config
LOG_LEVEL="${LOG_LEVEL:-INFO}"
LOG_FILE="${LOG_FILE:-}"
LOG_MAX_SIZE="${LOG_MAX_SIZE:-10485760}"  # 10MB
LOG_COLORS="${LOG_COLORS:-true}"

# Color codes
if [ "$LOG_COLORS" = true ] && [ -t 1 ]; then
    L_TRACE='\033[0;37m'
    L_DEBUG='\033[0;36m'
    L_INFO='\033[0;32m'
    L_WARNING='\033[1;33m'
    L_ERROR='\033[0;31m'
    L_CRITICAL='\033[1;31m'
    L_NC='\033[0m'
else
    L_TRACE='' L_DEBUG='' L_INFO='' L_WARNING='' L_ERROR='' L_CRITICAL='' L_NC=''
fi

# Rotate log if too large
rotate_log() {
    local logfile="$1"
    if [ -f "$logfile" ] && [ "$(stat -f%z "$logfile" 2>/dev/null || stat -c%s "$logfile")" -gt "$LOG_MAX_SIZE" ]; then
        mv "$logfile" "${logfile}.$(date +%Y%m%d_%H%M%S)"
        echo "Log rotated: ${logfile}"
    fi
}

# Main log function
_log() {
    local level="$1"
    local message="$2"
    local caller="${3:-}"
    
    # ตรวจสอบ log level
    local level_num="${LOG_LEVELS[$level]:-2}"
    local current_level_num="${LOG_LEVELS[$LOG_LEVEL]:-2}"
    
    [ "$level_num" -lt "$current_level_num" ] && return 0
    
    # สร้าง timestamp
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S.%3N' 2>/dev/null || date '+%Y-%m-%d %H:%M:%S')
    
    # เลือก color
    local color
    case "$level" in
        TRACE)    color="$L_TRACE" ;;
        DEBUG)    color="$L_DEBUG" ;;
        INFO)     color="$L_INFO" ;;
        WARNING)  color="$L_WARNING" ;;
        ERROR)    color="$L_ERROR" ;;
        CRITICAL) color="$L_CRITICAL" ;;
        *)        color="" ;;
    esac
    
    # สร้าง log entry
    local log_entry="[$timestamp] [$level] ${caller:+[$caller] }$message"
    local colored_entry="${color}${log_entry}${L_NC}"
    
    # เขียนไป stdout/stderr
    if [ "$level_num" -ge "${LOG_LEVELS[ERROR]}" ]; then
        echo -e "$colored_entry" >&2
    else
        echo -e "$colored_entry"
    fi
    
    # เขียนไป log file
    if [ -n "$LOG_FILE" ]; then
        rotate_log "$LOG_FILE"
        echo "$log_entry" >> "$LOG_FILE"
    fi
}

# Public log functions
log_trace()    { _log TRACE "$*" "${FUNCNAME[1]:-main}"; }
log_debug()    { _log DEBUG "$*" "${FUNCNAME[1]:-main}"; }
log_info()     { _log INFO "$*"; }
log_warning()  { _log WARNING "$*"; }
log_error()    { _log ERROR "$*"; }
log_critical() { _log CRITICAL "$*"; }

# ทดสอบ
LOG_LEVEL=DEBUG
LOG_FILE="/tmp/test.log"

log_trace    "Entering function"
log_debug    "Variable x = 42"
log_info     "Server started on port 8080"
log_warning  "Disk space below 20%"
log_error    "Failed to connect to database"
log_critical "System out of memory!"

echo ""
echo "Log file content:"
cat "$LOG_FILE" 2>/dev/null
```

---

## ขั้นตอนที่ 86: Interactive Menus

### 86.1 Simple Menu

```bash
#!/bin/bash

# Simple menu
show_menu() {
    echo ""
    echo "=== Main Menu ==="
    echo "1. Option One"
    echo "2. Option Two"
    echo "3. Option Three"
    echo "4. Exit"
    echo ""
    read -rp "Select option (1-4): " choice
    
    case "$choice" in
        1) echo "You chose Option One" ;;
        2) echo "You chose Option Two" ;;
        3) echo "You chose Option Three" ;;
        4) exit 0 ;;
        *) echo "Invalid choice" ;;
    esac
}

while true; do
    show_menu
done
```

### 86.2 Advanced Interactive Menu

```bash
#!/usr/bin/env bash
# =============================================================================
# Advanced Menu System
# =============================================================================

# Terminal setup
TERM_WIDTH=$(tput cols 2>/dev/null || echo 80)
CURSOR_UP='\033[A'
CURSOR_DOWN='\033[B'

# Colors
readonly C_SELECTED='\033[1;36m'   # Cyan bold
readonly C_NORMAL='\033[0;37m'     # Gray
readonly C_TITLE='\033[1;34m'      # Blue bold
readonly C_NC='\033[0m'

# Disable echo
stty -echo 2>/dev/null

# Enable echo on exit
trap 'stty echo 2>/dev/null; echo ""' EXIT INT TERM

# แสดง menu แบบ arrow key navigation
select_menu() {
    local title="$1"
    shift
    local options=("$@")
    local num_options="${#options[@]}"
    local selected=0
    
    # ซ่อน cursor
    tput civis 2>/dev/null
    
    # แสดง menu ครั้งแรก
    echo -e "${C_TITLE}$title${C_NC}"
    echo ""
    
    local render_menu() {
        for i in "${!options[@]}"; do
            if [ "$i" -eq "$selected" ]; then
                echo -e "  ${C_SELECTED}▶ ${options[$i]}${C_NC}"
            else
                echo -e "  ${C_NORMAL}  ${options[$i]}${C_NC}"
            fi
        done
    }
    
    render_menu
    
    # Input loop
    while true; do
        # อ่าน key
        local key
        IFS= read -r -s -n1 key 2>/dev/null
        
        # อ่าน escape sequence
        if [ "$key" = $'\x1b' ]; then
            read -r -s -n2 key2 2>/dev/null
            key="$key$key2"
        fi
        
        case "$key" in
            $'\x1b[A'|k)  # Up arrow or k
                ((selected--))
                [ "$selected" -lt 0 ] && selected=$((num_options - 1))
                ;;
            $'\x1b[B'|j)  # Down arrow or j
                ((selected++))
                [ "$selected" -ge "$num_options" ] && selected=0
                ;;
            $'\x0a'|$'\x0d'|' ')  # Enter or Space
                tput cnorm 2>/dev/null  # แสดง cursor
                echo ""
                echo "${options[$selected]}"
                return "$selected"
                ;;
            $'\x1b'|q)  # Escape or q
                tput cnorm 2>/dev/null
                return 255
                ;;
        esac
        
        # Redraw menu
        tput cuu "$num_options" 2>/dev/null || printf '\033[%dA' "$num_options"
        render_menu
    done
}

# ตัวอย่างการใช้งาน
stty echo 2>/dev/null

options=(
    "📦 Install packages"
    "⚙️  Configure settings"
    "📊 View reports"
    "🔧 Run diagnostics"
    "❌ Exit"
)

choice=$(select_menu "What would you like to do?" "${options[@]}")
selected=$?

if [ "$selected" -eq 255 ]; then
    echo "Cancelled"
elif [ "$selected" -eq $((${#options[@]} - 1)) ]; then
    echo "Goodbye!"
else
    echo "You selected: $choice"
fi
```

---

## ขั้นตอนที่ 87: Dialog Boxes

### 87.1 ใช้ dialog command

```bash
#!/bin/bash

# ตรวจสอบว่ามี dialog หรือ whiptail
if command -v dialog &>/dev/null; then
    DIALOG=dialog
elif command -v whiptail &>/dev/null; then
    DIALOG=whiptail
else
    echo "ไม่มี dialog หรือ whiptail"
    exit 1
fi

# Message box
$DIALOG --title "Message" \
    --msgbox "Installation complete!" \
    8 40

# Yes/No dialog
if $DIALOG --title "Confirm" \
    --yesno "Do you want to continue?" \
    8 40; then
    echo "User said YES"
else
    echo "User said NO"
fi

# Input dialog
name=$($DIALOG --title "Input" \
    --inputbox "Enter your name:" \
    8 40 "" \
    3>&1 1>&2 2>&3)
echo "Name: $name"

# Password dialog
password=$($DIALOG --title "Password" \
    --passwordbox "Enter password:" \
    8 40 \
    3>&1 1>&2 2>&3)
echo "Password entered (length: ${#password})"

# Menu dialog
choice=$($DIALOG --title "Menu" \
    --menu "Select option:" \
    15 50 5 \
    "1" "Option One" \
    "2" "Option Two" \
    "3" "Option Three" \
    3>&1 1>&2 2>&3)
echo "Choice: $choice"

# Checklist
selections=$($DIALOG --title "Select" \
    --checklist "Choose items:" \
    15 50 5 \
    "a" "Item A" off \
    "b" "Item B" on \
    "c" "Item C" off \
    3>&1 1>&2 2>&3)
echo "Selections: $selections"

# Progress bar
(
    for i in 0 20 40 60 80 100; do
        sleep 0.5
        echo "$i"
    done
) | $DIALOG --title "Progress" \
    --gauge "Processing..." \
    8 40 0

# File selection
file=$($DIALOG --title "File Select" \
    --fselect "$HOME/" \
    15 60 \
    3>&1 1>&2 2>&3)
echo "Selected file: $file"
```

---

## ขั้นตอนที่ 88: Workshop 04 - Interactive Configuration Script

### Workshop: System Configuration Wizard

```bash
#!/usr/bin/env bash
# =============================================================================
# Workshop 04: config_wizard.sh
# Interactive System Configuration Wizard
# =============================================================================

set -euo pipefail

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
NC='\033[0m'

# Config storage
declare -A CONFIG

# Helper functions
info()    { echo -e "${BLUE}ℹ${NC}  $*"; }
success() { echo -e "${GREEN}✓${NC}  $*"; }
warning() { echo -e "${YELLOW}⚠${NC}  $*"; }
error()   { echo -e "${RED}✗${NC}  $*" >&2; }

# Prompt with default
prompt_with_default() {
    local prompt="$1"
    local default="$2"
    local result
    
    read -rp "$prompt [${default}]: " result
    echo "${result:-$default}"
}

# Prompt yes/no
prompt_yesno() {
    local prompt="$1"
    local default="${2:-y}"
    local answer
    
    while true; do
        if [ "$default" = "y" ]; then
            read -rp "$prompt [Y/n]: " answer
            answer="${answer:-y}"
        else
            read -rp "$prompt [y/N]: " answer
            answer="${answer:-n}"
        fi
        
        case "${answer,,}" in
            y|yes) return 0 ;;
            n|no)  return 1 ;;
            *)     warning "กรุณาตอบ y หรือ n" ;;
        esac
    done
}

# Prompt from list
prompt_from_list() {
    local prompt="$1"
    shift
    local options=("$@")
    
    echo "$prompt"
    for i in "${!options[@]}"; do
        echo "  $((i+1)). ${options[$i]}"
    done
    
    local choice
    while true; do
        read -rp "เลือก (1-${#options[@]}): " choice
        if [[ "$choice" =~ ^[0-9]+$ ]] && \
           [ "$choice" -ge 1 ] && \
           [ "$choice" -le "${#options[@]}" ]; then
            echo "${options[$((choice-1))]}"
            return 0
        fi
        warning "กรุณาเลือก 1-${#options[@]}"
    done
}

# Validate IP
validate_ip() {
    local ip="$1"
    if [[ "$ip" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]]; then
        IFS='.' read -ra parts <<< "$ip"
        for part in "${parts[@]}"; do
            [ "$part" -gt 255 ] && return 1
        done
        return 0
    fi
    return 1
}

# Step 1: Welcome
wizard_welcome() {
    clear
    echo ""
    echo -e "${CYAN}╔══════════════════════════════════════╗${NC}"
    echo -e "${CYAN}║    System Configuration Wizard       ║${NC}"
    echo -e "${CYAN}╚══════════════════════════════════════╝${NC}"
    echo ""
    echo "ยินดีต้อนรับสู่ Configuration Wizard"
    echo "Wizard นี้จะช่วยตั้งค่าระบบของคุณ"
    echo ""
    
    if ! prompt_yesno "ต้องการดำเนินการต่อ?"; then
        echo "ออกจาก Wizard"
        exit 0
    fi
}

# Step 2: System Info
wizard_system_info() {
    echo ""
    echo -e "${CYAN}─── ข้อมูลระบบ ───${NC}"
    echo ""
    
    CONFIG[hostname]=$(prompt_with_default "Hostname" "$(hostname)")
    CONFIG[timezone]=$(prompt_with_default "Timezone" "Asia/Bangkok")
    CONFIG[locale]=$(prompt_with_default "Locale" "th_TH.UTF-8")
    
    success "บันทึกข้อมูลระบบแล้ว"
}

# Step 3: Network
wizard_network() {
    echo ""
    echo -e "${CYAN}─── การตั้งค่าเครือข่าย ───${NC}"
    echo ""
    
    local network_type
    network_type=$(prompt_from_list "ประเภทการเชื่อมต่อ:" "DHCP (อัตโนมัติ)" "Static IP (กำหนดเอง)")
    
    if [ "$network_type" = "Static IP (กำหนดเอง)" ]; then
        local ip
        while true; do
            read -rp "IP Address: " ip
            if validate_ip "$ip"; then
                break
            fi
            warning "IP Address ไม่ถูกต้อง"
        done
        
        CONFIG[ip_type]="static"
        CONFIG[ip_address]="$ip"
        CONFIG[netmask]=$(prompt_with_default "Netmask" "255.255.255.0")
        CONFIG[gateway]=$(prompt_with_default "Gateway" "$(echo "$ip" | cut -d. -f1-3).1")
        CONFIG[dns1]=$(prompt_with_default "DNS Primary" "8.8.8.8")
        CONFIG[dns2]=$(prompt_with_default "DNS Secondary" "8.8.4.4")
    else
        CONFIG[ip_type]="dhcp"
        info "ใช้ DHCP"
    fi
    
    success "บันทึกการตั้งค่าเครือข่ายแล้ว"
}

# Step 4: Services
wizard_services() {
    echo ""
    echo -e "${CYAN}─── บริการที่ต้องการ ───${NC}"
    echo ""
    
    local services=("SSH" "Nginx" "MySQL" "Redis" "Docker")
    declare -a selected_services=()
    
    for service in "${services[@]}"; do
        if prompt_yesno "ติดตั้ง $service?" "n"; then
            selected_services+=("$service")
        fi
    done
    
    CONFIG[services]="${selected_services[*]:-none}"
    success "บันทึกรายการ services แล้ว"
}

# Step 5: Users
wizard_users() {
    echo ""
    echo -e "${CYAN}─── การจัดการ Users ───${NC}"
    echo ""
    
    if prompt_yesno "ต้องการสร้าง admin user ใหม่?"; then
        local username
        read -rp "Username: " username
        CONFIG[admin_user]="$username"
        
        if prompt_yesno "ให้สิทธิ์ sudo?"; then
            CONFIG[admin_sudo]="yes"
        else
            CONFIG[admin_sudo]="no"
        fi
        
        success "บันทึกข้อมูล user แล้ว"
    fi
}

# Step 6: Summary and Confirm
wizard_summary() {
    echo ""
    echo -e "${CYAN}─── สรุปการตั้งค่า ───${NC}"
    echo ""
    
    printf "%-25s %s\n" "Hostname:"      "${CONFIG[hostname]:-N/A}"
    printf "%-25s %s\n" "Timezone:"      "${CONFIG[timezone]:-N/A}"
    printf "%-25s %s\n" "Locale:"        "${CONFIG[locale]:-N/A}"
    printf "%-25s %s\n" "Network Type:"  "${CONFIG[ip_type]:-N/A}"
    
    if [ "${CONFIG[ip_type]:-}" = "static" ]; then
        printf "%-25s %s\n" "IP Address:"    "${CONFIG[ip_address]:-N/A}"
        printf "%-25s %s\n" "Netmask:"       "${CONFIG[netmask]:-N/A}"
        printf "%-25s %s\n" "Gateway:"       "${CONFIG[gateway]:-N/A}"
    fi
    
    printf "%-25s %s\n" "Services:"      "${CONFIG[services]:-none}"
    printf "%-25s %s\n" "Admin User:"    "${CONFIG[admin_user]:-N/A}"
    echo ""
    
    if prompt_yesno "บันทึกการตั้งค่านี้?"; then
        wizard_save_config
    else
        warning "ยกเลิกการบันทึก"
    fi
}

# บันทึก config
wizard_save_config() {
    local config_file="/tmp/system_config_$(date +%Y%m%d_%H%M%S).conf"
    
    {
        echo "# System Configuration"
        echo "# Generated: $(date)"
        echo ""
        
        for key in "${!CONFIG[@]}"; do
            echo "${key^^}=\"${CONFIG[$key]}\""
        done
    } > "$config_file"
    
    success "บันทึก config ไปที่: $config_file"
    cat "$config_file"
}

# Main wizard
main() {
    wizard_welcome
    wizard_system_info
    wizard_network
    wizard_services
    wizard_users
    wizard_summary
    
    echo ""
    success "Configuration Wizard เสร็จสิ้น!"
}

main "$@"
```

---

## สรุป Part 04

### สิ่งที่เรียนรู้

1. ✅ echo vs printf และ formatted output
2. ✅ Colored output ด้วย ANSI codes
3. ✅ Progress bars และ spinners
4. ✅ read command และ input validation
5. ✅ Command line arguments (getopts)
6. ✅ File descriptors และ custom FDs
7. ✅ Advanced redirection
8. ✅ Here Documents
9. ✅ Logging framework
10. ✅ Interactive menus
11. ✅ Dialog boxes

### แบบฝึกหัด

**Easy:**
1. สร้าง script แสดงตารางข้อมูลแบบ formatted ด้วย printf
2. เขียนฟังก์ชัน log ที่เขียนทั้ง terminal และ log file

**Medium:**
3. สร้าง input form ที่มี validation สำหรับ user registration
4. สร้าง progress bar ที่แสดงขณะ copy ไฟล์

**Hard:**
5. สร้าง CLI tool ที่มี options หลายแบบด้วย getopts รวมกับ long options

---

**ต่อไป:** [Part 05 - เงื่อนไข if-else และ Operators](part-05.md)

*Part 04 จบแล้ว! พร้อมเรียน Part 05 →*
