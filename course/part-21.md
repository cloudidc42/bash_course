# Part 21: Testing และ Quality Assurance

## Module 2: Intermediate Level
### ขั้นตอนที่ 416-427: การทดสอบและประกันคุณภาพ

---

## ขั้นตอนที่ 416: Unit Testing Basics สำหรับ Bash

```bash
#!/usr/bin/env bash
# unit_testing_basics.sh

# ===== Simple Test Framework =====
TESTS_PASSED=0
TESTS_FAILED=0
TESTS_TOTAL=0

# Test assertion functions
assert_equal() {
    local description="$1"
    local expected="$2"
    local actual="$3"
    
    ((TESTS_TOTAL++))
    
    if [[ "$expected" == "$actual" ]]; then
        ((TESTS_PASSED++))
        echo "  PASS: $description"
    else
        ((TESTS_FAILED++))
        echo "  FAIL: $description"
        echo "        Expected: '$expected'"
        echo "        Actual:   '$actual'"
    fi
}

assert_not_equal() {
    local description="$1"
    local unexpected="$2"
    local actual="$3"
    
    ((TESTS_TOTAL++))
    
    if [[ "$unexpected" != "$actual" ]]; then
        ((TESTS_PASSED++))
        echo "  PASS: $description"
    else
        ((TESTS_FAILED++))
        echo "  FAIL: $description (should not equal '$unexpected')"
    fi
}

assert_contains() {
    local description="$1"
    local substring="$2"
    local string="$3"
    
    ((TESTS_TOTAL++))
    
    if [[ "$string" == *"$substring"* ]]; then
        ((TESTS_PASSED++))
        echo "  PASS: $description"
    else
        ((TESTS_FAILED++))
        echo "  FAIL: $description"
        echo "        Expected '$string' to contain '$substring'"
    fi
}

assert_true() {
    local description="$1"
    shift
    
    ((TESTS_TOTAL++))
    
    if "$@" 2>/dev/null; then
        ((TESTS_PASSED++))
        echo "  PASS: $description"
    else
        ((TESTS_FAILED++))
        echo "  FAIL: $description"
        echo "        Command failed: $*"
    fi
}

assert_false() {
    local description="$1"
    shift
    
    ((TESTS_TOTAL++))
    
    if ! "$@" 2>/dev/null; then
        ((TESTS_PASSED++))
        echo "  PASS: $description"
    else
        ((TESTS_FAILED++))
        echo "  FAIL: $description"
        echo "        Command should have failed: $*"
    fi
}

assert_exit_code() {
    local description="$1"
    local expected_code="$2"
    shift 2
    
    ((TESTS_TOTAL++))
    
    "$@" > /dev/null 2>&1
    local actual_code=$?
    
    if [[ $actual_code -eq $expected_code ]]; then
        ((TESTS_PASSED++))
        echo "  PASS: $description"
    else
        ((TESTS_FAILED++))
        echo "  FAIL: $description"
        echo "        Expected exit code: $expected_code"
        echo "        Actual exit code:   $actual_code"
    fi
}

assert_file_exists() {
    local description="$1"
    local file="$2"
    
    ((TESTS_TOTAL++))
    
    if [[ -f "$file" ]]; then
        ((TESTS_PASSED++))
        echo "  PASS: $description"
    else
        ((TESTS_FAILED++))
        echo "  FAIL: $description"
        echo "        File not found: $file"
    fi
}

assert_output() {
    local description="$1"
    local expected="$2"
    shift 2
    
    local actual
    actual=$("$@" 2>&1)
    assert_equal "$description" "$expected" "$actual"
}

# Test runner
run_test() {
    local test_name="$1"
    echo ""
    echo "Running: $test_name"
    "$test_name"
}

# Print summary
test_summary() {
    echo ""
    echo "======================================="
    echo "Test Results: $TESTS_PASSED/$TESTS_TOTAL passed"
    if [[ $TESTS_FAILED -gt 0 ]]; then
        echo "FAILED: $TESTS_FAILED tests"
        return 1
    else
        echo "ALL TESTS PASSED"
        return 0
    fi
}

# ===== Functions to Test =====
add_numbers() {
    echo $(( $1 + $2 ))
}

is_even() {
    (( $1 % 2 == 0 ))
}

to_uppercase() {
    echo "${1^^}"
}

trim_string() {
    local str="$1"
    str="${str#"${str%%[![:space:]]*}"}"
    str="${str%"${str##*[![:space:]]}"}"
    echo "$str"
}

# ===== Test Suites =====
test_arithmetic() {
    assert_output "add 2+3" "5" add_numbers 2 3
    assert_output "add 0+0" "0" add_numbers 0 0
    assert_output "add negative" "-1" add_numbers -3 2
    assert_output "large numbers" "1000" add_numbers 500 500
}

test_string_functions() {
    assert_output "uppercase hello" "HELLO" to_uppercase "hello"
    assert_output "uppercase mixed" "HELLO WORLD" to_uppercase "hello world"
    assert_output "trim spaces" "hello" trim_string "  hello  "
    assert_output "trim tabs" "world" trim_string $'\t\tworld\t\t'
}

test_predicates() {
    assert_true "2 is even" is_even 2
    assert_true "100 is even" is_even 100
    assert_false "3 is not even" is_even 3
    assert_false "7 is not even" is_even 7
}

# Run tests
run_test test_arithmetic
run_test test_string_functions
run_test test_predicates
test_summary
```

---

## ขั้นตอนที่ 417: Bats - Bash Automated Testing System

