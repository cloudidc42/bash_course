# Part 67: Database Engineering at Scale, NewSQL, Distributed Transactions และ Data Mesh

## ภาพรวม
Part นี้ครอบคลุม advanced database patterns: CockroachDB, TiDB, distributed transactions, CQRS materialized views และ Data Mesh architecture

---

## Step 616: CockroachDB — Distributed SQL + Multi-Region

```bash
cat > cockroachdb-setup.sh << 'SCRIPT'
#!/bin/bash
# CockroachDB: Distributed SQL with Multi-Region Survivability

set -euo pipefail

echo "=== CockroachDB Multi-Region Setup ==="

# ─── 1. Deploy CockroachDB on Kubernetes ──────────────────────────────────
helm repo add cockroachdb https://charts.cockroachdb.com
helm upgrade --install cockroachdb cockroachdb/cockroachdb \
  --namespace cockroachdb \
  --create-namespace \
  --values - << 'EOF'
conf:
  single-node: false
  cluster-name: payment-crdb

statefulset:
  replicas: 9    # 3 nodes per region × 3 regions

  resources:
    requests:
      cpu: 8
      memory: 32Gi
    limits:
      cpu: 16
      memory: 64Gi

  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule

storage:
  persistentVolume:
    size: 500Gi
    storageClass: gp3-io2

tls:
  enabled: true
  selfSigner:
    enabled: true

init:
  provisioning:
    enabled: true
    databases:
      - name: payments
        owners: ["payments_admin"]
    users:
      - name: payments_app
        password:
          secretKeyRef:
            name: crdb-app-secret
            key: password
        options: ["LOGIN"]
EOF

# ─── 2. Multi-Region Schema with Regional Tables ───────────────────────────
cat > crdb-multi-region.sql << 'EOF'
-- Configure multi-region database
ALTER DATABASE payments ADD REGION "ap-southeast-1";
ALTER DATABASE payments ADD REGION "us-east-1";
ALTER DATABASE payments ADD REGION "eu-west-1";
ALTER DATABASE payments SET PRIMARY REGION "ap-southeast-1";

-- Survivability goal: survive regional failure
ALTER DATABASE payments SURVIVE REGION FAILURE;

-- Global table: shared config data (low-write, read from anywhere)
CREATE TABLE payment_methods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING NOT NULL,
    provider STRING NOT NULL,
    config JSONB DEFAULT '{}',
    is_active BOOL DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT now()
) LOCALITY GLOBAL;  -- Cached in all regions

-- Regional-by-row: route data to user's home region
CREATE TABLE users (
    id UUID DEFAULT gen_random_uuid(),
    email STRING UNIQUE NOT NULL,
    tier STRING DEFAULT 'standard',
    home_region crdb_internal_region NOT NULL DEFAULT gateway_region(),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (home_region, id)
) LOCALITY REGIONAL BY ROW;  -- Rows in primary + replicated across regions

-- Regional-by-row: payments stay near user's region
CREATE TABLE payments (
    id UUID DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    amount DECIMAL(18, 6) NOT NULL,
    currency STRING(3) NOT NULL,
    status STRING NOT NULL DEFAULT 'pending',
    gateway_ref STRING,
    metadata JSONB DEFAULT '{}',
    crdb_region crdb_internal_region NOT NULL DEFAULT gateway_region(),
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (crdb_region, id),
    INDEX payments_user_idx (crdb_region, user_id, created_at DESC),
    INDEX payments_status_idx (crdb_region, status, created_at DESC)
) LOCALITY REGIONAL BY ROW;

-- Check: Strong consistency cross-region query
-- SELECT * FROM payments WHERE user_id = $1 ORDER BY created_at DESC LIMIT 10;
-- ↑ CockroachDB routes to correct region based on crdb_region column

-- Follower reads: near-realtime analytics (can read from local replica)
-- SET TRANSACTION AS OF SYSTEM TIME follower_read_timestamp();
-- SELECT COUNT(*), SUM(amount) FROM payments WHERE crdb_region = 'ap-southeast-1';

-- Multi-region transaction example
BEGIN;
  INSERT INTO payments (user_id, amount, currency, crdb_region)
  VALUES ($1, $2, $3, 'ap-southeast-1');
  
  UPDATE users 
  SET metadata = metadata || '{"last_payment_at": "NOW"}'
  WHERE id = $1 AND home_region = 'ap-southeast-1';
COMMIT;
-- CockroachDB ensures ACID across regions using Raft + distributed transactions
EOF

# ─── 3. Connection Pooling with PgBouncer ─────────────────────────────────
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: cockroachdb
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pgbouncer
  template:
    spec:
      containers:
        - name: pgbouncer
          image: pgbouncer/pgbouncer:1.23
          env:
            - name: DATABASES_HOST
              value: cockroachdb-public.cockroachdb.svc.cluster.local
            - name: DATABASES_PORT
              value: "26257"
            - name: DATABASES_DBNAME
              value: payments
            - name: PGBOUNCER_POOL_MODE
              value: transaction    # Best for high concurrency
            - name: PGBOUNCER_MAX_CLIENT_CONN
              value: "10000"
            - name: PGBOUNCER_DEFAULT_POOL_SIZE
              value: "50"
            - name: PGBOUNCER_MIN_POOL_SIZE
              value: "10"
            - name: PGBOUNCER_MAX_DB_CONNECTIONS
              value: "200"
            - name: PGBOUNCER_QUERY_TIMEOUT
              value: "30"
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2
              memory: 1Gi
EOF

echo "CockroachDB multi-region configured"
echo "=== Step 616 Complete: CockroachDB Multi-Region ==="
SCRIPT
chmod +x cockroachdb-setup.sh
echo "Script created: cockroachdb-setup.sh"
```

