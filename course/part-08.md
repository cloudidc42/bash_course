# Part 08: Arrays และ Data Structures
## หลักสูตร Bash Script - ขั้นตอนที่ 201-230

---

## ขั้นตอนที่ 201: Arrays ขั้นสูง

### 201.1 Array Operations ครบถ้วน

```bash
#!/bin/bash

# สร้าง arrays
declare -a numbers=(10 20 30 40 50)
declare -a fruits=("apple" "banana" "cherry")

# ==============================
# การเข้าถึงข้อมูล
# ==============================

echo "Index 0: ${numbers[0]}"         # 10
echo "Last: ${numbers[-1]}"           # 50
echo "All: ${numbers[@]}"             # 10 20 30 40 50
echo "Length: ${#numbers[@]}"         # 5
echo "Indices: ${!numbers[@]}"        # 0 1 2 3 4

# Slice
echo "Slice [1:3]: ${numbers[@]:1:3}" # 20 30 40
echo "From index 2: ${numbers[@]:2}"  # 30 40 50

# ==============================
# การแก้ไข array
# ==============================

# เพิ่มท้าย
numbers+=("60" "70")
echo "After append: ${numbers[@]}"    # 10 20 30 40 50 60 70

# เพิ่มที่ index ที่กำหนด
numbers[7]="80"
numbers[10]="100"  # เว้นช่วง index ได้!
echo "Sparse: ${numbers[@]}"

# ลบ element
unset 'numbers[2]'  # ลบ index 2
echo "After unset: ${numbers[@]}"

# แทนที่ด้วย slice
numbers=("${numbers[@]:0:2}" "99" "${numbers[@]:3}")

# ==============================
# Copy และ Merge
# ==============================

# Copy array
copy=("${numbers[@]}")

# Merge arrays
arr1=("a" "b")
arr2=("c" "d")
merged=("${arr1[@]}" "${arr2[@]}")
echo "Merged: ${merged[@]}"

# ==============================
# Sort array
# ==============================

unsorted=("banana" "apple" "cherry" "date")

# Sort ascending
IFS=$'\n' sorted=($(sort <<< "${unsorted[*]}")); unset IFS
echo "Sorted: ${sorted[@]}"

# Sort numerics
nums=(5 2 8 1 9 3)
IFS=$'\n' sorted_nums=($(sort -n <<< "${nums[*]}")); unset IFS
echo "Sorted nums: ${sorted_nums[@]}"

# Sort descending
IFS=$'\n' desc=($(sort -r <<< "${unsorted[*]}")); unset IFS
echo "Desc: ${desc[@]}"
```

### 201.2 Array Algorithms

