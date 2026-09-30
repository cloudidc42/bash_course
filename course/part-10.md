# Part 10: File Operations
## หลักสูตร Bash Script - ขั้นตอนที่ 261-290

---

## ขั้นตอนที่ 261: File Operations พื้นฐาน

### 261.1 Read และ Write Files

```bash
#!/bin/bash

# =============================
# Writing to Files
# =============================

# เขียนไฟล์ใหม่ (overwrite)
echo "Hello World" > /tmp/test.txt
cat > /tmp/test.txt << 'EOF'
Line 1
Line 2
Line 3
EOF

# Append ต่อท้ายไฟล์
echo "Line 4" >> /tmp/test.txt

# เขียนหลายบรรทัด
printf "Line A\nLine B\nLine C\n" > /tmp/test2.txt

# เขียนด้วย tee (แสดงและบันทึก)
echo "Hello" | tee /tmp/test3.txt
echo "Append" | tee -a /tmp/test3.txt

# =============================
# Reading Files
# =============================

# อ่านทั้งไฟล์
content=$(cat /tmp/test.txt)
echo "Content: $content"

# อ่านทีละบรรทัด (วิธีที่ถูกต้อง)
while IFS= read -r line; do
    echo "Line: $line"
done < /tmp/test.txt

# อ่านทีละบรรทัด + line number
line_num=0
while IFS= read -r line; do
    ((line_num++))
    printf "%3d: %s\n" "$line_num" "$line"
done < /tmp/test.txt

# อ่านบรรทัดแรก
first_line=$(head -1 /tmp/test.txt)
echo "First: $first_line"

# อ่านบรรทัดสุดท้าย
last_line=$(tail -1 /tmp/test.txt)
echo "Last: $last_line"

# อ่าน N บรรทัด
head -3 /tmp/test.txt

# อ่านตั้งแต่บรรทัดที่ N
tail -n +3 /tmp/test.txt
```

### 261.2 File Information

```bash
#!/bin/bash

# =============================
# File Metadata
# =============================

get_file_info() {
    local file="$1"
    
    if [ ! -e "$file" ]; then
        echo "ไม่พบไฟล์: $file"
        return 1
    fi
    
    echo "=== File Info: $file ==="
    
    # ประเภทไฟล์
    local type
    if [ -f "$file" ]; then
        type="regular file"
    elif [ -d "$file" ]; then
        type="directory"
    elif [ -L "$file" ]; then
        type="symbolic link"
    else
        type="other"
    fi
    echo "Type:       $type"
    
    # ขนาด
    local size
    size=$(stat -c%s "$file" 2>/dev/null || stat -f%z "$file")
    local human_size
    if (( size >= 1073741824 )); then
        human_size="$(echo "scale=1; $size/1073741824" | bc)G"
    elif (( size >= 1048576 )); then
        human_size="$(echo "scale=1; $size/1048576" | bc)M"
    elif (( size >= 1024 )); then
        human_size="$(echo "scale=1; $size/1024" | bc)K"
    else
        human_size="${size}B"
    fi
    echo "Size:       $size bytes ($human_size)"
    
    # Permissions
    local perms
    perms=$(stat -c%A "$file" 2>/dev/null || stat -f%Sp "$file")
    echo "Permissions: $perms"
    
    # Owner
    local owner
    owner=$(stat -c%U "$file" 2>/dev/null || stat -f%Su "$file")
    echo "Owner:      $owner"
    
    # Timestamps
    echo "Modified:   $(stat -c%y "$file" 2>/dev/null | cut -d. -f1)"
    echo "Accessed:   $(stat -c%x "$file" 2>/dev/null | cut -d. -f1)"
    echo "Created:    $(stat -c%w "$file" 2>/dev/null | cut -d. -f1)"
    
    # MIME type (ถ้ามี file command)
    if command -v file &>/dev/null; then
        echo "MIME type:  $(file --mime-type -b "$file")"
    fi
    
    # MD5/SHA checksum
    if [ -f "$file" ]; then
        if command -v md5sum &>/dev/null; then
            echo "MD5:        $(md5sum "$file" | cut -d' ' -f1)"
        fi
        if command -v sha256sum &>/dev/null; then
            echo "SHA256:     $(sha256sum "$file" | cut -d' ' -f1)"
        fi
    fi
}

get_file_info "/etc/hostname"
echo ""
get_file_info "/etc"
```

