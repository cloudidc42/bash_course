# Part 24: Database Operations

## Module 2: Intermediate Level
### ขั้นตอนที่ 437-447: การทำงานกับ Database

---

## ขั้นตอนที่ 437: PostgreSQL Operations

```bash
#!/usr/bin/env bash
# postgresql_ops.sh

# ===== PostgreSQL Integration =====

# Connection settings
PG_HOST="${DB_HOST:-localhost}"
PG_PORT="${DB_PORT:-5432}"
PG_USER="${DB_USER:-postgres}"
PG_PASS="${DB_PASSWORD:-}"
PG_NAME="${DB_NAME:-myapp}"

# Execute query
pg_exec() {
    local query="$1"
    PGPASSWORD="$PG_PASS" psql \
        -h "$PG_HOST" \
        -p "$PG_PORT" \
        -U "$PG_USER" \
        -d "$PG_NAME" \
        -c "$query" 2>&1
}

# Execute query and get output
pg_query() {
    local query="$1"
    local format="${2:-csv}"  # csv, unaligned, table
    
    PGPASSWORD="$PG_PASS" psql \
        -h "$PG_HOST" \
        -p "$PG_PORT" \
        -U "$PG_USER" \
        -d "$PG_NAME" \
        -A -F',' \
        -t \
        -c "$query" 2>/dev/null
}

# Check if table exists
pg_table_exists() {
    local schema="${2:-public}"
    local result
    result=$(pg_query "SELECT COUNT(*) FROM information_schema.tables 
                       WHERE table_schema='$schema' AND table_name='$1'")
    [[ "$result" == "1" ]]
}

# Get row count
pg_count() {
    local table="$1"
    local where="${2:-1=1}"
    pg_query "SELECT COUNT(*) FROM $table WHERE $where"
}

# Backup database
pg_backup() {
    local backup_dir="${1:-/tmp/pg_backup}"
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_file="$backup_dir/${PG_NAME}_${timestamp}.sql.gz"
    
    mkdir -p "$backup_dir"
    
    echo "Backing up $PG_NAME to $backup_file..."
    PGPASSWORD="$PG_PASS" pg_dump \
        -h "$PG_HOST" \
        -p "$PG_PORT" \
        -U "$PG_USER" \
        "$PG_NAME" | gzip > "$backup_file"
    
    if [[ $? -eq 0 ]]; then
        local size
        size=$(du -sh "$backup_file" | cut -f1)
        echo "Backup complete: $backup_file ($size)"
        echo "$backup_file"
    else
        echo "Backup failed!" >&2
        rm -f "$backup_file"
        return 1
    fi
}

# Restore from backup
pg_restore_backup() {
    local backup_file="$1"
    local target_db="${2:-$PG_NAME}"
    
    echo "Restoring from $backup_file to $target_db..."
    
    gunzip -c "$backup_file" | PGPASSWORD="$PG_PASS" psql \
        -h "$PG_HOST" \
        -p "$PG_PORT" \
        -U "$PG_USER" \
        "$target_db"
    
    echo "Restore complete"
}

# Migration system
MIGRATION_DIR="${MIGRATION_DIR:-./migrations}"

run_migrations() {
    # Create migrations table if not exists
    pg_exec "CREATE TABLE IF NOT EXISTS _migrations (
        id SERIAL PRIMARY KEY,
        filename VARCHAR(255) UNIQUE NOT NULL,
        applied_at TIMESTAMP DEFAULT NOW()
    );"
    
    local applied_count=0
    
    # Run pending migrations
    for migration_file in "$MIGRATION_DIR"/*.sql; do
        [[ -f "$migration_file" ]] || continue
        
        local filename
        filename=$(basename "$migration_file")
        
        # Check if already applied
        local count
        count=$(pg_query "SELECT COUNT(*) FROM _migrations WHERE filename='$filename'")
        
        if [[ "${count:-0}" -eq 0 ]]; then
            echo "Applying: $filename"
            
            if PGPASSWORD="$PG_PASS" psql \
                -h "$PG_HOST" -p "$PG_PORT" \
                -U "$PG_USER" -d "$PG_NAME" \
                -f "$migration_file" 2>&1; then
                
                pg_exec "INSERT INTO _migrations (filename) VALUES ('$filename');"
                ((applied_count++))
                echo "Applied: $filename"
            else
                echo "Migration failed: $filename" >&2
                return 1
            fi
        fi
    done
    
    echo "Migrations complete: $applied_count applied"
}

# Query builder (simple)
pg_select() {
    local table="$1"
    local columns="${2:-*}"
    local where="${3:-}"
    local limit="${4:-100}"
    local order="${5:-}"
    
    local query="SELECT $columns FROM $table"
    [[ -n "$where" ]] && query+=" WHERE $where"
    [[ -n "$order" ]] && query+=" ORDER BY $order"
    [[ -n "$limit" ]] && query+=" LIMIT $limit"
    
    pg_query "$query"
}

# Demo (requires PostgreSQL)
demo_postgresql() {
    if ! command -v psql &>/dev/null; then
        echo "psql not installed, showing example code only"
        echo ""
        echo "Example usage:"
        echo "  pg_query \"SELECT * FROM users LIMIT 5\""
        echo "  pg_exec \"INSERT INTO logs (message) VALUES ('test')\""
        echo "  pg_backup /var/backups"
        return
    fi
    
    echo "PostgreSQL connected: $(pg_query 'SELECT version()' | head -1)"
    echo "Tables: $(pg_query "SELECT table_name FROM information_schema.tables WHERE table_schema='public'" | wc -l)"
}

demo_postgresql
```

