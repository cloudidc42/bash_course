# Part 01: แนะนำ Bash Script และการติดตั้ง
## หลักสูตร Bash Script ตั้งแต่พื้นฐานสู่ระดับโลก

---

## ขั้นตอนที่ 1-20: รู้จัก Bash Script

### 1.1 Bash คืออะไร?

**Bash** ย่อมาจาก **Bourne Again Shell** เป็นโปรแกรมตัวแปลคำสั่ง (Command-line interpreter หรือ Shell) ที่พัฒนาโดย **Brian Fox** สำหรับ GNU Project ในปี 1989

```
ประวัติย่อ:
- 1971: Thompson Shell (sh) - Shell แรกของ Unix
- 1979: Bourne Shell (sh) - พัฒนาโดย Stephen Bourne
- 1989: Bash - Bourne Again Shell โดย Brian Fox
- ปัจจุบัน: Bash 5.x เป็น default shell บน Linux/macOS
```

### 1.2 ทำไมต้องเรียน Bash Script?

| ประโยชน์ | รายละเอียด |
|---------|-----------|
| **อัตโนมัติ** | ทำงานซ้ำๆ โดยอัตโนมัติ |
| **DevOps** | CI/CD, Docker, Kubernetes |
| **System Admin** | จัดการ server, backup, monitoring |
| **ประหยัดเวลา** | งาน 4 ชั่วโมง เหลือ 5 นาที |
| **ทุก Platform** | Linux, macOS, WSL บน Windows |
| **ฟรี** | Open Source ไม่มีค่าใช้จ่าย |

### 1.3 Bash ใช้ทำอะไรได้บ้าง?

```bash
# ตัวอย่างสิ่งที่ทำได้:
# 1. จัดการไฟล์และโฟลเดอร์
# 2. ประมวลผลข้อความขนาดใหญ่
# 3. Backup อัตโนมัติ
# 4. Monitor ระบบ
# 5. Deploy แอพพลิเคชัน
# 6. สร้าง Web Server
# 7. จัดการ Database
# 8. ทำงานกับ API
# 9. สร้าง CI/CD Pipeline
# 10. สร้าง CLI Tools
```

---

## ขั้นตอนที่ 2: การติดตั้งและตรวจสอบ

### 2.1 ตรวจสอบ Bash ที่มีอยู่

```bash
# ตรวจสอบ Bash version
bash --version

# ผลลัพธ์ที่ควรได้:
# GNU bash, version 5.2.15(1)-release (x86_64-pc-linux-gnu)
# Copyright (C) 2022 Free Software Foundation, Inc.

# ตรวจสอบตำแหน่ง Bash
which bash
# /bin/bash  หรือ  /usr/bin/bash

# ตรวจสอบ shell ปัจจุบัน
echo $SHELL
# /bin/bash

# ดู shell ทั้งหมดที่ติดตั้ง
cat /etc/shells
# /bin/sh
# /bin/bash
# /usr/bin/bash
# /bin/rbash
# /usr/bin/rbash
# /usr/bin/sh
# /bin/zsh
# /usr/bin/zsh
```

### 2.2 การติดตั้ง Bash บน Linux (Ubuntu/Debian)

```bash
# อัปเดต package list
sudo apt update

# ติดตั้ง Bash (ถ้ายังไม่มี)
sudo apt install bash

# อัปเกรดเป็นเวอร์ชันล่าสุด
sudo apt upgrade bash

# ติดตั้งเครื่องมือเสริม
sudo apt install -y \
    bash-completion \
    shellcheck \
    bats \
    curl \
    wget \
    jq \
    git \
    vim \
    nano
```

### 2.3 การติดตั้งบน macOS

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Bash เวอร์ชันใหม่
brew install bash

# ติดตั้งเครื่องมือเสริม
brew install \
    bash-completion@2 \
    shellcheck \
    bats-core \
    jq \
    wget

# เพิ่ม Bash ใหม่เข้า allowed shells
echo "/opt/homebrew/bin/bash" | sudo tee -a /etc/shells

# เปลี่ยน default shell
chsh -s /opt/homebrew/bin/bash
```

### 2.4 การติดตั้งบน Windows (WSL)

```powershell
# เปิด PowerShell as Administrator แล้วรัน:
wsl --install

# หรือติดตั้ง Ubuntu จาก Microsoft Store
# แล้วเปิด Ubuntu terminal