### 261.3 File Manipulation

```bash
#!/bin/bash

# =============================
# Advanced File Operations
# =============================

# Atomic write (เขียนแบบปลอดภัย)
atomic_write() {
    local target="$1"
    local content="$2"
    local temp_file
    
    # เขียนไป temp file ก่อน
    temp_file=$(mktemp "${target}.XXXXXX")
    
    echo "$content" > "$temp_file"
    
    # ตรวจสอบว่าเขียนสำเร็จ
    if [ $? -eq 0 ]; then
        # Move atomically
        mv -f "$temp_file" "$target"
        echo "Written atomically: $target"
    else
        rm -f "$temp_file"
        echo "Write failed!" >&2
        return 1
    fi
}

# Lock file สำหรับ concurrent access
with_lock() {
    local lockfile="$1"
    local timeout="${2:-10}"
    shift 2
    
    local fd=9
    eval "exec $fd>\"$lockfile\""
    
    # Try to get lock
    local elapsed=0
    while ! flock -n "$fd"; do
        if [ "$elapsed" -ge "$timeout" ]; then
            echo "Cannot acquire lock: $lockfile" >&2
            eval "exec ${fd}>&-"
            return 1
        fi
        sleep 0.1
        elapsed=$(echo "$elapsed + 0.1" | bc)
    done
    
    # Run command with lock held
    "$@"
    local result=$?
    
    # Release lock
    eval "exec ${fd}>&-"
    rm -f "$lockfile"
    
    return $result
}

# Safe file delete (ส่งไป trash)
safe_delete() {
    local file="$1"
    local trash_dir="${HOME}/.trash"
    
    if [ ! -e "$file" ]; then
        echo "ไม่พบไฟล์: $file"
        return 1
    fi
    
    mkdir -p "$trash_dir"
    
    local basename
    basename=$(basename "$file")
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)
    local trash_name="${trash_dir}/${basename}_${timestamp}"
    
    mv "$file" "$trash_name"
    echo "ย้ายไป trash: $trash_name"
}

# Copy with progress
copy_with_progress() {
    local src="$1"
    local dst="$2"
    
    if command -v rsync &>/dev/null; then
        rsync -ah --progress "$src" "$dst"
    else
        local size
        size=$(stat -c%s "$src" 2>/dev/null || stat -f%z "$src")
        echo "Copying $src → $dst ($(du -sh "$src" | cut -f1))"
        cp "$src" "$dst"
        echo "Done!"
    fi
}

# ทดสอบ
echo "=== File Operations Test ==="
atomic_write "/tmp/atomic_test.txt" "Hello from atomic write!"
cat /tmp/atomic_test.txt
```

---

## ขั้นตอนที่ 262: Directory Operations

### 262.1 Directory Traversal

