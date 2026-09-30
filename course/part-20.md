# Part 20: Performance Tuning และ Optimization

## Module 2: Intermediate Level
### ขั้นตอนที่ 404-415: การปรับแต่งประสิทธิภาพ Bash Script

---

## ขั้นตอนที่ 404: ทำความเข้าใจ Performance Bottlenecks

การเขียน Bash Script ที่เร็วและมีประสิทธิภาพต้องเข้าใจจุดที่ทำให้ช้า

```bash
#!/usr/bin/env bash
# performance_basics.sh

# ทดสอบเวลาการทำงาน - time command
time_command() {
    local description="$1"
    shift
    local start_time=$(date +%s%N)
    "$@"
    local end_time=$(date +%s%N)
    local elapsed=$(( (end_time - start_time) / 1000000 ))
    echo "[$description] เวลา: ${elapsed}ms"
}

# ตัวอย่าง: เปรียบเทียบวิธีการนับ
count_with_loop() {
    local count=0
    for i in $(seq 1 1000); do
        count=$((count + 1))
    done
    echo "Count: $count"
}

count_with_arithmetic() {
    local count=0
    for ((i=1; i<=1000; i++)); do
        ((count++))
    done
    echo "Count: $count"
}

echo "=== เปรียบเทียบประสิทธิภาพ ==="
time_command "seq + loop" count_with_loop
time_command "C-style loop" count_with_arithmetic

# ใช้ time builtin
echo ""
echo "=== ใช้ time command ==="
{ time count_with_loop; } 2>&1
{ time count_with_arithmetic; } 2>&1
```

---

## ขั้นตอนที่ 405: Subshell Overhead และการลด Fork

การเปิด subshell มีค่าใช้จ่ายสูง ควรหลีกเลี่ยงเมื่อทำได้

```bash
#!/usr/bin/env bash
# subshell_optimization.sh

# BAD: สร้าง subshell ทุกครั้ง
bad_string_upper() {
    echo "$1" | tr '[:lower:]' '[:upper:]'  # fork + exec tr
}

# GOOD: ใช้ Bash built-in
good_string_upper() {
    echo "${1^^}"  # ไม่มี subshell
}

# BAD: ใช้ $(command) ซ้ำๆ
bad_file_processing() {
    local file="$1"
    local lines=$(wc -l < "$file")    # fork
    local words=$(wc -w < "$file")    # fork อีก
    local chars=$(wc -c < "$file")    # fork อีก
    echo "Lines: $lines, Words: $words, Chars: $chars"
}

# GOOD: อ่านครั้งเดียว
good_file_processing() {
    local file="$1"
    local stats
    stats=$(wc -lwc < "$file")        # fork เดียว
    read -r lines words chars <<< "$stats"
    echo "Lines: $lines, Words: $words, Chars: $chars"
}

# BAD: $(cat file) - useless use of cat
bad_read_file() {
    local content=$(cat "$1")
    echo "$content"
}

# GOOD: อ่านโดยตรง
good_read_file() {
    local content
    content=$(<"$1")    # ไม่มี cat
    echo "$content"
}

# ทดสอบ basename/dirname
# BAD:
bad_get_basename() {
    echo $(basename "$1")  # fork
}

# GOOD:
good_get_basename() {
    echo "${1##*/}"  # parameter expansion
}

bad_get_dirname() {
    echo $(dirname "$1")  # fork
}

good_get_dirname() {
    local path="$1"
    echo "${path%/*}"  # parameter expansion
}

# เปรียบเทียบ
echo "=== String Operations ==="
for i in $(seq 1 100); do
    bad_string_upper "hello world" > /dev/null
done

for i in $(seq 1 100); do
    good_string_upper "hello world" > /dev/null
done

echo "basename comparison:"
path="/usr/local/bin/script.sh"
echo "Bad: $(bad_get_basename "$path")"
echo "Good: $(good_get_basename "$path")"
```

---

## ขั้นตอนที่ 406: การใช้ Built-in Commands แทน External Commands

```bash
#!/usr/bin/env bash
# builtin_vs_external.sh

# String operations ด้วย Parameter Expansion แทน sed/awk

# แทน sed สำหรับ simple substitution
str="Hello World Hello"

# BAD: fork sed
bad_replace() { echo "$1" | sed "s/$2/$3/g"; }

# GOOD: parameter expansion
good_replace() {
    local str="$1" from="$2" to="$3"
    echo "${str//$from/$to}"
}

# แทน grep สำหรับ simple match
bad_contains() {
    echo "$1" | grep -q "$2"
}

good_contains() {
    [[ "$1" == *"$2"* ]]
}

# แทน awk สำหรับ column extraction
bad_get_field() {
    echo "$1" | awk -F"$3" "{print \$$2}"
}

good_get_field() {
    local line="$1" field="$2" sep="$3"
    IFS="$sep" read -ra fields <<< "$line"
    echo "${fields[$((field-1))]}"
}

# แทน wc -l สำหรับนับ array elements
bad_count_lines() {
    echo "$1" | wc -l
}

array_count() {
    local -n arr_ref=$1
    echo "${#arr_ref[@]}"
}

# แทน head/tail
bad_first_line() {
    head -1 "$1"
}

good_first_line() {
    IFS= read -r line < "$1"
    echo "$line"
}

# แทน test -f / test -d
# Bad: [[ $(ls -la "$1" 2>/dev/null | wc -l) -gt 0 ]]
# Good: [[ -e "$1" ]]

# String length
# Bad: $(echo -n "$str" | wc -c)
# Good: ${#str}

# ทดสอบ
echo "Replace: $(good_replace "$str" "Hello" "Hi")"
echo "Contains: $(good_contains "$str" "World" && echo yes || echo no)"
echo "Field: $(good_get_field "a:b:c:d" 2 ":")"

# printf แทน echo สำหรับ portability
printf "Name: %s, Age: %d\n" "Alice" 30
```

