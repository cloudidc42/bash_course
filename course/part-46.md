# Part 46: Database Engineering at Scale

## Module 4: Professional Level — Database Architecture

### ขั้นตอนที่ 538: PostgreSQL Enterprise Cluster Management

**PostgreSQL** clustering ระดับ enterprise ด้วย Patroni และ PgBouncer

```bash
#!/bin/bash
# postgresql-enterprise-manager.sh - PostgreSQL Enterprise Cluster Management

set -euo pipefail

PG_NAMESPACE="${PG_NAMESPACE:-postgresql}"
PG_CLUSTER_NAME="${PG_CLUSTER_NAME:-production-pg}"
CNPG_VERSION="${CNPG_VERSION:-1.22.0}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_cnpg_operator() {
    log "Installing CloudNativePG Operator..."
    
    kubectl create namespace "${PG_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    kubectl apply -f "https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.0.yaml"
    
    kubectl rollout status deployment/cnpg-controller-manager \
        -n cnpg-system --timeout=120s
    
    log "CloudNativePG operator installed"
}

create_postgresql_cluster() {
    local instances="${1:-3}"
    local storage_size="${2:-100Gi}"
    local pg_version="${3:-16}"
    
    log "Creating PostgreSQL cluster: ${PG_CLUSTER_NAME} (${instances} instances)"
    
    kubectl create secret generic "${PG_CLUSTER_NAME}-superuser" \
        --from-literal=username=postgres \
        --from-literal=password="$(openssl rand -base64 32)" \
        -n "${PG_NAMESPACE}" \
        --dry-run=client -o yaml | kubectl apply -f -
    
    kubectl create secret generic "${PG_CLUSTER_NAME}-app-user" \
        --from-literal=username=app \
        --from-literal=password="$(openssl rand -base64 24)" \
        -n "${PG_NAMESPACE}" \
        --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: ${PG_CLUSTER_NAME}
  namespace: ${PG_NAMESPACE}
spec:
  instances: ${instances}
  imageName: ghcr.io/cloudnative-pg/postgresql:${pg_version}
  
  postgresql:
    parameters:
      max_connections: "300"
      shared_buffers: "4GB"
      effective_cache_size: "12GB"
      maintenance_work_mem: "512MB"
      checkpoint_completion_target: "0.9"
      wal_buffers: "64MB"
      default_statistics_target: "100"
      random_page_cost: "1.1"
      effective_io_concurrency: "200"
      work_mem: "16MB"
      min_wal_size: "1GB"
      max_wal_size: "4GB"
      max_worker_processes: "8"
      max_parallel_workers_per_gather: "4"
      max_parallel_workers: "8"
      max_parallel_maintenance_workers: "4"
      log_min_duration_statement: "1000"
      log_checkpoints: "on"
      log_connections: "on"
      log_disconnections: "on"
      log_lock_waits: "on"
      log_temp_files: "0"
      log_autovacuum_min_duration: "250ms"
      autovacuum_max_workers: "4"
      autovacuum_naptime: "10s"
      autovacuum_vacuum_threshold: "50"
      autovacuum_vacuum_scale_factor: "0.05"
      autovacuum_analyze_threshold: "50"
      autovacuum_analyze_scale_factor: "0.025"
      wal_level: logical
      max_wal_senders: "20"
      max_replication_slots: "20"
      wal_keep_size: "1GB"
      hot_standby_feedback: "on"
    pg_hba:
      - host all all 10.0.0.0/8 scram-sha-256
      - host replication streaming_replica all scram-sha-256
  
  superuserSecret:
    name: ${PG_CLUSTER_NAME}-superuser
  
  bootstrap:
    initdb:
      database: app
      owner: app
      secret:
        name: ${PG_CLUSTER_NAME}-app-user
      postInitSQL:
        - CREATE EXTENSION IF NOT EXISTS pg_stat_statements
        - CREATE EXTENSION IF NOT EXISTS pgcrypto
        - CREATE EXTENSION IF NOT EXISTS uuid-ossp
        - CREATE EXTENSION IF NOT EXISTS pg_partman
        - CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE
  
  storage:
    size: ${storage_size}
    storageClass: fast-ssd
  
  walStorage:
    size: 20Gi
    storageClass: fast-ssd
  
  resources:
    requests:
      cpu: "2"
      memory: "8Gi"
    limits:
      cpu: "4"
      memory: "16Gi"
  
  affinity:
    enablePodAntiAffinity: true
    topologyKey: kubernetes.io/hostname
    podAntiAffinityType: required
  
  backup:
    barmanObjectStore:
      destinationPath: "s3://postgresql-backups/${PG_CLUSTER_NAME}"
      s3Credentials:
        accessKeyId:
          name: pg-s3-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: pg-s3-credentials
          key: ACCESS_SECRET_KEY
      wal:
        compression: gzip
        encryption: AES256
        maxParallel: 8
      data:
        compression: gzip
        encryption: AES256
        immediateCheckpoint: false
        jobs: 4
    retentionPolicy: "30d"
  
  monitoring:
    enablePodMonitor: true
    
  externalClusters: []
EOF
    
    log "Waiting for PostgreSQL cluster..."
    kubectl wait cluster/${PG_CLUSTER_NAME} \
        --for=condition=Ready \
        --timeout=300s \
        -n "${PG_NAMESPACE}"
    
    log "PostgreSQL cluster created: ${PG_CLUSTER_NAME}"
}

setup_pgbouncer() {
    local pool_mode="${1:-transaction}"
    local max_connections="${2:-1000}"
    local pool_size="${3:-25}"
    
    log "Setting up PgBouncer connection pooler..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: ${PG_CLUSTER_NAME}-pooler
  namespace: ${PG_NAMESPACE}
spec:
  cluster:
    name: ${PG_CLUSTER_NAME}
  instances: 2
  type: rw
  pgbouncer:
    poolMode: ${pool_mode}
    parameters:
      max_client_conn: "${max_connections}"
      default_pool_size: "${pool_size}"
      min_pool_size: "5"
      reserve_pool_size: "5"
      reserve_pool_timeout: "5.0"
      max_db_connections: "100"
      max_user_connections: "100"
      server_check_delay: "30"
      server_check_query: "select 1"
      server_fast_close: "1"
      server_idle_timeout: "600"
      client_idle_timeout: "0"
      client_login_timeout: "60"
      query_timeout: "0"
      query_wait_timeout: "120"
      client_tls_sslmode: require
  resources:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
EOF
    
    log "PgBouncer pooler deployed"
}

create_postgresql_backup() {
    local backup_name="${1:-backup-$(date '+%Y%m%d-%H%M%S')}"
    
    log "Creating PostgreSQL backup: ${backup_name}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: ${backup_name}
  namespace: ${PG_NAMESPACE}
spec:
  method: barmanObjectStore
  cluster:
    name: ${PG_CLUSTER_NAME}
EOF
    
    kubectl wait backup/${backup_name} \
        --for=condition=Completed \
        --timeout=3600s \
        -n "${PG_NAMESPACE}"
    
    log "Backup completed: ${backup_name}"
}

setup_logical_replication() {
    local target_host="${1:-replica-pg.staging}"
    local replication_slot="${2:-logical_slot_1}"
    local publications="${3:-ALL TABLES}"
    
    log "Setting up logical replication to: ${target_host}"
    
    local primary_pod=$(kubectl get pod -n "${PG_NAMESPACE}" \
        -l "postgresql=${PG_CLUSTER_NAME},role=primary" \
        -o name | head -1 | cut -d/ -f2)
    
    kubectl exec -n "${PG_NAMESPACE}" "${primary_pod}" -- \
        psql -U postgres <<SQL
-- Create publication
CREATE PUBLICATION full_replication FOR ${publications};

-- Create replication slot
SELECT pg_create_logical_replication_slot('${replication_slot}', 'pgoutput');

-- Grant replication privilege
GRANT REPLICATION ON DATABASE app TO app;

SELECT pubname, puballtables, pubinsert, pubupdate, pubdelete 
FROM pg_publication;
SQL
    
    log "Logical replication configured"
}

run_pg_health_check() {
    log "=== PostgreSQL Health Check ==="
    
    local primary_pod=$(kubectl get pod -n "${PG_NAMESPACE}" \
        -l "postgresql=${PG_CLUSTER_NAME},role=primary" \
        -o name | head -1 | cut -d/ -f2)
    
    if [[ -z "${primary_pod}" ]]; then
        log "ERROR: Primary pod not found"
        return 1
    fi
    
    kubectl exec -n "${PG_NAMESPACE}" "${primary_pod}" -- \
        psql -U postgres <<'SQL'
-- Cluster status
SELECT * FROM pg_stat_replication \g

-- Active connections
SELECT count(*), state, wait_event_type, wait_event 
FROM pg_stat_activity 
GROUP BY state, wait_event_type, wait_event 
ORDER BY count DESC \g

-- Long running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
  AND state != 'idle' \g

-- Table bloat
SELECT relname, n_dead_tup, n_live_tup, 
       round(n_dead_tup::numeric/NULLIF(n_live_tup, 0)*100, 2) AS dead_ratio,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20 \g

-- Index usage
SELECT schemaname, relname, indexrelname, 
       idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC
LIMIT 20 \g
SQL
}

pg_performance_tuning() {
    local cpu_count="${1:-4}"
    local total_ram_gb="${2:-16}"
    
    log "Calculating PostgreSQL performance settings..."
    
    local shared_buffers=$(( total_ram_gb * 1024 / 4 ))
    local effective_cache=$(( total_ram_gb * 1024 * 3 / 4 ))
    local maintenance_work_mem=$(( total_ram_gb * 1024 / 16 ))
    local work_mem=$(( (total_ram_gb * 1024 / 4) / (cpu_count * 2) ))
    
    cat <<EOF
=== PostgreSQL Performance Settings ===
CPU Cores: ${cpu_count}
Total RAM: ${total_ram_gb}GB

Recommended Settings:
--------------------
shared_buffers = ${shared_buffers}MB
effective_cache_size = ${effective_cache}MB
maintenance_work_mem = ${maintenance_work_mem}MB
work_mem = ${work_mem}MB
max_connections = $(( cpu_count * 50 ))
max_worker_processes = ${cpu_count}
max_parallel_workers = ${cpu_count}
max_parallel_workers_per_gather = $(( cpu_count / 2 ))

WAL Settings:
------------
wal_buffers = 64MB
min_wal_size = 1GB
max_wal_size = 4GB
checkpoint_completion_target = 0.9

Autovacuum Settings:
-------------------
autovacuum_max_workers = $(( cpu_count / 2 ))
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 400
EOF
}

case "${1:-help}" in
    "install-operator") install_cnpg_operator ;;
    "create-cluster") create_postgresql_cluster "${2:-3}" "${3:-100Gi}" "${4:-16}" ;;
    "setup-pgbouncer") setup_pgbouncer "${2:-transaction}" "${3:-1000}" "${4:-25}" ;;
    "backup") create_postgresql_backup "${2:-}" ;;
    "logical-replication") setup_logical_replication "$2" "${3:-logical_slot_1}" "${4:-ALL TABLES}" ;;
    "health-check") run_pg_health_check ;;
    "tune") pg_performance_tuning "${2:-4}" "${3:-16}" ;;
    *) echo "Usage: $0 {install-operator|create-cluster|setup-pgbouncer|backup|logical-replication|health-check|tune}" ;;
esac
```