```bash
#!/bin/bash

# =============================
# Directory Operations
# =============================

# ดู directory tree
show_tree() {
    local dir="${1:-.}"
    local prefix="${2:-}"
    local max_depth="${3:-5}"
    local current_depth="${4:-0}"
    
    [ "$current_depth" -gt "$max_depth" ] && return
    
    local items=("$dir"/*)
    local count="${#items[@]}"
    
    for ((i=0; i<count; i++)); do
        local item="${items[$i]}"
        [ -e "$item" ] || continue
        
        local is_last=$([ "$i" -eq $((count-1)) ] && echo true || echo false)
        local connector
        local new_prefix
        
        if [ "$is_last" = true ]; then
            connector="└── "
            new_prefix="${prefix}    "
        else
            connector="├── "
            new_prefix="${prefix}│   "
        fi
        
        local item_name
        item_name=$(basename "$item")
        
        if [ -d "$item" ]; then
            echo "${prefix}${connector}📁 ${item_name}/"
            show_tree "$item" "$new_prefix" "$max_depth" $((current_depth+1))
        elif [ -L "$item" ]; then
            local target
            target=$(readlink "$item")
            echo "${prefix}${connector}🔗 ${item_name} → $target"
        else
            local size
            size=$(du -sh "$item" 2>/dev/null | cut -f1)
            echo "${prefix}${connector}📄 ${item_name} ($size)"
        fi
    done
}

# สร้าง directory structure สำหรับ project
create_project_structure() {
    local project_name="$1"
    local project_type="${2:-web}"
    
    echo "Creating project: $project_name ($project_type)"
    
    case "$project_type" in
        web)
            mkdir -p "$project_name"/{
                src/{components,pages,styles,utils},
                public/{images,fonts},
                tests,
                docs,
                .github/workflows,
                scripts
            }
            touch "$project_name"/{README.md,.gitignore,package.json}
            touch "$project_name"/src/{index.js,app.js}
            ;;
        python)
            mkdir -p "$project_name"/{
                src/{models,views,controllers,utils},
                tests/{unit,integration},
                docs,
                scripts,
                data/{raw,processed}
            }
            touch "$project_name"/{README.md,.gitignore,requirements.txt,setup.py}
            touch "$project_name"/src/__init__.py
            ;;
        bash)
            mkdir -p "$project_name"/{
                bin,lib,tests,docs,
                .github/workflows
            }
            touch "$project_name"/{README.md,.gitignore}
            touch "$project_name"/bin/main.sh
            touch "$project_name"/lib/utils.sh
            ;;
    esac
    
    echo "Project structure created:"
    show_tree "$project_name" "" 3
}

# create_project_structure "myapp" "bash"

# =============================
# Directory Statistics
# =============================

dir_stats() {
    local dir="${1:-.}"
    
    echo "=== Directory Stats: $dir ==="
    
    # Total size
    local total_size
    total_size=$(du -sh "$dir" 2>/dev/null | cut -f1)
    echo "Total size: $total_size"
    
    # Count files and directories
    local file_count dir_count
    file_count=$(find "$dir" -type f 2>/dev/null | wc -l)
    dir_count=$(find "$dir" -type d 2>/dev/null | wc -l)
    echo "Files: $file_count"
    echo "Directories: $dir_count"
    
    # Largest files
    echo ""
    echo "Largest files:"
    find "$dir" -type f 2>/dev/null -exec du -sh {} \; | \
        sort -rh | head -5 | \
        awk '{printf "  %-10s %s\n", $1, $2}'
    
    # File types
    echo ""
    echo "File types:"
    find "$dir" -type f 2>/dev/null | \
        sed 's/.*\.//' | \
        sort | uniq -c | \
        sort -rn | head -10 | \
        awk '{printf "  %-10s %d files\n", $2, $1}'
    
    # Most recently modified
    echo ""
    echo "Recently modified:"
    find "$dir" -type f 2>/dev/null -printf '%TY-%Tm-%Td %TH:%TM %p\n' | \
        sort -r | head -5 | \
        awk '{printf "  %s %s  %s\n", $1, $2, $3}'
}
```

---

## ขั้นตอนที่ 263: File Synchronization

### 263.1 Sync และ Backup