---

## ขั้นตอนที่ 407: Efficient File Reading และ Processing

```bash
#!/usr/bin/env bash
# efficient_file_reading.sh

# BAD: for loop กับ command substitution
bad_read_lines() {
    local file="$1"
    for line in $(cat "$file"); do  # แยก whitespace, glob expansion
        echo "Line: $line"
    done
}

# GOOD: while read loop
good_read_lines() {
    local file="$1"
    while IFS= read -r line; do
        echo "Line: $line"
    done < "$file"
}

# BETTER: process substitution สำหรับ pipeline
process_file_pipeline() {
    local file="$1"
    while IFS= read -r line; do
        # process line
        echo "Processed: ${line^^}"
    done < <(grep -v "^#" "$file")  # กรองก่อน
}

# อ่านทั้งไฟล์เข้า array
read_file_to_array() {
    local file="$1"
    mapfile -t lines < "$file"
    echo "Total lines: ${#lines[@]}"
    echo "First line: ${lines[0]}"
    echo "Last line: ${lines[-1]}"
}

# อ่านบางส่วน (head/tail equivalent)
read_first_n_lines() {
    local file="$1" n="$2"
    local count=0
    while IFS= read -r line && [[ $count -lt $n ]]; do
        echo "$line"
        ((count++))
    done < "$file"
}

# ประมวลผลไฟล์ขนาดใหญ่แบบ chunk
process_large_file() {
    local file="$1"
    local chunk_size=1000
    local line_num=0
    local chunk=()

    while IFS= read -r line; do
        chunk+=("$line")
        ((line_num++))
        
        if [[ ${#chunk[@]} -ge $chunk_size ]]; then
            # ประมวลผล chunk
            echo "Processing chunk ending at line $line_num"
            # process "${chunk[@]}"
            chunk=()
        fi
    done < "$file"
    
    # ประมวลผล chunk สุดท้าย
    if [[ ${#chunk[@]} -gt 0 ]]; then
        echo "Processing final chunk (${#chunk[@]} lines)"
    fi
}

# CSV processing ที่มีประสิทธิภาพ
process_csv() {
    local file="$1"
    local header_processed=false
    declare -a headers
    
    while IFS=',' read -ra fields; do
        if ! $header_processed; then
            headers=("${fields[@]}")
            header_processed=true
            continue
        fi
        
        # สร้าง associative array จาก row
        declare -A row
        for i in "${!headers[@]}"; do
            row["${headers[$i]}"]="${fields[$i]}"
        done
        
        echo "Name: ${row[name]}, Age: ${row[age]}"
        unset row
    done < "$file"
}

# สร้าง test file
create_test_file() {
    local file="$1"
    for i in $(seq 1 100); do
        echo "Line $i: This is test data number $i"
    done > "$file"
}

# Demo
test_file="/tmp/test_perf.txt"
create_test_file "$test_file"

echo "=== Reading file efficiently ==="
good_read_lines "$test_file" | head -3

echo ""
echo "=== Read to array ==="
read_file_to_array "$test_file"

echo ""
echo "=== First 3 lines ==="
read_first_n_lines "$test_file" 3

rm -f "$test_file"
```

---

## ขั้นตอนที่ 408: Caching และ Memoization

```bash
#!/usr/bin/env bash
# caching_memoization.sh

# Cache สำหรับ expensive operations
declare -A _cache

cache_get() {
    local key="$1"
    echo "${_cache[$key]:-}"
}

cache_set() {
    local key="$1" value="$2"
    _cache[$key]="$value"
}

cache_has() {
    local key="$1"
    [[ -n "${_cache[$key]+set}" ]]
}

# Memoized function
expensive_computation() {
    local input="$1"
    local cache_key="expensive:$input"
    
    if cache_has "$cache_key"; then
        echo "${_cache[$cache_key]}"
        return
    fi
    
    # Simulate expensive computation
    local result=$((input * input + input))
    sleep 0.1  # Simulate slow operation
    
    cache_set "$cache_key" "$result"
    echo "$result"
}

# Disk-based cache
DISK_CACHE_DIR="/tmp/bash_cache_$$"
mkdir -p "$DISK_CACHE_DIR"

disk_cache_get() {
    local key="$1"
    local cache_file="$DISK_CACHE_DIR/${key//\//_}"
    
    if [[ -f "$cache_file" ]]; then
        # ตรวจสอบ TTL (5 นาที)
        local age=$(($(date +%s) - $(stat -c %Y "$cache_file" 2>/dev/null || echo 0)))
        if [[ $age -lt 300 ]]; then
            cat "$cache_file"
            return 0
        fi
    fi
    return 1
}

disk_cache_set() {
    local key="$1" value="$2"
    local cache_file="$DISK_CACHE_DIR/${key//\//_}"
    echo "$value" > "$cache_file"
}

# Cache-aware function ที่ทำงานกับ external API หรือ command ช้าๆ
get_system_info_cached() {
    local key="system_info"
    local cached
    
    if cached=$(disk_cache_get "$key"); then
        echo "$cached"
        return
    fi
    
    # Expensive operation
    local info
    info=$(uname -a && uptime && df -h / | tail -1)
    
    disk_cache_set "$key" "$info"
    echo "$info"
}

# Function result cache ด้วย eval
declare -A _func_cache

memoize() {
    local func="$1"
    shift
    local args_key="${func}:$*"
    
    if [[ -n "${_func_cache[$args_key]+set}" ]]; then
        echo "${_func_cache[$args_key]}"
        return
    fi
    
    local result
    result=$("$func" "$@")
    _func_cache[$args_key]="$result"
    echo "$result"
}

# ตัวอย่าง: cache DNS lookups
resolve_hostname_cached() {
    local host="$1"
    memoize "getent" "hosts" "$host"
}

# Cleanup
cleanup_cache() {
    rm -rf "$DISK_CACHE_DIR"
}
trap cleanup_cache EXIT

# Demo
echo "=== Memoization Demo ==="
echo "First call (slow):"
time expensive_computation 42

echo "Second call (from cache):"
time expensive_computation 42

echo ""
echo "=== Disk Cache Demo ==="
echo "System Info (first fetch):"
get_system_info_cached | head -2

echo "System Info (from cache):"
get_system_info_cached | head -2
```

