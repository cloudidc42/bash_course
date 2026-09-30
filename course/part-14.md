# Part 14: Process Management

## Module 2: Intermediate Level (ระดับกลาง)

---

## ขั้นตอนที่ 347: Processes คืออะไร?

**Process** คือโปรแกรมที่กำลังทำงานในระบบ แต่ละ process มี PID (Process ID) เป็น unique identifier

```bash
#!/usr/bin/env bash
# process_intro.sh - แนะนำ Process Management

echo "=== Process Management ==="
echo ""

echo "1. ดู processes ที่กำลังทำงาน:"
ps aux | head -5

echo ""
echo "2. ดู process ของตัวเอง:"
echo "Current shell PID: $$"
echo "Parent PID: $PPID"

echo ""
echo "3. Process tree:"
pstree $$ 2>/dev/null || ps --ppid $$ 2>/dev/null || \
ps -o pid,ppid,cmd --ppid $$ 2>/dev/null || \
echo "ดูด้วย: pstree $$"

echo ""
echo "4. Process information:"
cat /proc/$$/status 2>/dev/null | head -10 || \
ps -p $$ -o pid,ppid,user,pcpu,pmem,cmd 2>/dev/null
```

---

## ขั้นตอนที่ 348: ps - Process Status

```bash
#!/usr/bin/env bash
# ps_command.sh - ps command

echo "=== ps Command ==="

echo "1. ps aux - แสดงทุก processes:"
ps aux | head -10
echo "..."

echo ""
echo "2. ps -ef - แสดง full format:"
ps -ef | head -10
echo "..."

echo ""
echo "3. ps ด้วย custom format:"
ps -eo pid,ppid,user,pcpu,pmem,vsz,rss,stat,start,time,cmd | head -10

echo ""
echo "4. ค้นหา process เฉพาะ:"
ps aux | grep "bash" | grep -v grep

echo ""
echo "5. Process Tree:"
ps auxf 2>/dev/null | head -20 || ps aux

echo ""
echo "6. ดู process ด้วย PID:"
ps -p $$ -o pid,ppid,user,pcpu,pmem,cmd

echo ""
echo "7. Sorted by CPU usage:"
ps aux --sort=-%cpu 2>/dev/null | head -5 || \
ps aux | sort -k3 -rn | head -5

echo ""
echo "8. Sorted by memory:"
ps aux --sort=-%mem 2>/dev/null | head -5 || \
ps aux | sort -k4 -rn | head -5

echo ""
echo "ps columns:"
echo "  USER  - owner"
echo "  PID   - Process ID"
echo "  %CPU  - CPU usage"
echo "  %MEM  - Memory usage"
echo "  VSZ   - Virtual memory size"
echo "  RSS   - Resident Set Size (actual RAM)"
echo "  STAT  - Process state"
echo "  START - Start time"
echo "  TIME  - CPU time used"
echo "  CMD   - Command"
```

---

## ขั้นตอนที่ 349: Process States

```bash
#!/usr/bin/env bash
# process_states.sh - Process States

echo "=== Process States ==="
echo ""
echo "Process States (STAT column):"
echo ""
echo "  R - Running (กำลังทำงาน)"
echo "  S - Sleeping, interruptible (รอ I/O)"
echo "  D - Sleeping, uninterruptible (รอ kernel)"
echo "  T - Stopped/Traced"
echo "  Z - Zombie (รอ parent collect exit status)"
echo "  X - Dead"
echo ""
echo "Additional flags:"
echo "  s - Session leader"
echo "  l - Multi-threaded"
echo "  + - Foreground process group"
echo "  < - High priority"
echo "  N - Low priority"
echo ""

echo "ดู process states:"
ps aux | awk '{print $8}' | sort | uniq -c | sort -rn

echo ""
echo "Zombie processes:"
ps aux | awk '$8 ~ /Z/ {print $0}'

echo ""
echo "การสร้าง zombie process (demo):"
cat << 'EOF'
# Zombie เกิดเมื่อ child process ตาย แต่ parent ยังไม่รับ exit status
# (ยังไม่ได้ call wait())

#!/usr/bin/env bash
create_zombie() {
    bash -c 'sleep 1 &
    wait_pid=$!
    # ไม่ call wait $wait_pid
    sleep 5'
}
EOF
```

---

## ขั้นตอนที่ 350: Background Processes