```bash
#!/usr/bin/env bash
# bats_setup.sh
# BATS คือ framework ยอดนิยมสำหรับ testing Bash

# การติดตั้ง BATS
install_bats() {
    if command -v bats &>/dev/null; then
        echo "BATS already installed: $(bats --version)"
        return
    fi
    
    # วิธีที่ 1: npm
    if command -v npm &>/dev/null; then
        npm install -g bats
        return
    fi
    
    # วิธีที่ 2: git clone
    git clone https://github.com/bats-core/bats-core.git /tmp/bats-core
    /tmp/bats-core/install.sh /usr/local
    
    # วิธีที่ 3: apt (Debian/Ubuntu)
    # sudo apt-get install bats
}

# BATS test file structure
create_bats_example() {
    cat > /tmp/example.bats << 'EOF'
#!/usr/bin/env bats

# Load helper libraries (ถ้ามี)
# load 'bats-support/load'
# load 'bats-assert/load'

# Setup - รันก่อน test แต่ละตัว
setup() {
    TEST_DIR="$(mktemp -d)"
    export TEST_DIR
}

# Teardown - รันหลัง test แต่ละตัว
teardown() {
    rm -rf "$TEST_DIR"
}

# Test cases
@test "addition using bc" {
    result="$(echo 2+2 | bc)"
    [ "$result" -eq 4 ]
}

@test "create file successfully" {
    run touch "$TEST_DIR/test.txt"
    [ "$status" -eq 0 ]
    [ -f "$TEST_DIR/test.txt" ]
}

@test "echo outputs correct string" {
    run echo "Hello, World!"
    [ "$status" -eq 0 ]
    [ "$output" = "Hello, World!" ]
}

@test "grep finds pattern" {
    echo "apple" > "$TEST_DIR/fruits.txt"
    echo "banana" >> "$TEST_DIR/fruits.txt"
    
    run grep "apple" "$TEST_DIR/fruits.txt"
    [ "$status" -eq 0 ]
    [[ "$output" =~ "apple" ]]
}

@test "command fails for nonexistent file" {
    run cat /nonexistent/file
    [ "$status" -ne 0 ]
}

@test "string manipulation" {
    local str="hello world"
    run bash -c "echo \"\${str^^}\"" -- "$str"
    [ "$output" = "HELLO WORLD" ]
}

# Parameterized tests ด้วย loop
for i in 1 2 3; do
    @test "iteration $i" {
        [ "$i" -gt 0 ]
    }
done
EOF
    echo "Created: /tmp/example.bats"
}

# โครงสร้าง test suite
create_test_structure() {
    mkdir -p /tmp/bash_tests/{test,lib,fixtures}
    
    # Test helper
    cat > /tmp/bash_tests/lib/helpers.bash << 'EOF'
# Test helpers

# Create temp directory
create_temp_dir() {
    mktemp -d
}

# Assert file contains text
file_contains() {
    local file="$1" text="$2"
    grep -q "$text" "$file"
}

# Mock command
mock_command() {
    local cmd="$1"
    local output="$2"
    eval "${cmd}() { echo '$output'; return 0; }"
}
EOF
    
    # Main test file
    cat > /tmp/bash_tests/test/main.bats << 'EOF'
#!/usr/bin/env bats

# Load helpers
load '../lib/helpers'

setup() {
    TEMP_DIR=$(create_temp_dir)
}

teardown() {
    rm -rf "$TEMP_DIR"
}

@test "helper creates temp dir" {
    [ -d "$TEMP_DIR" ]
}

@test "file_contains works" {
    echo "test content" > "$TEMP_DIR/file.txt"
    file_contains "$TEMP_DIR/file.txt" "content"
}
EOF
    
    echo "Test structure created at /tmp/bash_tests/"
}

# สร้างไฟล์
create_bats_example
create_test_structure

echo "BATS examples created"
echo "Run with: bats /tmp/example.bats"
echo "Run suite: bats /tmp/bash_tests/test/"
```

---

## ขั้นตอนที่ 418: Test Fixtures และ Mocking

