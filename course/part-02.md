# Part 02: คำสั่งพื้นฐาน Linux/Bash
## หลักสูตร Bash Script - ขั้นตอนที่ 21-50

---

## ขั้นตอนที่ 21: คำสั่งจัดการไฟล์และโฟลเดอร์

### 21.1 ls - แสดงรายการไฟล์

```bash
#!/bin/bash

# ls - list directory contents
ls                    # แสดงไฟล์ในปัจจุบัน
ls /tmp               # แสดงไฟล์ใน /tmp
ls -l                 # แสดงแบบ long format
ls -a                 # แสดงไฟล์ที่ซ่อน (. prefix)
ls -la                # รวมกัน: long + hidden
ls -lh                # ขนาดไฟล์แบบ human-readable
ls -lt                # เรียงตามเวลาแก้ไข (ใหม่สุดก่อน)
ls -ltr               # เรียงตามเวลาแก้ไข (เก่าสุดก่อน)
ls -lS                # เรียงตามขนาดไฟล์ (ใหญ่สุดก่อน)
ls -R                 # แสดงแบบ recursive
ls *.txt              # แสดงเฉพาะไฟล์ .txt
ls -d */              # แสดงเฉพาะ directory

# ตัวอย่าง output ของ ls -lh:
# total 48K
# drwxr-xr-x 2 user user 4.0K Sep 30 12:00 Documents
# -rw-r--r-- 1 user user  15K Sep 30 11:30 report.txt
# -rwxr-xr-x 1 user user 1.2K Sep 30 10:00 script.sh

# ความหมายของ permissions: -rwxr-xr-x
# -    = ไฟล์ธรรมดา (d=directory, l=symlink)
# rwx  = owner: read, write, execute
# r-x  = group: read, execute
# r-x  = others: read, execute
```

### 21.2 mkdir - สร้างโฟลเดอร์

```bash
# สร้างโฟลเดอร์เดียว
mkdir mydir

# สร้างหลายโฟลเดอร์
mkdir dir1 dir2 dir3

# สร้าง nested directories (-p)
mkdir -p projects/web/src/components

# สร้างพร้อมกำหนด permissions
mkdir -m 755 secure_dir

# ตัวอย่างการสร้าง project structure
setup_project() {
    local project_name="$1"
    
    mkdir -p "$project_name"/{
        src/{main,utils,tests},
        docs,
        config,
        scripts,
        .github/workflows
    }
    
    echo "Project '$project_name' structure created!"
    tree "$project_name" 2>/dev/null || find "$project_name" -type d | sort
}

setup_project "myapp"
```

### 21.3 cp - คัดลอกไฟล์

```bash
# คัดลอกไฟล์
cp source.txt destination.txt

# คัดลอกไปยัง directory
cp file.txt /tmp/

# คัดลอกหลายไฟล์
cp file1.txt file2.txt /tmp/

# คัดลอก directory ทั้งหมด (-r)
cp -r mydir /tmp/

# คัดลอกพร้อมรักษา attributes (-p)
cp -p original.txt backup.txt

# คัดลอกแบบ verbose (-v)
cp -v file.txt /backup/

# คัดลอกเฉพาะไฟล์ใหม่กว่า (-u)
cp -u source.txt dest.txt

# คัดลอกพร้อม progress (ใช้ rsync แทน)
rsync -ah --progress source/ destination/

# Interactive copy (ถามก่อน overwrite)
cp -i file.txt existing.txt
```

### 21.4 mv - ย้ายหรือเปลี่ยนชื่อไฟล์

```bash
# เปลี่ยนชื่อไฟล์
mv oldname.txt newname.txt

# ย้ายไฟล์ไปยัง directory
mv file.txt /tmp/

# ย้ายหลายไฟล์
mv file1.txt file2.txt /tmp/

# ย้าย directory
mv mydir /tmp/

# Move แบบ verbose
mv -v file.txt /backup/

# Move แบบ interactive (ถามก่อน overwrite)
mv -i source.txt dest.txt

# Rename ทุกไฟล์ .txt เป็น .bak
for f in *.txt; do
    mv "$f" "${f%.txt}.bak"
done
```

### 21.5 rm - ลบไฟล์