**สิ่งที่เรียนรู้:**
- CockroachDB: `SURVIVE REGION FAILURE` (ต้องการ 3 regions), `ADD REGION` DDL
- LOCALITY GLOBAL: config tables cached ใน all regions (low-write, fast-read anywhere)
- LOCALITY REGIONAL BY ROW: แต่ละ row routed ไป home region ด้วย `crdb_region` column
- Follower reads: near-realtime queries จาก local replica (1.5s lag, แต่ 0ms network)
- PgBouncer transaction mode: 10,000 client connections → 200 actual DB connections

---

## Step 617: Data Mesh Architecture — Domain-Oriented Data Products

```bash
cat > data-mesh.sh << 'SCRIPT'
#!/bin/bash
# Data Mesh: Domain-Oriented Data Products with Self-Service Infrastructure

set -euo pipefail

echo "=== Data Mesh Architecture ==="

# ─── 1. Data Product Definition ────────────────────────────────────────────
cat > data-product-spec.yaml << 'EOF'
# Data Product: Payments Analytics
# Owner: payments-team
# Domain: financial

apiVersion: datamesh.example.com/v1
kind: DataProduct
metadata:
  name: payment-analytics
  namespace: payments-domain
  labels:
    domain: financial
    owner: payments-team
    tier: gold   # gold/silver/bronze data quality tiers
  annotations:
    datamesh.example.com/steward: "data-eng@payments-team.example.com"
    datamesh.example.com/sla: "99.9"
    datamesh.example.com/freshness: "15m"   # Data freshness SLA

spec:
  description: "Aggregated payment analytics for business intelligence and reporting"
  version: "2.1.0"
  
  # Input ports (data sources)
  inputPorts:
    - name: raw-payments
      type: kafka-topic
      location: kafka://payments/payment-events
      schema: avro://schema-registry/payment-events-value
      
    - name: user-profiles
      type: data-product
      location: dp://users-domain/user-profiles
      version: ">=1.0.0"
  
  # Output ports (data product interfaces)
  outputPorts:
    - name: payment-summary-daily
      type: iceberg-table
      location: s3://data-lake/payments/summary-daily/
      format: iceberg
      schema:
        fields:
          - name: date
            type: date
          - name: currency
            type: string
          - name: total_volume
            type: decimal(18,6)
          - name: transaction_count
            type: bigint
          - name: unique_users
            type: bigint
          - name: avg_amount
            type: decimal(18,6)
          - name: failure_rate
            type: double
      
      sla:
        freshness: PT15M      # ISO 8601 duration: 15 minutes
        availability: 99.9%
        
    - name: payment-events-stream
      type: kafka-topic
      location: kafka://data-platform/payments.payment-analytics.enriched
      description: "Enriched payment events for downstream consumers"
      format: avro

  # Data quality checks
  quality:
    checks:
      - name: no-null-amounts
        type: not_null
        column: total_volume
        severity: error
        
      - name: positive-amounts
        type: range_check
        column: total_volume
        min: 0
        severity: error
        
      - name: freshness-check
        type: freshness
        column: date
        max_lag: PT20M
        severity: warning
        
      - name: row-count-anomaly
        type: anomaly_detection
        column: transaction_count
        threshold: 3.0    # sigma
        severity: warning

  # Access control
  accessControl:
    defaultPolicy: deny
    grants:
      - principal: group:data-analysts
        permissions: [read]
        ports: [payment-summary-daily]
        
      - principal: group:finance-team
        permissions: [read]
        ports: [payment-summary-daily]
        
      - principal: group:bi-platform
        permissions: [read]
        ports: [payment-summary-daily, payment-events-stream]
        
  # Infrastructure
  infrastructure:
    compute:
      type: flink
      config:
        parallelism: 8
        checkpoint_interval: 60s
        
    storage:
      catalog: glue
      warehouse: s3://data-lake/
      
  # Lineage
  lineage:
    upstream:
      - dp://payments/raw-events
      - dp://users/user-profiles
    downstream:
      - dp://analytics/executive-dashboard
      - dp://ml/fraud-features
EOF

# ─── 2. Data Mesh Infrastructure Controller ────────────────────────────────
cat > data_mesh_controller.py << 'PYEOF'
#!/usr/bin/env python3
"""
Data Mesh Infrastructure Controller.
Manages data product lifecycle: provisioning, monitoring, access control.
"""

import json
import yaml
import logging
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from enum import Enum

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DataProductStatus(Enum):
    DRAFT = "draft"
    PROVISIONING = "provisioning"
    ACTIVE = "active"
    DEGRADED = "degraded"
    DEPRECATED = "deprecated"

class DataQualityTier(Enum):
    GOLD = "gold"      # 99.9% accuracy, full schema enforcement
    SILVER = "silver"  # 95% accuracy, schema enforcement
    BRONZE = "bronze"  # Best effort, no guarantees

@dataclass
class DataProductHealth:
    product_name: str
    status: DataProductStatus
    freshness_minutes: float      # Actual data age in minutes
    freshness_sla_minutes: float  # SLA threshold
    row_count: int
    quality_score: float          # 0.0 - 1.0
    schema_violations: int
    last_updated: datetime = field(default_factory=datetime.utcnow)

    @property
    def is_sla_breached(self) -> bool:
        return self.freshness_minutes > self.freshness_sla_minutes

    @property
    def health_color(self) -> str:
        if self.status == DataProductStatus.ACTIVE and not self.is_sla_breached:
            if self.quality_score >= 0.99:
                return "GREEN"
            elif self.quality_score >= 0.95:
                return "YELLOW"
        return "RED"

@dataclass
class DataContract:
    """Data contract between data product producer and consumer."""
    product_name: str
    consumer: str
    output_port: str
    agreed_sla_minutes: int
    schema_version: str
    start_date: datetime = field(default_factory=datetime.utcnow)
    
    def validate_compatibility(self, current_schema: Dict) -> List[str]:
        """Check if current schema is backward compatible with contract."""
        violations = []
        # In production: compare schemas using Apache Avro or OpenAPI schema evolution rules
        return violations

class DataMeshController:
    def __init__(self):
        self.products: Dict[str, DataProductHealth] = {}
        self.contracts: List[DataContract] = []

    def register_product(self, spec: Dict):
        """Register a new data product from spec YAML."""
        name = spec["metadata"]["name"]
        sla_freshness = spec["spec"].get("outputPorts", [{}])[0].get(
            "sla", {}
        ).get("freshness", "PT60M")
        
        # Parse ISO 8601 duration (simplified)
        freshness_mins = int(sla_freshness.replace("PT", "").replace("M", ""))
        
        health = DataProductHealth(
            product_name=name,
            status=DataProductStatus.PROVISIONING,
            freshness_minutes=0,
            freshness_sla_minutes=freshness_mins,
            row_count=0,
            quality_score=1.0,
            schema_violations=0,
        )
        self.products[name] = health
        logger.info(f"Registered data product: {name}")

    def check_quality(self, product_name: str) -> Dict:
        """Run data quality checks for a product."""
        product = self.products.get(product_name)
        if not product:
            return {"error": "Product not found"}
        
        # Simulate quality checks
        import random
        random.seed(hash(product_name) + int(datetime.utcnow().timestamp() / 3600))
        
        checks = {
            "no_null_amounts": random.random() > 0.01,
            "positive_amounts": random.random() > 0.001,
            "freshness_check": product.freshness_minutes <= product.freshness_sla_minutes,
            "row_count_anomaly": random.random() > 0.05,
            "schema_compliance": random.random() > 0.001,
        }
        
        passed = sum(checks.values())
        product.quality_score = passed / len(checks)
        product.status = DataProductStatus.ACTIVE if product.quality_score >= 0.8 \
            else DataProductStatus.DEGRADED
        
        return {
            "product": product_name,
            "quality_score": product.quality_score,
            "health": product.health_color,
            "checks": checks,
            "freshness_minutes": product.freshness_minutes,
            "sla_breached": product.is_sla_breached,
        }

    def get_discovery_catalog(self) -> List[Dict]:
        """Return searchable data product catalog."""
        return [
            {
                "name": name,
                "status": health.status.value,
                "health": health.health_color,
                "quality_score": health.quality_score,
                "freshness_minutes": health.freshness_minutes,
                "last_updated": health.last_updated.isoformat(),
            }
            for name, health in self.products.items()
        ]

    def enforce_access_policy(self, product_name: str, principal: str, action: str) -> bool:
        """Enforce data mesh access control."""
        # Real impl: OPA or Ranger policy evaluation
        # For demo: simple allowlist
        allowed_readers = {
            "payment-analytics": [
                "group:data-analysts",
                "group:finance-team",
                "group:bi-platform",
            ]
        }
        
        allowed = allowed_readers.get(product_name, [])
        granted = principal in allowed or any(
            principal.startswith(a) for a in allowed
        )
        
        if not granted:
            logger.warning(f"Access DENIED: {principal} → {product_name} ({action})")
        else:
            logger.info(f"Access GRANTED: {principal} → {product_name} ({action})")
        
        return granted

# Demo
controller = DataMeshController()

# Load and register product spec
with open("data-product-spec.yaml") as f:
    spec = yaml.safe_load(f)

controller.register_product(spec)

# Simulate active product
controller.products["payment-analytics"].status = DataProductStatus.ACTIVE
controller.products["payment-analytics"].freshness_minutes = 8.5
controller.products["payment-analytics"].row_count = 1_234_567

# Quality check
print("=== Data Mesh Product Health Check ===\n")
quality = controller.check_quality("payment-analytics")
print(f"Product: {quality['product']}")
print(f"Health: {quality['health']}")
print(f"Quality Score: {quality['quality_score']:.1%}")
print(f"Freshness: {quality['freshness_minutes']:.1f}min (SLA breach: {quality['sla_breached']})")
print(f"Checks:")
for check, passed in quality["checks"].items():
    print(f"  {'✓' if passed else '✗'} {check}")

# Access control
print("\nAccess Control:")
test_cases = [
    ("group:data-analysts", "read"),
    ("group:engineering", "read"),
    ("group:finance-team", "write"),
]
for principal, action in test_cases:
    result = controller.enforce_access_policy("payment-analytics", principal, action)
PYEOF

pip install pyyaml 2>/dev/null | tail -1
python3 data_mesh_controller.py

echo "=== Step 617 Complete: Data Mesh Architecture ==="
SCRIPT
chmod +x data-mesh.sh
bash data-mesh.sh
echo "Script created: data-mesh.sh"
```