```bash
#!/usr/bin/env bash
# background.sh - Background Processes

echo "=== Background Processes ==="

echo "1. รัน process ใน background ด้วย &:"
sleep 5 &
bg_pid=$!
echo "Background PID: $bg_pid"

echo ""
echo "2. ดู background jobs:"
jobs

echo ""
echo "3. jobs -l (แสดง PID):"
jobs -l

echo ""
echo "4. นำกลับมา foreground:"
# fg %1   # นำ job 1 กลับมา foreground

echo ""
echo "5. ส่งไป background:"
# bg %1   # ส่ง job 1 ไป background

echo ""
echo "6. รอ background job:"
echo "กำลังรอ PID $bg_pid..."
wait $bg_pid
echo "Job เสร็จแล้ว, exit code: $?"

echo ""
echo "7. รันหลาย jobs พร้อมกัน:"
echo "Starting 3 background jobs..."
for i in 1 2 3; do
    (sleep $i && echo "Job $i done") &
    echo "Started job $i (PID: $!)"
done

echo "รอทุก jobs..."
wait
echo "ทุก jobs เสร็จแล้ว"

echo ""
echo "8. Kill background job:"
sleep 100 &
kill_pid=$!
echo "Started process $kill_pid"
kill $kill_pid
echo "Killed process $kill_pid"
```

---

## ขั้นตอนที่ 351: kill และ Signals

```bash
#!/usr/bin/env bash
# signals.sh - kill และ Signals

echo "=== Signals ==="
echo ""
echo "Signals สำคัญ:"
echo ""
echo "  1 SIGHUP    - Hangup (ปิด terminal)"
echo "  2 SIGINT    - Interrupt (Ctrl+C)"
echo "  3 SIGQUIT   - Quit (Ctrl+\\)"
echo "  9 SIGKILL   - Force kill (ไม่สามารถ block ได้)"
echo " 15 SIGTERM   - Terminate (graceful)"
echo " 17 SIGCHLD   - Child status changed"
echo " 18 SIGCONT   - Continue (หลัง SIGSTOP)"
echo " 19 SIGSTOP   - Stop (ไม่สามารถ block ได้)"
echo " 20 SIGTSTP   - Terminal stop (Ctrl+Z)"
echo ""

echo "1. ดู signals ทั้งหมด:"
kill -l | head -20

echo ""
echo "2. Send signal ด้วยชื่อ:"
# kill -SIGTERM PID
# kill -TERM PID
# kill -15 PID

echo ""
echo "3. Kill process gracefully:"
sleep 1000 &
test_pid=$!
echo "Started: $test_pid"

kill -SIGTERM $test_pid
sleep 0.1
if kill -0 $test_pid 2>/dev/null; then
    echo "Process ยังอยู่, force kill..."
    kill -SIGKILL $test_pid
else
    echo "Process terminated gracefully"
fi

echo ""
echo "4. Kill ทุก process ชื่อ:"
# killall process_name
# pkill pattern

echo ""
echo "5. pkill - kill by name/pattern:"
# pkill -9 firefox
# pkill -u username

echo ""
echo "6. pgrep - ค้นหา PID:"
pgrep bash | head -5

echo ""
echo "7. Graceful shutdown pattern:"
graceful_shutdown() {
    local pid=$1
    local timeout=${2:-10}
    
    echo "Sending SIGTERM to $pid..."
    kill -SIGTERM "$pid" 2>/dev/null || return 0
    
    local waited=0
    while kill -0 "$pid" 2>/dev/null; do
        if (( waited >= timeout )); then
            echo "Timeout! Force killing $pid..."
            kill -SIGKILL "$pid" 2>/dev/null
            return 1
        fi
        sleep 1
        (( waited++ ))
    done
    echo "Process $pid terminated"
    return 0
}
```

---

## ขั้นตอนที่ 352: trap - Handle Signals