```bash
#!/bin/bash

# =============================
# File Synchronization
# =============================

# Simple sync ด้วย rsync
sync_directories() {
    local src="$1"
    local dst="$2"
    local opts="${3:--av}"
    
    if ! command -v rsync &>/dev/null; then
        echo "ไม่มี rsync" >&2
        return 1
    fi
    
    echo "Syncing: $src → $dst"
    
    rsync $opts \
        --progress \
        --stats \
        --exclude='.git' \
        --exclude='node_modules' \
        --exclude='*.pyc' \
        --exclude='.DS_Store' \
        "$src/" "$dst/"
    
    echo "Sync complete"
}

# Incremental backup
incremental_backup() {
    local src="$1"
    local backup_base="$2"
    
    local date_stamp
    date_stamp=$(date '+%Y%m%d_%H%M%S')
    local backup_dir="${backup_base}/${date_stamp}"
    local latest_link="${backup_base}/latest"
    
    echo "Starting incremental backup: $src → $backup_dir"
    
    mkdir -p "$backup_dir"
    
    # ใช้ hard links กับ backup ก่อนหน้า
    local rsync_args="-a --stats"
    if [ -L "$latest_link" ]; then
        rsync_args+=" --link-dest=$(readlink -f "$latest_link")"
        echo "Using previous backup as reference: $(readlink "$latest_link")"
    fi
    
    rsync $rsync_args \
        --exclude='.git' \
        --exclude='*.tmp' \
        "$src/" "$backup_dir/"
    
    # อัปเดต latest symlink
    rm -f "$latest_link"
    ln -s "$backup_dir" "$latest_link"
    
    # คำนวณขนาด
    local size
    size=$(du -sh "$backup_dir" | cut -f1)
    echo "Backup complete: $backup_dir ($size)"
    
    # ลบ backup เก่า (เก็บไว้ 7 วัน)
    find "$backup_base" -maxdepth 1 -type d \
        -name "[0-9]*" \
        -mtime +7 \
        -exec echo "Removing old backup: {}" \; \
        -exec rm -rf {} \;
}

# Mirror ไฟล์ (ทำให้ dst เหมือน src)
mirror_directory() {
    local src="$1"
    local dst="$2"
    
    rsync -av --delete \
        --exclude='.git' \
        "$src/" "$dst/"
    
    echo "Mirror complete: $dst is now identical to $src"
}

# Watch directory for changes
watch_directory() {
    local dir="$1"
    local callback="$2"
    
    if command -v inotifywait &>/dev/null; then
        echo "Watching: $dir (using inotifywait)"
        inotifywait -m -r -e create,modify,delete,move "$dir" | \
        while IFS=' ' read -r directory event file; do
            echo "Change detected: $event $directory$file"
            "$callback" "$directory$file" "$event"
        done
    else
        echo "Watching: $dir (using polling)"
        local last_snapshot
        last_snapshot=$(find "$dir" -type f -printf '%p %T@\n' | md5sum)
        
        while true; do
            sleep 2
            local current_snapshot
            current_snapshot=$(find "$dir" -type f -printf '%p %T@\n' | md5sum)
            
            if [ "$current_snapshot" != "$last_snapshot" ]; then
                echo "Changes detected in $dir"
                "$callback" "$dir" "change"
                last_snapshot="$current_snapshot"
            fi
        done
    fi
}
```

---

## ขั้นตอนที่ 264: File Processing Pipeline

### 264.1 Batch File Processing

```bash
#!/bin/bash

# =============================
# Batch Processing
# =============================

# Process ไฟล์ด้วย transformations
process_files() {
    local src_dir="$1"
    local dst_dir="$2"
    local pattern="${3:-*.txt}"
    local transform_func="${4:-cat}"
    
    mkdir -p "$dst_dir"
    
    local processed=0
    local failed=0
    
    while IFS= read -r -d $'\0' file; do
        local basename
        basename=$(basename "$file")
        local dst_file="${dst_dir}/${basename}"
        
        echo -n "Processing: $basename... "
        
        if "$transform_func" "$file" > "$dst_file" 2>/dev/null; then
            echo "✓"
            ((processed++))
        else
            echo "✗"
            ((failed++))
        fi
        
    done < <(find "$src_dir" -name "$pattern" -type f -print0)
    
    echo ""
    echo "Processed: $processed files"
    echo "Failed: $failed files"
}

# Transformation functions
to_uppercase() {
    cat "$1" | tr '[:lower:]' '[:upper:]'
}

compress_file() {
    gzip -c "$1"
}

count_lines() {
    wc -l < "$1"
}

# CSV to JSON converter
csv_to_json() {
    local csv_file="$1"
    local first_line=true
    local headers=()
    
    echo "["
    local is_first_row=true
    
    while IFS=',' read -ra fields; do
        if [ "$first_line" = true ]; then
            headers=("${fields[@]}")
            first_line=false
            continue
        fi
        
        [ "$is_first_row" = false ] && echo ","
        is_first_row=false
        
        printf "  {"
        local first_field=true
        
        for i in "${!headers[@]}"; do
            [ "$first_field" = false ] && printf ", "
            first_field=false
            
            local key="${headers[$i]}"
            local value="${fields[$i]:-}"
            
            # Escape JSON strings
            value="${value//\\/\\\\}"
            value="${value//\"/\\\"}"
            
            printf '"%s": "%s"' "$key" "$value"
        done
        
        printf "}"
        
    done < "$csv_file"
    
    echo ""
    echo "]"
}

# สร้าง test CSV
cat > /tmp/test.csv << 'EOF'
Name,Age,City,Email
สมชาย,25,Bangkok,somchai@example.com
สมหญิง,30,Chiang Mai,somying@example.com
สมศักดิ์,35,Pattaya,somsak@example.com
EOF

echo "=== CSV to JSON ==="
csv_to_json /tmp/test.csv
```

---

## ขั้นตอนที่ 265: Workshop 10 - File Manager Application