# ตรวจสอบ version หลังติดตั้ง
bash --version
```

---

## ขั้นตอนที่ 3: Text Editor และ IDE

### 3.1 เลือก Editor

```bash
# Visual Studio Code (แนะนำสำหรับมือใหม่)
# ติดตั้ง extension: Bash IDE, ShellCheck

# Vim (สำหรับ server work)
vim myscript.sh

# Nano (ง่ายที่สุด)
nano myscript.sh

# Emacs
emacs myscript.sh

# Neovim (ทันสมัย)
nvim myscript.sh
```

### 3.2 ติดตั้ง ShellCheck (ตรวจสอบโค้ด)

```bash
# Ubuntu/Debian
sudo apt install shellcheck

# macOS
brew install shellcheck

# ใช้งาน ShellCheck
shellcheck myscript.sh

# ผลลัพธ์ตัวอย่าง:
# In myscript.sh line 3:
# echo $name
#      ^---^ SC2086 (info): Double quote to prevent globbing and word splitting.
```

---

## ขั้นตอนที่ 4: Bash Script แรก - Hello World

### 4.1 สร้างไฟล์ Script แรก

```bash
# สร้างไฟล์ใหม่
nano hello.sh

# หรือใช้ cat
cat > hello.sh << 'EOF'
#!/bin/bash
# สคริปต์แรกของฉัน
# วันที่: 2026-09-30

echo "สวัสดี โลก!"
echo "Hello, World!"
echo "ยินดีต้อนรับสู่หลักสูตร Bash Script"
EOF
```

### 4.2 ทำความเข้าใจ Shebang Line

```bash
#!/bin/bash
# ^^ นี่คือ Shebang line (บรรทัดแรกของ script)
# !/bin/bash บอกว่าให้ใช้ /bin/bash ในการรัน script นี้

# Shebang แบบต่างๆ:
#!/bin/bash          # ใช้ bash
#!/bin/sh            # ใช้ sh (POSIX shell)
#!/usr/bin/env bash  # ใช้ bash ที่อยู่ใน PATH (แนะนำ)
#!/usr/bin/env python3  # สำหรับ Python script
#!/usr/bin/env perl    # สำหรับ Perl script

# ทำไมต้อง #!/usr/bin/env bash ?
# เพราะ bash อาจอยู่ต่างตำแหน่งในแต่ละระบบ
# /usr/bin/env จะหา bash ใน PATH ให้อัตโนมัติ
```

### 4.3 ทำให้ Script รันได้

```bash
# ดูสิทธิ์ปัจจุบัน
ls -la hello.sh
# -rw-r--r-- 1 user user 89 Sep 30 12:00 hello.sh

# ให้สิทธิ์ execute
chmod +x hello.sh

# ดูสิทธิ์หลัง chmod
ls -la hello.sh
# -rwxr-xr-x 1 user user 89 Sep 30 12:00 hello.sh

# รัน script
./hello.sh

# หรือรันโดยเรียก bash โดยตรง (ไม่ต้อง chmod)
bash hello.sh

# หรือ
sh hello.sh
```

### 4.4 ผลลัพธ์ที่คาดหวัง

```
สวัสดี โลก!
Hello, World!
ยินดีต้อนรับสู่หลักสูตร Bash Script
```

---

## ขั้นตอนที่ 5: ทำความเข้าใจ Terminal

### 5.1 โครงสร้าง Prompt

```bash
user@hostname:~/Documents$ command

# user     = ชื่อผู้ใช้
# hostname = ชื่อเครื่อง
# ~        = home directory (/home/user)
# /Documents = ตำแหน่งปัจจุบัน
# $        = ผู้ใช้ทั่วไป
# #        = root (superuser)
```

### 5.2 คำสั่ง Navigation พื้นฐาน

```bash
# ดูตำแหน่งปัจจุบัน
pwd
# /home/user/Documents

# แสดงไฟล์และโฟลเดอร์
ls          # แสดงแบบปกติ
ls -l       # แสดงแบบละเอียด (long format)
ls -la      # แสดงทั้งไฟล์ที่ซ่อน
ls -lh      # แสดงขนาดไฟล์แบบอ่านง่าย
ls -lt      # เรียงตามเวลาแก้ไข

# เปลี่ยน directory
cd /home        # ไปที่ /home
cd ~            # ไปที่ home directory
cd ..           # ขึ้นไปหนึ่งระดับ
cd -            # กลับไป directory ก่อนหน้า
cd Documents    # เข้า Documents (relative path)