```bash
#!/usr/bin/env bash
# test_fixtures_mocking.sh

# ===== Test Fixtures =====
FIXTURE_DIR="/tmp/test_fixtures_$$"

setup_fixtures() {
    mkdir -p "$FIXTURE_DIR"/{data,config,logs}
    
    # สร้าง fixture files
    cat > "$FIXTURE_DIR/data/users.csv" << 'EOF'
id,name,email,age
1,Alice,alice@example.com,30
2,Bob,bob@example.com,25
3,Charlie,charlie@example.com,35
EOF
    
    cat > "$FIXTURE_DIR/config/app.conf" << 'EOF'
DB_HOST=localhost
DB_PORT=5432
DB_NAME=testdb
LOG_LEVEL=info
MAX_CONNECTIONS=10
EOF
    
    cat > "$FIXTURE_DIR/logs/app.log" << 'EOF'
2024-01-15 10:00:00 INFO Application started
2024-01-15 10:01:00 ERROR Database connection failed
2024-01-15 10:02:00 INFO Retrying connection
2024-01-15 10:03:00 ERROR Auth failed for user: testuser
2024-01-15 10:04:00 INFO Application ready
EOF
}

teardown_fixtures() {
    rm -rf "$FIXTURE_DIR"
}

# ===== Command Mocking =====

# Mock ด้วย function override
mock_curl() {
    local url="$1"
    case "$url" in
        *"api.example.com/users"*)
            echo '{"users": [{"id": 1, "name": "Alice"}]}'
            return 0
            ;;
        *"api.example.com/error"*)
            echo '{"error": "Not Found"}'
            return 404
            ;;
        *)
            echo '{"status": "unknown"}'
            return 0
            ;;
    esac
}

# Mock ด้วย PATH manipulation
setup_mock_bin() {
    local mock_dir="/tmp/mock_bin_$$"
    mkdir -p "$mock_dir"
    
    # สร้าง mock curl
    cat > "$mock_dir/curl" << 'EOF'
#!/usr/bin/env bash
echo '{"mocked": true}'
exit 0
EOF
    chmod +x "$mock_dir/curl"
    
    # เพิ่มไปยัง PATH
    export ORIGINAL_PATH="$PATH"
    export PATH="$mock_dir:$PATH"
    
    echo "$mock_dir"
}

teardown_mock_bin() {
    local mock_dir="$1"
    export PATH="$ORIGINAL_PATH"
    rm -rf "$mock_dir"
}

# Spy - บันทึกการเรียกใช้
declare -a CALL_LOG=()

spy_function() {
    local func_name="$1"
    local original_func
    original_func=$(declare -f "$func_name")
    
    # Rename original
    eval "original_${func_name}() ${original_func#*\(\)}"
    
    # Create spy
    eval "${func_name}() {
        CALL_LOG+=(\"${func_name}: \$*\")
        original_${func_name} \"\$@\"
    }"
}

# ===== Function Under Test =====
process_users_csv() {
    local file="$1"
    local count=0
    while IFS=',' read -r id name email age; do
        [[ "$id" == "id" ]] && continue  # Skip header
        ((count++))
    done < "$file"
    echo "$count"
}

load_config() {
    local file="$1"
    source "$file"
}

parse_log_errors() {
    local file="$1"
    grep "ERROR" "$file" | awk '{print $4, $NF}'
}

# ===== Tests =====
run_tests() {
    local passed=0 failed=0
    
    pass() { ((passed++)); echo "  PASS: $1"; }
    fail() { ((failed++)); echo "  FAIL: $1 - $2"; }
    
    echo "=== Fixture Tests ==="
    
    # Test CSV processing
    local user_count
    user_count=$(process_users_csv "$FIXTURE_DIR/data/users.csv")
    [[ "$user_count" -eq 3 ]] && pass "CSV has 3 users" || fail "CSV count" "got $user_count"
    
    # Test config loading
    load_config "$FIXTURE_DIR/config/app.conf"
    [[ "$DB_HOST" == "localhost" ]] && pass "Config DB_HOST" || fail "Config" "DB_HOST=$DB_HOST"
    [[ "$DB_PORT" == "5432" ]] && pass "Config DB_PORT" || fail "Config" "DB_PORT=$DB_PORT"
    
    # Test log parsing
    local errors
    errors=$(parse_log_errors "$FIXTURE_DIR/logs/app.log")
    [[ $(echo "$errors" | wc -l) -eq 2 ]] && pass "Found 2 errors" || fail "Error count" "got $(echo "$errors" | wc -l)"
    
    echo ""
    echo "=== Mock Tests ==="
    
    # Test with mocked curl
    curl() { mock_curl "$@"; }
    
    local api_result
    api_result=$(curl "https://api.example.com/users")
    [[ "$api_result" == *"Alice"* ]] && pass "Mock API returns user" || fail "Mock API" "$api_result"
    
    unset -f curl
    
    echo ""
    echo "Results: $passed passed, $failed failed"
    [[ $failed -eq 0 ]]
}

# Main
setup_fixtures
run_tests
exit_code=$?
teardown_fixtures
exit $exit_code
```

---

## ขั้นตอนที่ 419: Integration Testing