### ขั้นตอนที่ 539: MongoDB Atlas and Sharded Cluster Management

**MongoDB** sharded cluster สำหรับ document database ขนาด enterprise

```bash
#!/bin/bash
# mongodb-enterprise-manager.sh - MongoDB Enterprise Cluster Management

set -euo pipefail

MONGO_NAMESPACE="${MONGO_NAMESPACE:-mongodb}"
MONGO_CLUSTER_NAME="${MONGO_CLUSTER_NAME:-production-mongo}"
MONGO_OP_VERSION="${MONGO_OP_VERSION:-0.9.0}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_mongodb_operator() {
    log "Installing MongoDB Community Operator..."
    
    kubectl create namespace "${MONGO_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    helm repo add mongodb https://mongodb.github.io/helm-charts
    helm repo update
    
    helm upgrade --install community-operator mongodb/community-operator \
        --namespace "${MONGO_NAMESPACE}" \
        --set operator.watchNamespace="${MONGO_NAMESPACE}" \
        --wait
    
    log "MongoDB operator installed"
}

create_mongodb_replicaset() {
    local members="${1:-3}"
    local storage_size="${2:-100Gi}"
    local mongo_version="${3:-7.0.4}"
    
    log "Creating MongoDB ReplicaSet: ${MONGO_CLUSTER_NAME} (${members} members)"
    
    kubectl create secret generic "${MONGO_CLUSTER_NAME}-admin" \
        --from-literal=password="$(openssl rand -base64 32)" \
        -n "${MONGO_NAMESPACE}" \
        --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF | kubectl apply -f -
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: ${MONGO_CLUSTER_NAME}
  namespace: ${MONGO_NAMESPACE}
spec:
  members: ${members}
  type: ReplicaSet
  version: "${mongo_version}"
  
  security:
    authentication:
      modes: ["SCRAM"]
    tls:
      enabled: true
      caConfigMapRef:
        name: ${MONGO_CLUSTER_NAME}-ca
  
  users:
    - name: admin
      db: admin
      passwordSecretRef:
        name: ${MONGO_CLUSTER_NAME}-admin
        key: password
      roles:
        - name: clusterAdmin
          db: admin
        - name: userAdminAnyDatabase
          db: admin
        - name: readWriteAnyDatabase
          db: admin
        - name: dbAdminAnyDatabase
          db: admin
      scramCredentialsSecretName: admin-scram
    
    - name: app-user
      db: appdb
      passwordSecretRef:
        name: ${MONGO_CLUSTER_NAME}-app-user
        key: password
      roles:
        - name: readWrite
          db: appdb
        - name: dbAdmin
          db: appdb
      scramCredentialsSecretName: app-scram
  
  statefulSet:
    spec:
      volumeClaimTemplates:
        - metadata:
            name: data-volume
          spec:
            accessModes: [ReadWriteOnce]
            resources:
              requests:
                storage: ${storage_size}
            storageClassName: fast-ssd
        
        - metadata:
            name: logs-volume
          spec:
            accessModes: [ReadWriteOnce]
            resources:
              requests:
                storage: 10Gi
            storageClassName: standard
      
      template:
        spec:
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchLabels:
                      app: ${MONGO_CLUSTER_NAME}-svc
                  topologyKey: kubernetes.io/hostname
  
  additionalMongodConfig:
    operationProfiling:
      mode: slowOp
      slowOpThresholdMs: 100
      slowOpSampleRate: 0.5
    storage:
      wiredTiger:
        engineConfig:
          cacheSizeGB: 4
          journalCompressor: snappy
          directoryForIndexes: false
        collectionConfig:
          blockCompressor: snappy
        indexConfig:
          prefixCompression: true
    replication:
      oplogSizeMB: 10240
    net:
      maxIncomingConnections: 5000
    setParameter:
      transactionLifetimeLimitSeconds: 120
      cursorTimeoutMillis: 600000
      maxNumberOfTransactionOperationsInSingleOplogEntry: 4000

EOF
    
    log "MongoDB ReplicaSet creating: ${MONGO_CLUSTER_NAME}"
}

create_indexes() {
    local database="${1:-appdb}"
    local collection="${2:-users}"
    
    log "Creating optimized indexes for ${database}.${collection}"
    
    local primary_pod="${MONGO_CLUSTER_NAME}-0"
    local connection_string="mongodb://admin:$(kubectl get secret ${MONGO_CLUSTER_NAME}-admin -n ${MONGO_NAMESPACE} -o jsonpath='{.data.password}' | base64 -d)@${MONGO_CLUSTER_NAME}-svc.${MONGO_NAMESPACE}:27017/admin?replicaSet=${MONGO_CLUSTER_NAME}&tls=true"
    
    kubectl exec -n "${MONGO_NAMESPACE}" "${primary_pod}" -- \
        mongosh "${connection_string}" <<'SCRIPT'
use appdb

// Users collection indexes
db.users.createIndex({email: 1}, {unique: true, background: true})
db.users.createIndex({username: 1}, {unique: true, sparse: true})
db.users.createIndex({createdAt: -1}, {background: true})
db.users.createIndex({status: 1, createdAt: -1}, {background: true})
db.users.createIndex(
    {firstName: "text", lastName: "text", email: "text"},
    {weights: {firstName: 10, lastName: 8, email: 5}, name: "user_text_search"}
)

// Events collection - Time Series optimized
db.events.createIndex({userId: 1, timestamp: -1}, {background: true})
db.events.createIndex({type: 1, timestamp: -1}, {background: true})
db.events.createIndex({timestamp: 1}, {
    expireAfterSeconds: 7776000,
    background: true
})

// Products collection
db.products.createIndex({category: 1, price: 1})
db.products.createIndex({sku: 1}, {unique: true})
db.products.createIndex({tags: 1}, {background: true})
db.products.createIndex(
    {name: "text", description: "text", tags: "text"},
    {weights: {name: 20, tags: 10, description: 5}}
)

print("Indexes created successfully")
SCRIPT
    
    log "Indexes created for ${database}.${collection}"
}

setup_change_streams() {
    log "Setting up MongoDB change streams consumer..."
    
    cat <<'PYTHON' > /tmp/change_stream_consumer.py
"""
MongoDB Change Streams Consumer
Streams database changes to Kafka for real-time processing
"""
import asyncio
import json
from datetime import datetime
from motor.motor_asyncio import AsyncIOMotorClient
from aiokafka import AIOKafkaProducer

MONGO_URI = "mongodb://admin:password@production-mongo-svc.mongodb:27017/admin?replicaSet=production-mongo&tls=true"
KAFKA_BOOTSTRAP = "production-kafka-kafka-bootstrap.kafka:9092"
KAFKA_TOPIC = "mongodb.changes"

async def stream_changes(database: str, collection: str):
    """Stream MongoDB changes to Kafka"""
    mongo_client = AsyncIOMotorClient(MONGO_URI)
    kafka_producer = AIOKafkaProducer(
        bootstrap_servers=KAFKA_BOOTSTRAP,
        value_serializer=lambda v: json.dumps(v, default=str).encode('utf-8'),
        key_serializer=lambda k: k.encode('utf-8') if k else None,
        compression_type='snappy',
        acks='all',
        enable_idempotence=True
    )
    
    await kafka_producer.start()
    
    try:
        pipeline = [
            {'$match': {'operationType': {'$in': ['insert', 'update', 'delete', 'replace']}}},
            {'$addFields': {'fullDocument': {'$ifNull': ['$fullDocument', {}]}}}
        ]
        
        collection_obj = mongo_client[database][collection]
        
        async with collection_obj.watch(
            pipeline,
            full_document='updateLookup',
            full_document_before_change='whenAvailable'
        ) as change_stream:
            print(f"Watching changes on {database}.{collection}...")
            
            async for change in change_stream:
                event = {
                    'operation': change['operationType'],
                    'collection': collection,
                    'database': database,
                    'document_id': str(change['documentKey']['_id']),
                    'document': change.get('fullDocument', {}),
                    'timestamp': datetime.utcnow().isoformat(),
                    'resume_token': str(change['_id'])
                }
                
                doc_id = str(change['documentKey']['_id'])
                
                await kafka_producer.send_and_wait(
                    KAFKA_TOPIC,
                    key=f"{database}.{collection}.{doc_id}",
                    value=event
                )
                
                print(f"Sent change: {change['operationType']} on {doc_id}")
    finally:
        await kafka_producer.stop()
        mongo_client.close()

if __name__ == '__main__':
    asyncio.run(stream_changes('appdb', 'users'))
PYTHON
    
    log "Change stream consumer saved: /tmp/change_stream_consumer.py"
}

mongo_performance_analysis() {
    log "=== MongoDB Performance Analysis ==="
    
    local primary_pod="${MONGO_CLUSTER_NAME}-0"
    
    kubectl exec -n "${MONGO_NAMESPACE}" "${primary_pod}" -- \
        mongosh --quiet <<'SCRIPT'
// Server status
const status = db.serverStatus()
print("=== Server Status ===")
print("Version:", status.version)
print("Uptime:", status.uptimeEstimate, "seconds")
print("Connections current:", status.connections.current)
print("Connections available:", status.connections.available)
print("Ops per second:", JSON.stringify({
    insert: status.opcounters.insert,
    query: status.opcounters.query,
    update: status.opcounters.update,
    delete: status.opcounters.delete
}))

// Replica set status
const rsStatus = rs.status()
print("\n=== Replica Set Status ===")
rsStatus.members.forEach(m => {
    print(`  ${m.name}: ${m.stateStr} (health: ${m.health})`)
})

// Slow operations
const currentOps = db.currentOp({
    active: true,
    secs_running: {$gt: 5}
})
if (currentOps.inprog.length > 0) {
    print("\n=== Slow Operations (>5s) ===")
    currentOps.inprog.forEach(op => {
        print(`  opid: ${op.opid}, secs: ${op.secs_running}, ns: ${op.ns}`)
    })
}

// Index stats
use appdb
print("\n=== Index Usage Stats ===")
db.users.aggregate([{$indexStats: {}}]).forEach(stat => {
    print(`  ${stat.name}: ${stat.accesses.ops} accesses`)
})
SCRIPT
}

case "${1:-help}" in
    "install") install_mongodb_operator ;;
    "create-cluster") create_mongodb_replicaset "${2:-3}" "${3:-100Gi}" "${4:-7.0.4}" ;;
    "create-indexes") create_indexes "${2:-appdb}" "${3:-users}" ;;
    "change-streams") setup_change_streams ;;
    "analyze") mongo_performance_analysis ;;
    *) echo "Usage: $0 {install|create-cluster|create-indexes|change-streams|analyze}" ;;
esac
```