---

## ขั้นตอนที่ 409: Parallel Processing

```bash
#!/usr/bin/env bash
# parallel_processing.sh

# การประมวลผลแบบ parallel ด้วย background jobs
MAX_PARALLEL=${MAX_PARALLEL:-4}

# Semaphore สำหรับ parallel jobs
declare -a _pids=()

parallel_run() {
    local cmd="$1"
    shift
    
    # รอถ้า jobs เต็ม
    while [[ ${#_pids[@]} -ge $MAX_PARALLEL ]]; do
        for i in "${!_pids[@]}"; do
            if ! kill -0 "${_pids[$i]}" 2>/dev/null; then
                unset '_pids[$i]'
            fi
        done
        _pids=("${_pids[@]}")
        [[ ${#_pids[@]} -ge $MAX_PARALLEL ]] && sleep 0.1
    done
    
    "$cmd" "$@" &
    _pids+=($!)
}

wait_all() {
    for pid in "${_pids[@]}"; do
        wait "$pid" 2>/dev/null
    done
    _pids=()
}

# Process pool ที่มีประสิทธิภาพ
run_parallel_jobs() {
    local -a jobs=("$@")
    local -a pids=()
    local -a results=()
    local max_jobs=$MAX_PARALLEL
    
    for job in "${jobs[@]}"; do
        # จำกัดจำนวน parallel jobs
        while [[ $(jobs -r | wc -l) -ge $max_jobs ]]; do
            sleep 0.05
        done
        
        eval "$job" &
        pids+=($!)
    done
    
    # รอทุก job
    for pid in "${pids[@]}"; do
        wait "$pid"
        results+=($?)
    done
    
    echo "Results: ${results[*]}"
}

# Worker pool pattern
worker_pool() {
    local num_workers="$1"
    local input_file="$2"
    local output_file="$3"
    
    # สร้าง FIFO สำหรับ work queue
    local fifo="/tmp/work_queue_$$"
    mkfifo "$fifo"
    
    # Worker function
    worker() {
        local worker_id="$1"
        while IFS= read -r item; do
            # ประมวลผล item
            local result="${item^^}"  # ตัวอย่าง: แปลงเป็นตัวพิมพ์ใหญ่
            echo "Worker $worker_id: $result"
        done
        echo "Worker $worker_id done"
    }
    
    # เริ่ม workers
    local pids=()
    for ((i=1; i<=num_workers; i++)); do
        worker "$i" < "$fifo" >> "$output_file" &
        pids+=($!)
    done
    
    # ส่ง work items
    cat "$input_file" > "$fifo"
    
    # ปิด FIFO และรอ workers
    wait "${pids[@]}"
    rm -f "$fifo"
}

# Parallel download example
parallel_download() {
    local -a urls=("$@")
    local output_dir="/tmp/downloads_$$"
    mkdir -p "$output_dir"
    
    download_file() {
        local url="$1"
        local filename
        filename=$(basename "$url")
        echo "Downloading: $url"
        # curl -sL "$url" -o "$output_dir/$filename"
        sleep 1  # Simulate download
        echo "Done: $filename"
    }
    
    local pids=()
    for url in "${urls[@]}"; do
        download_file "$url" &
        pids+=($!)
        
        # ควบคุมจำนวน concurrent downloads
        if [[ ${#pids[@]} -ge $MAX_PARALLEL ]]; then
            wait "${pids[0]}"
            pids=("${pids[@]:1}")
        fi
    done
    
    wait "${pids[@]}"
    rm -rf "$output_dir"
}

# GNU Parallel integration (ถ้ามี)
use_gnu_parallel() {
    if ! command -v parallel &>/dev/null; then
        echo "GNU Parallel not installed"
        return 1
    fi
    
    # ตัวอย่าง: ประมวลผลไฟล์แบบ parallel
    # find . -name "*.log" | parallel --jobs 4 'gzip {}'
    # parallel --jobs 4 "process_item {}" ::: "${items[@]}"
    
    echo "GNU Parallel available"
}

# xargs parallel
xargs_parallel() {
    local -a items=("$@")
    printf '%s\n' "${items[@]}" | xargs -P "$MAX_PARALLEL" -I{} bash -c 'echo "Processing: {}"'
}

# Demo
echo "=== Parallel Processing Demo ==="
echo "Running 8 jobs with max $MAX_PARALLEL parallel..."

process_item() {
    local item="$1"
    sleep 0.5
    echo "Processed: $item"
}

for i in $(seq 1 8); do
    parallel_run process_item "item_$i"
done
wait_all
echo "All jobs completed"

echo ""
echo "=== xargs Parallel ==="
xargs_parallel "a" "b" "c" "d" "e" "f"
```

---

## ขั้นตอนที่ 410: Memory Optimization