```bash
#!/usr/bin/env bash
# integration_testing.sh

# ===== Integration Test Framework =====
INTEGRATION_LOG="/tmp/integration_test_$$.log"
TEST_ENV_DIR="/tmp/test_env_$$"

setup_test_environment() {
    mkdir -p "$TEST_ENV_DIR"/{bin,etc,var/log,tmp}
    
    # จำลอง environment
    export TEST_HOME="$TEST_ENV_DIR"
    export TEST_CONFIG="$TEST_ENV_DIR/etc/config"
    export TEST_LOG="$TEST_ENV_DIR/var/log/app.log"
    
    # สร้าง config
    cat > "$TEST_CONFIG" << EOF
APP_NAME=test_app
APP_ENV=test
LOG_FILE=$TEST_ENV_DIR/var/log/app.log
DATA_DIR=$TEST_ENV_DIR/var/data
EOF
    
    echo "Test environment ready: $TEST_ENV_DIR"
}

teardown_test_environment() {
    rm -rf "$TEST_ENV_DIR"
}

# ===== Test the whole flow =====
# Simulated application components
app_init() {
    source "$TEST_CONFIG"
    mkdir -p "$DATA_DIR"
    touch "$LOG_FILE"
    echo "$(date): Application initialized" >> "$LOG_FILE"
}

app_process_input() {
    local input="$1"
    echo "$(date): Processing: $input" >> "$TEST_LOG"
    
    # Validate
    if [[ -z "$input" ]]; then
        echo "ERROR: Empty input" >&2
        return 1
    fi
    
    # Process
    local result="${input^^}"
    echo "$result" >> "$DATA_DIR/results.txt"
    echo "$result"
}

app_get_results() {
    if [[ -f "$DATA_DIR/results.txt" ]]; then
        cat "$DATA_DIR/results.txt"
    fi
}

# ===== Integration Tests =====
declare -i IT_PASSED=0
declare -i IT_FAILED=0

it_assert() {
    local desc="$1"
    shift
    
    if "$@" 2>/dev/null; then
        ((IT_PASSED++))
        echo "  [PASS] $desc"
    else
        ((IT_FAILED++))
        echo "  [FAIL] $desc"
        return 1
    fi
}

test_full_workflow() {
    echo "=== Integration Test: Full Workflow ==="
    
    # Test 1: Initialize application
    it_assert "App initializes successfully" app_init
    it_assert "Config loaded" [[ -n "$APP_NAME" ]]
    it_assert "Log file created" [[ -f "$TEST_LOG" ]]
    it_assert "Data dir created" [[ -d "$DATA_DIR" ]]
    
    # Test 2: Process inputs
    local result1
    result1=$(app_process_input "hello")
    it_assert "Process hello returns HELLO" [[ "$result1" == "HELLO" ]]
    it_assert "Result written to file" grep -q "HELLO" "$DATA_DIR/results.txt"
    
    local result2
    result2=$(app_process_input "world")
    it_assert "Process world returns WORLD" [[ "$result2" == "WORLD" ]]
    
    # Test 3: Error handling
    it_assert "Empty input fails" ! app_process_input ""
    
    # Test 4: Retrieve results
    local all_results
    all_results=$(app_get_results)
    it_assert "Get results has HELLO" [[ "$all_results" == *"HELLO"* ]]
    it_assert "Get results has WORLD" [[ "$all_results" == *"WORLD"* ]]
    it_assert "Has 2 results" [[ $(echo "$all_results" | wc -l) -eq 2 ]]
    
    # Test 5: Log entries created
    it_assert "Log has init entry" grep -q "initialized" "$TEST_LOG"
    it_assert "Log has processing entries" grep -q "Processing" "$TEST_LOG"
}

test_concurrent_access() {
    echo ""
    echo "=== Integration Test: Concurrent Access ==="
    
    # รัน multiple processes พร้อมกัน
    local pids=()
    local results_dir="$TEST_ENV_DIR/concurrent_results"
    mkdir -p "$results_dir"
    
    for i in $(seq 1 5); do
        (
            app_process_input "item_$i" > "$results_dir/result_$i.txt" 2>&1
        ) &
        pids+=($!)
    done
    
    # รอทุก process
    for pid in "${pids[@]}"; do
        wait "$pid"
    done
    
    # ตรวจสอบผล
    local result_count
    result_count=$(ls "$results_dir"/*.txt 2>/dev/null | wc -l)
    it_assert "All 5 processes completed" [[ $result_count -eq 5 ]]
}

test_error_recovery() {
    echo ""
    echo "=== Integration Test: Error Recovery ==="
    
    # ทดสอบการ recover จาก error
    local saved_data_dir="$DATA_DIR"
    DATA_DIR="/nonexistent/path"
    
    # ควร fail gracefully
    it_assert "Fails with bad data dir" ! app_process_input "test"
    
    DATA_DIR="$saved_data_dir"
    it_assert "Recovers after fixing data dir" app_process_input "recovery_test"
}

print_integration_results() {
    echo ""
    echo "======================================="
    echo "Integration Test Results:"
    echo "  Passed: $IT_PASSED"
    echo "  Failed: $IT_FAILED"
    echo "  Total:  $((IT_PASSED + IT_FAILED))"
    
    if [[ $IT_FAILED -eq 0 ]]; then
        echo "  Status: ALL PASSED"
        return 0
    else
        echo "  Status: FAILURES DETECTED"
        return 1
    fi
}

# Main
setup_test_environment
test_full_workflow
test_concurrent_access
test_error_recovery
print_integration_results
exit_code=$?
teardown_test_environment
exit $exit_code
```

---

## ขั้นตอนที่ 420: Test Coverage และ Code Quality