### ขั้นตอนที่ 540: Redis Enterprise Cluster

**Redis** cluster ระดับ enterprise พร้อม high availability

```bash
#!/bin/bash
# redis-enterprise-manager.sh - Redis Enterprise Cluster Management

set -euo pipefail

REDIS_NAMESPACE="${REDIS_NAMESPACE:-redis}"
REDIS_CLUSTER_NAME="${REDIS_CLUSTER_NAME:-production-redis}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

deploy_redis_cluster() {
    local shards="${1:-3}"
    local replicas_per_shard="${2:-1}"
    local memory_gb="${3:-4}"
    
    log "Deploying Redis Cluster: ${shards} shards x ${replicas_per_shard} replicas"
    
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update
    
    kubectl create namespace "${REDIS_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    local total_nodes=$(( shards * (replicas_per_shard + 1) ))
    
    cat <<EOF > /tmp/redis-cluster-values.yaml
cluster:
  enabled: true
  slaveCount: ${replicas_per_shard}
  
nodes:
  replicas: ${total_nodes}

redis:
  port: 6379
  configmap: |
    maxmemory $(( memory_gb * 1024 ))mb
    maxmemory-policy allkeys-lru
    maxmemory-samples 10
    
    activerehashing yes
    aof-rewrite-incremental-fsync yes
    rdb-save-incremental-fsync yes
    
    lazyfree-lazy-eviction yes
    lazyfree-lazy-expire yes
    lazyfree-lazy-server-del yes
    replica-lazy-flush yes
    
    timeout 300
    tcp-keepalive 60
    
    appendonly yes
    appendfsync everysec
    no-appendfsync-on-rewrite no
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    
    slowlog-log-slower-than 10000
    slowlog-max-len 128
    
    latency-monitor-threshold 100
    
    protected-mode yes
    bind 0.0.0.0
    
    cluster-node-timeout 15000
    cluster-require-full-coverage no
    cluster-migration-barrier 1

persistence:
  enabled: true
  storageClass: fast-ssd
  size: 50Gi

resources:
  requests:
    cpu: "500m"
    memory: "${memory_gb}Gi"
  limits:
    cpu: "2"
    memory: "$(( memory_gb * 2 ))Gi"

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring

affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app.kubernetes.io/name
              operator: In
              values: [redis]
        topologyKey: kubernetes.io/hostname
EOF
    
    helm upgrade --install "${REDIS_CLUSTER_NAME}" bitnami/redis-cluster \
        --namespace "${REDIS_NAMESPACE}" \
        -f /tmp/redis-cluster-values.yaml \
        --wait --timeout=600s
    
    log "Redis Cluster deployed: ${REDIS_CLUSTER_NAME}"
}

configure_redis_sentinel() {
    local sentinels="${1:-3}"
    local quorum="${2:-2}"
    
    log "Deploying Redis with Sentinel for HA..."
    
    cat <<EOF > /tmp/redis-sentinel-values.yaml
architecture: replication

auth:
  enabled: true
  sentinel: true
  password: "$(openssl rand -base64 32)"

sentinel:
  enabled: true
  count: ${sentinels}
  quorum: ${quorum}
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "256Mi"

replica:
  replicaCount: 3
  resources:
    requests:
      cpu: "500m"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  persistence:
    enabled: true
    size: 20Gi
    storageClass: fast-ssd

master:
  resources:
    requests:
      cpu: "500m"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  persistence:
    enabled: true
    size: 20Gi
    storageClass: fast-ssd
  
  configuration: |
    maxmemory 3gb
    maxmemory-policy allkeys-lru
    appendonly yes
    appendfsync everysec
    slowlog-log-slower-than 10000

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
EOF
    
    helm upgrade --install "${REDIS_CLUSTER_NAME}-sentinel" bitnami/redis \
        --namespace "${REDIS_NAMESPACE}" \
        -f /tmp/redis-sentinel-values.yaml \
        --wait
    
    log "Redis Sentinel cluster deployed"
}

setup_redis_caching_patterns() {
    log "Implementing Redis caching patterns..."
    
    cat <<'PYTHON' > /tmp/redis_caching_patterns.py
"""
Redis Caching Patterns for Production Use
"""
import redis
import json
import time
import hashlib
from typing import Any, Optional, Callable
from functools import wraps

class RedisCache:
    def __init__(self, host: str = 'redis', port: int = 6379, db: int = 0):
        self.pool = redis.ConnectionPool(
            host=host, port=port, db=db,
            max_connections=100,
            decode_responses=True,
            socket_connect_timeout=5,
            socket_timeout=5,
            retry_on_timeout=True,
            health_check_interval=30
        )
        self.client = redis.Redis(connection_pool=self.pool)
    
    def cache_aside(self, key: str, ttl: int, fetch_func: Callable) -> Any:
        """Cache-aside pattern: check cache, then fetch if missing"""
        cached = self.client.get(key)
        if cached:
            return json.loads(cached)
        
        data = fetch_func()
        
        pipeline = self.client.pipeline()
        pipeline.setex(key, ttl, json.dumps(data, default=str))
        pipeline.execute()
        
        return data
    
    def write_through(self, key: str, data: Any, ttl: int, persist_func: Callable) -> None:
        """Write-through: update cache and db simultaneously"""
        with self.client.pipeline() as pipe:
            pipe.setex(key, ttl, json.dumps(data, default=str))
            pipe.execute()
        
        persist_func(data)
    
    def write_behind(self, key: str, data: Any, ttl: int) -> None:
        """Write-behind: update cache, async db write"""
        with self.client.pipeline() as pipe:
            pipe.setex(key, ttl, json.dumps(data, default=str))
            # Add to write queue
            pipe.lpush('write_queue', json.dumps({
                'key': key,
                'data': data,
                'timestamp': time.time()
            }))
            pipe.execute()
    
    def cache_decorator(self, ttl: int = 300, key_prefix: str = ''):
        """Decorator for caching function results"""
        def decorator(func: Callable):
            @wraps(func)
            def wrapper(*args, **kwargs):
                cache_key = f"{key_prefix}:{func.__name__}:{hashlib.md5(str(args + tuple(sorted(kwargs.items()))).encode()).hexdigest()}"
                
                cached = self.client.get(cache_key)
                if cached:
                    return json.loads(cached)
                
                result = func(*args, **kwargs)
                self.client.setex(cache_key, ttl, json.dumps(result, default=str))
                return result
            return wrapper
        return decorator
    
    def distributed_lock(self, lock_name: str, timeout: int = 30):
        """Distributed lock using Redis SET NX PX"""
        lock_key = f"lock:{lock_name}"
        lock_id = f"{time.time()}-{id(self)}"
        
        acquired = self.client.set(lock_key, lock_id, nx=True, px=timeout*1000)
        
        return acquired, lock_key, lock_id
    
    def release_lock(self, lock_key: str, lock_id: str) -> bool:
        """Release distributed lock atomically"""
        lua_script = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
        """
        result = self.client.eval(lua_script, 1, lock_key, lock_id)
        return bool(result)
    
    def rate_limiter(self, key: str, limit: int, window: int) -> tuple:
        """Sliding window rate limiter"""
        now = time.time()
        window_start = now - window
        
        with self.client.pipeline() as pipe:
            pipe.zremrangebyscore(key, 0, window_start)
            pipe.zcard(key)
            pipe.zadd(key, {str(now): now})
            pipe.expire(key, window + 1)
            results = pipe.execute()
        
        count = results[1]
        allowed = count < limit
        
        return allowed, count, limit
    
    def session_management(self, session_id: str, data: dict, ttl: int = 3600) -> None:
        """Session data management"""
        key = f"session:{session_id}"
        self.client.hset(key, mapping={k: json.dumps(v) for k, v in data.items()})
        self.client.expire(key, ttl)
    
    def get_session(self, session_id: str) -> Optional[dict]:
        """Retrieve session data"""
        key = f"session:{session_id}"
        data = self.client.hgetall(key)
        return {k: json.loads(v) for k, v in data.items()} if data else None
    
    def leaderboard(self, board_name: str, member: str, score: float) -> None:
        """Real-time leaderboard using sorted sets"""
        self.client.zadd(f"leaderboard:{board_name}", {member: score})
    
    def get_top_n(self, board_name: str, n: int = 10) -> list:
        """Get top N from leaderboard"""
        return self.client.zrevrangebyscore(
            f"leaderboard:{board_name}",
            '+inf', '-inf',
            start=0, num=n,
            withscores=True
        )

cache = RedisCache(host='production-redis-cluster.redis', port=6379)

@cache.cache_decorator(ttl=600, key_prefix='user')
def get_user_profile(user_id: str) -> dict:
    # Simulated DB fetch
    return {'id': user_id, 'name': 'User', 'email': 'user@example.com'}

print("Redis caching patterns loaded")
PYTHON
    
    log "Redis caching patterns: /tmp/redis_caching_patterns.py"
}

monitor_redis_cluster() {
    log "=== Redis Cluster Status ==="
    
    kubectl exec -n "${REDIS_NAMESPACE}" \
        "$(kubectl get pod -n "${REDIS_NAMESPACE}" -l "app.kubernetes.io/name=redis" -o name | head -1 | cut -d/ -f2)" -- \
        redis-cli cluster info 2>/dev/null || \
        kubectl get pods -n "${REDIS_NAMESPACE}"
    
    log "=== Redis Memory Stats ==="
    kubectl exec -n "${REDIS_NAMESPACE}" \
        "$(kubectl get pod -n "${REDIS_NAMESPACE}" -l "app.kubernetes.io/name=redis" -o name | head -1 | cut -d/ -f2)" -- \
        redis-cli info memory 2>/dev/null | grep -E "used_memory:|maxmemory:|mem_fragmentation_ratio:"
}

case "${1:-help}" in
    "deploy-cluster") deploy_redis_cluster "${2:-3}" "${3:-1}" "${4:-4}" ;;
    "deploy-sentinel") configure_redis_sentinel "${2:-3}" "${3:-2}" ;;
    "caching-patterns") setup_redis_caching_patterns ;;
    "monitor") monitor_redis_cluster ;;
    *) echo "Usage: $0 {deploy-cluster|deploy-sentinel|caching-patterns|monitor}" ;;
esac
```

---

## สรุป Part 46

ในส่วนนี้เราได้เรียนรู้:

| ขั้นตอน | หัวข้อ | เครื่องมือหลัก |
|---------|--------|----------------|
| 538 | PostgreSQL Enterprise Cluster | CloudNativePG, PgBouncer, Logical Replication, Backup |
| 539 | MongoDB Enterprise Cluster | Community Operator, Indexes, Change Streams |
| 540 | Redis Enterprise Cluster | Redis Cluster, Sentinel HA, Caching Patterns, Rate Limiting |

### ขั้นตอนต่อไป: Part 47 - Global Load Balancing and Traffic Management