# สร้าง directory
mkdir mydir
mkdir -p path/to/deep/dir    # สร้างทั้ง path

# ลบ directory
rmdir emptydir               # ลบ directory ว่าง
rm -rf mydir                 # ลบทั้ง directory และข้างใน (ระวัง!)
```

### 5.3 Keyboard Shortcuts ใน Terminal

```bash
# Ctrl+C    = หยุด process ที่กำลังทำงาน
# Ctrl+Z    = หยุด process ชั่วคราว
# Ctrl+D    = ออกจาก shell / EOF
# Ctrl+L    = ล้างหน้าจอ (เหมือน clear)
# Ctrl+A    = ไปต้นบรรทัด
# Ctrl+E    = ไปท้ายบรรทัด
# Ctrl+U    = ลบจากตำแหน่งปัจจุบันไปต้นบรรทัด
# Ctrl+K    = ลบจากตำแหน่งปัจจุบันไปท้ายบรรทัด
# Ctrl+W    = ลบคำก่อนหน้า
# Ctrl+R    = ค้นหาประวัติคำสั่ง
# Tab       = Auto-complete
# ↑↓        = เลื่อนดูประวัติคำสั่ง
# !!        = รันคำสั่งก่อนหน้า
# !$        = argument สุดท้ายของคำสั่งก่อนหน้า
# !string   = รันคำสั่งล่าสุดที่ขึ้นต้นด้วย string
```

---

## ขั้นตอนที่ 6: โครงสร้างพื้นฐานของ Bash Script

### 6.1 Template มาตรฐาน

```bash
#!/usr/bin/env bash
# =============================================================================
# Script Name: template.sh
# Description: คำอธิบายสิ่งที่ script ทำ
# Author:      ชื่อผู้เขียน
# Date:        2026-09-30
# Version:     1.0.0
# Usage:       ./template.sh [options] [arguments]
# =============================================================================

# --- ตั้งค่า Strict Mode ---
set -euo pipefail
# set -e  = หยุดทันทีเมื่อมี error
# set -u  = error เมื่อใช้ตัวแปรที่ไม่ได้กำหนด
# set -o pipefail = error เมื่อ pipe ล้มเหลว

# --- ค่าคงที่ ---
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly SCRIPT_VERSION="1.0.0"

# --- ตัวแปร Global ---
LOG_LEVEL="INFO"
DRY_RUN=false

# --- Functions ---
log() {
    local level="$1"
    local message="$2"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] [$level] $message"
}

usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS]

Options:
    -h, --help      แสดงวิธีใช้
    -v, --verbose   แสดงรายละเอียดเพิ่มเติม
    -n, --dry-run   ทดสอบโดยไม่ทำจริง

Examples:
    $SCRIPT_NAME
    $SCRIPT_NAME --verbose
    $SCRIPT_NAME --dry-run
EOF
}

main() {
    log "INFO" "เริ่มต้น $SCRIPT_NAME v$SCRIPT_VERSION"
    
    # โค้ดหลักอยู่ที่นี่
    echo "Hello from main!"
    
    log "INFO" "เสร็จสิ้น"
}

# --- Entry Point ---
main "$@"
```

### 6.2 Comments (คำอธิบาย)

```bash
#!/bin/bash

# นี่คือ single-line comment (บรรทัดเดียว)

# Comment หลายบรรทัด ใช้ # นำหน้าทุกบรรทัด
# บรรทัดที่ 1
# บรรทัดที่ 2
# บรรทัดที่ 3

echo "Hello" # comment ท้ายคำสั่งก็ได้

# Multi-line comment ด้วย : << 'COMMENT'
: << 'COMMENT'
นี่คือ multi-line comment
ที่ใช้เทคนิค heredoc
บรรทัดนี้จะถูกข้ามทั้งหมด
COMMENT

echo "หลัง comment"
```

### 6.3 Exit Codes

```bash
#!/bin/bash

# Exit codes มาตรฐาน:
# 0   = สำเร็จ
# 1   = Error ทั่วไป
# 2   = Misuse ของ shell commands
# 126 = ไม่มีสิทธิ์รัน
# 127 = command ไม่พบ
# 128 = exit argument ไม่ถูกต้อง
# 130 = script ถูก Ctrl+C
# 255 = Exit status นอกช่วง

# ออกด้วย exit code
exit 0    # สำเร็จ
exit 1    # ล้มเหลว