```bash
#!/usr/bin/env bash
# trap_signals.sh - trap command

echo "=== trap Command ==="

echo "1. Trap SIGINT (Ctrl+C):"
cat << 'SCRIPT'
#!/usr/bin/env bash
cleanup() {
    echo "Caught SIGINT! Cleaning up..."
    rm -f /tmp/myapp.lock
    exit 0
}
trap cleanup SIGINT

echo "Running (press Ctrl+C to stop)..."
while true; do
    sleep 1
done
SCRIPT

echo ""
echo "2. Trap EXIT (ทำงานเสมอเมื่อ script จบ):"
cat << 'SCRIPT'
#!/usr/bin/env bash
cleanup() {
    echo "Script ending, cleanup..."
    rm -rf "$TMPDIR"
}

TMPDIR=$(mktemp -d)
trap cleanup EXIT

echo "Working in $TMPDIR"
touch "$TMPDIR/work.tmp"
# ไม่ว่า script จะจบยังไง cleanup จะทำงาน
SCRIPT

echo ""
echo "3. Trap หลาย signals:"
cleanup() {
    local sig="${1:-EXIT}"
    echo "Caught signal: $sig"
    # cleanup code here
}

trap 'cleanup SIGINT' INT
trap 'cleanup SIGTERM' TERM
trap 'cleanup EXIT' EXIT

echo "Traps set up"

# ล้าง traps
trap - INT TERM EXIT
echo "Traps cleared"

echo ""
echo "4. trap ERR - ทำงานเมื่อ command fail:"
cat << 'SCRIPT'
#!/usr/bin/env bash
set -E  # ต้องมี -E สำหรับ trap ERR ทำงานใน functions

error_handler() {
    echo "Error at line $LINENO: $BASH_COMMAND"
}
trap error_handler ERR

false  # triggers error
echo "This won't print"
SCRIPT

echo ""
echo "5. trap DEBUG - ทำงานก่อนทุก command:"
cat << 'SCRIPT'
#!/usr/bin/env bash
trace() {
    echo "TRACE: $BASH_COMMAND"
}
trap trace DEBUG

x=5
echo "x = $x"
y=$((x * 2))
echo "y = $y"
SCRIPT
```

---

## ขั้นตอนที่ 353: Process Priority - nice และ renice

```bash
#!/usr/bin/env bash
# nice_renice.sh - Process Priority

echo "=== Process Priority ==="
echo ""
echo "Nice values: -20 (highest priority) ถึง 19 (lowest priority)"
echo "Default nice value: 0"
echo ""

echo "1. ดู nice value ของ process:"
ps -o pid,ni,cmd -p $$

echo ""
echo "2. รัน command ด้วย nice value:"
echo "  nice -n 10 command        # nice value 10 (lower priority)"
echo "  nice -n -10 command       # nice value -10 (higher, ต้องเป็น root)"
echo "  nice command              # default nice value 10"

echo ""
echo "3. เปลี่ยน nice value ของ process ที่รันอยู่:"
echo "  renice -n 5 -p PID       # เปลี่ยน PID"
echo "  renice -n 5 -u username  # เปลี่ยนทุก processes ของ user"

echo ""
echo "4. demo nice:"
echo "รัน process พร้อม nice value 19 (low priority):"
nice -n 19 sleep 5 &
nice_pid=$!
echo "PID: $nice_pid"
ps -o pid,ni,cmd -p $nice_pid
kill $nice_pid 2>/dev/null

echo ""
echo "5. ionice - I/O Priority:"
echo "  ionice -c 1 -n 0 command  # Real-time I/O"
echo "  ionice -c 2 -n 7 command  # Best-effort, low"
echo "  ionice -c 3 command       # Idle I/O"

echo ""
echo "6. cpulimit - จำกัด CPU usage:"
echo "  cpulimit -l 50 command    # จำกัดที่ 50% CPU"
echo "  cpulimit -p PID -l 30     # จำกัด PID ที่ 30%"
```

---

## ขั้นตอนที่ 354: top และ htop

```bash
#!/usr/bin/env bash
# monitoring_tools.sh - Monitoring Tools

echo "=== Process Monitoring Tools ==="
echo ""

echo "1. top - real-time monitoring:"
echo "   top -b -n 1 | head -20   # batch mode, 1 iteration"
top -b -n 1 2>/dev/null | head -20 || \
ps aux --sort=-%cpu 2>/dev/null | head -10

echo ""
echo "2. top คำสั่งภายใน:"
cat << 'EOF'
ขณะที่ top ทำงาน:
  q - quit
  k - kill process
  r - renice
  u - filter by user
  o - filter
  1 - CPU breakdown
  M - sort by memory
  P - sort by CPU
  h - help
EOF

echo ""
echo "3. htop (ถ้าติดตั้ง):"
if command -v htop &>/dev/null; then
    echo "htop ติดตั้งแล้ว - รัน: htop"
else
    echo "ติดตั้ง htop: sudo apt install htop"
fi

echo ""
echo "4. watch - อัปเดตแบบ real-time:"
echo "  watch -n 1 'ps aux | head -10'"
echo "  watch -d -n 2 df -h   # highlight changes"

echo ""
echo "5. atop - advanced monitor:"
if command -v atop &>/dev/null; then
    atop -b 2>/dev/null | head -20
else
    echo "ติดตั้ง: sudo apt install atop"
fi

echo ""
echo "6. iotop - I/O monitoring:"
if command -v iotop &>/dev/null; then
    echo "iotop ติดตั้งแล้ว - รัน: sudo iotop"
else
    echo "ติดตั้ง: sudo apt install iotop"
fi
```