### Workshop: Full-Featured File Manager

```bash
#!/usr/bin/env bash
# =============================================================================
# Workshop 10: file_manager_pro.sh
# Professional File Manager
# =============================================================================

set -euo pipefail

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
NC='\033[0m'

# Config
CURRENT_DIR="$PWD"
CLIPBOARD=""
CLIPBOARD_OP=""

# Display
cols=$(tput cols 2>/dev/null || echo 80)
lines=$(tput lines 2>/dev/null || echo 24)

# ============================
# File Operations
# ============================

list_directory() {
    local dir="${1:-$CURRENT_DIR}"
    
    echo -e "${CYAN}${BOLD}Directory: $dir${NC}"
    echo -e "${BLUE}$(printf '%.0s─' $(seq 1 "$cols"))${NC}"
    
    printf "%-5s %-10s %-10s %-20s %s\n" "Type" "Size" "Date" "Permissions" "Name"
    echo "$(printf '%.0s─' $(seq 1 "$cols"))"
    
    # Parent directory
    echo -e "${BLUE}..  (parent)${NC}"
    
    # List files
    local items=()
    while IFS= read -r -d $'\0' item; do
        items+=("$item")
    done < <(find "$dir" -maxdepth 1 -mindepth 1 -print0 | sort -z)
    
    for item in "${items[@]}"; do
        local name
        name=$(basename "$item")
        local size=""
        local date=""
        local perms=""
        local type_icon=""
        local color="$NC"
        
        # Get info
        if [ -d "$item" ]; then
            type_icon="📁"
            color="$BLUE"
            size="<dir>"
        elif [ -L "$item" ]; then
            type_icon="🔗"
            color="$CYAN"
            local target
            target=$(readlink "$item" 2>/dev/null)
            size="→ $target"
        elif [ -x "$item" ]; then
            type_icon="⚙"
            color="$GREEN"
            size=$(du -sh "$item" 2>/dev/null | cut -f1)
        else
            type_icon="📄"
            size=$(du -sh "$item" 2>/dev/null | cut -f1)
        fi
        
        # Format date
        date=$(stat -c%y "$item" 2>/dev/null | cut -d' ' -f1)
        perms=$(stat -c%A "$item" 2>/dev/null)
        
        printf "${color}%-5s %-10s %-10s %-20s %s${NC}\n" \
            "$type_icon" "$size" "$date" "$perms" "$name"
    done
    
    echo ""
    echo -e "Items: ${#items[@]} | Current: $CURRENT_DIR"
}

# Copy file/directory
copy_item() {
    local src="$1"
    local dst="$2"
    
    [ -e "$src" ] || { echo -e "${RED}ไม่พบ: $src${NC}"; return 1; }
    
    if [ -d "$src" ]; then
        cp -r "$src" "$dst"
    else
        cp "$src" "$dst"
    fi
    
    echo -e "${GREEN}✓ คัดลอก: $src → $dst${NC}"
}

# Move file/directory
move_item() {
    local src="$1"
    local dst="$2"
    
    [ -e "$src" ] || { echo -e "${RED}ไม่พบ: $src${NC}"; return 1; }
    
    mv "$src" "$dst"
    echo -e "${GREEN}✓ ย้าย: $src → $dst${NC}"
}

# Delete with confirmation
delete_item() {
    local item="$1"
    
    [ -e "$item" ] || { echo -e "${RED}ไม่พบ: $item${NC}"; return 1; }
    
    echo -e "${RED}จะลบ: $item${NC}"
    read -rp "ยืนยัน? (y/N): " confirm
    
    if [ "${confirm,,}" = "y" ]; then
        rm -rf "$item"
        echo -e "${GREEN}✓ ลบแล้ว: $item${NC}"
    else
        echo "ยกเลิก"
    fi
}

# Search files
search_files() {
    local dir="${1:-.}"
    local pattern="$2"
    local search_content="${3:-false}"
    
    echo -e "${CYAN}ค้นหา: '$pattern' ใน $dir${NC}"
    echo ""
    
    local count=0
    
    # ค้นหาตามชื่อ
    while IFS= read -r -d $'\0' file; do
        echo -e "  ${GREEN}📄${NC} $file"
        ((count++))
    done < <(find "$dir" -name "*$pattern*" -print0 2>/dev/null)
    
    # ค้นหาตามเนื้อหา
    if [ "$search_content" = true ] && command -v grep &>/dev/null; then
        echo ""
        echo -e "${CYAN}ไฟล์ที่มีเนื้อหา '$pattern':${NC}"
        while IFS= read -r file; do
            echo -e "  ${YELLOW}📝${NC} $file"
            ((count++))
        done < <(grep -rl "$pattern" "$dir" 2>/dev/null)
    fi
    
    echo ""
    echo "พบ $count รายการ"
}

# File info
show_file_info() {
    local file="$1"
    
    [ -e "$file" ] || { echo -e "${RED}ไม่พบ: $file${NC}"; return 1; }
    
    echo -e "${CYAN}=== ข้อมูลไฟล์: $file ===${NC}"
    
    echo "ชื่อ:       $(basename "$file")"
    echo "ที่อยู่:   $(dirname "$(realpath "$file")")"
    echo "ชนิด:      $(file --mime-type -b "$file" 2>/dev/null || echo "unknown")"
    echo "ขนาด:      $(du -sh "$file" 2>/dev/null | cut -f1) ($(stat -c%s "$file" 2>/dev/null) bytes)"
    echo "สิทธิ์:    $(stat -c%A "$file" 2>/dev/null)"
    echo "เจ้าของ:   $(stat -c%U "$file" 2>/dev/null)"
    echo "แก้ไขล่าสุด: $(stat -c%y "$file" 2>/dev/null | cut -d. -f1)"
    
    if [ -f "$file" ]; then
        echo "บรรทัด:    $(wc -l < "$file")"
        echo "คำ:        $(wc -w < "$file")"
        
        echo ""
        echo "MD5:       $(md5sum "$file" 2>/dev/null | cut -d' ' -f1)"
        
        echo ""
        echo "5 บรรทัดแรก:"
        head -5 "$file" | sed 's/^/  /'
    fi
}

# Compress files
compress_files() {
    local output="$1"
    shift
    local files=("$@")
    
    echo "บีบอัด: ${files[*]} → $output"
    
    case "$output" in
        *.tar.gz|*.tgz)
            tar -czf "$output" "${files[@]}"
            ;;
        *.tar.bz2|*.tbz2)
            tar -cjf "$output" "${files[@]}"
            ;;
        *.zip)
            zip -r "$output" "${files[@]}"
            ;;
        *.gz)
            gzip -c "${files[0]}" > "$output"
            ;;
        *)
            echo -e "${RED}ไม่รองรับ format: $output${NC}"
            return 1
            ;;
    esac
    
    local size
    size=$(du -sh "$output" | cut -f1)
    echo -e "${GREEN}✓ บีบอัดสำเร็จ: $output ($size)${NC}"
}

# ============================
# Interactive Shell
# ============================

show_help() {
    echo ""
    echo -e "${CYAN}=== คำสั่งที่ใช้ได้ ===${NC}"
    echo "  ls [dir]        แสดงไฟล์"
    echo "  cd <dir>        เปลี่ยน directory"
    echo "  cp <src> <dst>  คัดลอกไฟล์"
    echo "  mv <src> <dst>  ย้ายไฟล์"
    echo "  rm <path>       ลบไฟล์"
    echo "  info <file>     ดูข้อมูลไฟล์"
    echo "  search <pattern> ค้นหาไฟล์"
    echo "  zip <out> <files...> บีบอัดไฟล์"
    echo "  mkdir <dir>     สร้าง directory"
    echo "  pwd             แสดง directory ปัจจุบัน"
    echo "  tree [dir]      แสดง tree"
    echo "  help            แสดง help"
    echo "  exit            ออก"
    echo ""
}

run_interactive() {
    echo -e "${CYAN}${BOLD}"
    echo "╔══════════════════════════════════╗"
    echo "║     File Manager Pro v1.0        ║"
    echo "╚══════════════════════════════════╝"
    echo -e "${NC}"
    
    show_help
    list_directory "$CURRENT_DIR"
    
    while true; do
        echo ""
        echo -ne "${YELLOW}fm:${CYAN}${CURRENT_DIR}${NC}> "
        
        read -r cmd_line || break
        
        [ -z "$cmd_line" ] && continue
        
        # Parse command
        read -ra parts <<< "$cmd_line"
        local cmd="${parts[0]}"
        local args=("${parts[@]:1}")
        
        case "$cmd" in
            ls|dir)
                list_directory "${args[0]:-$CURRENT_DIR}"
                ;;
            cd)
                local target="${args[0]:-$HOME}"
                if [ -d "$target" ]; then
                    CURRENT_DIR=$(realpath "$target")
                    list_directory "$CURRENT_DIR"
                else
                    echo -e "${RED}ไม่พบ directory: $target${NC}"
                fi
                ;;
            cp|copy)
                [ ${#args[@]} -ge 2 ] || { echo "Usage: cp <src> <dst>"; continue; }
                copy_item "${args[0]}" "${args[1]}"
                ;;
            mv|move)
                [ ${#args[@]} -ge 2 ] || { echo "Usage: mv <src> <dst>"; continue; }
                move_item "${args[0]}" "${args[1]}"
                ;;
            rm|del)
                [ ${#args[@]} -ge 1 ] || { echo "Usage: rm <path>"; continue; }
                delete_item "${args[0]}"
                ;;
            info)
                [ ${#args[@]} -ge 1 ] || { echo "Usage: info <file>"; continue; }
                show_file_info "${args[0]}"
                ;;
            search|find)
                [ ${#args[@]} -ge 1 ] || { echo "Usage: search <pattern>"; continue; }
                search_files "$CURRENT_DIR" "${args[0]}"
                ;;
            zip)
                [ ${#args[@]} -ge 2 ] || { echo "Usage: zip <output> <files...>"; continue; }
                compress_files "${args[0]}" "${args[@]:1}"
                ;;
            mkdir)
                [ ${#args[@]} -ge 1 ] || { echo "Usage: mkdir <dir>"; continue; }
                mkdir -p "${args[0]}"
                echo -e "${GREEN}✓ สร้าง directory: ${args[0]}${NC}"
                ;;
            pwd)
                echo "$CURRENT_DIR"
                ;;
            tree)
                show_tree "${args[0]:-$CURRENT_DIR}" "" 3
                ;;
            help|?)
                show_help
                ;;
            exit|quit|q)
                echo -e "${GREEN}ออกจาก File Manager${NC}"
                break
                ;;
            "")
                ;;
            *)
                # ลองรัน command จริง
                if command -v "$cmd" &>/dev/null; then
                    eval "$cmd_line"
                else
                    echo -e "${RED}ไม่รู้จักคำสั่ง: $cmd${NC}"
                    echo "พิมพ์ 'help' เพื่อดูคำสั่งที่ใช้ได้"
                fi
                ;;
        esac
    done
}

# Run
run_interactive
```