---

## ขั้นตอนที่ 438: MySQL/MariaDB Operations

```bash
#!/usr/bin/env bash
# mysql_ops.sh

# ===== MySQL/MariaDB Integration =====

MYSQL_HOST="${DB_HOST:-localhost}"
MYSQL_PORT="${DB_PORT:-3306}"
MYSQL_USER="${DB_USER:-root}"
MYSQL_PASS="${DB_PASSWORD:-}"
MYSQL_DB="${DB_NAME:-myapp}"

# MySQL options
MYSQL_OPTS="-h $MYSQL_HOST -P $MYSQL_PORT -u $MYSQL_USER"
[[ -n "$MYSQL_PASS" ]] && MYSQL_OPTS+=" -p$MYSQL_PASS"

# Execute query
mysql_exec() {
    local query="$1"
    mysql $MYSQL_OPTS "$MYSQL_DB" -e "$query" 2>&1
}

# Query with output
mysql_query() {
    local query="$1"
    mysql $MYSQL_OPTS "$MYSQL_DB" \
        --batch --silent \
        -e "$query" 2>/dev/null
}

# Check connection
mysql_connected() {
    mysql $MYSQL_OPTS -e "SELECT 1" "$MYSQL_DB" &>/dev/null
}

# Export table to CSV
mysql_export_csv() {
    local table="$1"
    local output="$2"
    local where="${3:-1=1}"
    
    mysql $MYSQL_OPTS "$MYSQL_DB" \
        --batch --silent \
        -e "SELECT * FROM $table WHERE $where" 2>/dev/null | \
        awk 'NR==1{for(i=1;i<=NF;i++) printf "%s%s",$i,(i<NF?",":"\n"); next}
             {for(i=1;i<=NF;i++) printf "\"%s\"%s",$i,(i<NF?",":"\n")}' > "$output"
    
    echo "Exported: $output ($(wc -l < "$output") rows)"
}

# Import CSV to table
mysql_import_csv() {
    local csv_file="$1"
    local table="$2"
    
    mysql_exec "LOAD DATA LOCAL INFILE '$csv_file'
                INTO TABLE $table
                FIELDS TERMINATED BY ','
                ENCLOSED BY '\"'
                LINES TERMINATED BY '\n'
                IGNORE 1 ROWS;"
    
    echo "Imported: $csv_file into $table"
}

# Database backup
mysql_backup() {
    local backup_dir="${1:-/tmp/mysql_backup}"
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_file="$backup_dir/${MYSQL_DB}_${timestamp}.sql.gz"
    
    mkdir -p "$backup_dir"
    
    echo "Backing up $MYSQL_DB..."
    mysqldump $MYSQL_OPTS "$MYSQL_DB" | gzip > "$backup_file"
    
    if [[ $? -eq 0 ]]; then
        echo "Backup: $backup_file ($(du -sh "$backup_file" | cut -f1))"
        echo "$backup_file"
    else
        rm -f "$backup_file"
        return 1
    fi
}

# Monitor MySQL performance
mysql_status() {
    echo "=== MySQL Status ==="
    mysql_exec "SHOW GLOBAL STATUS LIKE 'Threads_connected';"
    mysql_exec "SHOW GLOBAL STATUS LIKE 'Slow_queries';"
    mysql_exec "SHOW GLOBAL STATUS LIKE 'Questions';"
    mysql_exec "SHOW PROCESSLIST;"
}

# Optimize tables
mysql_optimize() {
    echo "Optimizing tables in $MYSQL_DB..."
    mysql_query "SHOW TABLES;" | while read -r table; do
        echo "Optimizing: $table"
        mysql_exec "OPTIMIZE TABLE $table;"
    done
}

# Demo
if command -v mysql &>/dev/null; then
    echo "MySQL available"
    mysql_connected && echo "Connected" || echo "Not connected"
else
    echo "MySQL not installed"
    echo "Example: mysql_query \"SELECT * FROM users LIMIT 5\""
fi
```