```bash
# ลบไฟล์
rm file.txt

# ลบหลายไฟล์
rm file1.txt file2.txt file3.txt

# ลบ directory ที่ว่าง
rmdir emptydir

# ลบ directory และทุกอย่างข้างใน (ระวัง!)
rm -rf mydir

# Interactive deletion (ถามก่อนลบ)
rm -i file.txt

# ลบแบบ verbose
rm -v file.txt

# ลบไฟล์ที่ขึ้นต้นด้วย -
rm -- -strange-filename.txt
rm ./-strange-filename.txt

# ลบไฟล์ .log ที่เก่ากว่า 30 วัน
find /var/log -name "*.log" -mtime +30 -delete

# Function ลบอย่างปลอดภัย
safe_remove() {
    local file="$1"
    if [ -e "$file" ]; then
        trash_dir="$HOME/.trash"
        mkdir -p "$trash_dir"
        mv "$file" "$trash_dir/"
        echo "ย้าย '$file' ไป trash แล้ว"
    else
        echo "ไม่พบไฟล์: $file"
        return 1
    fi
}
```

---

## ขั้นตอนที่ 22: คำสั่งดูและแก้ไขเนื้อหาไฟล์

### 22.1 cat - แสดงเนื้อหาไฟล์

```bash
# แสดงเนื้อหาไฟล์
cat file.txt

# แสดงพร้อม line numbers
cat -n file.txt

# แสดงพร้อม line numbers (เฉพาะบรรทัดที่ไม่ว่าง)
cat -b file.txt

# แสดง non-printing characters
cat -A file.txt

# รวมหลายไฟล์
cat file1.txt file2.txt file3.txt

# สร้างไฟล์ด้วย cat
cat > newfile.txt << 'EOF'
บรรทัดที่ 1
บรรทัดที่ 2
บรรทัดที่ 3
EOF

# ต่อท้ายไฟล์ (append)
cat >> existing.txt << 'EOF'
เพิ่มบรรทัดนี้
EOF
```

### 22.2 head และ tail - ดูส่วนต้นและส่วนท้าย

```bash
# head - ดู 10 บรรทัดแรก (default)
head file.txt

# ดู N บรรทัดแรก
head -n 20 file.txt
head -20 file.txt    # shorthand

# tail - ดู 10 บรรทัดสุดท้าย (default)
tail file.txt

# ดู N บรรทัดสุดท้าย
tail -n 20 file.txt

# ดูแบบ real-time (follow)
tail -f /var/log/syslog

# follow พร้อม retry เมื่อไฟล์ถูกหมุน (rotate)
tail -F /var/log/syslog

# ดูตั้งแต่บรรทัดที่ N (ใช้ + นำหน้า)
tail -n +5 file.txt    # ดูตั้งแต่บรรทัดที่ 5

# ตัวอย่าง: ดู log แบบ real-time พร้อม filter
tail -f /var/log/nginx/access.log | grep "ERROR"
```

### 22.3 less และ more - ดูไฟล์แบบ page

```bash
# less - ดูแบบ interactive (แนะนำ)
less largefile.txt

# ใน less:
# Space    = หน้าถัดไป
# b        = หน้าก่อนหน้า
# /text    = ค้นหา text
# n        = ค้นหาถัดไป
# N        = ค้นหาก่อนหน้า
# g        = ไปต้นไฟล์
# G        = ไปท้ายไฟล์
# q        = ออก
# h        = help

# less พร้อม line numbers
less -N file.txt

# more - ดูแบบเก่า (ไปข้างหน้าอย่างเดียว)
more file.txt
```

### 22.4 grep - ค้นหาข้อความ

```bash
# ค้นหาคำในไฟล์
grep "error" logfile.txt

# ค้นหาแบบ case-insensitive
grep -i "error" logfile.txt

# แสดง line numbers
grep -n "error" logfile.txt

# ค้นหาแบบ recursive ในทุกไฟล์
grep -r "error" /var/log/

# ค้นหาด้วย Regular Expression
grep -E "error|warning|critical" logfile.txt

# แสดงเฉพาะชื่อไฟล์ที่มี match
grep -l "error" /var/log/*.log

# แสดงบรรทัดที่ NOT match (-v)
grep -v "^#" config.txt    # ซ่อน comment lines

# นับจำนวน matches
grep -c "error" logfile.txt

# แสดง N บรรทัดก่อนและหลัง match
grep -B 2 -A 2 "error" logfile.txt

# แสดง N บรรทัดรอบๆ match
grep -C 3 "error" logfile.txt

# ค้นหา word ทั้งคำ (-w)
grep -w "log" file.txt    # จะได้ "log" แต่ไม่ได้ "logging"

# ตัวอย่าง pipeline
cat /var/log/auth.log | grep "Failed" | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" | sort | uniq -c | sort -rn | head -10
```