# ตรวจสอบ exit code ของคำสั่งก่อนหน้า
ls /tmp
echo "Exit code: $?"    # $? เก็บ exit code ล่าสุด

# ตัวอย่างการใช้
if ls /nonexistent 2>/dev/null; then
    echo "มีไฟล์"
else
    echo "ไม่มีไฟล์ (exit code: $?)"
fi
```

---

## ขั้นตอนที่ 7: Workshop 01 - สร้าง Script แรก

### Workshop: สร้าง System Info Script

```bash
#!/usr/bin/env bash
# =============================================================================
# Workshop 01: system_info.sh
# ดูข้อมูลระบบพื้นฐาน
# =============================================================================

echo "=================================="
echo "    ข้อมูลระบบ (System Info)"
echo "=================================="
echo ""

echo "📅 วันที่และเวลา:"
date '+%Y-%m-%d %H:%M:%S %Z'
echo ""

echo "💻 ชื่อเครื่อง (Hostname):"
hostname
echo ""

echo "👤 ผู้ใช้ปัจจุบัน:"
echo "  Username: $(whoami)"
echo "  User ID:  $(id -u)"
echo "  Groups:   $(groups)"
echo ""

echo "🐧 ระบบปฏิบัติการ:"
if [ -f /etc/os-release ]; then
    source /etc/os-release
    echo "  OS:      $PRETTY_NAME"
fi
echo "  Kernel:  $(uname -r)"
echo "  Arch:    $(uname -m)"
echo ""

echo "⚙️  CPU:"
if command -v lscpu &>/dev/null; then
    echo "  $(lscpu | grep 'Model name' | sed 's/Model name:\s*//')"
    echo "  Cores: $(nproc)"
fi
echo ""

echo "💾 Memory:"
if command -v free &>/dev/null; then
    free -h | grep -E 'Mem:|Swap:'
fi
echo ""

echo "💿 Disk:"
df -h / | tail -1 | awk '{print "  Total:", $2, "| Used:", $3, "| Free:", $4, "| Use%:", $5}'
echo ""

echo "🌐 Network:"
ip addr show 2>/dev/null | grep 'inet ' | grep -v '127.0.0.1' | awk '{print "  IP:", $2}' || \
ifconfig 2>/dev/null | grep 'inet ' | grep -v '127.0.0.1' | awk '{print "  IP:", $2}'
echo ""

echo "⚡ Uptime:"
uptime -p 2>/dev/null || uptime
echo ""

echo "=================================="
echo "    ตรวจสอบเสร็จสิ้น"
echo "=================================="
```

บันทึกไฟล์นี้เป็น `system_info.sh` แล้วรัน:

```bash
chmod +x system_info.sh
./system_info.sh
```

ผลลัพธ์ตัวอย่าง:
```
==================================
    ข้อมูลระบบ (System Info)
==================================

📅 วันที่และเวลา:
2026-09-30 12:00:00 UTC

💻 ชื่อเครื่อง (Hostname):
myserver

👤 ผู้ใช้ปัจจุบัน:
  Username: ubuntu
  User ID:  1000
  Groups:   ubuntu adm sudo

🐧 ระบบปฏิบัติการ:
  OS:      Ubuntu 24.04.1 LTS
  Kernel:  6.8.0-45-generic
  Arch:    x86_64
...
```

---

## ขั้นตอนที่ 8: การ Debug Script

### 8.1 วิธี Debug พื้นฐาน

```bash
#!/bin/bash

# วิธี 1: ใช้ -x flag (trace mode)
# รัน: bash -x myscript.sh

# วิธี 2: เพิ่ม set -x ใน script
set -x    # เปิด debug mode
echo "Hello"
set +x    # ปิด debug mode

# วิธี 3: ใช้ -x เฉพาะส่วน
{
    set -x
    # โค้ดที่ต้องการ debug
    name="World"
    echo "Hello $name"
    set +x
} 

# วิธี 4: ใช้ -v (verbose - แสดงทุกบรรทัดก่อนรัน)
# bash -v myscript.sh

# วิธี 5: ใช้ -xv รวมกัน
# bash -xv myscript.sh
```

### 8.2 Debugging ด้วย echo

```bash
#!/bin/bash

# เพิ่ม debug messages
DEBUG=true

debug() {
    if [ "$DEBUG" = true ]; then
        echo "[DEBUG] $*" >&2
    fi
}

name="สวัสดี"
debug "ตัวแปร name = $name"