---

## สรุป Module 1 (Part 01-10)

### ทบทวนสิ่งที่เรียนรู้

| Part | หัวข้อ | ทักษะหลัก |
|------|--------|-----------|
| 01 | แนะนำ Bash | ติดตั้ง, Hello World, Debugging |
| 02 | คำสั่งพื้นฐาน | ls, cp, mv, rm, grep, find |
| 03 | ตัวแปร | String, Integer, Array, Associative Array |
| 04 | Input/Output | read, printf, Redirection, Heredoc |
| 05 | เงื่อนไข | if-else, case, Operators |
| 06 | Loops | for, while, until, break, continue |
| 07 | Functions | Parameters, Return, Library, Patterns |
| 08 | Arrays | Algorithms, Stack, Queue, Graph |
| 09 | Strings | Processing, Regex, Template |
| 10 | Files | Read/Write, Sync, Backup |

### Module 1 Final Project

สร้าง **Personal Task Manager** ที่รวมทุกความรู้:

```bash
#!/usr/bin/env bash
# task_manager.sh - Personal Task Manager

# Features:
# 1. บันทึก/ดู/แก้ไข/ลบ tasks
# 2. กำหนด priority และ due date
# 3. Filter/Search tasks
# 4. Export เป็น CSV หรือ JSON
# 5. Statistics

# Implementation ใช้:
# - Arrays สำหรับเก็บ tasks
# - Associative arrays สำหรับ task data
# - Files สำหรับ persistence
# - Functions สำหรับ CRUD operations
# - Regex สำหรับ validation
# - Loops สำหรับ display
```

---

**ต่อไป:** [Part 11 - Regular Expressions และ grep](part-11.md)

*Module 1 จบแล้ว! เข้าสู่ Module 2 ระดับกลาง →*