---

## ขั้นตอนที่ 23: คำสั่งค้นหาไฟล์

### 23.1 find - ค้นหาไฟล์

```bash
# ค้นหาไฟล์ตามชื่อ
find /home -name "*.txt"

# ค้นหาแบบ case-insensitive
find /home -iname "*.TXT"

# ค้นหาเฉพาะไฟล์ (ไม่รวม directory)
find /home -type f -name "*.sh"

# ค้นหาเฉพาะ directory
find /home -type d -name "project*"

# ค้นหาตามขนาดไฟล์
find /var -size +100M              # ใหญ่กว่า 100MB
find /var -size -1k                # เล็กกว่า 1KB
find /var -size +1M -size -10M    # ระหว่าง 1MB ถึง 10MB

# ค้นหาตามเวลาแก้ไข
find /home -mtime -7               # แก้ไขใน 7 วันที่ผ่านมา
find /home -mtime +30              # แก้ไขมากกว่า 30 วันที่ผ่านมา
find /home -newer reference.txt    # ใหม่กว่า reference.txt

# ค้นหาตาม permissions
find /home -perm 755
find /home -perm -u+x             # มี execute permission สำหรับ owner

# ค้นหาตาม owner
find /home -user john
find /var -group www-data

# ค้นหาและรันคำสั่ง (-exec)
find /tmp -name "*.tmp" -exec rm {} \;
find /home -name "*.sh" -exec chmod +x {} \;

# ค้นหาและรันคำสั่งแบบ batch (-exec ... +)
find /tmp -name "*.log" -exec gzip {} +

# ค้นหาและแสดงผลแบบ detailed
find /home -name "*.txt" -ls

# ค้นหาไฟล์ว่าง
find /home -empty -type f

# ค้นหาและลบไฟล์เก่า
find /var/log -name "*.log" -mtime +30 -delete

# ตัวอย่าง: หาไฟล์ที่ใหญ่ที่สุด 10 ไฟล์
find / -type f -printf '%s %p\n' 2>/dev/null | sort -rn | head -10
```

### 23.2 locate - ค้นหาจาก database

```bash
# ค้นหาไฟล์ (เร็วกว่า find)
locate "*.txt"

# อัปเดต database ก่อนใช้
sudo updatedb

# ค้นหาแบบ case-insensitive
locate -i "README"

# จำกัดจำนวนผลลัพธ์
locate -n 10 "*.sh"

# ค้นหา exact match
locate -b "\profile"
```

### 23.3 which และ whereis

```bash
# หา path ของ command
which bash
# /bin/bash

which python3
# /usr/bin/python3

# หาทุก path ที่มี command
which -a python
# /usr/bin/python
# /usr/local/bin/python

# whereis - หา binary, source, man pages
whereis bash
# bash: /bin/bash /etc/bash.bashrc /usr/share/man/man1/bash.1.gz

# type - ดูว่า command เป็นอะไร
type ls
# ls is aliased to `ls --color=auto'

type cd
# cd is a shell builtin

type grep
# grep is /bin/grep
```

---

## ขั้นตอนที่ 24: คำสั่ง Text Processing

### 24.1 wc - นับบรรทัด คำ ตัวอักษร

```bash
# นับบรรทัด (-l), คำ (-w), ตัวอักษร (-c)
wc file.txt
# 10  50  250 file.txt (lines words chars)

# นับเฉพาะบรรทัด
wc -l file.txt

# นับเฉพาะคำ
wc -w file.txt

# นับเฉพาะ bytes
wc -c file.txt

# นับเฉพาะตัวอักษร (multi-byte aware)
wc -m file.txt

# นับหลายไฟล์
wc -l *.txt

# ใน pipeline
cat file.txt | wc -l
grep "error" logfile.txt | wc -l
```

### 24.2 sort - เรียงข้อมูล

```bash
# เรียงแบบปกติ
sort file.txt