```bash
#!/usr/bin/env bash
# test_coverage.sh

# ===== Code Coverage Analysis =====
# ใช้ kcov หรือ bashcov สำหรับ coverage

# วิธีใช้ kcov
setup_kcov() {
    if ! command -v kcov &>/dev/null; then
        echo "Install kcov: sudo apt-get install kcov"
        return 1
    fi
    
    # รัน script พร้อม coverage
    # kcov --include-path=. /tmp/coverage_report ./my_script.sh
    echo "kcov available"
}

# Manual coverage tracking
declare -A _covered_lines
COVERAGE_SCRIPT=""

enable_coverage_tracking() {
    COVERAGE_SCRIPT="$1"
    
    # ใช้ DEBUG trap เพื่อ track lines
    trap 'track_line "$BASH_SOURCE" "$LINENO"' DEBUG
}

track_line() {
    local source="$1"
    local line="$2"
    
    if [[ "$source" == "$COVERAGE_SCRIPT" ]]; then
        _covered_lines["$line"]=1
    fi
}

coverage_report() {
    echo "Lines covered: ${#_covered_lines[@]}"
    echo "Covered lines: ${!_covered_lines[*]}"
}

# ===== Code Quality Checks =====

# ShellCheck integration
run_shellcheck() {
    local file="$1"
    
    if ! command -v shellcheck &>/dev/null; then
        echo "ShellCheck not installed"
        echo "Install: sudo apt-get install shellcheck"
        return 1
    fi
    
    echo "Running ShellCheck on $file..."
    if shellcheck "$file"; then
        echo "ShellCheck: PASSED"
        return 0
    else
        echo "ShellCheck: FAILED"
        return 1
    fi
}

# Custom lint checks
lint_bash_script() {
    local file="$1"
    local errors=0
    
    echo "=== Linting: $file ==="
    
    # Check 1: #!/usr/bin/env bash shebang
    if ! head -1 "$file" | grep -q "^#!/"; then
        echo "  WARN: Missing shebang line"
    fi
    
    # Check 2: set -euo pipefail
    if ! grep -q "set -.*e" "$file"; then
        echo "  WARN: Missing 'set -e' (exit on error)"
    fi
    
    # Check 3: Local variables in functions
    local func_start=false
    while IFS= read -r line; do
        if [[ "$line" =~ ^[a-zA-Z_]+\(\) ]]; then
            func_start=true
        fi
        if $func_start && [[ "$line" =~ ^[[:space:]]+[a-zA-Z_]+=.*$ ]] && \
           ! [[ "$line" =~ ^[[:space:]]+(local|declare|readonly) ]]; then
            echo "  WARN: Non-local variable assignment in function: $line"
            ((errors++))
        fi
        [[ "$line" == "}" ]] && func_start=false
    done < "$file"
    
    # Check 4: Unused variables
    # หา ตัวแปรที่ declare แต่ไม่ใช้ (simplified)
    while IFS= read -r line; do
        if [[ "$line" =~ local[[:space:]]+([a-zA-Z_]+)= ]]; then
            local varname="${BASH_REMATCH[1]}"
            local occurrences
            occurrences=$(grep -c "$varname" "$file" 2>/dev/null || echo 0)
            if [[ $occurrences -le 1 ]]; then
                echo "  WARN: Possibly unused variable: $varname"
            fi
        fi
    done < "$file"
    
    # Check 5: Command injection risks
    while IFS= read -r line; do
        if [[ "$line" =~ eval.*\$ ]] && ! [[ "$line" =~ ^[[:space:]]*# ]]; then
            echo "  SECURITY: Potential command injection with eval: $line"
            ((errors++))
        fi
    done < "$file"
    
    echo "Lint complete: $errors error(s) found"
    return $errors
}

# Complexity analysis
analyze_complexity() {
    local file="$1"
    
    echo "=== Complexity Analysis: $file ==="
    
    # Count functions
    local func_count
    func_count=$(grep -c "^[a-zA-Z_]*()" "$file" 2>/dev/null || echo 0)
    echo "Functions: $func_count"
    
    # Count if statements
    local if_count
    if_count=$(grep -c "^\s*if\b" "$file" 2>/dev/null || echo 0)
    echo "If statements: $if_count"
    
    # Count loops
    local loop_count
    loop_count=$(grep -cE "^\s*(for|while|until)\b" "$file" 2>/dev/null || echo 0)
    echo "Loops: $loop_count"
    
    # Count case statements
    local case_count
    case_count=$(grep -c "^\s*case\b" "$file" 2>/dev/null || echo 0)
    echo "Case statements: $case_count"
    
    # Total lines
    local total_lines
    total_lines=$(wc -l < "$file")
    echo "Total lines: $total_lines"
    
    # Average function length (rough estimate)
    if [[ $func_count -gt 0 ]]; then
        local avg_func_len
        avg_func_len=$((total_lines / func_count))
        echo "Avg function length: ~$avg_func_len lines"
    fi
}

# ===== Test Generator =====
generate_tests() {
    local script="$1"
    local output="$2"
    
    echo "#!/usr/bin/env bash" > "$output"
    echo "# Auto-generated tests for $script" >> "$output"
    echo "" >> "$output"
    echo "source '$script'" >> "$output"
    echo "" >> "$output"
    
    # Extract functions
    grep "^[a-zA-Z_]*()" "$script" | while read -r func_def; do
        local func_name="${func_def%%(*}"
        cat >> "$output" << EOF

test_${func_name}() {
    # TODO: Add tests for $func_name
    echo "Testing $func_name..."
    # assert_equal "test description" "expected" "\$(${func_name} args)"
}

EOF
    done
    
    echo "Generated test skeleton: $output"
}

# Demo
echo "=== Code Quality Demo ==="

# สร้าง test script
test_script="/tmp/quality_test_$$.sh"
cat > "$test_script" << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

# Calculate factorial
factorial() {
    local n="$1"
    if [[ $n -le 1 ]]; then
        echo 1
    else
        echo $(( n * $(factorial $((n-1))) ))
    fi
}

# Greet user
greet() {
    local name="$1"
    echo "Hello, $name!"
}
SCRIPT

echo ""
echo "Analyzing: $test_script"
analyze_complexity "$test_script"

echo ""
echo "Linting:"
lint_bash_script "$test_script"

echo ""
echo "Generating tests:"
generate_tests "$test_script" "/tmp/generated_tests_$$.sh"
cat "/tmp/generated_tests_$$.sh"

rm -f "$test_script" "/tmp/generated_tests_$$.sh"
```

---

## ขั้นตอนที่ 421: Continuous Testing Pipeline