---

## ขั้นตอนที่ 355: Subshells และ Process Substitution

```bash
#!/usr/bin/env bash
# subshells.sh - Subshells

echo "=== Subshells ==="

echo "1. Subshell ด้วย ():"
echo "Changes in subshell ไม่กระทบ parent:"
x=10
(x=20; echo "In subshell: x=$x")
echo "In parent: x=$x"

echo ""
echo "2. Subshell inherit variables:"
PARENT_VAR="hello"
(echo "Subshell sees: $PARENT_VAR")

echo ""
echo "3. Command substitution เป็น subshell:"
result=$(echo "This runs in subshell")
echo "Result: $result"

echo ""
echo "4. Process substitution <():"
echo "Compare sorted files:"
diff <(echo -e "b\na\nc" | sort) <(echo -e "c\na\nb" | sort)

echo ""
echo "5. Process substitution >():"
echo "Redirect to multiple files:"
echo "Hello World" | tee >(tr '[:upper:]' '[:lower:]' > /tmp/lower.txt) \
                         >(tr '[:lower:]' '[:upper:]' > /tmp/upper.txt)

echo ""
echo "ผลลัพธ์:"
echo "Lower: $(cat /tmp/lower.txt)"
echo "Upper: $(cat /tmp/upper.txt)"

echo ""
echo "6. Coprocess:"
coproc my_proc { cat; }
echo "Hello" >&"${my_proc[1]}"
read -r response <&"${my_proc[0]}"
echo "Response: $response"
kill "${my_proc_PID}" 2>/dev/null

echo ""
echo "7. Explicit subshell vs function:"
modify_in_subshell() {
    (VAR="changed_in_subshell")
}

modify_in_function() {
    VAR="changed_in_function"
}

VAR="original"
modify_in_subshell
echo "After subshell: $VAR"  # ยังเป็น "original"

modify_in_function
echo "After function: $VAR"  # เปลี่ยนเป็น "changed_in_function"
```

---

## ขั้นตอนที่ 356: nohup และ Detached Processes

```bash
#!/usr/bin/env bash
# nohup.sh - nohup และ detached processes

echo "=== nohup และ Detached Processes ==="

echo "1. nohup - ป้องกัน SIGHUP:"
echo "  nohup command &"
echo "  nohup command > output.log 2>&1 &"
echo ""
echo "  เมื่อใช้ nohup:"
echo "  - process จะยังทำงานแม้ปิด terminal"
echo "  - output ไปที่ nohup.out โดย default"

echo ""
echo "2. disown - detach job จาก shell:"
echo "  command &       # รันใน background"
echo "  disown %1       # detach job 1"
echo "  disown -h %1    # protect from SIGHUP ยังอยู่ใน jobs list"

echo ""
echo "3. screen - terminal multiplexer:"
cat << 'EOF'
screen commands:
  screen           - เริ่ม session ใหม่
  screen -ls       - list sessions
  screen -r name   - reattach
  
ภายใน screen:
  Ctrl+A d    - detach
  Ctrl+A c    - new window
  Ctrl+A n    - next window
  Ctrl+A "    - list windows
EOF

echo ""
echo "4. tmux - modern terminal multiplexer:"
cat << 'EOF'
tmux commands:
  tmux                  - เริ่ม session
  tmux new -s name      - session พร้อมชื่อ
  tmux ls               - list sessions
  tmux attach -t name   - reattach
  
ภายใน tmux:
  Ctrl+B d    - detach
  Ctrl+B c    - new window
  Ctrl+B n    - next window
  Ctrl+B %    - split vertical
  Ctrl+B "    - split horizontal
EOF

echo ""
echo "5. setsid - รันใน new session:"
echo "  setsid command &"
echo "  สร้าง session ใหม่, ไม่มี controlling terminal"

echo ""
echo "6. daemon function (เขียนเอง):"
daemonize() {
    local cmd="$@"
    
    # Fork process
    (
        # Detach from terminal
        exec setsid "$cmd" \
            </dev/null \
            >/tmp/daemon.log \
            2>&1 &
        echo $!
    )
}

echo "Daemonizing sleep command..."
pid=$(daemonize sleep 30)
echo "Daemon PID: $pid"
sleep 0.5
ps -p "$pid" -o pid,ppid,cmd 2>/dev/null || echo "Process running"
kill "$pid" 2>/dev/null
```