---

## ขั้นตอนที่ 439: SQLite Operations

```bash
#!/usr/bin/env bash
# sqlite_ops.sh

# ===== SQLite Integration =====
# SQLite เหมาะสำหรับ scripts ที่ต้องการ database แต่ไม่อยากตั้ง server

SQLITE_DB="${SQLITE_DB:-/tmp/app_$$.db}"

# Execute SQL
sqlite_exec() {
    local query="$1"
    local db="${2:-$SQLITE_DB}"
    sqlite3 "$db" "$query" 2>&1
}

# Query with output
sqlite_query() {
    local query="$1"
    local db="${2:-$SQLITE_DB}"
    sqlite3 -separator ',' "$db" "$query" 2>/dev/null
}

# Initialize database schema
sqlite_init() {
    local schema_file="${1:-}"
    
    if [[ -n "$schema_file" && -f "$schema_file" ]]; then
        sqlite3 "$SQLITE_DB" < "$schema_file"
    fi
    
    echo "Database initialized: $SQLITE_DB"
}

# ===== Key-Value Store using SQLite =====
KV_DB="/tmp/kv_store_$$.db"

kv_init() {
    sqlite3 "$KV_DB" << 'SQL'
CREATE TABLE IF NOT EXISTS kv_store (
    key TEXT PRIMARY KEY,
    value TEXT,
    expires_at INTEGER,
    created_at INTEGER DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER DEFAULT (strftime('%s', 'now'))
);
CREATE INDEX IF NOT EXISTS idx_expires ON kv_store(expires_at);
SQL
    echo "KV store initialized: $KV_DB"
}

kv_set() {
    local key="$1"
    local value="$2"
    local ttl="${3:-}"  # seconds
    
    local expires_sql="NULL"
    if [[ -n "$ttl" ]]; then
        expires_sql="strftime('%s', 'now') + $ttl"
    fi
    
    sqlite3 "$KV_DB" << SQL
INSERT OR REPLACE INTO kv_store (key, value, expires_at, updated_at)
VALUES ('$(sqlite3 ':memory:' "SELECT replace('$key', '''', ''''||'''')")', 
        '$(sqlite3 ':memory:' "SELECT replace('$value', '''', ''''||'''')")',
        $expires_sql,
        strftime('%s', 'now'));
SQL
}

kv_get() {
    local key="$1"
    sqlite3 "$KV_DB" \
        "SELECT value FROM kv_store 
         WHERE key='$key' 
         AND (expires_at IS NULL OR expires_at > strftime('%s', 'now'))" 2>/dev/null
}

kv_delete() {
    local key="$1"
    sqlite3 "$KV_DB" "DELETE FROM kv_store WHERE key='$key'" 2>/dev/null
}

kv_exists() {
    local result
    result=$(sqlite3 "$KV_DB" \
        "SELECT COUNT(*) FROM kv_store 
         WHERE key='$1' 
         AND (expires_at IS NULL OR expires_at > strftime('%s', 'now'))" 2>/dev/null)
    [[ "$result" == "1" ]]
}

kv_cleanup_expired() {
    sqlite3 "$KV_DB" \
        "DELETE FROM kv_store WHERE expires_at IS NOT NULL AND expires_at <= strftime('%s', 'now')"
    echo "Expired entries removed"
}

kv_list() {
    echo "=== KV Store Contents ==="
    sqlite3 -column -header "$KV_DB" \
        "SELECT key, value, 
                CASE WHEN expires_at IS NULL THEN 'never' 
                     ELSE datetime(expires_at, 'unixepoch')
                END as expires,
                datetime(created_at, 'unixepoch') as created
         FROM kv_store 
         WHERE expires_at IS NULL OR expires_at > strftime('%s', 'now')
         ORDER BY key" 2>/dev/null
}

# ===== Queue using SQLite =====
QUEUE_DB="/tmp/queue_$$.db"

queue_init() {
    sqlite3 "$QUEUE_DB" << 'SQL'
CREATE TABLE IF NOT EXISTS job_queue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    queue_name TEXT DEFAULT 'default',
    payload TEXT NOT NULL,
    status TEXT DEFAULT 'pending',
    priority INTEGER DEFAULT 0,
    created_at INTEGER DEFAULT (strftime('%s', 'now')),
    started_at INTEGER,
    completed_at INTEGER,
    error TEXT,
    attempts INTEGER DEFAULT 0
);
CREATE INDEX IF NOT EXISTS idx_queue_status ON job_queue(queue_name, status, priority);
SQL
}

queue_push() {
    local payload="$1"
    local queue="${2:-default}"
    local priority="${3:-0}"
    
    local safe_payload="${payload//\'/\'\'}"
    sqlite3 "$QUEUE_DB" \
        "INSERT INTO job_queue (queue_name, payload, priority) 
         VALUES ('$queue', '$safe_payload', $priority)" 2>/dev/null
    echo "Job queued (ID: $(sqlite3 "$QUEUE_DB" 'SELECT last_insert_rowid()'))"
}

queue_pop() {
    local queue="${1:-default}"
    
    # Get and lock next job
    local job_id
    job_id=$(sqlite3 "$QUEUE_DB" \
        "SELECT id FROM job_queue 
         WHERE queue_name='$queue' AND status='pending'
         ORDER BY priority DESC, id ASC
         LIMIT 1" 2>/dev/null)
    
    [[ -z "$job_id" ]] && return 1
    
    # Mark as processing
    sqlite3 "$QUEUE_DB" \
        "UPDATE job_queue 
         SET status='processing', started_at=strftime('%s','now'), attempts=attempts+1
         WHERE id=$job_id AND status='pending'" 2>/dev/null
    
    # Return payload
    sqlite3 "$QUEUE_DB" "SELECT payload FROM job_queue WHERE id=$job_id" 2>/dev/null
    echo "JOB_ID=$job_id"
}

queue_complete() {
    local job_id="$1"
    sqlite3 "$QUEUE_DB" \
        "UPDATE job_queue 
         SET status='completed', completed_at=strftime('%s','now')
         WHERE id=$job_id" 2>/dev/null
}

queue_fail() {
    local job_id="$1"
    local error="${2:-unknown error}"
    sqlite3 "$QUEUE_DB" \
        "UPDATE job_queue 
         SET status='failed', error='${error//\'/\'\'}', completed_at=strftime('%s','now')
         WHERE id=$job_id" 2>/dev/null
}

queue_stats() {
    echo "=== Queue Statistics ==="
    sqlite3 -column -header "$QUEUE_DB" \
        "SELECT queue_name, status, COUNT(*) as count 
         FROM job_queue 
         GROUP BY queue_name, status
         ORDER BY queue_name, status" 2>/dev/null
}

# ===== Demo =====
if ! command -v sqlite3 &>/dev/null; then
    echo "sqlite3 not installed"
    echo "Install: sudo apt-get install sqlite3"
    exit 0
fi

echo "=== SQLite Demo ==="

# KV Store demo
echo ""
echo "--- Key-Value Store ---"
kv_init
kv_set "user:1:name" "Alice"
kv_set "user:1:email" "alice@example.com"
kv_set "session:abc123" "user_id=1" 3600  # Expires in 1 hour
kv_set "temp_key" "temp_value" 1  # Expires in 1 second

echo "user:1:name = $(kv_get "user:1:name")"
echo "session:abc123 = $(kv_get "session:abc123")"
kv_exists "user:1:name" && echo "user:1:name exists" || echo "not found"
kv_list

# Queue demo
echo ""
echo "--- Job Queue ---"
queue_init
queue_push '{"type":"email","to":"alice@example.com","subject":"Hello"}'
queue_push '{"type":"sms","to":"+1234567890","message":"Test"}' "sms" 10  # Higher priority
queue_push '{"type":"webhook","url":"https://example.com/hook","data":{"event":"test"}}'

queue_stats

echo ""
echo "Processing jobs:"
while IFS= read -r output; do
    if [[ "$output" =~ ^JOB_ID=(.+) ]]; then
        job_id="${BASH_REMATCH[1]}"
        echo "Completing job $job_id"
        queue_complete "$job_id"
    else
        echo "Job payload: $output"
    fi
done < <(queue_pop)

queue_stats

rm -f "$KV_DB" "$QUEUE_DB" "$SQLITE_DB"
```