```bash
#!/bin/bash

# ===================================
# Search Algorithms
# ===================================

# Linear search
linear_search() {
    local target="$1"
    shift
    local arr=("$@")
    
    for ((i=0; i<${#arr[@]}; i++)); do
        if [ "${arr[$i]}" = "$target" ]; then
            echo "$i"
            return 0
        fi
    done
    
    echo "-1"
    return 1
}

# Binary search (array ต้องเรียงแล้ว)
binary_search() {
    local target="$1"
    shift
    local arr=("$@")
    local left=0
    local right=$(( ${#arr[@]} - 1 ))
    
    while [ "$left" -le "$right" ]; do
        local mid=$(( (left + right) / 2 ))
        
        if [ "${arr[$mid]}" -eq "$target" ]; then
            echo "$mid"
            return 0
        elif [ "${arr[$mid]}" -lt "$target" ]; then
            left=$(( mid + 1 ))
        else
            right=$(( mid - 1 ))
        fi
    done
    
    echo "-1"
    return 1
}

# ทดสอบ
nums=(1 3 5 7 9 11 13 15 17 19)
echo "Linear search 7: $(linear_search 7 "${nums[@]}")"
echo "Binary search 7: $(binary_search 7 "${nums[@]}")"
echo "Binary search 6: $(binary_search 6 "${nums[@]}")"

# ===================================
# Sort Algorithms
# ===================================

# Bubble sort
bubble_sort() {
    local -a arr=("$@")
    local n="${#arr[@]}"
    local swapped
    
    for ((i=0; i<n-1; i++)); do
        swapped=false
        for ((j=0; j<n-i-1; j++)); do
            if (( arr[j] > arr[j+1] )); then
                local temp="${arr[$j]}"
                arr[$j]="${arr[$((j+1))]}"
                arr[$((j+1))]="$temp"
                swapped=true
            fi
        done
        [ "$swapped" = false ] && break
    done
    
    echo "${arr[@]}"
}

# Quick sort
quicksort() {
    local -a arr=("$@")
    local n="${#arr[@]}"
    
    if [ "$n" -le 1 ]; then
        echo "${arr[@]}"
        return
    fi
    
    local pivot="${arr[0]}"
    local -a less=()
    local -a greater=()
    
    for ((i=1; i<n; i++)); do
        if (( arr[i] <= pivot )); then
            less+=("${arr[$i]}")
        else
            greater+=("${arr[$i]}")
        fi
    done
    
    local sorted_less sorted_greater
    if [ ${#less[@]} -gt 0 ]; then
        sorted_less=$(quicksort "${less[@]}")
    fi
    if [ ${#greater[@]} -gt 0 ]; then
        sorted_greater=$(quicksort "${greater[@]}")
    fi
    
    echo ${sorted_less:+"$sorted_less"} "$pivot" ${sorted_greater:+"$sorted_greater"}
}

# ทดสอบ
test_arr=(64 34 25 12 22 11 90)
echo "Bubble sort: $(bubble_sort "${test_arr[@]}")"

IFS=' ' read -ra result <<< "$(quicksort "${test_arr[@]}")"
echo "Quick sort: ${result[@]}"
```

---

## ขั้นตอนที่ 202: Data Structures

### 202.1 Stack Implementation

```bash
#!/bin/bash

# Stack implementation
declare -a _STACK=()
declare -g _STACK_NAME=""

stack.new() {
    local name="$1"
    _STACK_NAME="$name"
    declare -ga "_STACK_${name}=()"
}

stack.push() {
    local name="$1"
    local value="$2"
    local ref="_STACK_${name}"
    eval "${ref}+=(\"$value\")"
}

stack.pop() {
    local name="$1"
    local ref="_STACK_${name}"
    
    eval "local size=\${#${ref}[@]}"
    if [ "$size" -eq 0 ]; then
        echo "Stack empty!" >&2
        return 1
    fi
    
    eval "local top=\${${ref}[-1]}"
    eval "unset '${ref}[-1]'"
    echo "$top"
}

stack.peek() {
    local name="$1"
    local ref="_STACK_${name}"
    
    eval "local size=\${#${ref}[@]}"
    if [ "$size" -eq 0 ]; then
        echo "Stack empty!" >&2
        return 1
    fi
    
    eval "echo \${${ref}[-1]}"
}

stack.size() {
    local name="$1"
    local ref="_STACK_${name}"
    eval "echo \${#${ref}[@]}"
}

stack.is_empty() {
    local name="$1"
    [ "$(stack.size "$name")" -eq 0 ]
}

stack.dump() {
    local name="$1"
    local ref="_STACK_${name}"
    
    echo "Stack '$name' (size: $(stack.size "$name")):"
    eval "
        local size=\${#${ref}[@]}
        for ((i=size-1; i>=0; i--)); do
            echo \"  [\$i] \${${ref}[\$i]}\"
        done
    "
}

# ทดสอบ Stack
echo "=== Stack Test ==="
stack.new "mystack"
stack.push "mystack" "first"
stack.push "mystack" "second"
stack.push "mystack" "third"
stack.dump "mystack"
echo "Pop: $(stack.pop "mystack")"
echo "Peek: $(stack.peek "mystack")"
echo "Size: $(stack.size "mystack")"

# ใช้ Stack แก้ปัญหา: ตรวจสอบ balanced brackets
check_brackets() {
    local str="$1"
    local stack=()
    
    declare -A pairs=([")"]=1 ["]"]=1 ["}"]=1)
    declare -A open_close=(["("]=")" ["["]="]" ["{"]="}")
    
    for ((i=0; i<${#str}; i++)); do
        local char="${str:$i:1}"
        
        case "$char" in
            "("| "["| "{")
                stack+=("$char")
                ;;
            ")"| "]"| "}")
                if [ ${#stack[@]} -eq 0 ]; then
                    echo "ไม่สมดุล: '$char' ที่ตำแหน่ง $i ไม่มี opening bracket"
                    return 1
                fi
                
                local top="${stack[-1]}"
                local expected="${open_close[$top]}"
                
                if [ "$char" != "$expected" ]; then
                    echo "ไม่สมดุล: ต้องการ '$expected' แต่พบ '$char'"
                    return 1
                fi
                
                unset 'stack[-1]'
                ;;
        esac
    done
    
    if [ ${#stack[@]} -gt 0 ]; then
        echo "ไม่สมดุล: มี ${#stack[@]} opening brackets ที่ไม่มีคู่"
        return 1
    fi
    
    echo "สมดุล"
    return 0
}

echo ""
echo "=== Bracket Check ==="
check_brackets "(()[]{})"
check_brackets "(()"
check_brackets "([)]"
check_brackets "{[()]}"
```