---

## ขั้นตอนที่ 357: Parallel Processing

```bash
#!/usr/bin/env bash
# parallel_processing.sh - Parallel Processing

echo "=== Parallel Processing ==="

# ==================== Basic Parallel ====================
echo "1. รันหลาย jobs พร้อมกัน:"
do_work() {
    local id=$1
    local delay=$2
    sleep "$delay"
    echo "Job $id completed (${delay}s)"
}

echo "Sequential:"
time {
    do_work 1 1
    do_work 2 1
    do_work 3 1
}

echo ""
echo "Parallel:"
time {
    do_work 1 1 &
    do_work 2 1 &
    do_work 3 1 &
    wait
}

echo ""
echo "2. จำกัดจำนวน parallel jobs:"
run_parallel() {
    local max_jobs=$1
    shift
    local items=("$@")
    local running=0
    local pids=()
    
    for item in "${items[@]}"; do
        while (( running >= max_jobs )); do
            # รอ job ใด job หนึ่งเสร็จ
            for pid in "${pids[@]}"; do
                if ! kill -0 "$pid" 2>/dev/null; then
                    running=$((running - 1))
                    # ลบ pid ออกจาก array
                    pids=("${pids[@]/$pid}")
                fi
            done
            sleep 0.1
        done
        
        (sleep 0.5; echo "  Processed: $item") &
        pids+=($!)
        running=$((running + 1))
        echo "  Started: $item (running: $running)"
    done
    
    wait
}

echo ""
echo "Parallel processing กับ max 2 jobs:"
run_parallel 2 "item_a" "item_b" "item_c" "item_d" "item_e"

echo ""
echo "3. GNU parallel (ถ้าติดตั้ง):"
if command -v parallel &>/dev/null; then
    echo "1 2 3 4 5" | tr ' ' '\n' | \
        parallel -j 3 'echo "Processing {}"'
else
    echo "ติดตั้ง: sudo apt install parallel"
    echo ""
    echo "GNU parallel syntax:"
    echo "  parallel -j 4 command ::: item1 item2 item3"
    echo "  cat list.txt | parallel -j 4 process_item {}"
    echo "  parallel -j 4 --progress command ::: *"
fi

echo ""
echo "4. xargs สำหรับ parallel:"
echo -e "a\nb\nc\nd\ne" | \
    xargs -P 3 -I{} bash -c 'sleep 0.3; echo "Done: {}"'
```

---

## ขั้นตอนที่ 358: Process Monitoring Script