---

## ขั้นตอนที่ 440: Database Abstraction Layer

```bash
#!/usr/bin/env bash
# db_abstraction.sh

# ===== Database Abstraction Layer =====
# รองรับ PostgreSQL, MySQL, SQLite ด้วย interface เดียวกัน

DB_DRIVER="${DB_DRIVER:-sqlite}"  # sqlite, postgresql, mysql
DB_DSN="${DB_DSN:-/tmp/app_$$.db}"

# Initialize connection
db_connect() {
    case "$DB_DRIVER" in
        sqlite)
            [[ ! -f "$DB_DSN" ]] && touch "$DB_DSN"
            echo "Connected to SQLite: $DB_DSN"
            ;;
        postgresql)
            # Test connection
            PGPASSWORD="$DB_PASSWORD" psql "$DB_DSN" -c '\q' 2>/dev/null || {
                echo "PostgreSQL connection failed" >&2
                return 1
            }
            echo "Connected to PostgreSQL"
            ;;
        mysql)
            mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASSWORD" \
                -e "SELECT 1" "$DB_NAME" &>/dev/null || {
                echo "MySQL connection failed" >&2
                return 1
            }
            echo "Connected to MySQL"
            ;;
        *)
            echo "Unknown driver: $DB_DRIVER" >&2
            return 1
            ;;
    esac
}

# Execute query
db_exec() {
    local query="$1"
    
    case "$DB_DRIVER" in
        sqlite)
            sqlite3 "$DB_DSN" "$query"
            ;;
        postgresql)
            PGPASSWORD="$DB_PASSWORD" psql "$DB_DSN" -c "$query"
            ;;
        mysql)
            mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASSWORD" "$DB_NAME" -e "$query"
            ;;
    esac
}

# Query with results
db_query() {
    local query="$1"
    
    case "$DB_DRIVER" in
        sqlite)
            sqlite3 -separator ',' "$DB_DSN" "$query" 2>/dev/null
            ;;
        postgresql)
            PGPASSWORD="$DB_PASSWORD" psql "$DB_DSN" -A -F',' -t -c "$query" 2>/dev/null
            ;;
        mysql)
            mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASSWORD" "$DB_NAME" \
                --batch --silent -e "$query" 2>/dev/null | tr '\t' ','
            ;;
    esac
}

# Create table (cross-database)
db_create_table() {
    local table_name="$1"
    shift
    # columns format: "name:type:constraints"
    
    local columns_sql=""
    for col_def in "$@"; do
        local name="${col_def%%:*}"
        local rest="${col_def#*:}"
        local type="${rest%%:*}"
        local constraints="${rest#*:}"
        
        # Translate types
        case "$DB_DRIVER:$type" in
            sqlite:serial)    type="INTEGER" ; constraints="PRIMARY KEY AUTOINCREMENT $constraints" ;;
            postgresql:serial) type="SERIAL" ;;
            mysql:serial)     type="INT AUTO_INCREMENT" ; constraints="$constraints PRIMARY KEY" ;;
        esac
        
        columns_sql+="${columns_sql:+, }$name $type $constraints"
    done
    
    db_exec "CREATE TABLE IF NOT EXISTS $table_name ($columns_sql);"
}

# Insert record
db_insert() {
    local table="$1"
    shift
    
    local keys="" values=""
    while [[ $# -gt 0 ]]; do
        local kv="$1"
        local k="${kv%%=*}"
        local v="${kv#*=}"
        
        keys+="${keys:+,}$k"
        values+="${values:+,}'${v//\'/\'\'}'"
        shift
    done
    
    db_exec "INSERT INTO $table ($keys) VALUES ($values);"
}

# Select records
db_select() {
    local table="$1"
    local columns="${2:-*}"
    local where="${3:-}"
    local limit="${4:-}"
    
    local query="SELECT $columns FROM $table"
    [[ -n "$where" ]] && query+=" WHERE $where"
    [[ -n "$limit" ]] && query+=" LIMIT $limit"
    
    db_query "$query"
}

# Update records
db_update() {
    local table="$1"
    local set_clause="$2"
    local where="${3:-1=1}"
    
    db_exec "UPDATE $table SET $set_clause WHERE $where;"
}

# Delete records
db_delete() {
    local table="$1"
    local where="${2:-1=1}"
    
    db_exec "DELETE FROM $table WHERE $where;"
}

# Transaction support
db_transaction() {
    local commands=("$@")
    
    case "$DB_DRIVER" in
        sqlite)
            {
                echo "BEGIN TRANSACTION;"
                for cmd in "${commands[@]}"; do
                    echo "$cmd;"
                done
                echo "COMMIT;"
            } | sqlite3 "$DB_DSN"
            ;;
        postgresql|mysql)
            {
                echo "BEGIN;"
                for cmd in "${commands[@]}"; do
                    echo "$cmd;"
                done
                echo "COMMIT;"
            } | db_exec ""
            ;;
    esac
}

# ===== Demo =====
if ! command -v sqlite3 &>/dev/null; then
    echo "sqlite3 not available for demo"
    exit 0
fi

echo "=== Database Abstraction Demo (SQLite) ==="

DB_DRIVER=sqlite
DB_DSN="/tmp/abstract_db_$$.db"

db_connect

# Create tables
db_create_table "users" \
    "id:serial:" \
    "name:TEXT:NOT NULL" \
    "email:TEXT:UNIQUE NOT NULL" \
    "created_at:INTEGER:DEFAULT (strftime('%s','now'))"

db_create_table "posts" \
    "id:serial:" \
    "user_id:INTEGER:NOT NULL" \
    "title:TEXT:NOT NULL" \
    "content:TEXT:"

echo ""
echo "Inserting data..."
db_insert "users" "name=Alice" "email=alice@example.com"
db_insert "users" "name=Bob" "email=bob@example.com"
db_insert "users" "name=Charlie" "email=charlie@example.com"

db_insert "posts" "user_id=1" "title=First Post" "content=Hello World"
db_insert "posts" "user_id=1" "title=Second Post" "content=More content"
db_insert "posts" "user_id=2" "title=Bob Post" "content=Bob writes here"

echo ""
echo "Users:"
db_select "users" "id,name,email" | while IFS=',' read -r id name email; do
    printf "  [%s] %s <%s>\n" "$id" "$name" "$email"
done

echo ""
echo "Posts by user 1:"
db_select "posts" "id,title" "user_id=1" | while IFS=',' read -r id title; do
    printf "  [%s] %s\n" "$id" "$title"
done

echo ""
echo "Update user 2:"
db_update "users" "name='Robert'" "id=2"
db_select "users" "name" "id=2"

echo ""
echo "Delete user 3:"
db_delete "users" "id=3"
echo "Remaining users: $(db_query 'SELECT COUNT(*) FROM users')"

rm -f "$DB_DSN"
```