```bash
#!/usr/bin/env bash
# continuous_testing.sh

# ===== Test Pipeline =====
set -euo pipefail

PIPELINE_LOG="/tmp/pipeline_$$.log"
PIPELINE_FAILED=false

# Pipeline step runner
pipeline_step() {
    local step_name="$1"
    shift
    
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Step: $step_name"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    
    local start_time
    start_time=$(date +%s)
    
    if "$@" 2>&1 | tee -a "$PIPELINE_LOG"; then
        local duration=$(( $(date +%s) - start_time ))
        echo "✓ $step_name PASSED (${duration}s)"
        echo ""
    else
        local duration=$(( $(date +%s) - start_time ))
        echo "✗ $step_name FAILED (${duration}s)"
        PIPELINE_FAILED=true
        
        # Continue or abort based on step
        if [[ "${ABORT_ON_FAILURE:-true}" == "true" ]]; then
            echo "Aborting pipeline due to failure"
            return 1
        fi
    fi
}

# ===== Pipeline Stages =====

# Stage 1: Syntax Check
stage_syntax_check() {
    local scripts_dir="${1:-.}"
    local failed=0
    
    echo "Checking syntax of all .sh files..."
    while IFS= read -r -d '' script; do
        if bash -n "$script" 2>&1; then
            echo "  OK: $script"
        else
            echo "  FAIL: $script"
            ((failed++))
        fi
    done < <(find "$scripts_dir" -name "*.sh" -print0 2>/dev/null)
    
    [[ $failed -eq 0 ]]
}

# Stage 2: Lint
stage_lint() {
    local scripts_dir="${1:-.}"
    
    if command -v shellcheck &>/dev/null; then
        echo "Running ShellCheck..."
        find "$scripts_dir" -name "*.sh" -exec shellcheck {} \; 2>&1
    else
        echo "ShellCheck not available, skipping lint"
        return 0
    fi
}

# Stage 3: Unit Tests
stage_unit_tests() {
    local test_dir="${1:-./tests}"
    
    if [[ ! -d "$test_dir" ]]; then
        echo "No test directory found: $test_dir"
        return 0
    fi
    
    local passed=0 failed=0
    
    while IFS= read -r -d '' test_file; do
        echo "Running: $test_file"
        if bash "$test_file"; then
            ((passed++))
        else
            ((failed++))
        fi
    done < <(find "$test_dir" -name "test_*.sh" -print0 2>/dev/null)
    
    echo "Unit Tests: $passed passed, $failed failed"
    [[ $failed -eq 0 ]]
}

# Stage 4: Integration Tests
stage_integration_tests() {
    echo "Running integration tests..."
    # Run integration test suite
    # bash ./tests/integration/run_all.sh
    echo "Integration tests complete"
}

# Stage 5: Code Coverage
stage_coverage() {
    echo "Checking code coverage..."
    
    if command -v kcov &>/dev/null; then
        # kcov --include-path=./src /tmp/coverage_report bash ./tests/run_all.sh
        echo "Coverage report generated"
    else
        echo "kcov not available, skipping coverage"
    fi
}

# Stage 6: Security Scan
stage_security_scan() {
    local dir="${1:-.}"
    local issues=0
    
    echo "Scanning for security issues..."
    
    # ตรวจหา eval ที่อันตราย
    while IFS= read -r -d '' file; do
        if grep -nE "eval.*\\\$" "$file" 2>/dev/null | grep -v "^#"; then
            echo "  SECURITY WARNING in $file: potentially unsafe eval"
            ((issues++))
        fi
        
        # ตรวจหา hardcoded passwords
        if grep -nE "(password|passwd|secret)\s*=\s*[\"'][^\"']+[\"']" "$file" 2>/dev/null | \
           grep -vi "example\|placeholder\|your_"; then
            echo "  SECURITY WARNING in $file: possible hardcoded credential"
            ((issues++))
        fi
    done < <(find "$dir" -name "*.sh" -print0 2>/dev/null)
    
    if [[ $issues -eq 0 ]]; then
        echo "No security issues found"
    fi
    
    return 0  # Don't fail pipeline for warnings
}

# ===== Main Pipeline =====
run_test_pipeline() {
    local project_dir="${1:-.}"
    
    echo "╔══════════════════════════════════════╗"
    echo "║     Bash Script Test Pipeline        ║"
    echo "╚══════════════════════════════════════╝"
    echo "Project: $project_dir"
    echo "Time: $(date)"
    echo ""
    
    # สร้าง test files ชั่วคราว
    local test_dir="$project_dir/tests"
    mkdir -p "$test_dir"
    
    cat > "$test_dir/test_basic.sh" << 'EOF'
#!/usr/bin/env bash
echo "Running basic tests..."
[[ 2 -eq $((1+1)) ]] && echo "Math OK" || exit 1
[[ "HELLO" == "${$(echo hello)^^}" ]] 2>/dev/null || \
    [[ "HELLO" == "$(echo "hello" | tr '[:lower:]' '[:upper:]')" ]] && echo "String OK" || exit 1
echo "All basic tests passed"
EOF
    chmod +x "$test_dir/test_basic.sh"
    
    # Run pipeline stages
    ABORT_ON_FAILURE=false
    
    pipeline_step "Syntax Check" stage_syntax_check "$project_dir"
    pipeline_step "Security Scan" stage_security_scan "$project_dir"
    pipeline_step "Unit Tests" stage_unit_tests "$test_dir"
    
    # Pipeline summary
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Pipeline Summary"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Log: $PIPELINE_LOG"
    
    if $PIPELINE_FAILED; then
        echo "Status: PIPELINE FAILED"
        rm -rf "$test_dir"
        return 1
    else
        echo "Status: PIPELINE PASSED"
        rm -rf "$test_dir"
        return 0
    fi
}

run_test_pipeline "/tmp/pipeline_demo_$$"
rm -f "$PIPELINE_LOG"
```

---

## ขั้นตอนที่ 422: Test-Driven Development (TDD) ด้วย Bash