result=$(echo "$name" | wc -c)
debug "ความยาว = $result"

echo "ผลลัพธ์: $result"
```

### 8.3 ใช้ ShellCheck

```bash
# ตรวจสอบ script ด้วย ShellCheck
shellcheck myscript.sh

# ตรวจสอบแบบละเอียด
shellcheck --severity=style myscript.sh

# ผลลัพธ์ตัวอย่าง:
# In myscript.sh line 5:
# for f in $(ls *.txt); do
#           ^---------^ SC2045 (error): Iterating over ls output is fragile.
# Did you mean:
# for f in *.txt; do
```

---

## ขั้นตอนที่ 9: Bash vs Other Shells

### 9.1 เปรียบเทียบ Shells

```bash
# Bash (Bourne Again Shell) - ที่นิยมมากที่สุด
#!/bin/bash
array=(1 2 3)           # Bash arrays
echo "${array[@]}"

# Zsh (Z Shell) - คล้าย Bash แต่มีฟีเจอร์เพิ่ม
#!/bin/zsh
array=(1 2 3)
echo "${array[@]}"

# Fish (Friendly Interactive Shell) - ง่ายต่อการใช้
#!/usr/bin/fish
set array 1 2 3
echo $array

# POSIX sh - compatibility สูงสุด
#!/bin/sh
# ไม่มี arrays ใน POSIX sh
```

### 9.2 เมื่อไหร่ใช้อะไร?

```bash
# ใช้ #!/usr/bin/env bash เมื่อ:
# - ต้องการ Bash features (arrays, [[ ]], etc.)
# - รู้แน่ว่าระบบมี bash

# ใช้ #!/bin/sh เมื่อ:
# - ต้องการ portability สูง
# - จะรันบน embedded systems
# - ต้องการ POSIX compliance

# แนะนำสำหรับหลักสูตรนี้: #!/usr/bin/env bash
```

---

## ขั้นตอนที่ 10: สรุปและแบบฝึกหัด

### สิ่งที่เรียนรู้ใน Part 01

1. ✅ Bash คืออะไรและทำไมต้องเรียน
2. ✅ วิธีติดตั้งและตรวจสอบ Bash
3. ✅ Text editors สำหรับเขียน script
4. ✅ สร้างและรัน Bash script แรก
5. ✅ Shebang line และ comments
6. ✅ Exit codes
7. ✅ โครงสร้างพื้นฐานของ script
8. ✅ การ Debug
9. ✅ เปรียบเทียบ Shells ต่างๆ

### แบบฝึกหัด

**Easy (ง่าย):**
1. สร้าง script ที่แสดง "Hello, [ชื่อของคุณ]!"
2. สร้าง script ที่แสดงวันที่และเวลาปัจจุบัน
3. ตรวจสอบว่าเครื่องของคุณมี Bash version อะไร

**Medium (ปานกลาง):**
4. ปรับปรุง system_info.sh ให้แสดงข้อมูลเพิ่มเติม
5. สร้าง script ที่สร้าง directory structure สำหรับ project ใหม่

**Hard (ยาก):**
6. สร้าง script ที่รับชื่อโฟลเดอร์เป็น argument แล้วแสดงสรุปไฟล์ในนั้น

### เฉลยแบบฝึกหัดข้อ 1-3

```bash
# ข้อ 1: Hello with name
#!/bin/bash
YOUR_NAME="สมชาย"
echo "Hello, $YOUR_NAME!"

# ข้อ 2: Date and time
#!/bin/bash
echo "วันที่: $(date '+%d/%m/%Y')"
echo "เวลา:   $(date '+%H:%M:%S')"

# ข้อ 3: Bash version
#!/bin/bash
echo "Bash version: $BASH_VERSION"
# หรือ
bash --version | head -1
```

---

## คำสั่งที่ควรจำ (Part 01)

```bash
bash --version      # ตรวจสอบ version
which bash          # หา path ของ bash
chmod +x file.sh    # ให้สิทธิ์รัน
./script.sh         # รัน script
bash script.sh      # รัน script ด้วย bash
bash -x script.sh   # รันพร้อม debug
shellcheck file.sh  # ตรวจสอบ syntax
echo $?             # ดู exit code ล่าสุด
```

---

**ต่อไป:** [Part 02 - คำสั่งพื้นฐาน Linux/Bash](part-02.md)

*Part 01 จบแล้ว! พร้อมเรียน Part 02 →*