### 202.2 Queue Implementation

```bash
#!/bin/bash

# Queue implementation (FIFO)
declare -a _QUEUE=()

queue.enqueue() {
    local value="$1"
    _QUEUE+=("$value")
    echo "Enqueued: $value"
}

queue.dequeue() {
    if [ ${#_QUEUE[@]} -eq 0 ]; then
        echo "Queue empty!" >&2
        return 1
    fi
    
    local front="${_QUEUE[0]}"
    _QUEUE=("${_QUEUE[@]:1}")
    echo "$front"
}

queue.front()    { [ ${#_QUEUE[@]} -gt 0 ] && echo "${_QUEUE[0]}" || echo "empty"; }
queue.size()     { echo "${#_QUEUE[@]}"; }
queue.is_empty() { [ ${#_QUEUE[@]} -eq 0 ]; }

queue.dump() {
    echo "Queue (size: ${#_QUEUE[@]}):"
    for i in "${!_QUEUE[@]}"; do
        printf "  [%d] %s\n" "$i" "${_QUEUE[$i]}"
    done
}

# ทดสอบ Queue
echo "=== Queue Test ==="
queue.enqueue "task1"
queue.enqueue "task2"
queue.enqueue "task3"
queue.dump

echo ""
echo "Processing queue:"
while ! queue.is_empty; do
    task=$(queue.dequeue)
    echo "Processing: $task"
done
```

### 202.3 Linked List Simulation

```bash
#!/bin/bash

# Linked List ด้วย associative arrays
declare -A _LIST_DATA=()
declare -A _LIST_NEXT=()
declare -g _LIST_HEAD=""
declare -g _LIST_TAIL=""
declare -g _LIST_SIZE=0
declare -g _LIST_COUNTER=0

list.add() {
    local value="$1"
    local node_id="node_$((_LIST_COUNTER++))"
    
    _LIST_DATA[$node_id]="$value"
    _LIST_NEXT[$node_id]=""
    
    if [ -z "$_LIST_HEAD" ]; then
        _LIST_HEAD="$node_id"
        _LIST_TAIL="$node_id"
    else
        _LIST_NEXT[$_LIST_TAIL]="$node_id"
        _LIST_TAIL="$node_id"
    fi
    
    ((_LIST_SIZE++))
}

list.traverse() {
    local current="$_LIST_HEAD"
    while [ -n "$current" ]; do
        echo "${_LIST_DATA[$current]}"
        current="${_LIST_NEXT[$current]}"
    done
}

list.size() { echo "$_LIST_SIZE"; }

# ทดสอบ
echo "=== Linked List ==="
list.add "first"
list.add "second"
list.add "third"
list.add "fourth"

echo "List contents:"
list.traverse

echo "Size: $(list.size)"
```