```bash
#!/usr/bin/env bash
# tdd_example.sh

# ===== TDD Workflow =====
# 1. RED: เขียน test ที่ fail ก่อน
# 2. GREEN: เขียน code ให้ test pass
# 3. REFACTOR: ปรับปรุง code

# ===== Test Harness =====
PASS=0
FAIL=0
declare -a FAILED_TESTS=()

it() {
    local description="$1"
    shift
    
    if "$@" 2>/dev/null; then
        ((PASS++))
        printf "\033[32m  ✓ %s\033[0m\n" "$description"
    else
        ((FAIL++))
        FAILED_TESTS+=("$description")
        printf "\033[31m  ✗ %s\033[0m\n" "$description"
    fi
}

describe() {
    echo ""
    echo "▶ $1"
}

expect_output() {
    local expected="$1"
    shift
    local actual
    actual=$("$@" 2>&1)
    [[ "$actual" == "$expected" ]]
}

# ===== Step 1: Write Tests First (RED) =====

describe "Stack implementation"
# stack_push, stack_pop, stack_peek, stack_size, stack_is_empty

# ===== Step 2: Implement Stack (GREEN) =====

declare -a _stack=()

stack_push() {
    _stack+=("$1")
}

stack_pop() {
    if [[ ${#_stack[@]} -eq 0 ]]; then
        echo "Error: Stack underflow" >&2
        return 1
    fi
    local top="${_stack[-1]}"
    unset '_stack[-1]'
    echo "$top"
}

stack_peek() {
    if [[ ${#_stack[@]} -eq 0 ]]; then
        echo "Error: Stack is empty" >&2
        return 1
    fi
    echo "${_stack[-1]}"
}

stack_size() {
    echo "${#_stack[@]}"
}

stack_is_empty() {
    [[ ${#_stack[@]} -eq 0 ]]
}

stack_clear() {
    _stack=()
}

# ===== Run Tests =====
it "new stack is empty" stack_is_empty

stack_push "a"
it "after push, not empty" ! stack_is_empty
it "size is 1 after one push" expect_output "1" stack_size

stack_push "b"
stack_push "c"
it "size is 3 after three pushes" expect_output "3" stack_size
it "peek returns top element" expect_output "c" stack_peek

local popped
popped=$(stack_pop)
it "pop returns c" [[ "$popped" == "c" ]]
it "size decreases after pop" expect_output "2" stack_size
it "peek shows b after popping c" expect_output "b" stack_peek

stack_clear
it "cleared stack is empty" stack_is_empty
it "pop on empty stack fails" ! stack_pop

# ===== More TDD: Queue =====
describe "Queue implementation"

declare -a _queue=()

queue_enqueue() { _queue+=("$1"); }

queue_dequeue() {
    if [[ ${#_queue[@]} -eq 0 ]]; then
        echo "Error: Queue empty" >&2
        return 1
    fi
    local front="${_queue[0]}"
    _queue=("${_queue[@]:1}")
    echo "$front"
}

queue_front() {
    [[ ${#_queue[@]} -gt 0 ]] && echo "${_queue[0]}" || return 1
}

queue_size() { echo "${#_queue[@]}"; }
queue_is_empty() { [[ ${#_queue[@]} -eq 0 ]]; }

it "new queue is empty" queue_is_empty
queue_enqueue "first"
queue_enqueue "second"
queue_enqueue "third"
it "queue size is 3" expect_output "3" queue_size
it "FIFO: front is first" expect_output "first" queue_front

local dequeued
dequeued=$(queue_dequeue)
it "dequeue returns first" [[ "$dequeued" == "first" ]]
it "front is now second" expect_output "second" queue_front

# ===== Summary =====
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━"
echo "Test Results: $PASS/$((PASS+FAIL)) passed"
if [[ ${#FAILED_TESTS[@]} -gt 0 ]]; then
    echo "Failed:"
    for test in "${FAILED_TESTS[@]}"; do
        echo "  - $test"
    done
    exit 1
fi
echo "ALL TESTS PASSED"
```

---

## ขั้นตอนที่ 423: Property-Based Testing

```bash
#!/usr/bin/env bash
# property_based_testing.sh

# Property-based testing: ทดสอบ properties/invariants แทน specific cases

# ===== Random Test Data Generators =====
random_int() {
    local min="${1:-0}"
    local max="${2:-100}"
    echo $(( RANDOM % (max - min + 1) + min ))
}

random_string() {
    local length="${1:-10}"
    cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c "$length" 2>/dev/null || \
        tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c "$length"
}

random_array() {
    local size="${1:-5}"
    local -a arr=()
    for ((i=0; i<size; i++)); do
        arr+=("$(random_int)")
    done
    echo "${arr[*]}"
}

# ===== Properties to Test =====

# Function under test
sort_array() {
    local -a arr=("$@")
    printf '%s\n' "${arr[@]}" | sort -n
}

reverse_string() {
    echo "$1" | rev
}

is_palindrome() {
    local str="$1"
    [[ "$str" == "$(echo "$str" | rev)" ]]
}

# Properties (invariants ที่ต้องเป็นจริงเสมอ)
property_sort_idempotent() {
    # sort(sort(x)) == sort(x)
    local arr=("$@")
    local sorted1 sorted2
    sorted1=$(sort_array "${arr[@]}")
    sorted2=$(echo "$sorted1" | sort_array $(echo "$sorted1"))
    [[ "$sorted1" == "$sorted2" ]]
}

property_sort_preserves_count() {
    # count(sort(x)) == count(x)
    local arr=("$@")
    local original_count=${#arr[@]}
    local sorted_count
    sorted_count=$(sort_array "${arr[@]}" | wc -l)
    [[ $original_count -eq $sorted_count ]]
}

property_double_reverse_is_identity() {
    # rev(rev(s)) == s
    local str="$1"
    local double_reversed
    double_reversed=$(reverse_string "$(reverse_string "$str")")
    [[ "$str" == "$double_reversed" ]]
}

property_palindrome_reversed_equals_self() {
    # is_palindrome(s) => rev(s) == s
    local str="$1"
    if is_palindrome "$str"; then
        [[ "$str" == "$(reverse_string "$str")" ]]
    else
        return 0  # Property doesn't apply
    fi
}

# Property test runner
run_property_tests() {
    local property="$1"
    local generator="$2"
    local runs="${3:-50}"
    local passed=0 failed=0
    
    echo "Property: $property ($runs runs)"
    
    for ((i=1; i<=runs; i++)); do
        local test_input
        test_input=$(eval "$generator")
        
        if $property $test_input 2>/dev/null; then
            ((passed++))
        else
            ((failed++))
            echo "  FAIL with input: $test_input"
            if [[ $failed -ge 3 ]]; then
                echo "  (stopping after 3 failures)"
                break
            fi
        fi
    done
    
    echo "  Results: $passed/$((passed+failed)) passed"
    [[ $failed -eq 0 ]]
}

# ===== Run Property Tests =====
echo "=== Property-Based Tests ==="

echo ""
run_property_tests \
    "property_sort_preserves_count" \
    'random_array 5' \
    30

echo ""
run_property_tests \
    "property_double_reverse_is_identity" \
    'random_string 10' \
    50

# Shrinking: find minimal failing input
shrink_failure() {
    local property="$1"
    local failing_input="$*"
    
    echo "Shrinking failure: '$failing_input'"
    
    # Try shorter versions
    local current="$failing_input"
    while [[ ${#current} -gt 1 ]]; do
        local shorter="${current:0:$((${#current}-1))}"
        if ! $property "$shorter" 2>/dev/null; then
            current="$shorter"
            echo "  Shrunk to: '$current'"
        else
            break
        fi
    done
    
    echo "Minimal failing input: '$current'"
}

echo ""
echo "Property tests complete"
```