---

## ขั้นตอนที่ 441: Redis Operations

```bash
#!/usr/bin/env bash
# redis_ops.sh

# ===== Redis Integration =====
REDIS_HOST="${REDIS_HOST:-localhost}"
REDIS_PORT="${REDIS_PORT:-6379}"
REDIS_AUTH="${REDIS_AUTH:-}"
REDIS_DB="${REDIS_DB:-0}"

# Send command to Redis
redis_cmd() {
    if command -v redis-cli &>/dev/null; then
        local args=(-h "$REDIS_HOST" -p "$REDIS_PORT" -n "$REDIS_DB")
        [[ -n "$REDIS_AUTH" ]] && args+=(-a "$REDIS_AUTH")
        redis-cli "${args[@]}" "$@" 2>/dev/null
    else
        echo "redis-cli not available"
        return 1
    fi
}

# String operations
redis_set() { redis_cmd SET "$1" "$2"; }
redis_get() { redis_cmd GET "$1"; }
redis_del() { redis_cmd DEL "$1"; }
redis_exists() { [[ "$(redis_cmd EXISTS "$1")" == "1" ]]; }
redis_expire() { redis_cmd EXPIRE "$1" "$2"; }
redis_setex() { redis_cmd SETEX "$1" "$2" "$3"; }  # set with TTL

# Counter operations
redis_incr() { redis_cmd INCR "$1"; }
redis_incrby() { redis_cmd INCRBY "$1" "$2"; }
redis_decr() { redis_cmd DECR "$1"; }

# Hash operations
redis_hset() { redis_cmd HSET "$1" "$2" "$3"; }
redis_hget() { redis_cmd HGET "$1" "$2"; }
redis_hgetall() { redis_cmd HGETALL "$1"; }
redis_hdel() { redis_cmd HDEL "$1" "$2"; }
redis_hmset() { redis_cmd HMSET "$@"; }

# List operations
redis_lpush() { redis_cmd LPUSH "$1" "$2"; }
redis_rpush() { redis_cmd RPUSH "$1" "$2"; }
redis_lpop() { redis_cmd LPOP "$1"; }
redis_rpop() { redis_cmd RPOP "$1"; }
redis_llen() { redis_cmd LLEN "$1"; }
redis_lrange() { redis_cmd LRANGE "$1" "${2:-0}" "${3:--1}"; }

# Set operations
redis_sadd() { redis_cmd SADD "$1" "$2"; }
redis_srem() { redis_cmd SREM "$1" "$2"; }
redis_smembers() { redis_cmd SMEMBERS "$1"; }
redis_sismember() { [[ "$(redis_cmd SISMEMBER "$1" "$2")" == "1" ]]; }
redis_scard() { redis_cmd SCARD "$1"; }

# Sorted set operations
redis_zadd() { redis_cmd ZADD "$1" "$2" "$3"; }  # key score member
redis_zrange() { redis_cmd ZRANGE "$1" "${2:-0}" "${3:--1}" WITHSCORES; }
redis_zrangebyscore() { redis_cmd ZRANGEBYSCORE "$1" "$2" "$3"; }
redis_zscore() { redis_cmd ZSCORE "$1" "$2"; }
redis_zrank() { redis_cmd ZRANK "$1" "$2"; }

# Pub/Sub
redis_publish() { redis_cmd PUBLISH "$1" "$2"; }
redis_subscribe() { redis_cmd SUBSCRIBE "$1"; }

# Pattern-based key search
redis_keys() {
    local pattern="${1:-*}"
    redis_cmd KEYS "$pattern"
}

# Flush
redis_flushdb() { redis_cmd FLUSHDB; }

# ===== Session Store =====
SESSION_TTL=3600  # 1 hour

session_create() {
    local user_id="$1"
    local session_id
    session_id=$(cat /proc/sys/kernel/random/uuid 2>/dev/null || \
                 openssl rand -hex 16)
    
    redis_hmset "session:$session_id" \
        "user_id" "$user_id" \
        "created_at" "$(date +%s)" \
        "last_activity" "$(date +%s)"
    
    redis_expire "session:$session_id" "$SESSION_TTL"
    echo "$session_id"
}

session_get() {
    local session_id="$1"
    redis_hgetall "session:$session_id"
}

session_touch() {
    local session_id="$1"
    redis_hset "session:$session_id" "last_activity" "$(date +%s)"
    redis_expire "session:$session_id" "$SESSION_TTL"
}

session_destroy() {
    local session_id="$1"
    redis_del "session:$session_id"
}

# ===== Rate Limiter =====
rate_limit_check() {
    local key="$1"
    local max_requests="${2:-10}"
    local window_seconds="${3:-60}"
    
    local full_key="ratelimit:$key"
    local current
    current=$(redis_cmd INCR "$full_key")
    
    if [[ "$current" == "1" ]]; then
        redis_expire "$full_key" "$window_seconds"
    fi
    
    if [[ ${current:-0} -le $max_requests ]]; then
        echo "OK: $current/$max_requests requests"
        return 0
    else
        local ttl
        ttl=$(redis_cmd TTL "$full_key")
        echo "RATE_LIMITED: $current/$max_requests requests. Reset in ${ttl}s"
        return 1
    fi
}

# ===== Demo =====
if ! command -v redis-cli &>/dev/null; then
    echo "redis-cli not installed"
    echo "Demo skipped. Install: sudo apt-get install redis-tools"
    exit 0
fi

# Test connection
if ! redis_cmd PING | grep -q "PONG"; then
    echo "Redis not available (not running)"
    exit 0
fi

echo "=== Redis Demo ==="
echo "Connected to Redis: $(redis_cmd INFO server | grep redis_version)"

# String operations
redis_set "greeting" "Hello, Redis!"
echo "Get: $(redis_get "greeting")"

# Counter
redis_set "page_views" "0"
for i in $(seq 1 5); do redis_incr "page_views" > /dev/null; done
echo "Page views: $(redis_get "page_views")"

# Hash (user profile)
redis_hmset "user:1" name "Alice" email "alice@example.com" age "30"
echo "User: $(redis_hget "user:1" "name") ($(redis_hget "user:1" "email"))"

# List (recent activity)
for action in "login" "view_page" "add_to_cart" "checkout"; do
    redis_rpush "user:1:activity" "$action"
done
echo "Recent activity: $(redis_lrange "user:1:activity")"

# Set (unique visitors)
for ip in "1.2.3.4" "5.6.7.8" "1.2.3.4" "9.10.11.12"; do
    redis_sadd "unique_visitors" "$ip"
done
echo "Unique visitors: $(redis_scard "unique_visitors")"

# Sorted set (leaderboard)
redis_zadd "leaderboard" 1000 "Alice"
redis_zadd "leaderboard" 850 "Bob"
redis_zadd "leaderboard" 1200 "Charlie"
echo "Leaderboard:"
redis_zrange "leaderboard" 0 -1 | paste - - | awk '{printf "  %s: %s\n", $1, $2}'

# Rate limiting
echo ""
echo "Rate limiting (max 3 req/min):"
for i in $(seq 1 5); do
    result=$(rate_limit_check "api:user:1" 3 60)
    echo "  Request $i: $result"
done

# Cleanup
redis_del "greeting" "page_views" "user:1" "user:1:activity" \
          "unique_visitors" "leaderboard" > /dev/null
```

---

## สรุป Part 24

### สิ่งที่เรียนรู้ (Steps 437-441):

| Step | Database | Operations |
|------|----------|------------|
| 437 | PostgreSQL | query, backup, migration |
| 438 | MySQL/MariaDB | export CSV, optimize, status |
| 439 | SQLite | KV store, job queue |
| 440 | Abstraction Layer | cross-database interface |
| 441 | Redis | strings, hash, list, set, pub/sub |

### Use Cases:
- **SQLite**: scripts, local storage, testing
- **PostgreSQL/MySQL**: production applications
- **Redis**: caching, sessions, rate limiting, queues

**ขั้นตอนต่อไป**: Part 25 - Advanced Scripting Patterns (จบ Module 2)