```bash
#!/usr/bin/env bash
# process_monitor.sh - Process Monitor

set -euo pipefail

echo "=== Process Monitor ==="

# Configuration
MONITOR_INTERVAL=5
MAX_CPU_PERCENT=80
MAX_MEM_PERCENT=90

# ==================== Monitor Functions ====================

check_process() {
    local process_name="$1"
    local pids
    pids=$(pgrep -x "$process_name" 2>/dev/null || true)
    
    if [[ -z "$pids" ]]; then
        echo "  [DOWN] $process_name is not running"
        return 1
    else
        echo "  [UP] $process_name (PIDs: $(echo $pids | tr '\n' ' '))"
        return 0
    fi
}

check_cpu() {
    local threshold="${1:-$MAX_CPU_PERCENT}"
    
    # หา processes ที่ใช้ CPU มากเกินไป
    ps aux | awk -v thresh="$threshold" 'NR>1 && $3+0 > thresh {
        printf "  HIGH CPU: PID %-7s %-15s CPU: %.1f%%\n", $2, $11, $3
    }'
}

check_memory() {
    local threshold="${1:-$MAX_MEM_PERCENT}"
    
    # ตรวจสอบ total memory
    if [[ -f /proc/meminfo ]]; then
        local total free available used_pct
        total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
        available=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
        used_pct=$(awk "BEGIN {printf \"%.0f\", (1 - $available/$total) * 100}")
        
        printf "  Memory: %s%% used" "$used_pct"
        if (( used_pct > threshold )); then
            echo " [WARNING!]"
        else
            echo " [OK]"
        fi
    fi
    
    # Processes ที่ใช้ memory มาก
    ps aux | awk 'NR>1 && $4+0 > 5 {
        printf "  HIGH MEM: PID %-7s %-15s MEM: %.1f%%\n", $2, $11, $4
    }' | head -5
}

check_zombie() {
    local zombies
    zombies=$(ps aux | awk '$8 ~ /Z/ {count++} END {print count+0}')
    
    if (( zombies > 0 )); then
        echo "  [WARNING] $zombies zombie process(es) found:"
        ps aux | awk '$8 ~ /Z/ {print "  PID:", $2, "CMD:", $11}'
    else
        echo "  [OK] No zombie processes"
    fi
}

get_process_info() {
    local pid=$1
    
    if [[ ! -d "/proc/$pid" ]]; then
        echo "  Process $pid not found"
        return 1
    fi
    
    echo "  PID: $pid"
    
    # Name
    if [[ -f "/proc/$pid/comm" ]]; then
        echo "  Name: $(cat /proc/$pid/comm)"
    fi
    
    # Status
    if [[ -f "/proc/$pid/status" ]]; then
        awk '/^(State|VmRSS|VmSize|Threads):/ {print "  " $0}' "/proc/$pid/status"
    fi
    
    # CPU/Memory from ps
    ps -p "$pid" -o pid,pcpu,pmem,cmd --no-header 2>/dev/null | \
        awk '{printf "  CPU: %.1f%% MEM: %.1f%%\n", $2, $3}'
}

# ==================== Main Monitor ====================
monitor_once() {
    echo ""
    echo "=== System Status $(date '+%Y-%m-%d %H:%M:%S') ==="
    
    echo ""
    echo "Critical Services:"
    for service in sshd cron systemd; do
        check_process "$service" 2>/dev/null || true
    done
    
    echo ""
    echo "Resource Usage:"
    check_cpu 50
    check_memory 70
    
    echo ""
    echo "Zombie Processes:"
    check_zombie
    
    echo ""
    echo "Top 5 CPU Consumers:"
    ps aux --sort=-%cpu 2>/dev/null | awk 'NR>1 && NR<=6 {
        printf "  %-8s %-20s CPU: %.1f%% MEM: %.1f%%\n", $2, substr($11,1,20), $3, $4
    }' || ps aux | sort -k3 -rn | awk 'NR>1 && NR<=6 {
        printf "  %-8s %-20s CPU: %.1f%% MEM: %.1f%%\n", $2, substr($11,1,20), $3, $4
    }'
}

monitor_once

echo ""
echo "=== Process Info สำหรับ current shell ==="
get_process_info $$
```

---

## ขั้นตอนที่ 359: cron - Scheduled Tasks

```bash
#!/usr/bin/env bash
# cron_intro.sh - cron Job Scheduling

echo "=== cron Job Scheduling ==="
echo ""

echo "cron format:"
echo ""
echo "  * * * * * command"
echo "  │ │ │ │ │"
echo "  │ │ │ │ └── Day of week (0-7, 0=Sun)"
echo "  │ │ │ └──── Month (1-12)"
echo "  │ │ └────── Day of month (1-31)"
echo "  │ └──────── Hour (0-23)"
echo "  └────────── Minute (0-59)"
echo ""

echo "ตัวอย่าง cron expressions:"
echo ""
echo "  * * * * *           ทุกนาที"
echo "  0 * * * *           ทุกชั่วโมง (นาที 0)"
echo "  0 0 * * *           ทุกวันเที่ยงคืน"
echo "  0 0 * * 0           ทุกอาทิตย์เที่ยงคืน"
echo "  0 0 1 * *           วันแรกของทุกเดือน"
echo "  */5 * * * *         ทุก 5 นาที"
echo "  0 9-17 * * 1-5      ทุกชั่วโมง 9-17 วันจันทร์-ศุกร์"
echo "  0 0,12 * * *        เที่ยงคืนและเที่ยงวัน"
echo "  @hourly             = 0 * * * *"
echo "  @daily              = 0 0 * * *"
echo "  @weekly             = 0 0 * * 0"
echo "  @monthly            = 0 0 1 * *"
echo "  @reboot             ตอน boot"
echo ""

echo "1. ดู crontab:"
crontab -l 2>/dev/null || echo "(no crontab)"

echo ""
echo "2. แก้ไข crontab:"
echo "  crontab -e   # แก้ไขด้วย editor"
echo "  crontab -l   # list"
echo "  crontab -r   # ลบทั้งหมด (ระวัง!)"

echo ""
echo "3. System cron directories:"
echo "  /etc/cron.hourly/"
echo "  /etc/cron.daily/"
echo "  /etc/cron.weekly/"
echo "  /etc/cron.monthly/"
echo "  /etc/crontab"
echo "  /etc/cron.d/"

echo ""
echo "4. สร้าง cron job สำหรับ backup:"
cat << 'EOF'
# Backup script - save as /etc/cron.d/myapp-backup
# ทุกวัน 2:30am
30 2 * * * root /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
EOF

echo ""
echo "5. Best practices:"
echo "  - ใช้ full path ใน commands"
echo "  - redirect output ไปยัง log file"
echo "  - test script ก่อนใส่ใน cron"
echo "  - ระวัง environment variables ต่างจาก shell"
echo "  - เพิ่ม MAILTO=user@example.com เพื่อรับ error email"
```