---

## ขั้นตอนที่ 424: Test Reports และ Documentation

```bash
#!/usr/bin/env bash
# test_reports.sh

# ===== JUnit XML Report Generator =====
generate_junit_xml() {
    local output_file="$1"
    local test_suite_name="$2"
    shift 2
    local test_results=("$@")  # Array ของ "name:status:duration"
    
    local passed=0 failed=0 total=0
    
    # Count
    for result in "${test_results[@]}"; do
        ((total++))
        local status="${result#*:}"
        status="${status%%:*}"
        [[ "$status" == "pass" ]] && ((passed++)) || ((failed++))
    done
    
    cat > "$output_file" << EOF
<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
  <testsuite name="$test_suite_name" tests="$total" failures="$failed" 
             time="0" timestamp="$(date -u +%Y-%m-%dT%H:%M:%S)">
EOF
    
    for result in "${test_results[@]}"; do
        local name="${result%%:*}"
        local rest="${result#*:}"
        local status="${rest%%:*}"
        local duration="${rest#*:}"
        
        if [[ "$status" == "pass" ]]; then
            cat >> "$output_file" << EOF
    <testcase name="$name" classname="$test_suite_name" time="$duration"/>
EOF
        else
            cat >> "$output_file" << EOF
    <testcase name="$name" classname="$test_suite_name" time="$duration">
      <failure message="Test failed">Test '$name' failed</failure>
    </testcase>
EOF
        fi
    done
    
    cat >> "$output_file" << EOF
  </testsuite>
</testsuites>
EOF
    
    echo "JUnit XML report: $output_file"
}

# ===== HTML Report =====
generate_html_report() {
    local output_file="$1"
    local title="$2"
    shift 2
    local test_results=("$@")
    
    local passed=0 failed=0 total=0
    
    for result in "${test_results[@]}"; do
        ((total++))
        [[ "${result#*:}" == pass:* ]] && ((passed++)) || ((failed++))
    done
    
    local pass_rate=0
    [[ $total -gt 0 ]] && pass_rate=$((passed * 100 / total))
    
    cat > "$output_file" << EOF
<!DOCTYPE html>
<html>
<head>
<title>$title</title>
<style>
body { font-family: Arial, sans-serif; margin: 20px; }
.pass { color: green; }
.fail { color: red; }
.summary { background: #f0f0f0; padding: 10px; margin: 10px 0; }
table { border-collapse: collapse; width: 100%; }
th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
th { background-color: #4CAF50; color: white; }
tr:nth-child(even) { background-color: #f2f2f2; }
</style>
</head>
<body>
<h1>$title</h1>
<div class="summary">
  <strong>Total:</strong> $total |
  <span class="pass">Passed: $passed</span> |
  <span class="fail">Failed: $failed</span> |
  Pass Rate: $pass_rate%
</div>
<table>
<tr><th>Test Name</th><th>Status</th><th>Duration</th></tr>
EOF
    
    for result in "${test_results[@]}"; do
        local name="${result%%:*}"
        local rest="${result#*:}"
        local status="${rest%%:*}"
        local duration="${rest#*:}"
        local class
        [[ "$status" == "pass" ]] && class="pass" || class="fail"
        local symbol
        [[ "$status" == "pass" ]] && symbol="✓" || symbol="✗"
        
        echo "<tr><td>$name</td><td class='$class'>$symbol $status</td><td>${duration}ms</td></tr>" >> "$output_file"
    done
    
    cat >> "$output_file" << EOF
</table>
<p>Generated: $(date)</p>
</body>
</html>
EOF
    
    echo "HTML report: $output_file"
}

# Demo
test_results=(
    "test_addition:pass:5"
    "test_subtraction:pass:3"
    "test_multiplication:fail:8"
    "test_string_upper:pass:2"
    "test_file_exists:fail:12"
)

generate_junit_xml "/tmp/test_results.xml" "BashUnitTests" "${test_results[@]}"
generate_html_report "/tmp/test_results.html" "Bash Test Report" "${test_results[@]}"

echo ""
echo "Reports generated:"
cat /tmp/test_results.xml
```

---

## สรุป Part 21

### สิ่งที่เรียนรู้ (Steps 416-424):

| Step | หัวข้อ | เครื่องมือ/เทคนิค |
|------|--------|------------------|
| 416 | Unit Testing Basics | Custom test framework, assert functions |
| 417 | BATS Framework | bats-core, setup/teardown |
| 418 | Fixtures & Mocking | mock commands, spy functions |
| 419 | Integration Testing | End-to-end workflow tests |
| 420 | Code Quality | shellcheck, lint, complexity |
| 421 | CI Pipeline | Multi-stage pipeline |
| 422 | TDD | Red-Green-Refactor cycle |
| 423 | Property-Based Testing | Invariants, random inputs |
| 424 | Test Reports | JUnit XML, HTML reports |

**ขั้นตอนต่อไป**: Part 22 - Logging และ Monitoring