### 202.4 Hash Table

```bash
#!/bin/bash

# Hash Table (Bash associative arrays เป็น hash table อยู่แล้ว)
# แต่เราสร้าง wrapper ที่มี statistics

declare -A _HASH=()
declare -i _HASH_OPERATIONS=0

hash.set() {
    local key="$1"
    local value="$2"
    _HASH["$key"]="$value"
    ((_HASH_OPERATIONS++))
}

hash.get() {
    local key="$1"
    local default="${2:-}"
    echo "${_HASH[$key]:-$default}"
    ((_HASH_OPERATIONS++))
}

hash.has() {
    [[ "${_HASH[$1]+exists}" ]]
}

hash.delete() {
    unset '_HASH[$1]'
    ((_HASH_OPERATIONS++))
}

hash.size()  { echo "${#_HASH[@]}"; }
hash.keys()  { echo "${!_HASH[@]}"; }
hash.values(){ echo "${_HASH[@]}"; }

hash.dump() {
    echo "Hash Table (${#_HASH[@]} entries, $_HASH_OPERATIONS ops):"
    for key in "${!_HASH[@]}"; do
        printf "  %-20s → %s\n" "$key" "${_HASH[$key]}"
    done
}

# Word frequency counter using hash
count_words() {
    local text="$1"
    declare -A word_freq=()
    
    for word in $text; do
        # lowercase
        word="${word,,}"
        # remove punctuation
        word="${word//[^a-zA-Z0-9]/}"
        [ -z "$word" ] && continue
        
        ((word_freq[$word]++))
    done
    
    # sort by frequency
    echo "Word frequencies:"
    for word in "${!word_freq[@]}"; do
        echo "${word_freq[$word]} $word"
    done | sort -rn | head -10 | while read -r count word; do
        printf "  %-20s %d\n" "$word" "$count"
    done
}

text="the quick brown fox jumps over the lazy dog the fox"
count_words "$text"
```

---

## ขั้นตอนที่ 203: Working with Complex Data

### 203.1 Matrix Operations

```bash
#!/bin/bash

# Matrix using 1D array
declare -a MATRIX=()
declare -i MATRIX_ROWS=0
declare -i MATRIX_COLS=0

matrix.create() {
    MATRIX_ROWS="$1"
    MATRIX_COLS="$2"
    MATRIX=()
    
    # Initialize with zeros
    for ((i=0; i<MATRIX_ROWS * MATRIX_COLS; i++)); do
        MATRIX[$i]=0
    done
}

matrix.set() {
    local row="$1"
    local col="$2"
    local value="$3"
    MATRIX[$((row * MATRIX_COLS + col))]="$value"
}

matrix.get() {
    local row="$1"
    local col="$2"
    echo "${MATRIX[$((row * MATRIX_COLS + col))]}"
}

matrix.print() {
    echo "Matrix ${MATRIX_ROWS}x${MATRIX_COLS}:"
    for ((i=0; i<MATRIX_ROWS; i++)); do
        printf "  ["
        for ((j=0; j<MATRIX_COLS; j++)); do
            printf " %4d" "${MATRIX[$((i * MATRIX_COLS + j))]}"
        done
        printf " ]\n"
    done
}

# สร้างและใช้งาน matrix
matrix.create 3 3
matrix.set 0 0 1; matrix.set 0 1 2; matrix.set 0 2 3
matrix.set 1 0 4; matrix.set 1 1 5; matrix.set 1 2 6
matrix.set 2 0 7; matrix.set 2 1 8; matrix.set 2 2 9

matrix.print

# Matrix transpose
matrix.transpose() {
    local rows="$MATRIX_ROWS"
    local cols="$MATRIX_COLS"
    local -a temp=("${MATRIX[@]}")
    
    MATRIX_ROWS="$cols"
    MATRIX_COLS="$rows"
    
    for ((i=0; i<rows; i++)); do
        for ((j=0; j<cols; j++)); do
            MATRIX[$((j * rows + i))]="${temp[$((i * cols + j))]}"
        done
    done
}

echo ""
echo "Transposed:"
matrix.transpose
matrix.print
```