# เรียงแบบกลับ
sort -r file.txt

# เรียงแบบตัวเลข
sort -n numbers.txt

# เรียงแบบกลับ + ตัวเลข
sort -rn numbers.txt

# เรียงตาม column (ใช้ -k)
sort -k2 data.txt           # เรียงตาม column 2
sort -k2 -k3 data.txt       # เรียงตาม column 2 แล้ว 3
sort -k2n data.txt          # เรียงตาม column 2 แบบตัวเลข

# กำหนด delimiter
sort -t: -k3 -n /etc/passwd    # เรียงตาม UID (field 3, delimit by :)

# เรียงแบบไม่สนใจ case
sort -f file.txt

# ลบ duplicates ระหว่างเรียง
sort -u file.txt

# Human-readable sort (1K, 2M, 3G)
du -sh /* 2>/dev/null | sort -h

# เรียงแบบ stable
sort -s -k1,1 data.txt

# ตัวอย่าง: หา top 10 IPs จาก access.log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

### 24.3 uniq - จัดการ duplicates

```bash
# ลบบรรทัดที่ซ้ำกัน (ต้องเรียงก่อน)
sort file.txt | uniq

# นับจำนวนซ้ำ
sort file.txt | uniq -c

# แสดงเฉพาะที่ซ้ำ
sort file.txt | uniq -d

# แสดงเฉพาะที่ไม่ซ้ำ
sort file.txt | uniq -u

# ไม่สนใจ case
sort file.txt | uniq -i

# skip N fields ก่อนเปรียบเทียบ
sort data.txt | uniq -f 1    # skip field 1

# ตัวอย่างใช้งานจริง
# หา IPs ที่ login ผิดพลาดมากที่สุด
grep "Failed password" /var/log/auth.log | \
    awk '{print $(NF-3)}' | \
    sort | \
    uniq -c | \
    sort -rn | \
    head -20
```

### 24.4 cut - ตัดข้อมูลตาม column

```bash
# ตัดตาม character position
cut -c1-5 file.txt           # เอา character 1-5
cut -c1,3,5 file.txt         # เอา character 1, 3, 5
cut -c5- file.txt            # เอาตั้งแต่ character 5 เป็นต้นไป

# ตัดตาม field (delimiter)
cut -d: -f1 /etc/passwd      # เอา field 1 (username)
cut -d: -f1,3 /etc/passwd    # เอา field 1 และ 3
cut -d, -f2 data.csv         # CSV: เอา column 2

# ตัวอย่าง: ดู hostname และ IP
hostname -I | cut -d' ' -f1

# ตัวอย่าง: ดูรายชื่อ users
cut -d: -f1 /etc/passwd | sort
```

### 24.5 tr - แปลงตัวอักษร

```bash
# แปลง lowercase เป็น uppercase
echo "hello world" | tr 'a-z' 'A-Z'
# HELLO WORLD

# แปลง uppercase เป็น lowercase
echo "HELLO WORLD" | tr 'A-Z' 'a-z'
# hello world

# ลบตัวอักษรที่ระบุ (-d)
echo "h3ll0 w0rld" | tr -d '0-9'
# hll wrld

# ลบ whitespace ซ้ำ (-s)
echo "too   many   spaces" | tr -s ' '
# too many spaces

# แทนที่ตัวอักษร
echo "hello" | tr 'aeiou' '*'
# h*ll*

# แปลง newlines เป็น spaces
tr '\n' ' ' < file.txt

# ลบ non-printable characters
tr -cd '[:print:]' < file.txt

# ตัวอย่าง: แปลง CSV เป็น TSV
tr ',' '\t' < data.csv > data.tsv
```

---

## ขั้นตอนที่ 25: คำสั่ง System Information

### 25.1 uname - ข้อมูล OS

```bash
# ชื่อ OS
uname -s
# Linux

# Hostname
uname -n
# myserver

# Kernel version
uname -r
# 6.8.0-45-generic

# Machine hardware
uname -m
# x86_64

# ทั้งหมด
uname -a
# Linux myserver 6.8.0-45-generic #45-Ubuntu SMP x86_64 GNU/Linux
```

### 25.2 top, htop, ps - ดู processes

```bash
# top - ดู processes แบบ real-time
top

# htop - top ที่ดีกว่า (ต้องติดตั้ง)
htop

# ps - ดู processes snapshot
ps                        # processes ของ user นี้
ps aux                    # ทุก processes
ps aux | grep nginx       # หา nginx process
ps -ef                    # full format

# ดู process tree
pstree
ps auxf

# ดู processes เรียงตาม CPU
ps aux --sort=-%cpu | head -10

# ดู processes เรียงตาม Memory
ps aux --sort=-%mem | head -10

# ดู process โดย PID
ps -p 1234

# Kill process
kill 1234           # ส่ง SIGTERM
kill -9 1234        # ส่ง SIGKILL (บังคับ)
killall nginx       # kill ทุก process ชื่อ nginx
pkill -u john       # kill ทุก process ของ user john

# ดู process โดย command name
pgrep nginx
pgrep -a nginx      # แสดง command ด้วย
```

### 25.3 df และ du - ดูพื้นที่ disk

```bash
# df - disk free space
df                    # ทุก filesystem
df -h                 # human-readable
df -H                 # SI units (1K=1000)
df /home              # เฉพาะ /home
df -T                 # แสดง filesystem type

# du - disk usage
du file.txt           # ขนาดไฟล์
du -h file.txt        # human-readable
du -sh /home          # รวม /home
du -sh *              # ขนาดทุกอย่างในปัจจุบัน
du -sh /* 2>/dev/null | sort -h    # เรียงตามขนาด

# ดู 10 directory ที่ใหญ่ที่สุด
du -sh /var/* 2>/dev/null | sort -rh | head -10

# ดูไฟล์ที่ใหญ่ที่สุด
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -20
```

### 25.4 free - ดู Memory

```bash
# ดู memory
free
# แสดง: total, used, free, shared, buff/cache, available

# human-readable
free -h

# human-readable แบบ SI
free -H

# อัปเดตทุก 2 วินาที
free -h -s 2

# ตัวอย่าง: ดูเฉพาะ RAM ที่ใช้
free | grep Mem | awk '{printf "Memory: %.1f%% used (%s/%s)\n", ($3/$2)*100, $3, $2}'
```

---

## ขั้นตอนที่ 26: คำสั่ง Network

### 26.1 ping, curl, wget

```bash
# ping - ทดสอบการเชื่อมต่อ
ping google.com
ping -c 4 google.com        # ping 4 ครั้ง
ping -i 0.5 google.com      # interval 0.5 วินาที

# curl - HTTP client
curl http://example.com                          # GET request
curl -I http://example.com                       # Headers only
curl -o output.html http://example.com           # บันทึกเป็นไฟล์
curl -L http://example.com                       # Follow redirects
curl -X POST -d "data=value" http://api.com      # POST request
curl -H "Authorization: Bearer TOKEN" http://api.com  # Custom header

# wget - download files
wget http://example.com/file.zip
wget -q http://example.com/file.zip              # quiet mode
wget -c http://example.com/file.zip              # resume download
wget -O custom_name.zip http://example.com/file  # custom filename

# ip - network info
ip addr show                  # ดู IP addresses
ip route show                 # ดู routing table
ip link show                  # ดู network interfaces

# netstat / ss - network connections
ss -tuln                      # listening ports
ss -tunap                     # all connections
netstat -tunap                # เหมือน ss (เก่า)
```

---

## ขั้นตอนที่ 27: Redirection และ Pipes

### 27.1 Output Redirection

```bash
# > ส่ง output ไปไฟล์ (overwrite)
ls > files.txt

# >> ต่อท้ายไฟล์ (append)
echo "new line" >> files.txt

# 2> ส่ง error ไปไฟล์
ls /nonexistent 2> errors.txt

# 2>&1 รวม stdout และ stderr
ls /nonexistent > output.txt 2>&1

# &> รวม stdout และ stderr (bash shorthand)
ls /nonexistent &> output.txt

# /dev/null - ทิ้ง output
command > /dev/null           # ทิ้ง stdout
command 2>/dev/null           # ทิ้ง stderr
command &>/dev/null           # ทิ้งทั้งคู่

# ตัวอย่างการใช้งาน
{
    echo "เริ่มต้น: $(date)"
    ls /home/
    echo "เสร็จ: $(date)"
} > /tmp/output.txt 2>&1
```

### 27.2 Input Redirection

```bash
# < อ่านจากไฟล์
sort < unsorted.txt

# << Here Document
cat << EOF
บรรทัดที่ 1
บรรทัดที่ 2
EOF

# <<< Here String
grep "pattern" <<< "this is a string with pattern in it"

# ตัวอย่าง: อ่าน input ทีละบรรทัด
while IFS= read -r line; do
    echo "บรรทัด: $line"
done < file.txt
```

### 27.3 Pipes

```bash
# | ส่ง output ของคำสั่งหนึ่งไปเป็น input ของอีกคำสั่ง
ls | grep ".txt"
cat file.txt | grep "error" | wc -l

# ตัวอย่าง pipeline ซับซ้อน
# หา process ที่ใช้ memory มากสุด
ps aux | sort -k4rn | head -5 | awk '{print $4"%", $11}'

# นับจำนวนไฟล์ต่อประเภท
find /home -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn

# ดู top 10 words ในไฟล์
cat document.txt | tr -s ' ' '\n' | sort | uniq -c | sort -rn | head -10

# tee - แสดงและบันทึกพร้อมกัน
ls | tee filelist.txt
command | tee -a logfile.txt | grep "error"
```

---

## ขั้นตอนที่ 28: Permissions และ Ownership

### 28.1 chmod - เปลี่ยน Permissions

```bash
# Symbolic mode
chmod +x file.sh          # เพิ่ม execute
chmod -x file.sh          # ลบ execute
chmod +r file.txt         # เพิ่ม read
chmod u+x file.sh         # เพิ่ม execute เฉพาะ owner
chmod g+w file.txt        # เพิ่ม write สำหรับ group
chmod o-r file.txt        # ลบ read สำหรับ others
chmod a+r file.txt        # เพิ่ม read สำหรับทุกคน (a=all)

# Numeric mode (Octal)
chmod 755 file.sh         # rwxr-xr-x
chmod 644 file.txt        # rw-r--r--
chmod 600 secret.txt      # rw-------
chmod 777 shared/         # rwxrwxrwx (ระวัง!)

# Octal ความหมาย:
# 4 = read (r)
# 2 = write (w)  
# 1 = execute (x)
# ตัวอย่าง: 7 = 4+2+1 = rwx, 5 = 4+0+1 = r-x, 4 = r--

# Recursive
chmod -R 755 /var/www/html

# เฉพาะไฟล์
find /var/www -type f -exec chmod 644 {} \;

# เฉพาะ directory
find /var/www -type d -exec chmod 755 {} \;
```

### 28.2 chown - เปลี่ยน Owner

```bash
# เปลี่ยน owner
chown user file.txt

# เปลี่ยน owner และ group
chown user:group file.txt

# เปลี่ยนเฉพาะ group
chown :group file.txt
# หรือ
chgrp group file.txt

# Recursive
chown -R user:group /var/www

# เปลี่ยน ownership ของไฟล์
sudo chown root:root /etc/hosts
```

---

## ขั้นตอนที่ 29: Environment Variables

### 29.1 ดูและตั้งค่า Variables

```bash
# ดู environment variables ทั้งหมด
env
printenv

# ดูค่าตัวแปรเฉพาะ
echo $HOME
echo $PATH
echo $USER
echo $SHELL
echo $PWD
echo $OLDPWD

# ตัวแปร system สำคัญ
echo "HOME:    $HOME"
echo "PATH:    $PATH"
echo "USER:    $USER"
echo "SHELL:   $SHELL"
echo "PWD:     $PWD"
echo "LANG:    $LANG"
echo "TERM:    $TERM"
echo "PS1:     $PS1"

# ตั้งค่าตัวแปร
NAME="สมชาย"
export NAME    # ส่งต่อไป child processes

# หรือ ตั้งค่าพร้อม export ในบรรทัดเดียว
export API_KEY="my-secret-key"

# ตั้งค่าชั่วคราวสำหรับ command เดียว
HTTP_PROXY=http://proxy:3128 curl http://example.com

# unset ตัวแปร
unset NAME
```

### 29.2 PATH Variable

```bash
# ดู PATH ปัจจุบัน
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# เพิ่มใน PATH
export PATH="$PATH:/home/user/scripts"
export PATH="/usr/local/custom/bin:$PATH"    # เพิ่มต้น PATH

# เพิ่มแบบถาวร (เพิ่มใน ~/.bashrc หรือ ~/.bash_profile)
echo 'export PATH="$PATH:$HOME/bin"' >> ~/.bashrc
source ~/.bashrc

# ตรวจสอบว่าโปรแกรมอยู่ใน PATH ไหม
if command -v python3 &>/dev/null; then
    echo "Python3 พบใน PATH"
else
    echo "ไม่พบ Python3"
fi
```

---

## ขั้นตอนที่ 30: Workshop 02 - File Management Script

### Workshop: สร้าง File Manager Script

```bash
#!/usr/bin/env bash
# =============================================================================
# Workshop 02: file_manager.sh
# จัดการไฟล์แบบอัตโนมัติ
# =============================================================================

set -euo pipefail

readonly SCRIPT_NAME="$(basename "$0")"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'  # No Color

log_info()    { echo -e "${GREEN}[INFO]${NC} $*"; }
log_warning() { echo -e "${YELLOW}[WARN]${NC} $*"; }
log_error()   { echo -e "${RED}[ERROR]${NC} $*" >&2; }

# Function: จัดเรียงไฟล์ตามประเภท
organize_files() {
    local source_dir="${1:-.}"
    local target_dir="${2:-$source_dir/organized}"
    
    log_info "จัดเรียงไฟล์จาก: $source_dir"
    
    declare -A type_dirs=(
        ["images"]="jpg jpeg png gif bmp svg webp"
        ["documents"]="pdf doc docx txt md odt"
        ["videos"]="mp4 avi mkv mov wmv flv"
        ["audio"]="mp3 wav flac aac ogg wma"
        ["archives"]="zip tar gz bz2 7z rar"
        ["code"]="sh py js ts html css json xml"
        ["data"]="csv tsv sql db sqlite"
    )
    
    local moved=0
    
    for category in "${!type_dirs[@]}"; do
        local extensions="${type_dirs[$category]}"
        
        for ext in $extensions; do
            # หาไฟล์ที่มีนามสกุลนี้
            while IFS= read -r -d $'\0' file; do
                local dest_dir="$target_dir/$category"
                mkdir -p "$dest_dir"
                
                local filename
                filename=$(basename "$file")
                local dest_file="$dest_dir/$filename"
                
                # ถ้ามีไฟล์ชื่อเดิมแล้ว เพิ่ม timestamp
                if [ -e "$dest_file" ]; then
                    local timestamp
                    timestamp=$(date '+%Y%m%d_%H%M%S')
                    dest_file="$dest_dir/${filename%.*}_${timestamp}.${filename##*.}"
                fi
                
                mv "$file" "$dest_file"
                log_info "  ย้าย: $filename → $category/"
                ((moved++))
                
            done < <(find "$source_dir" -maxdepth 1 -type f -iname "*.${ext}" -print0)
        done
    done
    
    log_info "ย้ายไฟล์ทั้งหมด: $moved ไฟล์"
}

# Function: หาไฟล์ duplicate
find_duplicates() {
    local dir="${1:-.}"
    
    log_info "ค้นหาไฟล์ duplicate ใน: $dir"
    
    declare -A checksums
    local duplicates=0
    
    while IFS= read -r -d $'\0' file; do
        local checksum
        checksum=$(md5sum "$file" | cut -d' ' -f1)
        
        if [ "${checksums[$checksum]+exists}" ]; then
            log_warning "Duplicate พบ:"
            log_warning "  Original: ${checksums[$checksum]}"
            log_warning "  Duplicate: $file"
            ((duplicates++))
        else
            checksums[$checksum]="$file"
        fi
        
    done < <(find "$dir" -type f -print0)
    
    if [ "$duplicates" -eq 0 ]; then
        log_info "ไม่พบ duplicates"
    else
        log_warning "พบ $duplicates ไฟล์ที่ซ้ำ"
    fi
}

# Function: สร้าง backup
create_backup() {
    local source="${1}"
    local backup_dir="${2:-$HOME/backups}"
    local timestamp
    timestamp=$(date '+%Y%m%d_%H%M%S')
    local backup_name
    backup_name="$(basename "$source")_backup_${timestamp}.tar.gz"
    
    mkdir -p "$backup_dir"
    
    log_info "สร้าง backup: $source → $backup_dir/$backup_name"
    
    if tar -czf "$backup_dir/$backup_name" "$source" 2>/dev/null; then
        local size
        size=$(du -sh "$backup_dir/$backup_name" | cut -f1)
        log_info "Backup สำเร็จ! ขนาด: $size"
    else
        log_error "Backup ล้มเหลว!"
        return 1
    fi
}

# Function: ล้างไฟล์ temporary
cleanup_temp() {
    local dir="${1:-.}"
    local patterns=("*.tmp" "*.temp" "*.bak" "*~" "*.swp" ".DS_Store")
    local removed=0
    
    log_info "ล้างไฟล์ temporary ใน: $dir"
    
    for pattern in "${patterns[@]}"; do
        while IFS= read -r -d $'\0' file; do
            rm -f "$file"
            log_info "  ลบ: $file"
            ((removed++))
        done < <(find "$dir" -name "$pattern" -print0)
    done
    
    log_info "ลบไฟล์ทั้งหมด: $removed ไฟล์"
}

# Main menu
show_menu() {
    echo ""
    echo -e "${BLUE}=== File Manager ===${NC}"
    echo "1. จัดเรียงไฟล์ตามประเภท"
    echo "2. ค้นหาไฟล์ duplicate"
    echo "3. สร้าง Backup"
    echo "4. ล้างไฟล์ temporary"
    echo "5. ออก"
    echo ""
    read -rp "เลือก (1-5): " choice
    
    case "$choice" in
        1)
            read -rp "ระบุ directory (Enter = ปัจจุบัน): " dir
            organize_files "${dir:-.}"
            ;;
        2)
            read -rp "ระบุ directory (Enter = ปัจจุบัน): " dir
            find_duplicates "${dir:-.}"
            ;;
        3)
            read -rp "ระบุไฟล์/directory ที่ต้องการ backup: " source
            create_backup "$source"
            ;;
        4)
            read -rp "ระบุ directory (Enter = ปัจจุบัน): " dir
            cleanup_temp "${dir:-.}"
            ;;
        5)
            log_info "ออกจากโปรแกรม"
            exit 0
            ;;
        *)
            log_error "ตัวเลือกไม่ถูกต้อง"
            ;;
    esac
}

# Main
main() {
    while true; do
        show_menu
    done
}

main "$@"
```

---

## สรุป Part 02

### คำสั่งสำคัญที่เรียนรู้

| คำสั่ง | การใช้งาน |
|--------|----------|
| `ls -la` | แสดงไฟล์ทั้งหมดแบบละเอียด |
| `find` | ค้นหาไฟล์ตามเงื่อนไข |
| `grep` | ค้นหาข้อความในไฟล์ |
| `sort` | เรียงข้อมูล |
| `uniq` | จัดการ duplicates |
| `cut` | ตัดข้อความตาม column |
| `tr` | แปลงตัวอักษร |
| `wc` | นับบรรทัด/คำ/ตัวอักษร |
| `chmod` | เปลี่ยน permissions |
| `chown` | เปลี่ยน owner |

### แบบฝึกหัด Part 02

**Easy:**
1. หาไฟล์ทั้งหมดใน /etc ที่มีนามสกุล .conf
2. แสดง 5 directory ที่ใหญ่ที่สุดใน /var
3. นับจำนวน user ใน /etc/passwd

**Medium:**
4. สร้าง script ที่หาไฟล์ซ้ำโดยใช้ md5sum
5. สร้าง script ที่ backup ไฟล์เก่ากว่า 7 วัน

**Hard:**
6. สร้าง script Monitor disk usage และส่ง alert เมื่อเกิน 80%

### เฉลยแบบฝึกหัด

```bash
# ข้อ 1: หาไฟล์ .conf
find /etc -name "*.conf" 2>/dev/null

# ข้อ 2: 5 directory ใหญ่สุด
du -sh /var/* 2>/dev/null | sort -rh | head -5

# ข้อ 3: นับ users
wc -l < /etc/passwd
# หรือ
grep -c "^[^#]" /etc/passwd
```

---

**ต่อไป:** [Part 03 - ตัวแปรและประเภทข้อมูล](part-03.md)

*Part 02 จบแล้ว! พร้อมเรียน Part 03 →*