```bash
#!/usr/bin/env bash
# memory_optimization.sh

# การจัดการ memory ใน Bash

# 1. ใช้ local variables เสมอใน functions
bad_function() {
    # ตัวแปรเหล่านี้เป็น global
    result=""
    temp=""
    count=0
}

good_function() {
    # ตัวแปรเหล่านี้เป็น local
    local result=""
    local temp=""
    local count=0
}

# 2. Unset ตัวแปรขนาดใหญ่เมื่อไม่ใช้แล้ว
process_large_data() {
    local large_data
    # อ่านข้อมูลขนาดใหญ่
    large_data=$(cat /dev/urandom | head -c 10000 | base64 2>/dev/null || echo "test data")
    
    # ประมวลผล
    local result="${#large_data}"
    
    # ปล่อย memory
    unset large_data
    
    echo "Processed: $result bytes"
}

# 3. Streaming แทน loading ทั้งหมด
# BAD: โหลดทั้งไฟล์
bad_count_words() {
    local content
    content=$(cat "$1")  # โหลดทั้งหมด
    echo "$content" | wc -w
}

# GOOD: streaming
good_count_words() {
    wc -w < "$1"  # ไม่โหลดเข้า memory
}

# 4. Array memory management
manage_arrays() {
    # สร้าง array ขนาดใหญ่
    local -a big_array
    for i in $(seq 1 1000); do
        big_array+=("item_$i")
    done
    
    echo "Array size: ${#big_array[@]}"
    
    # ประมวลผลและลบเมื่อเสร็จ
    local sum=0
    for item in "${big_array[@]}"; do
        ((sum++))
    done
    
    # ปล่อย memory
    unset big_array
    echo "Sum: $sum"
}

# 5. ใช้ nameref แทน copying arrays ขนาดใหญ่
process_array_efficient() {
    local -n array_ref=$1  # nameref - ไม่ copy
    local total=0
    
    for item in "${array_ref[@]}"; do
        ((total++))
    done
    
    echo "Total items: $total"
}

# 6. จำกัดขนาด array
bounded_queue() {
    local -a queue=()
    local max_size="$1"
    
    enqueue() {
        queue+=("$1")
        # เก็บแค่ N items ล่าสุด
        if [[ ${#queue[@]} -gt $max_size ]]; then
            queue=("${queue[@]: -$max_size}")
        fi
    }
    
    dequeue() {
        if [[ ${#queue[@]} -eq 0 ]]; then
            return 1
        fi
        echo "${queue[0]}"
        queue=("${queue[@]:1}")
    }
    
    for i in $(seq 1 20); do
        enqueue "item_$i"
    done
    
    echo "Queue size (max $max_size): ${#queue[@]}"
    echo "Items: ${queue[*]}"
}

# 7. Memory-efficient string building
# BAD: concatenation ใน loop
bad_string_build() {
    local result=""
    for i in $(seq 1 100); do
        result="${result}item_${i},"  # ช้ามากสำหรับ string ใหญ่
    done
    echo "${result%,}"
}

# GOOD: ใช้ array แล้ว join
good_string_build() {
    local -a parts=()
    for i in $(seq 1 100); do
        parts+=("item_${i}")
    done
    local IFS=','
    echo "${parts[*]}"
}

# Demo
echo "=== Memory Optimization Demo ==="
echo "Process large data:"
process_large_data

echo ""
echo "Manage arrays:"
manage_arrays

echo ""
echo "Bounded queue (max 5):"
bounded_queue 5

echo ""
echo "String building comparison:"
time bad_string_build > /dev/null
time good_string_build > /dev/null
```

---

## ขั้นตอนที่ 411: I/O Optimization

```bash
#!/usr/bin/env bash
# io_optimization.sh

# 1. Buffered output ด้วย printf แทน echo ซ้ำๆ
# BAD: echo ทุก record
bad_output() {
    for i in $(seq 1 1000); do
        echo "Record $i: data_$i"
    done
}

# GOOD: สะสมแล้ว output ครั้งเดียว
good_output() {
    local -a lines=()
    for i in $(seq 1 1000); do
        lines+=("Record $i: data_$i")
    done
    printf '%s\n' "${lines[@]}"
}

# BETTER: ใช้ heredoc หรือ printf format string
better_output() {
    printf 'Record %d: data_%d\n' $(seq 1 2 2000) 2>/dev/null || {
        for i in $(seq 1 1000); do
            printf 'Record %d: data_%d\n' "$i" "$i"
        done
    }
}

# 2. ใช้ tee สำหรับ multiple outputs
write_multiple() {
    local data="$1"
    echo "$data" | tee file1.txt file2.txt > /dev/null
    # แทน:
    # echo "$data" > file1.txt
    # echo "$data" > file2.txt
}

# 3. Batch file operations
batch_file_rename() {
    local dir="$1"
    local old_ext="$2"
    local new_ext="$3"
    
    # BAD: mv ทีละไฟล์
    # find "$dir" -name "*.$old_ext" | while read f; do mv "$f" "${f%.$old_ext}.$new_ext"; done
    
    # GOOD: ใช้ rename หรือ mmv ถ้ามี
    if command -v rename &>/dev/null; then
        rename "s/\.$old_ext$/.$new_ext/" "$dir"/*."$old_ext" 2>/dev/null
    else
        # หรือ loop แต่ลด overhead
        while IFS= read -r -d '' file; do
            mv "$file" "${file%.$old_ext}.$new_ext"
        done < <(find "$dir" -name "*.$old_ext" -print0)
    fi
}

# 4. ใช้ /dev/stdin /dev/stdout /dev/stderr
filter_and_transform() {
    # Pipeline ที่มีประสิทธิภาพ
    grep "ERROR" /var/log/syslog 2>/dev/null | 
        awk '{print $1, $2, $NF}' | 
        sort -u | 
        head -20
}

# 5. อ่านและเขียนพร้อมกัน
process_stream() {
    local input="$1"
    local output="$2"
    
    # ประมวลผล in-place แบบ memory efficient
    awk '{print toupper($0)}' "$input" > "$output"
    
    # หรือ in-place ด้วย temp file
    local tmp=$(mktemp)
    awk '{print toupper($0)}' "$input" > "$tmp" && mv "$tmp" "$input"
}

# 6. Disk I/O reduction ด้วย tmpfs
use_tmpfs() {
    local tmpfs_dir="/dev/shm"  # Linux shared memory
    
    if [[ -d "$tmpfs_dir" ]]; then
        local work_dir="$tmpfs_dir/bash_work_$$"
        mkdir -p "$work_dir"
        
        # ทำงานใน memory filesystem
        echo "Working in tmpfs: $work_dir"
        
        # สร้างและประมวลผลไฟล์ใน memory
        for i in $(seq 1 10); do
            echo "data_$i" > "$work_dir/file_$i.txt"
        done
        
        # รวมผล
        cat "$work_dir"/*.txt
        
        # Cleanup
        rm -rf "$work_dir"
    else
        echo "tmpfs not available, using /tmp"
    fi
}

# 7. Checkpointing สำหรับ long-running scripts
CHECKPOINT_FILE="/tmp/script_checkpoint_$$"

save_checkpoint() {
    local step="$1"
    local data="$2"
    echo "$step:$data" > "$CHECKPOINT_FILE"
}

load_checkpoint() {
    if [[ -f "$CHECKPOINT_FILE" ]]; then
        IFS=':' read -r step data < "$CHECKPOINT_FILE"
        echo "$step $data"
        return 0
    fi
    return 1
}

resume_from_checkpoint() {
    local start_step=1
    
    if checkpoint=$(load_checkpoint); then
        read -r start_step data <<< "$checkpoint"
        echo "Resuming from step $start_step"
    fi
    
    for ((step=start_step; step<=10; step++)); do
        echo "Processing step $step"
        save_checkpoint "$step" "completed"
        sleep 0.1
    done
    
    rm -f "$CHECKPOINT_FILE"
}

# Demo
echo "=== I/O Optimization Demo ==="
echo "Buffered output timing:"
time good_output > /dev/null

echo ""
echo "Using tmpfs:"
use_tmpfs

echo ""
echo "Checkpoint/Resume:"
resume_from_checkpoint
```