**สิ่งที่เรียนรู้:**
- Data Product spec: inputPorts/outputPorts/quality/accessControl/lineage
- GOLD/SILVER/BRONZE tiers: quality guarantees level
- Data Contract: producer-consumer agreement on SLA, schema version compatibility
- Quality checks: not_null, range_check, freshness (ISO 8601 duration), anomaly_detection (3σ)
- Discovery Catalog: searchable product registry with health status
- Access control: deny-by-default, explicit grants per output port

---

## Step 618: Advanced PostgreSQL — Partitioning + Performance Tuning

```bash
cat > postgres-advanced.sh << 'SCRIPT'
#!/bin/bash
# Advanced PostgreSQL: Partitioning, Query Optimization, pg_partman

set -euo pipefail

echo "=== Advanced PostgreSQL Engineering ==="

# ─── 1. Range Partitioning + pg_partman ───────────────────────────────────
cat > postgres-partitioning.sql << 'EOF'
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS pg_partman;
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pg_trgm;       -- For fuzzy text search
CREATE EXTENSION IF NOT EXISTS btree_gin;     -- For GIN indexes on scalar types

-- ── Partitioned payments table ────────────────────────────────────────────
CREATE TABLE payments (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    amount DECIMAL(18, 6) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    gateway_ref TEXT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Primary index on partition key
CREATE INDEX payments_created_at_idx ON payments(created_at DESC);
CREATE INDEX payments_user_id_idx ON payments(user_id, created_at DESC);
CREATE INDEX payments_status_idx ON payments(status, created_at DESC) WHERE status IN ('pending', 'processing');
-- GIN index for JSONB
CREATE INDEX payments_metadata_gin ON payments USING GIN(metadata);

-- Create partitions using pg_partman (automatic monthly partitions)
SELECT partman.create_parent(
    p_parent_table => 'public.payments',
    p_control => 'created_at',
    p_type => 'range',
    p_interval => 'monthly',        -- Monthly partitions
    p_premake => 3,                  -- Pre-create 3 future partitions
    p_retention => '24 months',     -- Keep 24 months of data
    p_retention_keep_table => false -- Drop old partitions
);

-- Configure pg_partman maintenance
UPDATE partman.part_config
SET
    retention = '24 months',
    retention_keep_table = false,
    infinite_time_partitions = true,
    automatic_maintenance = 'on'
WHERE parent_table = 'public.payments';

-- ── Automatic partition maintenance via pg_cron ───────────────────────────
-- CREATE EXTENSION IF NOT EXISTS pg_cron;
-- SELECT cron.schedule('partman-run-maintenance', '0 * * * *',
--     'SELECT partman.run_maintenance(p_analyze := false)');

-- ── Columnar storage for analytics (using citus columnar) ─────────────────
-- CREATE TABLE payment_events_columnar (...) USING columnar;
-- SELECT alter_table_set_access_method('payment_aggregates_monthly', 'columnar');

-- ── Query optimization examples ───────────────────────────────────────────

-- Slow query: full table scan
-- SELECT * FROM payments WHERE status = 'pending' AND created_at > NOW() - INTERVAL '1 hour';

-- Optimized with partial index:
CREATE INDEX payments_pending_recent ON payments(created_at DESC)
WHERE status = 'pending' AND created_at > NOW() - INTERVAL '7 days';

-- Covering index for common API query (avoids heap access)
CREATE INDEX payments_user_summary ON payments(user_id, created_at DESC)
INCLUDE (id, amount, currency, status);  -- Covers SELECT id, amount, currency, status

-- ── BRIN index for time-series (very small, sequential data) ──────────────
CREATE INDEX payments_created_brin ON payments USING BRIN(created_at)
WITH (pages_per_range = 128);

-- ── Statistics for query planner ─────────────────────────────────────────
ALTER TABLE payments ALTER COLUMN status SET STATISTICS 500;  -- More histogram buckets
ALTER TABLE payments ALTER COLUMN currency SET STATISTICS 300;
ANALYZE payments;

-- ── Materialized view for dashboard ───────────────────────────────────────
CREATE MATERIALIZED VIEW payment_daily_summary AS
SELECT
    DATE_TRUNC('day', created_at) AS date,
    currency,
    COUNT(*) AS transaction_count,
    COUNT(DISTINCT user_id) AS unique_users,
    SUM(amount) AS total_volume,
    AVG(amount) AS avg_amount,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY amount) AS p99_amount,
    COUNT(*) FILTER (WHERE status = 'failed') AS failed_count
FROM payments
WHERE created_at >= NOW() - INTERVAL '90 days'
GROUP BY DATE_TRUNC('day', created_at), currency
WITH DATA;

CREATE UNIQUE INDEX ON payment_daily_summary(date, currency);

-- Refresh concurrently (no lock!)
-- SELECT pg_catalog.pg_refresh_materialized_view_concurrently('payment_daily_summary');

-- Schedule via pg_cron: every 15 minutes
-- SELECT cron.schedule('refresh-payment-summary', '*/15 * * * *',
--     'REFRESH MATERIALIZED VIEW CONCURRENTLY payment_daily_summary');
EOF

# ─── 2. PostgreSQL Configuration Tuning ───────────────────────────────────
cat > postgresql.conf.optimized << 'EOF'
# PostgreSQL 16 Production Tuning (256GB RAM, 64 vCPUs, NVMe SSD)

# ── Memory ──────────────────────────────────────────────────────────────
shared_buffers = '64GB'              # 25% of RAM
effective_cache_size = '192GB'       # 75% of RAM (hint for planner)
work_mem = '256MB'                   # Per sort/hash operation (limit per query!)
maintenance_work_mem = '8GB'        # For VACUUM, CREATE INDEX
huge_pages = try
wal_buffers = '64MB'                 # WAL buffer size

# ── Parallelism ───────────────────────────────────────────────────────────
max_worker_processes = 64
max_parallel_workers = 32
max_parallel_workers_per_gather = 16
max_parallel_maintenance_workers = 8

# ── I/O ───────────────────────────────────────────────────────────────────
effective_io_concurrency = 256      # NVMe SSDs
random_page_cost = 1.1              # SSD (vs default 4.0 for HDD)
seq_page_cost = 1.0

# ── Checkpoints ───────────────────────────────────────────────────────────
checkpoint_completion_target = 0.9
checkpoint_timeout = '15min'
min_wal_size = '2GB'
max_wal_size = '8GB'

# ── Query Planning ────────────────────────────────────────────────────────
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on
jit_above_cost = 100000

# ── Connection ────────────────────────────────────────────────────────────
max_connections = 500               # Use PgBouncer for more
idle_in_transaction_session_timeout = '30s'
lock_timeout = '10s'
statement_timeout = '120s'

# ── Logging ───────────────────────────────────────────────────────────────
log_min_duration_statement = 1000  # Log queries >1 second
log_checkpoints = on
log_autovacuum_min_duration = 250  # Log autovacuum runs >250ms

# ── VACUUM ────────────────────────────────────────────────────────────────
autovacuum_max_workers = 10
autovacuum_naptime = '10s'
autovacuum_vacuum_scale_factor = 0.02  # 2% (default 20%)
autovacuum_analyze_scale_factor = 0.01 # 1%
autovacuum_vacuum_cost_limit = 800     # Higher = faster vacuum
EOF

# ─── 3. Query Performance Analyzer ────────────────────────────────────────
cat > pg_performance_analyzer.py << 'PYEOF'
#!/usr/bin/env python3
"""Analyze PostgreSQL query performance using pg_stat_statements."""

import json
from dataclasses import dataclass
from typing import List

@dataclass
class SlowQuery:
    query: str
    calls: int
    total_time_ms: float
    mean_time_ms: float
    rows: float
    cache_hit_ratio: float

def get_slow_queries() -> List[SlowQuery]:
    """Sample slow queries (real: query pg_stat_statements)."""
    return [
        SlowQuery(
            query="SELECT * FROM payments WHERE user_id = $1 ORDER BY created_at DESC LIMIT 20",
            calls=45230,
            total_time_ms=2_250_000,
            mean_time_ms=49.7,
            rows=20,
            cache_hit_ratio=0.72,
        ),
        SlowQuery(
            query="SELECT COUNT(*), SUM(amount) FROM payments WHERE created_at > $1",
            calls=1203,
            total_time_ms=180_450,
            mean_time_ms=150.0,
            rows=1,
            cache_hit_ratio=0.45,
        ),
    ]

def analyze(queries: List[SlowQuery]):
    print("=== PostgreSQL Slow Query Analysis ===\n")
    
    for q in sorted(queries, key=lambda x: x.total_time_ms, reverse=True):
        print(f"Query: {q.query[:80]}...")
        print(f"  Calls: {q.calls:,}")
        print(f"  Total time: {q.total_time_ms/1000:.1f}s")
        print(f"  Mean time: {q.mean_time_ms:.1f}ms")
        print(f"  Cache hit ratio: {q.cache_hit_ratio:.1%}")
        
        # Recommendations
        if q.cache_hit_ratio < 0.85:
            print("  → [WARN] Low cache hit ratio — increase shared_buffers or add index")
        if q.mean_time_ms > 100:
            print("  → [ACTION] Run EXPLAIN ANALYZE to identify missing index or bad plan")
        if "SELECT *" in q.query:
            print("  → [BEST PRACTICE] Avoid SELECT * — specify needed columns for covering index")
        print()

analyze(get_slow_queries())
PYEOF

python3 pg_performance_analyzer.py

echo "=== Step 618 Complete: Advanced PostgreSQL ==="
SCRIPT
chmod +x postgres-advanced.sh
bash postgres-advanced.sh
echo "Script created: postgres-advanced.sh"
```

**สิ่งที่เรียนรู้:**
- `pg_partman` monthly range partitioning: automatic partition creation + retention (24 months drop)
- Partial index (WHERE clause): `payments_pending_recent` เฉพาะ pending ใน 7 วัน
- Covering index (INCLUDE): หลีกเลี่ยง heap access สำหรับ common queries
- BRIN index: เหมาะกับ append-only time-series (เล็กมาก)
- `REFRESH MATERIALIZED VIEW CONCURRENTLY`: refresh โดยไม่ lock reads
- PostgreSQL tuning: `shared_buffers` 25% RAM, `random_page_cost=1.1` สำหรับ NVMe, `autovacuum_vacuum_scale_factor=0.02`

---

## สรุป Part 67

| Step | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|----------------|
| 616 | CockroachDB Multi-Region | LOCALITY GLOBAL/REGIONAL BY ROW, SURVIVE REGION FAILURE, follower reads |
| 617 | Data Mesh | Data Product spec, quality tiers, contracts, discovery catalog |
| 618 | Advanced PostgreSQL | pg_partman, partial/covering/BRIN indexes, materialized views, tuning |

**ขั้นตอนต่อไป: Part 68 — Advanced Observability, eBPF Networking, Service Mesh Internals และ Network Debugging**