---

## ขั้นตอนที่ 360: Workshop - Process Manager

```bash
#!/usr/bin/env bash
# process_manager.sh - Workshop: Process Manager

set -euo pipefail

readonly SCRIPT_NAME="process_manager"
readonly PID_DIR="/tmp/pm_pids"
readonly LOG_DIR="/tmp/pm_logs"

mkdir -p "$PID_DIR" "$LOG_DIR"

# ==================== Process Manager ====================

start_process() {
    local name="$1"
    local cmd="${@:2}"
    local pid_file="$PID_DIR/${name}.pid"
    local log_file="$LOG_DIR/${name}.log"
    
    if [[ -f "$pid_file" ]]; then
        local pid
        pid=$(cat "$pid_file")
        if kill -0 "$pid" 2>/dev/null; then
            echo "[ERROR] $name already running (PID: $pid)"
            return 1
        fi
    fi
    
    echo "[START] Starting $name..."
    eval "$cmd" >> "$log_file" 2>&1 &
    local pid=$!
    echo "$pid" > "$pid_file"
    
    sleep 0.2
    if kill -0 "$pid" 2>/dev/null; then
        echo "[OK] $name started (PID: $pid)"
    else
        echo "[ERROR] $name failed to start"
        rm -f "$pid_file"
        return 1
    fi
}

stop_process() {
    local name="$1"
    local pid_file="$PID_DIR/${name}.pid"
    
    if [[ ! -f "$pid_file" ]]; then
        echo "[ERROR] $name is not running (no pid file)"
        return 1
    fi
    
    local pid
    pid=$(cat "$pid_file")
    
    if ! kill -0 "$pid" 2>/dev/null; then
        echo "[WARN] $name (PID: $pid) not running, removing pid file"
        rm -f "$pid_file"
        return 0
    fi
    
    echo "[STOP] Stopping $name (PID: $pid)..."
    kill -SIGTERM "$pid"
    
    local waited=0
    while kill -0 "$pid" 2>/dev/null; do
        if (( waited >= 5 )); then
            echo "[WARN] Force killing $name..."
            kill -SIGKILL "$pid" 2>/dev/null
            break
        fi
        sleep 1
        (( waited++ ))
    done
    
    rm -f "$pid_file"
    echo "[OK] $name stopped"
}

status_process() {
    local name="$1"
    local pid_file="$PID_DIR/${name}.pid"
    
    if [[ ! -f "$pid_file" ]]; then
        echo "[STOPPED] $name"
        return 1
    fi
    
    local pid
    pid=$(cat "$pid_file")
    
    if kill -0 "$pid" 2>/dev/null; then
        local cpu mem
        read -r cpu mem < <(ps -p "$pid" -o pcpu,pmem --no-header 2>/dev/null | awk '{print $1, $2}' || echo "0 0")
        printf "[RUNNING] %-15s PID: %-8s CPU: %5s%% MEM: %5s%%\n" "$name" "$pid" "$cpu" "$mem"
        return 0
    else
        echo "[DEAD] $name (PID: $pid, process not found)"
        rm -f "$pid_file"
        return 1
    fi
}

list_processes() {
    echo ""
    echo "Process Manager Status:"
    echo "----------------------"
    
    if [[ -z "$(ls "$PID_DIR"/*.pid 2>/dev/null)" ]]; then
        echo "No processes registered"
        return
    fi
    
    for pid_file in "$PID_DIR"/*.pid; do
        local name
        name=$(basename "$pid_file" .pid)
        status_process "$name"
    done
}

restart_process() {
    local name="$1"
    stop_process "$name"
    sleep 1
    # ต้องมี cmd สำหรับ restart - สำหรับ demo ใช้ simple loop
    echo "[RESTART] $name restarted"
}

tail_logs() {
    local name="$1"
    local lines="${2:-20}"
    local log_file="$LOG_DIR/${name}.log"
    
    if [[ -f "$log_file" ]]; then
        tail -n "$lines" "$log_file"
    else
        echo "No logs for $name"
    fi
}

# ==================== Demo ====================
echo "=== Process Manager Demo ==="
echo ""

echo "1. Starting worker processes..."
start_process "worker_1" "bash -c 'while true; do echo \"\$(date): worker_1 running\"; sleep 2; done'"
start_process "worker_2" "bash -c 'while true; do echo \"\$(date): worker_2 running\"; sleep 3; done'"

sleep 1

echo ""
echo "2. Status:"
list_processes

echo ""
echo "3. Logs:"
sleep 2
echo "Worker 1 logs:"
tail_logs "worker_1" 3

echo ""
echo "4. Stopping processes..."
stop_process "worker_1"
stop_process "worker_2"

echo ""
echo "5. Final status:"
list_processes

# Cleanup
rm -rf "$PID_DIR" "$LOG_DIR"
```