---

## ขั้นตอนที่ 412: String Processing Optimization

```bash
#!/usr/bin/env bash
# string_optimization.sh

# ใช้ Parameter Expansion ให้เต็มที่แทน external commands

# 1. String slicing
str="Hello, World!"

# Length
echo "Length: ${#str}"

# Substring
echo "Chars 7-11: ${str:7:5}"

# Find and replace (first)
echo "Replace first: ${str/World/Bash}"

# Find and replace (all)
echo "Replace all: ${str//l/L}"

# Remove prefix
path="/usr/local/bin/script.sh"
echo "No prefix /usr/: ${path#/usr/}"    # shortest match
echo "No prefix /usr: ${path##/usr*/}"   # longest match

# Remove suffix
echo "No .sh: ${path%.sh}"               # shortest match
echo "No /bin/*: ${path%/bin/*}"         # shortest match from end

# Upper/Lower case (Bash 4+)
word="hello world"
echo "Upper: ${word^^}"
echo "Lower: ${word,,}"
echo "Capitalize: ${word^}"
echo "First char upper: ${word^}"

# 2. Pattern matching
filename="report_2024_01_15.csv"

# Extract date part
date_part="${filename:7:10}"
echo "Date: $date_part"

# Check extension
ext="${filename##*.}"
echo "Extension: $ext"

# Check prefix
[[ "$filename" == report_* ]] && echo "Is a report"

# 3. String splitting
csv_line="John,Doe,30,Engineer"
IFS=',' read -ra fields <<< "$csv_line"
echo "First: ${fields[0]}, Last: ${fields[1]}"

# Split into chars
str="bash"
chars=()
for ((i=0; i<${#str}; i++)); do
    chars+=("${str:$i:1}")
done
echo "Chars: ${chars[*]}"

# 4. Trim whitespace (no fork)
trim() {
    local str="$1"
    # Remove leading whitespace
    str="${str#"${str%%[![:space:]]*}"}"
    # Remove trailing whitespace
    str="${str%"${str##*[![:space:]]}"}"
    echo "$str"
}

echo "Trimmed: '$(trim "  hello world  ")'"

# 5. Repeat string
repeat_str() {
    local str="$1" times="$2"
    local result=""
    for ((i=0; i<times; i++)); do
        result+="$str"
    done
    echo "$result"
}

echo "Repeated: $(repeat_str "ab" 5)"

# 6. Pad string
pad_left() {
    local str="$1" width="$2" char="${3:- }"
    printf "%${width}s" "$str" | tr ' ' "$char"
}

pad_right() {
    local str="$1" width="$2" char="${3:- }"
    printf "%-${width}s" "$str" | tr ' ' "$char"
}

echo "Pad left:  '$(pad_left "Hi" 10 "-")'"
echo "Pad right: '$(pad_right "Hi" 10 "-")'"

# 7. URL encoding/decoding ด้วย Bash
url_encode() {
    local string="$1"
    local encoded=""
    local char
    
    for ((i=0; i<${#string}; i++)); do
        char="${string:$i:1}"
        case "$char" in
            [a-zA-Z0-9._~-]) encoded+="$char" ;;
            *) printf -v encoded '%s%%%02X' "$encoded" "'$char" ;;
        esac
    done
    echo "$encoded"
}

url_decode() {
    local url="$1"
    printf '%b' "${url//%/\\x}"
}

echo "URL encode: $(url_encode "hello world & more")"
echo "URL decode: $(url_decode "hello%20world%20%26%20more")"

# 8. JSON-safe string escape
json_escape() {
    local str="$1"
    str="${str//\\/\\\\}"  # escape backslash
    str="${str//\"/\\\"}"  # escape double quote
    str="${str//$'\n'/\\n}"  # escape newline
    str="${str//$'\t'/\\t}"  # escape tab
    str="${str//$'\r'/\\r}"  # escape CR
    echo "$str"
}

echo "JSON escape: $(json_escape 'He said "hello"')"
```