### 203.2 Graph Representation

```bash
#!/bin/bash

# Graph using adjacency list
declare -A _GRAPH=()

graph.add_vertex() {
    local vertex="$1"
    _GRAPH["$vertex"]="${_GRAPH[$vertex]:-}"
}

graph.add_edge() {
    local from="$1"
    local to="$2"
    
    graph.add_vertex "$from"
    graph.add_vertex "$to"
    
    _GRAPH["$from"]+=" $to"
}

graph.neighbors() {
    local vertex="$1"
    echo "${_GRAPH[$vertex]}"
}

graph.vertices() {
    echo "${!_GRAPH[@]}"
}

# BFS (Breadth-First Search)
graph.bfs() {
    local start="$1"
    declare -A visited=()
    local -a queue=("$start")
    local -a result=()
    
    visited[$start]=1
    
    while [ ${#queue[@]} -gt 0 ]; do
        local current="${queue[0]}"
        queue=("${queue[@]:1}")
        result+=("$current")
        
        for neighbor in $(graph.neighbors "$current"); do
            if [ -z "${visited[$neighbor]+x}" ]; then
                visited[$neighbor]=1
                queue+=("$neighbor")
            fi
        done
    done
    
    echo "${result[@]}"
}

# DFS (Depth-First Search)
graph.dfs() {
    local start="$1"
    declare -A visited=()
    
    _dfs_helper() {
        local node="$1"
        visited[$node]=1
        echo -n "$node "
        
        for neighbor in $(graph.neighbors "$node"); do
            if [ -z "${visited[$neighbor]+x}" ]; then
                _dfs_helper "$neighbor"
            fi
        done
    }
    
    _dfs_helper "$start"
    echo ""
}

# สร้าง graph ตัวอย่าง
echo "=== Graph ==="
graph.add_edge "A" "B"
graph.add_edge "A" "C"
graph.add_edge "B" "D"
graph.add_edge "C" "D"
graph.add_edge "D" "E"

echo "Vertices: $(graph.vertices)"
echo "BFS from A: $(graph.bfs "A")"
echo -n "DFS from A: "
graph.dfs "A"
```

---

## ขั้นตอนที่ 204: Workshop 08 - Shopping Cart System

### Workshop: ระบบตะกร้าสินค้า