---

## ขั้นตอนที่ 361: สรุป Part 14 - Process Management

```bash
#!/usr/bin/env bash
# summary_process.sh

echo "=== สรุป Process Management ==="
echo ""
echo "Commands หลัก:"
echo "  ps aux        - ดู processes"
echo "  top/htop      - real-time monitor"
echo "  kill -SIGNAL  - ส่ง signal"
echo "  killall/pkill - kill by name"
echo "  pgrep         - ค้นหา PID"
echo "  nice/renice   - priority"
echo "  nohup         - detach from terminal"
echo "  jobs/bg/fg    - job control"
echo ""
echo "Signals หลัก:"
echo "   2 SIGINT  - Ctrl+C"
echo "   9 SIGKILL - force kill"
echo "  15 SIGTERM - graceful kill"
echo "  18 SIGCONT - continue"
echo "  19 SIGSTOP - stop"
echo ""
echo "Patterns:"
echo "  command &        - background"
echo "  wait \$!         - รอ background"
echo "  trap cleanup INT - handle Ctrl+C"
echo "  trap cleanup EXIT - always run"
echo "  nohup cmd &      - detached"
echo ""
echo "Next: Part 15 - Networking และ APIs"
```

---

## แบบฝึกหัด Part 14

### แบบฝึกหัดที่ 1: Service Watchdog
สร้าง watchdog script ที่:
- Monitor process list
- ถ้า process ตาย ให้ restart โดยอัตโนมัติ
- ส่ง notification (email หรือ log)
- บันทึก restart history
- รองรับ max restart limit

### แบบฝึกหัดที่ 2: Resource Monitor
Monitor ทรัพยากรระบบ:
- เก็บ CPU/Memory/Disk ทุก 30 วินาที
- แจ้งเตือนเมื่อเกิน threshold
- สร้าง report รายวัน
- Cleanup logs เก่า

### แบบฝึกหัดที่ 3: Parallel File Processor
ประมวลผลไฟล์แบบ parallel:
- รับ directory path
- ประมวลผลทุกไฟล์ใน directory
- จำกัด concurrent processes
- รายงาน progress และ errors

---

## สรุป

| หัวข้อ | Steps |
|--------|-------|
| Process introduction | 347 |
| ps command | 348 |
| Process states | 349 |
| Background processes | 350 |
| kill & signals | 351 |
| trap | 352 |
| nice/renice | 353 |
| top/htop | 354 |
| Subshells | 355 |
| nohup/detach | 356 |
| Parallel processing | 357 |
| Process monitoring | 358 |
| cron scheduling | 359 |
| Workshop | 360 |

**ขั้นตอนต่อไป**: Part 15 - Networking และ APIs