---

## ขั้นตอนที่ 413: Profiling และ Benchmarking

```bash
#!/usr/bin/env bash
# profiling.sh

# 1. ใช้ PS4 สำหรับ tracing
enable_profiling() {
    PS4='+ $(date "+%s%N") ${BASH_SOURCE}:${LINENO}: '
    exec 3>&2 2>/tmp/bashprofile.txt
    set -x
}

disable_profiling() {
    set +x
    exec 2>&3 3>&-
    
    # วิเคราะห์ผล
    sort -t' ' -k2 -n /tmp/bashprofile.txt | tail -20
}

# 2. Simple benchmarking framework
benchmark() {
    local name="$1"
    local iterations="${2:-100}"
    shift 2
    
    local start end elapsed avg
    start=$(date +%s%N)
    
    for ((i=0; i<iterations; i++)); do
        "$@" > /dev/null 2>&1
    done
    
    end=$(date +%s%N)
    elapsed=$(( (end - start) / 1000000 ))
    avg=$(echo "scale=2; $elapsed / $iterations" | bc 2>/dev/null || echo "$((elapsed / iterations))")
    
    printf "%-30s Total: %6dms  Avg: %6sms  (%d iterations)\n" \
        "$name" "$elapsed" "$avg" "$iterations"
}

# 3. Function call overhead measurement
empty_function() { :; }

function_with_local() {
    local a b c
    :
}

function_with_echo() {
    echo "test"
}

# 4. String operation benchmarks
test_string_ops() {
    local str="Hello World Hello World Hello World"
    
    # Test substitution methods
    test_sed() { echo "$str" | sed 's/Hello/Hi/g'; }
    test_pe() { echo "${str//Hello/Hi}"; }
    
    benchmark "sed substitution" 100 test_sed
    benchmark "parameter expansion" 100 test_pe
}

# 5. Array vs string performance
test_array_string() {
    # Array operations
    test_array_append() {
        local -a arr=()
        for i in $(seq 1 100); do
            arr+=("item_$i")
        done
    }
    
    # String operations
    test_string_concat() {
        local str=""
        for i in $(seq 1 100); do
            str+="item_$i "
        done
    }
    
    benchmark "array append" 10 test_array_append
    benchmark "string concat" 10 test_string_concat
}

# 6. Command substitution overhead
test_cmd_substitution() {
    test_date_cmd() {
        local d=$(date +%s)
    }
    
    test_printf_date() {
        printf -v d '%(%s)T' -1
    }
    
    benchmark "date command" 50 test_date_cmd
    benchmark "printf date" 50 test_printf_date
}

# 7. Profiling with BASH_XTRACEFD
profile_script() {
    local script="$1"
    local profile_output="/tmp/profile_$$.txt"
    
    bash -x "$script" 2>"$profile_output"
    
    echo "Profile output:"
    echo "Most called lines:"
    sort "$profile_output" | uniq -c | sort -rn | head -10
    
    rm -f "$profile_output"
}

# 8. Memory usage tracking
measure_memory() {
    local pid=$$
    local before after
    
    before=$(awk '/VmRSS/{print $2}' /proc/$pid/status 2>/dev/null || echo 0)
    
    "$@"
    
    after=$(awk '/VmRSS/{print $2}' /proc/$pid/status 2>/dev/null || echo 0)
    echo "Memory change: $((after - before)) kB"
}

# Demo
echo "=== Benchmarking Demo ==="
echo ""

echo "Function call overhead:"
benchmark "empty function" 1000 empty_function
benchmark "function with locals" 1000 function_with_local
benchmark "function with echo" 1000 function_with_echo

echo ""
echo "String operations:"
test_string_ops

echo ""
echo "Command substitution:"
test_cmd_substitution
```

---

## ขั้นตอนที่ 414: Script Startup Optimization