```bash
#!/usr/bin/env bash
# =============================================================================
# Workshop 08: shopping_cart.sh
# ระบบตะกร้าสินค้า
# =============================================================================

set -euo pipefail

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
NC='\033[0m'

# ==============================
# Product Database
# ==============================
declare -A PRODUCTS=(
    ["P001"]="Apple:15.00:100"
    ["P002"]="Banana:8.50:200"
    ["P003"]="Cherry:45.00:50"
    ["P004"]="Dragon Fruit:80.00:30"
    ["P005"]="Elderberry:120.00:20"
)

# ==============================
# Shopping Cart
# ==============================
declare -A CART=()        # product_id -> quantity
declare -A CART_PRICE=()  # product_id -> price at time of add

# Get product info
get_product_name()  { echo "${PRODUCTS[$1]}" | cut -d: -f1; }
get_product_price() { echo "${PRODUCTS[$1]}" | cut -d: -f2; }
get_product_stock() { echo "${PRODUCTS[$1]}" | cut -d: -f3; }

# Add to cart
cart.add() {
    local product_id="$1"
    local qty="${2:-1}"
    
    # ตรวจสอบว่ามีสินค้า
    if [ -z "${PRODUCTS[$product_id]:-}" ]; then
        echo -e "${RED}ไม่พบสินค้า: $product_id${NC}"
        return 1
    fi
    
    local stock
    stock=$(get_product_stock "$product_id")
    local current_qty="${CART[$product_id]:-0}"
    local new_qty=$((current_qty + qty))
    
    # ตรวจสอบ stock
    if [ "$new_qty" -gt "$stock" ]; then
        echo -e "${RED}สินค้าไม่พอ: มีเพียง $stock ชิ้น${NC}"
        return 1
    fi
    
    CART[$product_id]="$new_qty"
    CART_PRICE[$product_id]=$(get_product_price "$product_id")
    
    local name
    name=$(get_product_name "$product_id")
    echo -e "${GREEN}✓ เพิ่ม '$name' x$qty ในตะกร้า${NC}"
}

# Remove from cart
cart.remove() {
    local product_id="$1"
    local qty="${2:-}"
    
    if [ -z "${CART[$product_id]:-}" ]; then
        echo -e "${RED}ไม่มีสินค้านี้ในตะกร้า${NC}"
        return 1
    fi
    
    local name
    name=$(get_product_name "$product_id")
    
    if [ -z "$qty" ]; then
        unset "CART[$product_id]"
        unset "CART_PRICE[$product_id]"
        echo -e "${YELLOW}ลบ '$name' ออกจากตะกร้าแล้ว${NC}"
    else
        local current_qty="${CART[$product_id]}"
        local new_qty=$((current_qty - qty))
        
        if [ "$new_qty" -le 0 ]; then
            unset "CART[$product_id]"
            unset "CART_PRICE[$product_id]"
        else
            CART[$product_id]="$new_qty"
        fi
        echo -e "${YELLOW}ลด '$name' เหลือ $((new_qty > 0 ? new_qty : 0)) ชิ้น${NC}"
    fi
}

# Calculate total
cart.total() {
    local total=0
    
    for product_id in "${!CART[@]}"; do
        local qty="${CART[$product_id]}"
        local price="${CART_PRICE[$product_id]}"
        local subtotal
        subtotal=$(echo "$qty * $price" | bc)
        total=$(echo "$total + $subtotal" | bc)
    done
    
    echo "$total"
}

# Display cart
cart.display() {
    echo ""
    echo -e "${CYAN}╔══════════════════════════════════════════════════════╗${NC}"
    echo -e "${CYAN}║                    ตะกร้าสินค้า                     ║${NC}"
    echo -e "${CYAN}╚══════════════════════════════════════════════════════╝${NC}"
    
    if [ ${#CART[@]} -eq 0 ]; then
        echo -e "  ${YELLOW}ตะกร้าว่างเปล่า${NC}"
        return
    fi
    
    printf "\n  %-8s %-20s %8s %10s %12s\n" "รหัส" "ชื่อสินค้า" "ราคา" "จำนวน" "รวม"
    echo "  ──────────────────────────────────────────────────────"
    
    local grand_total=0
    
    for product_id in "${!CART[@]}"; do
        local qty="${CART[$product_id]}"
        local price="${CART_PRICE[$product_id]}"
        local name
        name=$(get_product_name "$product_id")
        local subtotal
        subtotal=$(echo "scale=2; $qty * $price" | bc)
        grand_total=$(echo "scale=2; $grand_total + $subtotal" | bc)
        
        printf "  %-8s %-20s %8.2f %10d %12.2f\n" \
            "$product_id" "$name" "$price" "$qty" "$subtotal"
    done
    
    echo "  ──────────────────────────────────────────────────────"
    printf "  %-40s %12.2f\n" "รวมทั้งหมด (THB):" "$grand_total"
    echo ""
}

# Checkout
cart.checkout() {
    if [ ${#CART[@]} -eq 0 ]; then
        echo -e "${RED}ตะกร้าว่าง!${NC}"
        return 1
    fi
    
    cart.display
    
    local total
    total=$(cart.total)
    
    echo -e "${YELLOW}ยืนยันการสั่งซื้อ? ยอดรวม: ${total} THB${NC}"
    read -rp "ยืนยัน (y/n): " confirm
    
    if [ "${confirm,,}" = "y" ]; then
        # สร้าง order
        local order_id="ORD$(date +%Y%m%d%H%M%S)"
        local order_file="/tmp/${order_id}.txt"
        
        {
            echo "Order ID: $order_id"
            echo "Date: $(date '+%Y-%m-%d %H:%M:%S')"
            echo ""
            
            for product_id in "${!CART[@]}"; do
                local name qty price subtotal
                name=$(get_product_name "$product_id")
                qty="${CART[$product_id]}"
                price="${CART_PRICE[$product_id]}"
                subtotal=$(echo "scale=2; $qty * $price" | bc)
                echo "$name x$qty @ $price = $subtotal"
            done
            
            echo ""
            echo "Total: $total THB"
        } > "$order_file"
        
        # ล้างตะกร้า
        CART=()
        CART_PRICE=()
        
        echo -e "${GREEN}✓ สั่งซื้อสำเร็จ! Order ID: $order_id${NC}"
        echo -e "  บันทึกที่: $order_file"
    else
        echo "ยกเลิกการสั่งซื้อ"
    fi
}

# Show products
show_products() {
    echo ""
    echo -e "${BLUE}=== รายการสินค้า ===${NC}"
    printf "  %-8s %-20s %10s %8s\n" "รหัส" "ชื่อ" "ราคา" "คงเหลือ"
    echo "  ─────────────────────────────────────────"
    
    for id in $(echo "${!PRODUCTS[@]}" | tr ' ' '\n' | sort); do
        local name price stock
        name=$(get_product_name "$id")
        price=$(get_product_price "$id")
        stock=$(get_product_stock "$id")
        
        local stock_color="$NC"
        (( stock < 20 )) && stock_color="$YELLOW"
        (( stock < 5 ))  && stock_color="$RED"
        
        printf "  %-8s %-20s %10s ${stock_color}%8s${NC}\n" \
            "$id" "$name" "$price" "$stock"
    done
    echo ""
}

# Main program
main() {
    echo -e "${CYAN}ยินดีต้อนรับสู่ร้านค้าออนไลน์${NC}"
    
    while true; do
        echo ""
        echo -e "${BLUE}=== เมนูหลัก ===${NC}"
        echo "1. ดูสินค้าทั้งหมด"
        echo "2. เพิ่มสินค้าในตะกร้า"
        echo "3. ดูตะกร้าสินค้า"
        echo "4. ลบสินค้าออกจากตะกร้า"
        echo "5. ชำระเงิน"
        echo "6. ออก"
        echo ""
        read -rp "เลือก: " choice
        
        case "$choice" in
            1)  show_products ;;
            2)
                show_products
                read -rp "รหัสสินค้า: " pid
                read -rp "จำนวน: " qty
                cart.add "$pid" "$qty"
                ;;
            3)  cart.display ;;
            4)
                cart.display
                read -rp "รหัสสินค้าที่ต้องการลบ: " pid
                read -rp "จำนวน (Enter = ลบทั้งหมด): " qty
                cart.remove "$pid" "${qty:-}"
                ;;
            5)  cart.checkout ;;
            6)  echo "ขอบคุณที่ใช้บริการ!"; exit 0 ;;
            *)  echo -e "${RED}ตัวเลือกไม่ถูกต้อง${NC}" ;;
        esac
    done
}

main "$@"
```

---

## สรุป Part 08

### สิ่งที่เรียนรู้

1. ✅ Advanced array operations
2. ✅ Search algorithms (linear, binary)
3. ✅ Sort algorithms (bubble, quick)
4. ✅ Stack data structure
5. ✅ Queue data structure
6. ✅ Linked list simulation
7. ✅ Hash table
8. ✅ Matrix operations
9. ✅ Graph representation และ traversal

---

**ต่อไป:** [Part 09 - String Manipulation](part-09.md)

*Part 08 จบแล้ว! พร้อมเรียน Part 09 →*