```bash
#!/usr/bin/env bash
# startup_optimization.sh

# 1. Lazy loading - โหลดเฉพาะส่วนที่จำเป็น
declare -A _loaded_modules

load_module() {
    local module="$1"
    local module_file="${MODULES_DIR:-/etc/bash_modules}/${module}.sh"
    
    if [[ -z "${_loaded_modules[$module]+set}" ]]; then
        if [[ -f "$module_file" ]]; then
            source "$module_file"
            _loaded_modules[$module]=1
            echo "Loaded module: $module"
        else
            echo "Module not found: $module" >&2
            return 1
        fi
    fi
}

# Auto-load ด้วย command_not_found_handle
command_not_found_handle() {
    local cmd="$1"
    local module
    
    # หา module ที่อาจมี command นี้
    for module_file in "${MODULES_DIR:-/etc/bash_modules}"/*.sh; do
        if grep -q "^${cmd}()" "$module_file" 2>/dev/null; then
            module=$(basename "$module_file" .sh)
            load_module "$module"
            "$cmd" "${@:2}"
            return $?
        fi
    done
    
    echo "$cmd: command not found" >&2
    return 127
}

# 2. ตรวจสอบ dependencies ครั้งเดียวตอนเริ่ม
declare -A _checked_deps

check_dependency() {
    local cmd="$1"
    
    if [[ -z "${_checked_deps[$cmd]+set}" ]]; then
        if command -v "$cmd" &>/dev/null; then
            _checked_deps[$cmd]="$(command -v "$cmd")"
        else
            echo "Missing dependency: $cmd" >&2
            _checked_deps[$cmd]=""
            return 1
        fi
    fi
    
    [[ -n "${_checked_deps[$cmd]}" ]]
}

require_deps() {
    local missing=()
    for dep in "$@"; do
        check_dependency "$dep" || missing+=("$dep")
    done
    
    if [[ ${#missing[@]} -gt 0 ]]; then
        echo "Error: Missing required tools: ${missing[*]}" >&2
        echo "Install with: apt-get install ${missing[*]}" >&2
        exit 1
    fi
}

# 3. Configuration caching
CONFIG_CACHE_FILE="/tmp/config_cache_$$.sh"

load_config_once() {
    local config_file="$1"
    
    if [[ ! -f "$CONFIG_CACHE_FILE" || "$config_file" -nt "$CONFIG_CACHE_FILE" ]]; then
        # Re-parse config
        echo "# Generated: $(date)" > "$CONFIG_CACHE_FILE"
        while IFS='=' read -r key value; do
            [[ "$key" =~ ^[[:space:]]*# ]] && continue
            [[ -z "$key" ]] && continue
            # Sanitize
            key="${key//[^a-zA-Z0-9_]/}"
            echo "export ${key}=${value}" >> "$CONFIG_CACHE_FILE"
        done < "$config_file"
    fi
    
    source "$CONFIG_CACHE_FILE"
}

# 4. ใช้ hash สำหรับ command lookup caching
cache_commands() {
    # hash ทำให้ bash cache path ของ commands
    hash -r  # reset cache
    
    # Pre-cache commands ที่ใช้บ่อย
    local cmds=(awk sed grep find sort uniq cut tr)
    for cmd in "${cmds[@]}"; do
        hash "$cmd" 2>/dev/null
    done
    
    echo "Cached $(hash | wc -l) command paths"
}

# 5. ลด source calls
# BAD: source ทุกครั้งที่ใช้
bad_load_lib() {
    source /usr/local/lib/my_functions.sh
    my_function
}

# GOOD: source ครั้งเดียว
_LIB_LOADED=false
good_load_lib() {
    if ! $_LIB_LOADED; then
        source /usr/local/lib/my_functions.sh 2>/dev/null || true
        _LIB_LOADED=true
    fi
    my_function 2>/dev/null || echo "function not available"
}

# 6. Fast initialization
fast_init() {
    # ตั้งค่า IFS, locale, etc. ครั้งเดียว
    export LC_ALL=C  # เร็วกว่า UTF-8 สำหรับ text processing
    export LANG=C
    
    # ปิด features ที่ไม่ใช้
    set +o noglob    # disable globbing ถ้าไม่ใช้
    # set +o monitor  # disable job control ถ้าไม่ใช้
    
    echo "Fast init complete"
}

# Demo
echo "=== Startup Optimization Demo ==="
echo ""

echo "Check dependencies:"
require_deps bash awk sed grep

echo ""
echo "Cache commands:"
cache_commands

echo ""
echo "Fast init:"
fast_init
```

---

## ขั้นตอนที่ 415: Performance Testing Framework

```bash
#!/usr/bin/env bash
# perf_test_framework.sh
# Performance Testing Framework สำหรับ Bash Scripts

set -euo pipefail

# ===== Configuration =====
PERF_LOG="/tmp/perf_results_$$.csv"
PERF_ITERATIONS=100
PERF_WARMUP=5

# ===== Colors =====
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# ===== Core Functions =====

perf_start() {
    echo "timestamp,test_name,iteration,duration_ms,memory_kb" > "$PERF_LOG"
    echo -e "${BLUE}Performance Test Suite Started${NC}"
    echo "Log: $PERF_LOG"
    echo ""
}

perf_test() {
    local name="$1"
    local iterations="${2:-$PERF_ITERATIONS}"
    shift 2
    
    echo -e "${YELLOW}Testing: $name${NC}"
    
    # Warmup
    for ((i=0; i<PERF_WARMUP; i++)); do
        "$@" > /dev/null 2>&1
    done
    
    # Actual test
    local total_ms=0
    local min_ms=999999
    local max_ms=0
    local timestamp
    
    for ((iter=1; iter<=iterations; iter++)); do
        local start end duration_ms mem_kb
        
        start=$(date +%s%N)
        mem_before=$(awk '/VmRSS/{print $2}' /proc/$$/status 2>/dev/null || echo 0)
        
        "$@" > /dev/null 2>&1
        
        end=$(date +%s%N)
        mem_after=$(awk '/VmRSS/{print $2}' /proc/$$/status 2>/dev/null || echo 0)
        
        duration_ms=$(( (end - start) / 1000000 ))
        mem_kb=$((mem_after - mem_before))
        timestamp=$(date +%s)
        
        echo "$timestamp,$name,$iter,$duration_ms,$mem_kb" >> "$PERF_LOG"
        
        total_ms=$((total_ms + duration_ms))
        [[ $duration_ms -lt $min_ms ]] && min_ms=$duration_ms
        [[ $duration_ms -gt $max_ms ]] && max_ms=$duration_ms
    done
    
    local avg_ms=$((total_ms / iterations))
    
    printf "  Min: %dms  Max: %dms  Avg: %dms  Total: %dms\n" \
        "$min_ms" "$max_ms" "$avg_ms" "$total_ms"
    
    # Performance rating
    if [[ $avg_ms -lt 10 ]]; then
        echo -e "  Rating: ${GREEN}EXCELLENT${NC} (< 10ms)"
    elif [[ $avg_ms -lt 50 ]]; then
        echo -e "  Rating: ${GREEN}GOOD${NC} (< 50ms)"
    elif [[ $avg_ms -lt 200 ]]; then
        echo -e "  Rating: ${YELLOW}OK${NC} (< 200ms)"
    else
        echo -e "  Rating: ${RED}SLOW${NC} (>= 200ms)"
    fi
    echo ""
}

compare_tests() {
    local baseline_name="$1"
    local comparison_name="$2"
    
    local baseline_avg comparison_avg
    
    baseline_avg=$(awk -F',' -v name="$baseline_name" \
        '$2==name {sum+=$4; count++} END {print (count>0)?int(sum/count):0}' \
        "$PERF_LOG")
    
    comparison_avg=$(awk -F',' -v name="$comparison_name" \
        '$2==name {sum+=$4; count++} END {print (count>0)?int(sum/count):0}' \
        "$PERF_LOG")
    
    if [[ $baseline_avg -gt 0 && $comparison_avg -gt 0 ]]; then
        local speedup
        speedup=$(echo "scale=2; $baseline_avg / $comparison_avg" | bc 2>/dev/null || echo "N/A")
        
        echo -e "${BLUE}Comparison: $baseline_name vs $comparison_name${NC}"
        echo "  $baseline_name avg: ${baseline_avg}ms"
        echo "  $comparison_name avg: ${comparison_avg}ms"
        
        if (( $(echo "$speedup > 1" | bc -l 2>/dev/null || echo 0) )); then
            echo -e "  ${GREEN}${comparison_name} is ${speedup}x faster${NC}"
        else
            speedup=$(echo "scale=2; $comparison_avg / $baseline_avg" | bc 2>/dev/null || echo "N/A")
            echo -e "  ${RED}${comparison_name} is ${speedup}x slower${NC}"
        fi
        echo ""
    fi
}

perf_report() {
    echo -e "${BLUE}=== Performance Report ===${NC}"
    echo ""
    
    if [[ -f "$PERF_LOG" ]]; then
        echo "Summary by test:"
        awk -F',' 'NR>1 {
            sum[$2]+=$4; count[$2]++; 
            if (min[$2]=="" || $4<min[$2]) min[$2]=$4
            if (max[$2]=="" || $4>max[$2]) max[$2]=$4
        }
        END {
            printf "%-30s %8s %8s %8s\n", "Test", "Avg(ms)", "Min(ms)", "Max(ms)"
            printf "%-30s %8s %8s %8s\n", "----", "-------", "-------", "-------"
            for (t in sum) {
                printf "%-30s %8d %8d %8d\n", t, int(sum[t]/count[t]), min[t], max[t]
            }
        }' "$PERF_LOG" | sort -t' ' -k2 -n
    fi
    
    rm -f "$PERF_LOG"
}

# ===== Test Functions =====

# Functions to test
test_echo() { echo "Hello World"; }
test_printf() { printf "Hello World\n"; }
test_string_upper_cmd() { echo "hello world" | tr '[:lower:]' '[:upper:]'; }
test_string_upper_builtin() { local s="hello world"; echo "${s^^}"; }
test_subshell_date() { local d=$(date +%s); }
test_printf_date() { printf -v d '%(%s)T' -1; }
test_array_loop() { 
    local -a arr=($(seq 1 100))
    local sum=0
    for i in "${arr[@]}"; do ((sum+=i)); done
}
test_arithmetic_loop() {
    local sum=0
    for ((i=1; i<=100; i++)); do ((sum+=i)); done
}

# ===== Main =====
perf_start

perf_test "echo" 200 test_echo
perf_test "printf" 200 test_printf
compare_tests "echo" "printf"

perf_test "string_upper_command" 50 test_string_upper_cmd
perf_test "string_upper_builtin" 50 test_string_upper_builtin
compare_tests "string_upper_command" "string_upper_builtin"

perf_test "subshell_date" 100 test_subshell_date
perf_test "printf_date" 100 test_printf_date
compare_tests "subshell_date" "printf_date"

perf_test "array_loop" 20 test_array_loop
perf_test "arithmetic_loop" 20 test_arithmetic_loop
compare_tests "array_loop" "arithmetic_loop"

perf_report

echo ""
echo "=== สรุป Performance Best Practices ==="
echo "1. ใช้ Parameter Expansion แทน external commands"
echo "2. หลีกเลี่ยง Subshell (Command Substitution) ที่ไม่จำเป็น"
echo "3. ใช้ C-style loops สำหรับตัวเลข"
echo "4. ใช้ printf แทน echo สำหรับ portability"
echo "5. Cache ผลลัพธ์ที่คำนวณซ้ำ"
echo "6. ใช้ Parallel Processing สำหรับ independent tasks"
echo "7. อ่าน/เขียนไฟล์แบบ streaming ไม่ใช่ load ทั้งหมด"
echo "8. Benchmark ก่อน optimize"
```

---

## สรุป Part 20

### สิ่งที่เรียนรู้ในบทนี้ (Steps 404-415):

| Step | หัวข้อ | เทคนิคหลัก |
|------|--------|------------|
| 404 | Performance Basics | time command, profiling basics |
| 405 | Subshell Optimization | ลด fork, built-in operations |
| 406 | Built-in Commands | parameter expansion แทน sed/awk |
| 407 | File Reading | while read loop, mapfile, streaming |
| 408 | Caching/Memoization | memory cache, disk cache, TTL |
| 409 | Parallel Processing | background jobs, worker pool, xargs |
| 410 | Memory Optimization | local vars, unset, streaming |
| 411 | I/O Optimization | buffered output, tmpfs, checkpointing |
| 412 | String Optimization | parameter expansion, URL encode |
| 413 | Profiling/Benchmarking | PS4 tracing, benchmark framework |
| 414 | Startup Optimization | lazy loading, dependency cache |
| 415 | Performance Framework | complete testing suite |

### Performance Checklist:
- [ ] ใช้ `${var^^}` แทน `echo "$var" | tr`
- [ ] ใช้ `while IFS= read -r` แทน `for line in $(cat)`
- [ ] ใช้ `$(<file)` แทน `$(cat file)`
- [ ] Cache expensive computations
- [ ] ใช้ parallel processing สำหรับ I/O-bound tasks
- [ ] Profile ก่อน optimize

**ขั้นตอนต่อไป**: Part 21 - Testing และ Quality Assurance
