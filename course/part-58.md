# Part 58: Real-Time Analytics, Event Streaming และ Digital Twin

## Module 5: World-Class Level (ต่อ)

---

## ขั้นตอนที่ 581: Real-Time Analytics Platform

### Real-Time Analytics Architecture

```
Real-Time Analytics Pipeline:
                                          ┌──────────────┐
Events ──► Kafka ──► Flink Streaming ──►  │  ClickHouse  │ ──► Superset
                         │                │  (Real-time) │     (Dashboards)
                         │                └──────────────┘
                         │
                         ▼
                   Materialized Views      ┌──────────────┐
                   (Aggregations)     ──►  │   Grafana    │
                                           │  (Metrics)   │
                                           └──────────────┘
```

### `realtime-analytics.sh`

```bash
#!/usr/bin/env bash
# realtime-analytics.sh — Real-Time Analytics with ClickHouse + Flink
set -euo pipefail

LOG_FILE="/var/log/realtime-analytics.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Kafka Streams Real-Time Processing ────────────────────────────────────

setup_kafka_streams() {
  log "=== Setting up Kafka Streams Real-Time Processing ==="

  # Kafka Streams Application: Session Analytics
  cat <<'JAVA' > /tmp/SessionAnalytics.java
package com.example.analytics;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.streams.state.WindowStore;

import java.time.Duration;
import java.util.Properties;

public class SessionAnalytics {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "session-analytics");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-cluster:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 10 * 1024 * 1024L);
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);
        props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 8);

        StreamsBuilder builder = new StreamsBuilder();

        // ── 1. Parse raw events ────────────────────────────────────────────────
        KStream<String, UserEvent> events = builder
            .stream("user-events", Consumed.with(Serdes.String(), EventSerdes.USER_EVENT))
            .filter((key, event) -> event != null && event.getUserId() != null)
            .mapValues(EventEnricher::enrich);

        // ── 2. Session windows (30-minute inactivity gap) ─────────────────────
        SessionWindowedKStream<String, UserEvent> sessionWindows = events
            .selectKey((k, v) -> v.getUserId())
            .groupByKey(Grouped.with(Serdes.String(), EventSerdes.USER_EVENT))
            .windowedBy(SessionWindows.ofInactivityGapAndGrace(
                Duration.ofMinutes(30),
                Duration.ofMinutes(5)
            ));

        KTable<Windowed<String>, SessionMetrics> sessionMetrics = sessionWindows.aggregate(
            SessionMetrics::new,
            (key, event, agg) -> agg.add(event),
            (key, agg1, agg2) -> agg1.merge(agg2),
            Materialized.<String, SessionMetrics, SessionStore<Bytes, byte[]>>as("session-store")
                .withRetention(Duration.ofHours(2))
        );

        // ── 3. Output to ClickHouse via Kafka sink ────────────────────────────
        sessionMetrics.toStream()
            .map((key, metrics) -> KeyValue.pair(
                key.key(),
                metrics.toJson(key.window().startTime(), key.window().endTime())
            ))
            .to("session-metrics", Produced.with(Serdes.String(), Serdes.String()));

        // ── 4. Real-time revenue aggregation (1-minute tumbling windows) ───────
        KStream<String, UserEvent> purchaseEvents = events
            .filter((k, v) -> "purchase".equals(v.getEventType()));

        purchaseEvents
            .selectKey((k, v) -> v.getCountry() + ":" + v.getProductCategory())
            .groupByKey(Grouped.with(Serdes.String(), EventSerdes.USER_EVENT))
            .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(1), Duration.ofSeconds(30)))
            .aggregate(
                RevenueMetrics::new,
                (key, event, agg) -> agg.add(event.getAmount()),
                Materialized.as("revenue-1m-store")
            )
            .toStream()
            .map((key, metrics) -> KeyValue.pair(
                key.key() + ":" + key.window().startTime().toEpochMilli(),
                metrics.toJson(key.window().startTime())
            ))
            .to("revenue-metrics-1m");

        // ── 5. Anomaly detection stream ────────────────────────────────────────
        events
            .filter((k, v) -> v.getAmount() != null && v.getAmount() > 10000)  // large transactions
            .mapValues(v -> new AnomalyAlert(v, "LARGE_TRANSACTION", "HIGH"))
            .to("anomaly-alerts", Produced.with(Serdes.String(), EventSerdes.ANOMALY_ALERT));

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.setUncaughtExceptionHandler((exception) -> {
            System.err.println("Uncaught exception in Kafka Streams: " + exception.getMessage());
            return StreamsUncaughtExceptionHandler.StreamThreadExceptionResponse.REPLACE_THREAD;
        });

        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
        streams.start();
    }
}
JAVA

  # Deploy Kafka Streams app to K8s
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: session-analytics
  namespace: data-engineering
spec:
  replicas: 3
  selector:
    matchLabels:
      app: session-analytics
  template:
    metadata:
      labels:
        app: session-analytics
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      containers:
        - name: session-analytics
          image: ghcr.io/example/session-analytics:v1.0.0
          ports:
            - containerPort: 8080
              name: metrics
          env:
            - name: KAFKA_BOOTSTRAP_SERVERS
              value: kafka-cluster:9092
            - name: KAFKA_STREAMS_NUM_THREADS
              value: "8"
            - name: JVM_OPTS
              value: "-Xms2g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=50"
          resources:
            requests:
              cpu: "2"
              memory: 4Gi
            limits:
              cpu: "4"
              memory: 8Gi
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 15
EOF

  log "✓ Kafka Streams real-time processing configured"
}

# ─── 2. ClickHouse Real-Time Ingestion ────────────────────────────────────────

setup_clickhouse_realtime() {
  log "=== Setting up ClickHouse Real-Time Ingestion ==="

  # ClickHouse Kafka Engine (native Kafka consumer)
  cat <<'SQL' > /tmp/clickhouse_kafka_tables.sql
-- Kafka engine table (consumer)
CREATE TABLE IF NOT EXISTS analytics.events_kafka ON CLUSTER analytics
(
    customer_id UInt64,
    event_type  String,
    event_time  DateTime64(3),
    properties  String,  -- JSON
    amount_usd  Nullable(Float64),
    country     String,
    session_id  String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka-cluster:9092',
    kafka_topic_list = 'user-events',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow',
    kafka_num_consumers = 4,
    kafka_max_block_size = 65536,
    kafka_flush_interval_ms = 1000;

-- Target MergeTree table
CREATE TABLE IF NOT EXISTS analytics.events ON CLUSTER analytics
(
    event_id    UUID DEFAULT generateUUIDv4(),
    customer_id UInt64,
    event_type  LowCardinality(String),
    event_date  Date MATERIALIZED toDate(event_time),
    event_time  DateTime64(3, 'UTC'),
    properties  Map(String, String) MATERIALIZED JSONExtractKeysAndValues(properties_raw, 'String'),
    properties_raw String,
    amount_usd  Nullable(Float64),
    country     LowCardinality(String),
    session_id  String,
    INDEX idx_customer customer_id TYPE bloom_filter GRANULARITY 4,
    INDEX idx_session session_id TYPE bloom_filter GRANULARITY 2
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/analytics/events',
    '{replica}'
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, customer_id, event_type)
TTL event_date + INTERVAL 2 YEAR;

-- Materialized View to move data from Kafka to MergeTree
CREATE MATERIALIZED VIEW IF NOT EXISTS analytics.events_mv ON CLUSTER analytics
TO analytics.events
AS
SELECT
    customer_id,
    event_type,
    event_time,
    properties AS properties_raw,
    amount_usd,
    country,
    session_id
FROM analytics.events_kafka;

-- Real-time dashboard view (last 5 minutes)
CREATE VIEW IF NOT EXISTS analytics.realtime_dashboard AS
SELECT
    toStartOfMinute(event_time) as minute,
    event_type,
    country,
    count() as events_count,
    uniq(customer_id) as unique_users,
    uniq(session_id) as active_sessions,
    sum(amount_usd) as revenue
FROM analytics.events
WHERE event_time >= now() - INTERVAL 5 MINUTE
GROUP BY minute, event_type, country
ORDER BY minute DESC, events_count DESC;
SQL

  log "✓ ClickHouse real-time ingestion configured"
}

# ─── 3. Apache Superset Dashboard ────────────────────────────────────────────

setup_superset() {
  log "=== Setting up Apache Superset ==="

  helm repo add superset https://apache.github.io/superset
  helm upgrade --install superset superset/superset \
    --namespace data-engineering \
    --set "configOverrides.secret=SECRET_KEY='${SUPERSET_SECRET_KEY}'" \
    --set "init.adminUser.username=admin" \
    --set "init.adminUser.password=${SUPERSET_ADMIN_PASSWORD}" \
    --set "init.adminUser.email=admin@example.com" \
    --set "supersetNode.replicaCount=3" \
    --set "supersetWorker.replicaCount=4" \
    --set "redis.enabled=true" \
    --set "postgresql.enabled=true"

  # Auto-create ClickHouse connection
  cat <<'PYTHON' > /tmp/setup_superset_db.py
"""Configure Superset database connections and dashboards via API."""
import requests
import json

SUPERSET_URL = "http://superset:8088"

session = requests.Session()

# Login
login_resp = session.post(f"{SUPERSET_URL}/api/v1/security/login", json={
    "username": "admin",
    "password": "admin",
    "provider": "db",
    "refresh": True,
})
token = login_resp.json()["access_token"]
session.headers.update({"Authorization": f"Bearer {token}"})

# Add ClickHouse database
db_payload = {
    "database_name": "ClickHouse Analytics",
    "sqlalchemy_uri": "clickhousedb://analytics:password@analytics-cluster:8123/analytics",
    "extra": json.dumps({
        "metadata_params": {},
        "engine_params": {},
        "cost_estimate_enabled": True,
    }),
    "expose_in_sqllab": True,
    "allow_run_async": True,
    "allow_ctas": False,
    "allow_cvas": False,
    "force_ctas_schema": None,
}

db_resp = session.post(f"{SUPERSET_URL}/api/v1/database/", json=db_payload)
print(f"Database created: {db_resp.status_code}")

# Create Real-Time Events chart
chart_payload = {
    "slice_name": "Real-Time Events (Last 5 min)",
    "viz_type": "echarts_timeseries_bar",
    "datasource_id": 1,
    "datasource_type": "table",
    "params": json.dumps({
        "metrics": ["count"],
        "groupby": ["event_type"],
        "time_range": "Last 5 minutes",
        "time_grain_sqla": "PT1M",
        "row_limit": 1000,
    }),
}
session.post(f"{SUPERSET_URL}/api/v1/chart/", json=chart_payload)
print("Charts created")
PYTHON

  log "✓ Apache Superset configured"
}

main() {
  log "Starting Real-Time Analytics Platform..."
  setup_kafka_streams
  setup_clickhouse_realtime
  setup_superset
  log "✓ Real-Time Analytics Platform complete"
}
main "$@"
```

---

## ขั้นตอนที่ 582: Event-Driven Architecture with CQRS/Event Sourcing

### CQRS + Event Sourcing Pattern

```
CQRS / Event Sourcing Architecture:
                                                
  ┌────────────────┐     Commands      ┌──────────────────┐
  │   Client App   │ ──────────────►   │ Command Service  │
  └────────────────┘                   └────────┬─────────┘
                                                │ Events
                                                ▼
  ┌────────────────┐                   ┌──────────────────┐
  │  Read Model    │ ◄── Projections ─ │  Event Store     │
  │  (PostgreSQL)  │                   │  (EventStoreDB)  │
  └────────────────┘                   └────────┬─────────┘
                                                │ Publish
                                                ▼
  ┌────────────────┐                   ┌──────────────────┐
  │  Query Service │                   │  Event Bus       │
  │  (Read-only)   │                   │  (Kafka)         │
  └────────────────┘                   └──────────────────┘
```

### `event-sourcing-platform.sh`

```bash
#!/usr/bin/env bash
# event-sourcing-platform.sh — CQRS/Event Sourcing with EventStoreDB
set -euo pipefail

LOG_FILE="/var/log/event-sourcing.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. EventStoreDB Deployment ───────────────────────────────────────────────

setup_eventstoredb() {
  log "=== Setting up EventStoreDB ==="

  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: eventstoredb
  namespace: event-sourcing
spec:
  serviceName: eventstoredb-headless
  replicas: 3
  selector:
    matchLabels:
      app: eventstoredb
  template:
    metadata:
      labels:
        app: eventstoredb
    spec:
      containers:
        - name: eventstoredb
          image: eventstore/eventstore:23.10.0-bookworm-slim
          ports:
            - containerPort: 1113
              name: tcp
            - containerPort: 2113
              name: http
          env:
            - name: EVENTSTORE_CLUSTER_SIZE
              value: "3"
            - name: EVENTSTORE_DISCOVERY_DNS
              value: eventstoredb-headless
            - name: EVENTSTORE_GOSSIP_SEED
              value: "eventstoredb-0.eventstoredb-headless:2113,eventstoredb-1.eventstoredb-headless:2113,eventstoredb-2.eventstoredb-headless:2113"
            - name: EVENTSTORE_EXT_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: EVENTSTORE_HTTP_PORT
              value: "2113"
            - name: EVENTSTORE_ENABLE_ATOM_PUB_OVER_HTTP
              value: "true"
            - name: EVENTSTORE_MEM_DB
              value: "false"
            - name: EVENTSTORE_DB
              value: /var/lib/eventstore/data
            - name: EVENTSTORE_LOG
              value: /var/log/eventstore
          resources:
            requests:
              cpu: "1"
              memory: 2Gi
            limits:
              cpu: "4"
              memory: 8Gi
          volumeMounts:
            - name: data
              mountPath: /var/lib/eventstore
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        storageClassName: fast-ssd
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 100Gi
---
apiVersion: v1
kind: Service
metadata:
  name: eventstoredb-headless
  namespace: event-sourcing
spec:
  clusterIP: None
  selector:
    app: eventstoredb
  ports:
    - port: 2113
      name: http
    - port: 1113
      name: tcp
---
apiVersion: v1
kind: Service
metadata:
  name: eventstoredb
  namespace: event-sourcing
spec:
  selector:
    app: eventstoredb
  ports:
    - port: 2113
      name: http
    - port: 1113
      name: tcp
  type: ClusterIP
EOF

  log "✓ EventStoreDB cluster deployed"
}

# ─── 2. CQRS Command/Query Services ──────────────────────────────────────────

setup_cqrs_services() {
  log "=== Setting up CQRS Services ==="

  # Command Service (Python)
  cat <<'PYTHON' > /tmp/command_service.py
"""CQRS Command Service with EventStoreDB."""
import asyncio
import json
import uuid
from dataclasses import dataclass, field, asdict
from datetime import datetime
from typing import Any
from fastapi import FastAPI, HTTPException
from esdbclient import EventStoreDBClient, NewEvent, StreamState
import structlog

logger = structlog.get_logger()
app = FastAPI(title="Order Command Service")

client = EventStoreDBClient(uri="esdb://eventstoredb:2113?tls=false")

# ── Domain Events ─────────────────────────────────────────────────────────────

@dataclass
class OrderPlaced:
    order_id: str
    customer_id: str
    items: list[dict]
    total_amount: float
    currency: str
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat())

    def to_event(self) -> NewEvent:
        return NewEvent(
            type="OrderPlaced",
            data=json.dumps(asdict(self)).encode(),
            metadata=json.dumps({"version": "1.0", "schema": "order"}).encode(),
        )

@dataclass
class OrderConfirmed:
    order_id: str
    confirmed_by: str
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat())

    def to_event(self) -> NewEvent:
        return NewEvent(
            type="OrderConfirmed",
            data=json.dumps(asdict(self)).encode(),
        )

@dataclass
class OrderShipped:
    order_id: str
    tracking_number: str
    carrier: str
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat())

    def to_event(self) -> NewEvent:
        return NewEvent(
            type="OrderShipped",
            data=json.dumps(asdict(self)).encode(),
        )

# ── Command Handlers ──────────────────────────────────────────────────────────

class OrderCommandHandler:
    def __init__(self, es_client: EventStoreDBClient):
        self.client = es_client

    async def place_order(self, cmd: dict) -> str:
        order_id = str(uuid.uuid4())
        stream = f"order-{order_id}"

        # Validate business rules
        if not cmd.get("items"):
            raise ValueError("Order must have at least one item")
        if cmd.get("total_amount", 0) <= 0:
            raise ValueError("Order total must be positive")

        event = OrderPlaced(
            order_id=order_id,
            customer_id=cmd["customer_id"],
            items=cmd["items"],
            total_amount=cmd["total_amount"],
            currency=cmd.get("currency", "USD"),
        )

        await asyncio.get_event_loop().run_in_executor(
            None,
            lambda: self.client.append_to_stream(
                stream_name=stream,
                current_version=StreamState.NO_STREAM,
                events=[event.to_event()],
            )
        )

        logger.info("order_placed", order_id=order_id, customer_id=cmd["customer_id"])
        return order_id

    async def confirm_order(self, order_id: str, confirmed_by: str) -> None:
        stream = f"order-{order_id}"

        # Load aggregate to verify it exists and get current version
        events_list = list(self.client.read_stream(stream))
        if not events_list:
            raise ValueError(f"Order {order_id} not found")

        current_version = len(events_list) - 1

        # Check idempotency — reject if already confirmed
        event_types = [e.type for e in events_list]
        if "OrderConfirmed" in event_types:
            logger.warning("order_already_confirmed", order_id=order_id)
            return

        event = OrderConfirmed(order_id=order_id, confirmed_by=confirmed_by)

        await asyncio.get_event_loop().run_in_executor(
            None,
            lambda: self.client.append_to_stream(
                stream_name=stream,
                current_version=current_version,
                events=[event.to_event()],
            )
        )

        logger.info("order_confirmed", order_id=order_id)

# ── API Endpoints ─────────────────────────────────────────────────────────────

handler = OrderCommandHandler(client)

@app.post("/orders", status_code=201)
async def place_order(cmd: dict):
    try:
        order_id = await handler.place_order(cmd)
        return {"order_id": order_id, "status": "placed"}
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@app.post("/orders/{order_id}/confirm")
async def confirm_order(order_id: str, body: dict):
    try:
        await handler.confirm_order(order_id, body.get("confirmed_by", "system"))
        return {"order_id": order_id, "status": "confirmed"}
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "ok"}
PYTHON

  # Query Service (Read Model with PostgreSQL)
  cat <<'PYTHON' > /tmp/query_service.py
"""CQRS Query Service — Read Model Projections."""
import asyncio
import json
from fastapi import FastAPI, Depends
import asyncpg
from esdbclient import EventStoreDBClient
import structlog

logger = structlog.get_logger()
app = FastAPI(title="Order Query Service")

# ── Projection: Build Read Model from Events ──────────────────────────────────

class OrderProjection:
    def __init__(self, db_pool: asyncpg.Pool):
        self.db = db_pool

    async def handle_order_placed(self, data: dict) -> None:
        await self.db.execute("""
            INSERT INTO orders (
                order_id, customer_id, status, total_amount,
                currency, items, created_at
            ) VALUES ($1, $2, $3, $4, $5, $6, $7)
            ON CONFLICT (order_id) DO NOTHING
        """,
            data["order_id"], data["customer_id"], "placed",
            data["total_amount"], data["currency"],
            json.dumps(data["items"]), data["timestamp"]
        )

    async def handle_order_confirmed(self, data: dict) -> None:
        await self.db.execute("""
            UPDATE orders
            SET status = 'confirmed',
                confirmed_at = $2,
                confirmed_by = $3,
                updated_at = now()
            WHERE order_id = $1
        """, data["order_id"], data["timestamp"], data["confirmed_by"])

    async def handle_order_shipped(self, data: dict) -> None:
        await self.db.execute("""
            UPDATE orders
            SET status = 'shipped',
                tracking_number = $2,
                carrier = $3,
                shipped_at = $4,
                updated_at = now()
            WHERE order_id = $1
        """, data["order_id"], data["tracking_number"],
            data["carrier"], data["timestamp"])

async def start_projection_listener(db_pool: asyncpg.Pool):
    """Subscribe to $all stream and project events to read model."""
    es_client = EventStoreDBClient(uri="esdb://eventstoredb:2113?tls=false")
    projection = OrderProjection(db_pool)

    handlers = {
        "OrderPlaced": projection.handle_order_placed,
        "OrderConfirmed": projection.handle_order_confirmed,
        "OrderShipped": projection.handle_order_shipped,
    }

    # Read checkpoint
    last_position = await db_pool.fetchval(
        "SELECT last_position FROM projection_checkpoints WHERE name = 'order-read-model'"
    )

    logger.info("starting_projection", last_position=last_position)

    for event in es_client.subscribe_to_all(
        from_position=last_position,
        filter_include=list(handlers.keys()),
    ):
        handler = handlers.get(event.type)
        if handler:
            data = json.loads(event.data)
            await handler(data)

            # Update checkpoint
            await db_pool.execute("""
                INSERT INTO projection_checkpoints (name, last_position, updated_at)
                VALUES ('order-read-model', $1, now())
                ON CONFLICT (name) DO UPDATE
                SET last_position = $1, updated_at = now()
            """, event.commit_position)

            logger.debug("event_projected", event_type=event.type, position=event.commit_position)

# ── Query API ─────────────────────────────────────────────────────────────────

@app.get("/orders/{order_id}")
async def get_order(order_id: str, db=Depends(lambda: app.state.db)):
    row = await db.fetchrow("SELECT * FROM orders WHERE order_id = $1", order_id)
    if not row:
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="Order not found")
    return dict(row)

@app.get("/customers/{customer_id}/orders")
async def get_customer_orders(
    customer_id: str,
    limit: int = 20,
    offset: int = 0,
    db=Depends(lambda: app.state.db),
):
    rows = await db.fetch("""
        SELECT order_id, status, total_amount, currency, created_at, shipped_at
        FROM orders
        WHERE customer_id = $1
        ORDER BY created_at DESC
        LIMIT $2 OFFSET $3
    """, customer_id, limit, offset)
    return [dict(r) for r in rows]

@app.get("/orders/{order_id}/history")
async def get_order_event_history(order_id: str):
    """Return full event history for an order."""
    es_client = EventStoreDBClient(uri="esdb://eventstoredb:2113?tls=false")
    stream = f"order-{order_id}"
    events = []
    for e in es_client.read_stream(stream):
        events.append({
            "type": e.type,
            "data": json.loads(e.data),
            "position": e.stream_position,
        })
    return events
PYTHON

  log "✓ CQRS command/query services configured"
}

# ─── 3. Kafka Integration for Event Publishing ────────────────────────────────

setup_event_bridge() {
  log "=== Setting up EventStore → Kafka Bridge ==="

  cat <<'PYTHON' > /tmp/event_bridge.py
"""Bridge: EventStoreDB → Kafka for downstream consumers."""
import asyncio
import json
import logging
from confluent_kafka import Producer
from esdbclient import EventStoreDBClient

logger = logging.getLogger(__name__)

KAFKA_CONF = {
    "bootstrap.servers": "kafka-cluster:9092",
    "acks": "all",
    "retries": 5,
    "retry.backoff.ms": 500,
    "enable.idempotence": True,
    "compression.type": "lz4",
    "linger.ms": 10,
    "batch.size": 65536,
}

TOPIC_MAP = {
    "OrderPlaced": "orders.placed",
    "OrderConfirmed": "orders.confirmed",
    "OrderShipped": "orders.shipped",
    "OrderCancelled": "orders.cancelled",
    "PaymentProcessed": "payments.processed",
    "PaymentFailed": "payments.failed",
}

class EventBridge:
    def __init__(self):
        self.es_client = EventStoreDBClient(uri="esdb://eventstoredb:2113?tls=false")
        self.producer = Producer(KAFKA_CONF)
        self._running = False

    def _delivery_callback(self, err, msg):
        if err:
            logger.error("Kafka delivery failed", extra={"error": str(err), "topic": msg.topic()})
        else:
            logger.debug("Event delivered", extra={"topic": msg.topic(), "offset": msg.offset()})

    def run(self):
        self._running = True
        logger.info("Starting EventStore → Kafka bridge")

        for event in self.es_client.subscribe_to_all(
            filter_include=list(TOPIC_MAP.keys())
        ):
            if not self._running:
                break

            topic = TOPIC_MAP.get(event.type)
            if not topic:
                continue

            try:
                payload = {
                    "event_id": str(event.id),
                    "event_type": event.type,
                    "stream": event.stream_name,
                    "position": event.commit_position,
                    "data": json.loads(event.data),
                    "metadata": json.loads(event.metadata) if event.metadata else {},
                }

                self.producer.produce(
                    topic=topic,
                    key=event.stream_name.encode(),
                    value=json.dumps(payload).encode(),
                    callback=self._delivery_callback,
                )
                self.producer.poll(0)

            except Exception as e:
                logger.error("Failed to bridge event", extra={"error": str(e), "event_type": event.type})

        self.producer.flush()
        logger.info("EventBridge stopped")

if __name__ == "__main__":
    bridge = EventBridge()
    bridge.run()
PYTHON

  log "✓ EventStore-Kafka bridge configured"
}

main() {
  log "Starting Event Sourcing Platform..."
  setup_eventstoredb
  setup_cqrs_services
  setup_event_bridge
  log "✓ CQRS/Event Sourcing Platform complete"
}
main "$@"
```

---

## ขั้นตอนที่ 583: Digital Twin Platform

### Digital Twin Architecture

```
Digital Twin Architecture:
┌─────────────────────────────────────────────────────────────┐
│                   Physical World                             │
│  IoT Sensors │ Industrial Machines │ Infrastructure │ Humans │
└──────────────────────┬──────────────────────────────────────┘
                       │ MQTT / OPC-UA / REST
                       ▼
┌─────────────────────────────────────────────────────────────┐
│               Edge Gateway Layer                             │
│   Eclipse Mosquitto │ Azure IoT Edge │ AWS Greengrass        │
└──────────────────────┬──────────────────────────────────────┘
                       │ Kafka / AMQP
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Digital Twin Engine                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Twin State Manager (Redis + TimescaleDB)            │   │
│  │  Simulation Engine (Physics Models)                  │   │
│  │  ML Prediction Service (PyTorch)                     │   │
│  │  Alert & Anomaly Engine                              │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────────────┘
                       │ WebSocket / GraphQL
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Visualization & Control                         │
│      3D Viewer (Three.js) │ Grafana │ Custom Dashboard       │
└─────────────────────────────────────────────────────────────┘
```

### `digital-twin-platform.sh`

```bash
#!/usr/bin/env bash
# digital-twin-platform.sh — Digital Twin Platform
set -euo pipefail

LOG_FILE="/var/log/digital-twin.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. TimescaleDB for Time-Series Data ─────────────────────────────────────

setup_timescaledb() {
  log "=== Setting up TimescaleDB ==="

  helm repo add timescale https://charts.timescale.com/
  helm upgrade --install timescaledb timescale/timescaledb-single \
    --namespace digital-twin \
    --create-namespace \
    --set replicaCount=3 \
    --set patroni.postgresql.parameters.max_connections=500 \
    --set patroni.postgresql.parameters.shared_buffers=4GB \
    --set patroni.postgresql.parameters.work_mem=64MB \
    --set resources.requests.cpu=2 \
    --set resources.requests.memory=8Gi \
    --set persistentVolumes.data.size=500Gi

  # TimescaleDB Schema for Digital Twins
  cat <<'SQL' > /tmp/timescaledb_schema.sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS timescaledb_toolkit;

-- Digital Twin definitions
CREATE TABLE IF NOT EXISTS twins (
    twin_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name         TEXT NOT NULL UNIQUE,
    twin_type    TEXT NOT NULL,  -- 'machine', 'building', 'vehicle', 'process'
    physical_id  TEXT UNIQUE,
    metadata     JSONB DEFAULT '{}',
    created_at   TIMESTAMPTZ DEFAULT now(),
    updated_at   TIMESTAMPTZ DEFAULT now()
);

-- Time-series telemetry
CREATE TABLE IF NOT EXISTS telemetry (
    time         TIMESTAMPTZ NOT NULL,
    twin_id      UUID REFERENCES twins(twin_id),
    metric_name  TEXT NOT NULL,
    value        DOUBLE PRECISION,
    unit         TEXT,
    quality      SMALLINT DEFAULT 100,  -- 0-100 data quality score
    tags         JSONB DEFAULT '{}'
);

-- Convert to hypertable
SELECT create_hypertable('telemetry', 'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- Compression policy (compress chunks older than 7 days)
SELECT add_compression_policy('telemetry', INTERVAL '7 days');

-- Retention policy (keep data for 2 years)
SELECT add_retention_policy('telemetry', INTERVAL '2 years');

-- Continuous aggregate: 1-minute averages
CREATE MATERIALIZED VIEW telemetry_1min
WITH (timescaledb.continuous, timescaledb.materialized_only=false) AS
SELECT
    time_bucket('1 minute', time) AS bucket,
    twin_id,
    metric_name,
    avg(value) AS avg_value,
    min(value) AS min_value,
    max(value) AS max_value,
    count(*) AS sample_count,
    stddev(value) AS std_dev
FROM telemetry
GROUP BY bucket, twin_id, metric_name
WITH NO DATA;

SELECT add_continuous_aggregate_policy('telemetry_1min',
    start_offset => INTERVAL '1 hour',
    end_offset   => INTERVAL '1 minute',
    schedule_interval => INTERVAL '30 seconds'
);

-- Continuous aggregate: 1-hour aggregates
CREATE MATERIALIZED VIEW telemetry_1hour
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', bucket) AS hour,
    twin_id,
    metric_name,
    avg(avg_value) AS avg_value,
    min(min_value) AS min_value,
    max(max_value) AS max_value,
    sum(sample_count) AS total_samples
FROM telemetry_1min
GROUP BY hour, twin_id, metric_name
WITH NO DATA;

-- Twin state table (latest values)
CREATE TABLE IF NOT EXISTS twin_state (
    twin_id      UUID REFERENCES twins(twin_id),
    metric_name  TEXT,
    value        DOUBLE PRECISION,
    unit         TEXT,
    updated_at   TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (twin_id, metric_name)
);

-- Alert configurations
CREATE TABLE IF NOT EXISTS alert_rules (
    rule_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    twin_id      UUID REFERENCES twins(twin_id),
    metric_name  TEXT NOT NULL,
    condition    TEXT NOT NULL,  -- 'gt', 'lt', 'eq', 'between'
    threshold    DOUBLE PRECISION NOT NULL,
    threshold2   DOUBLE PRECISION,  -- for 'between'
    severity     TEXT DEFAULT 'warning',  -- 'info', 'warning', 'critical'
    enabled      BOOLEAN DEFAULT true
);

-- Anomalies
CREATE TABLE IF NOT EXISTS anomalies (
    id           BIGSERIAL PRIMARY KEY,
    time         TIMESTAMPTZ NOT NULL,
    twin_id      UUID REFERENCES twins(twin_id),
    metric_name  TEXT,
    value        DOUBLE PRECISION,
    expected     DOUBLE PRECISION,
    deviation    DOUBLE PRECISION,
    severity     TEXT,
    acknowledged BOOLEAN DEFAULT false
);

SELECT create_hypertable('anomalies', 'time', if_not_exists => TRUE);
SQL

  log "✓ TimescaleDB configured"
}

# ─── 2. Digital Twin Service ──────────────────────────────────────────────────

create_twin_service() {
  log "=== Creating Digital Twin Service ==="

  cat <<'PYTHON' > /tmp/digital_twin_service.py
"""Digital Twin Engine — state management, simulation, anomaly detection."""
import asyncio
import json
import uuid
import numpy as np
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import Any, Optional
import asyncpg
import redis.asyncio as redis
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, HTTPException
from pydantic import BaseModel
import structlog

logger = structlog.get_logger()
app = FastAPI(title="Digital Twin Service")

# ── Twin Models ───────────────────────────────────────────────────────────────

@dataclass
class TwinState:
    twin_id: str
    name: str
    twin_type: str
    metrics: dict[str, float] = field(default_factory=dict)
    metadata: dict = field(default_factory=dict)
    last_update: datetime = field(default_factory=datetime.utcnow)
    health_score: float = 100.0
    anomalies: list = field(default_factory=list)

class TelemetryPoint(BaseModel):
    twin_id: str
    metrics: dict[str, float]
    timestamp: Optional[datetime] = None
    quality: int = 100

# ── Twin Manager ──────────────────────────────────────────────────────────────

class DigitalTwinManager:
    def __init__(self, db_pool: asyncpg.Pool, redis_client: redis.Redis):
        self.db = db_pool
        self.redis = redis_client
        self._websocket_clients: dict[str, set[WebSocket]] = {}
        self._anomaly_models: dict[str, Any] = {}

    async def ingest_telemetry(self, point: TelemetryPoint) -> dict:
        """Ingest sensor data, update twin state, run anomaly detection."""
        timestamp = point.timestamp or datetime.utcnow()

        # Update Redis state (sub-millisecond reads)
        state_key = f"twin:{point.twin_id}:state"
        pipeline = self.redis.pipeline()
        for metric, value in point.metrics.items():
            pipeline.hset(state_key, metric, value)
        pipeline.hset(state_key, "_updated_at", timestamp.isoformat())
        pipeline.expire(state_key, 86400)  # TTL: 24h
        await pipeline.execute()

        # Write to TimescaleDB
        async with self.db.acquire() as conn:
            await conn.executemany("""
                INSERT INTO telemetry (time, twin_id, metric_name, value, quality)
                VALUES ($1, $2, $3, $4, $5)
            """, [
                (timestamp, point.twin_id, name, value, point.quality)
                for name, value in point.metrics.items()
            ])

            # Update twin_state (upsert)
            await conn.executemany("""
                INSERT INTO twin_state (twin_id, metric_name, value, updated_at)
                VALUES ($1, $2, $3, $4)
                ON CONFLICT (twin_id, metric_name) DO UPDATE
                SET value = $3, updated_at = $4
            """, [
                (point.twin_id, name, value, timestamp)
                for name, value in point.metrics.items()
            ])

        # Run anomaly detection
        anomalies = await self._detect_anomalies(point.twin_id, point.metrics)

        # Compute health score
        health_score = self._compute_health_score(point.metrics, anomalies)

        # Broadcast to WebSocket subscribers
        await self._broadcast_update(point.twin_id, {
            "twin_id": point.twin_id,
            "timestamp": timestamp.isoformat(),
            "metrics": point.metrics,
            "health_score": health_score,
            "anomalies": anomalies,
        })

        return {
            "status": "ok",
            "health_score": health_score,
            "anomalies": len(anomalies),
        }

    async def _detect_anomalies(self, twin_id: str, metrics: dict[str, float]) -> list:
        """Statistical anomaly detection using Z-score on rolling window."""
        anomalies = []

        async with self.db.acquire() as conn:
            for metric_name, current_value in metrics.items():
                # Get rolling statistics (last 24 hours)
                stats = await conn.fetchrow("""
                    SELECT avg(value) as mean, stddev(value) as std
                    FROM telemetry_1min
                    WHERE twin_id = $1
                      AND metric_name = $2
                      AND bucket >= now() - INTERVAL '24 hours'
                """, twin_id, metric_name)

                if not stats or stats['std'] is None or stats['std'] < 1e-9:
                    continue

                z_score = abs((current_value - stats['mean']) / stats['std'])

                if z_score > 3.5:  # > 3.5 sigma
                    severity = "critical" if z_score > 5 else "warning"
                    anomalies.append({
                        "metric": metric_name,
                        "value": current_value,
                        "expected": stats['mean'],
                        "z_score": z_score,
                        "severity": severity,
                    })

                    # Persist anomaly
                    await conn.execute("""
                        INSERT INTO anomalies (time, twin_id, metric_name, value, expected, deviation, severity)
                        VALUES (now(), $1, $2, $3, $4, $5, $6)
                    """, twin_id, metric_name, current_value, stats['mean'],
                        abs(current_value - stats['mean']), severity)

        return anomalies

    def _compute_health_score(self, metrics: dict, anomalies: list) -> float:
        """Compute 0-100 health score based on anomalies."""
        if not anomalies:
            return 100.0
        score = 100.0
        for anomaly in anomalies:
            if anomaly["severity"] == "critical":
                score -= 30.0
            elif anomaly["severity"] == "warning":
                score -= 10.0
        return max(0.0, score)

    async def simulate_future_state(self, twin_id: str, horizon_hours: int = 24) -> dict:
        """Simple physics-based prediction model."""
        async with self.db.acquire() as conn:
            rows = await conn.fetch("""
                SELECT metric_name,
                       avg(avg_value) as mean,
                       stddev(avg_value) as std,
                       regr_slope(avg_value, extract(epoch from bucket)) as trend
                FROM telemetry_1hour
                WHERE twin_id = $1
                  AND hour >= now() - INTERVAL '7 days'
                GROUP BY metric_name
            """, twin_id)

        predictions = {}
        now = datetime.utcnow()

        for row in rows:
            metric = row['metric_name']
            trend = row['trend'] or 0
            mean = row['mean'] or 0
            std = row['std'] or 0

            # Linear trend extrapolation with noise
            future_times = [now + timedelta(hours=h) for h in range(1, horizon_hours + 1)]
            future_values = [
                mean + trend * (3600 * h) + np.random.normal(0, std * 0.1)
                for h in range(1, horizon_hours + 1)
            ]

            predictions[metric] = {
                "timestamps": [t.isoformat() for t in future_times],
                "values": future_values,
                "confidence": 0.80,
            }

        return predictions

    async def subscribe_websocket(self, twin_id: str, ws: WebSocket):
        await ws.accept()
        if twin_id not in self._websocket_clients:
            self._websocket_clients[twin_id] = set()
        self._websocket_clients[twin_id].add(ws)
        logger.info("websocket_connected", twin_id=twin_id)

    async def unsubscribe_websocket(self, twin_id: str, ws: WebSocket):
        if twin_id in self._websocket_clients:
            self._websocket_clients[twin_id].discard(ws)

    async def _broadcast_update(self, twin_id: str, data: dict):
        clients = self._websocket_clients.get(twin_id, set())
        if not clients:
            return
        message = json.dumps(data)
        disconnected = set()
        for ws in clients:
            try:
                await ws.send_text(message)
            except WebSocketDisconnect:
                disconnected.add(ws)
        for ws in disconnected:
            clients.discard(ws)

# ── API Routes ────────────────────────────────────────────────────────────────

@app.post("/twins/{twin_id}/telemetry")
async def ingest_telemetry(twin_id: str, payload: dict):
    point = TelemetryPoint(twin_id=twin_id, metrics=payload.get("metrics", {}))
    result = await app.state.twin_manager.ingest_telemetry(point)
    return result

@app.get("/twins/{twin_id}/state")
async def get_twin_state(twin_id: str):
    state = await app.state.redis.hgetall(f"twin:{twin_id}:state")
    if not state:
        raise HTTPException(status_code=404, detail="Twin not found")
    return {k.decode(): v.decode() for k, v in state.items()}

@app.get("/twins/{twin_id}/simulate")
async def simulate_future(twin_id: str, horizon_hours: int = 24):
    predictions = await app.state.twin_manager.simulate_future_state(twin_id, horizon_hours)
    return {"twin_id": twin_id, "horizon_hours": horizon_hours, "predictions": predictions}

@app.websocket("/twins/{twin_id}/stream")
async def twin_stream(twin_id: str, ws: WebSocket):
    await app.state.twin_manager.subscribe_websocket(twin_id, ws)
    try:
        while True:
            await ws.receive_text()  # keepalive
    except WebSocketDisconnect:
        await app.state.twin_manager.unsubscribe_websocket(twin_id, ws)
PYTHON

  log "✓ Digital Twin service created"
}

# ─── 3. MQTT IoT Gateway ─────────────────────────────────────────────────────

setup_mqtt_gateway() {
  log "=== Setting up MQTT IoT Gateway ==="

  # Eclipse Mosquitto MQTT Broker
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mosquitto
  namespace: digital-twin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mosquitto
  template:
    metadata:
      labels:
        app: mosquitto
    spec:
      containers:
        - name: mosquitto
          image: eclipse-mosquitto:2.0.18
          ports:
            - containerPort: 1883  # MQTT
            - containerPort: 9001  # WebSocket
          volumeMounts:
            - name: config
              mountPath: /mosquitto/config
            - name: data
              mountPath: /mosquitto/data
      volumes:
        - name: config
          configMap:
            name: mosquitto-config
        - name: data
          emptyDir: {}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mosquitto-config
  namespace: digital-twin
data:
  mosquitto.conf: |
    listener 1883
    protocol mqtt
    listener 9001
    protocol websockets

    # Auth
    allow_anonymous false
    password_file /mosquitto/config/passwords

    # Persistence
    persistence true
    persistence_location /mosquitto/data/

    # Logging
    log_type all
    log_dest stdout

    # Message limits
    max_inflight_messages 100
    max_queued_messages 1000
    message_size_limit 1048576
EOF

  # MQTT → Digital Twin Bridge
  cat <<'PYTHON' > /tmp/mqtt_bridge.py
"""MQTT → Digital Twin Engine bridge."""
import asyncio
import json
import logging
import paho.mqtt.client as mqtt
import httpx

logger = logging.getLogger(__name__)

TWIN_SERVICE_URL = "http://digital-twin-service:8080"
MQTT_BROKER = "mosquitto"
MQTT_PORT = 1883

# Topic pattern: sensors/{twin_id}/{metric_name}
TOPIC_PATTERN = "sensors/+/telemetry"

async def forward_to_twin_engine(twin_id: str, metrics: dict):
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f"{TWIN_SERVICE_URL}/twins/{twin_id}/telemetry",
            json={"metrics": metrics},
            timeout=5.0,
        )
        if resp.status_code != 200:
            logger.error(f"Twin engine error: {resp.status_code}")

def on_message(client, userdata, msg):
    try:
        topic_parts = msg.topic.split("/")
        if len(topic_parts) < 3:
            return

        twin_id = topic_parts[1]
        payload = json.loads(msg.payload.decode())

        # Run async forwarding in event loop
        loop = asyncio.get_event_loop()
        loop.run_until_complete(forward_to_twin_engine(twin_id, payload.get("metrics", {})))

    except Exception as e:
        logger.error(f"MQTT message processing failed: {e}")

def start_bridge():
    client = mqtt.Client()
    client.username_pw_set("bridge", "bridge-password")
    client.on_message = on_message
    client.connect(MQTT_BROKER, MQTT_PORT, keepalive=60)
    client.subscribe(TOPIC_PATTERN, qos=1)
    logger.info(f"MQTT bridge started, subscribed to {TOPIC_PATTERN}")
    client.loop_forever()

if __name__ == "__main__":
    start_bridge()
PYTHON

  log "✓ MQTT IoT gateway configured"
}

main() {
  log "Starting Digital Twin Platform..."
  setup_timescaledb
  create_twin_service
  setup_mqtt_gateway
  log ""
  log "╔══════════════════════════════════════════════════════╗"
  log "║         Digital Twin Platform Summary                 ║"
  log "╠══════════════════════════════════════════════════════╣"
  log "║  Time-Series DB: TimescaleDB (HA, compression)       ║"
  log "║  Twin Engine: FastAPI + Z-score anomaly detection    ║"
  log "║  State Store: Redis (sub-ms reads)                   ║"
  log "║  Simulation: Linear trend extrapolation              ║"
  log "║  IoT Gateway: Eclipse Mosquitto (MQTT)               ║"
  log "║  Real-time: WebSocket streaming                      ║"
  log "╚══════════════════════════════════════════════════════╝"
}
main "$@"
```

---

## ขั้นตอนที่ 584: Green Computing และ Sustainable Infrastructure

### Green Computing Framework

```
Sustainable Infrastructure:
┌─────────────────────────────────────────────────────────┐
│               Carbon-Aware Workload Scheduler            │
│                                                          │
│  ┌─────────────────┐    ┌─────────────────────────────┐ │
│  │ Carbon Intensity │    │    Workload Scheduler       │ │
│  │   API (Watt)     │ ─► │  - Shift to low-carbon time│ │
│  │  (Grid CO2/kWh) │    │  - Prefer renewable regions │ │
│  └─────────────────┘    └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│              Energy Efficiency Optimizer                  │
│                                                          │
│  CPU Power Capping │ GPU Throttling │ Node Sleep          │
│  Right-sizing VPA  │ KEDA Scale-to-Zero │ Spot Instances │
└─────────────────────────────────────────────────────────┘
```

### `green-computing.sh`

```bash
#!/usr/bin/env bash
# green-computing.sh — Carbon-Aware and Energy-Efficient Kubernetes
set -euo pipefail

LOG_FILE="/var/log/green-computing.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Kepler — Kubernetes Energy Profiler ───────────────────────────────────

setup_kepler() {
  log "=== Setting up Kepler Energy Profiler ==="

  helm repo add kepler https://sustainable-computing-io.github.io/kepler-helm-chart
  helm upgrade --install kepler kepler/kepler \
    --namespace kepler-system \
    --create-namespace \
    --set serviceMonitor.enabled=true \
    --set serviceMonitor.labels.release=kube-prometheus \
    --set extraEnvVars[0].name=ENABLE_EBPF_CGROUPID \
    --set extraEnvVars[0].value="true"

  # Grafana dashboard for energy consumption
  cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: kepler-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  kepler.json: |
    {
      "title": "Kubernetes Energy Consumption",
      "panels": [
        {
          "title": "Total Power Consumption (W)",
          "type": "stat",
          "targets": [{
            "expr": "sum(irate(kepler_container_joules_total[5m]))"
          }]
        },
        {
          "title": "Power by Namespace",
          "type": "timeseries",
          "targets": [{
            "expr": "sum by (container_namespace) (irate(kepler_container_joules_total[5m]))"
          }]
        },
        {
          "title": "Carbon Emissions (gCO2eq/h)",
          "type": "stat",
          "targets": [{
            "expr": "sum(irate(kepler_container_joules_total[5m])) * on() group_left() carbon_intensity_grams_per_kwh / 1000 * 3600"
          }]
        },
        {
          "title": "CPU Energy Efficiency (GFLOPS/W)",
          "type": "timeseries",
          "targets": [{
            "expr": "sum(rate(container_cpu_usage_seconds_total[5m])) / sum(irate(kepler_container_core_joules_total[5m]))"
          }]
        }
      ]
    }
EOF

  log "✓ Kepler energy profiler deployed"
}

# ─── 2. Carbon-Aware Scheduler ────────────────────────────────────────────────

setup_carbon_aware_scheduler() {
  log "=== Setting up Carbon-Aware Workload Scheduler ==="

  cat <<'PYTHON' > /tmp/carbon_scheduler.py
"""Carbon-Aware Kubernetes Workload Scheduler."""
import asyncio
import json
import logging
import httpx
from datetime import datetime, timezone
from kubernetes import client as k8s_client, config as k8s_config
from kubernetes.client import V1CronJob

logger = logging.getLogger(__name__)

WATTTIME_API = "https://api2.watttime.org/v3"
CARBON_THRESHOLD_HIGH = 250  # gCO2eq/kWh — defer batch jobs above this

class CarbonAwareScheduler:
    def __init__(self):
        k8s_config.load_incluster_config()
        self.k8s_batch = k8s_client.BatchV1Api()
        self.k8s_apps = k8s_client.AppsV1Api()
        self._watttime_token = None

    async def get_carbon_intensity(self, region: str = "CAISO_NORTH") -> float:
        """Get real-time grid carbon intensity from WattTime API."""
        if not self._watttime_token:
            await self._login_watttime()

        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{WATTTIME_API}/signal-index",
                params={"region": region, "signal_type": "co2_moer"},
                headers={"Authorization": f"Bearer {self._watttime_token}"},
            )
            data = resp.json()
            return data.get("data", [{}])[0].get("value", 200)

    async def _login_watttime(self):
        import os
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{WATTTIME_API}/login",
                auth=(os.environ["WATTTIME_USER"], os.environ["WATTTIME_PASSWORD"]),
            )
            self._watttime_token = resp.json()["token"]

    async def schedule_carbon_aware(self, job_name: str, namespace: str = "default") -> dict:
        """Decide whether to run a batch job based on carbon intensity."""
        carbon_intensity = await self.get_carbon_intensity()

        logger.info(f"Carbon intensity: {carbon_intensity:.1f} gCO2/kWh")

        if carbon_intensity < CARBON_THRESHOLD_HIGH:
            logger.info(f"Low carbon ({carbon_intensity:.0f}): Running job {job_name}")
            await self._resume_cronjob(job_name, namespace)
            return {"action": "run", "carbon_intensity": carbon_intensity}
        else:
            logger.info(f"High carbon ({carbon_intensity:.0f}): Deferring job {job_name}")
            await self._suspend_cronjob(job_name, namespace)
            return {"action": "defer", "carbon_intensity": carbon_intensity}

    async def _suspend_cronjob(self, name: str, namespace: str):
        suspend = True
        self.k8s_batch.patch_namespaced_cron_job(
            name=name, namespace=namespace,
            body={"spec": {"suspend": suspend}},
        )

    async def _resume_cronjob(self, name: str, namespace: str):
        self.k8s_batch.patch_namespaced_cron_job(
            name=name, namespace=namespace,
            body={"spec": {"suspend": False}},
        )

    async def scale_for_carbon(self, namespace: str = "data-engineering"):
        """Scale down non-critical workloads during high-carbon periods."""
        carbon_intensity = await self.get_carbon_intensity()
        hour = datetime.now(timezone.utc).hour

        # Scale down during peak carbon hours (typically 3pm-9pm local)
        is_peak_carbon = 15 <= hour <= 21

        deployments = self.k8s_apps.list_namespaced_deployment(
            namespace=namespace,
            label_selector="carbon-aware=true",
        )

        for deploy in deployments.items:
            current_replicas = deploy.spec.replicas
            annotations = deploy.metadata.annotations or {}
            min_replicas = int(annotations.get("carbon-aware/min-replicas", "1"))
            normal_replicas = int(annotations.get("carbon-aware/normal-replicas", str(current_replicas)))

            if is_peak_carbon and carbon_intensity > CARBON_THRESHOLD_HIGH:
                target = min_replicas
                action = "scale_down"
            else:
                target = normal_replicas
                action = "scale_up"

            if current_replicas != target:
                self.k8s_apps.patch_namespaced_deployment(
                    name=deploy.metadata.name,
                    namespace=namespace,
                    body={"spec": {"replicas": target}},
                )
                logger.info(
                    f"{action}: {deploy.metadata.name} "
                    f"{current_replicas}→{target} replicas "
                    f"(carbon: {carbon_intensity:.0f} gCO2/kWh)"
                )

        return {"carbon_intensity": carbon_intensity, "action_taken": action}

if __name__ == "__main__":
    scheduler = CarbonAwareScheduler()
    asyncio.run(scheduler.scale_for_carbon())
PYTHON

  # Kubernetes CronJob: Carbon-aware scheduling check every 15 minutes
  cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: carbon-scheduler
  namespace: platform-system
spec:
  schedule: "*/15 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: carbon-scheduler
          containers:
            - name: scheduler
              image: ghcr.io/example/carbon-scheduler:latest
              command: [python, carbon_scheduler.py]
              env:
                - name: WATTTIME_USER
                  valueFrom:
                    secretKeyRef:
                      name: watttime-credentials
                      key: username
                - name: WATTTIME_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: watttime-credentials
                      key: password
              resources:
                requests:
                  cpu: 100m
                  memory: 128Mi
          restartPolicy: OnFailure
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: carbon-scheduler
  namespace: platform-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: carbon-scheduler
rules:
  - apiGroups: [apps]
    resources: [deployments]
    verbs: [get, list, patch, update]
  - apiGroups: [batch]
    resources: [cronjobs]
    verbs: [get, list, patch, update]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: carbon-scheduler
subjects:
  - kind: ServiceAccount
    name: carbon-scheduler
    namespace: platform-system
roleRef:
  kind: ClusterRole
  name: carbon-scheduler
  apiGroup: rbac.authorization.k8s.io
EOF

  log "✓ Carbon-aware scheduler configured"
}

# ─── 3. Power Efficiency Policies ────────────────────────────────────────────

setup_power_efficiency() {
  log "=== Setting up Power Efficiency Policies ==="

  # Node power capping via Intel RAPL (Runtime Average Power Limiting)
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: power-manager
  namespace: kepler-system
spec:
  selector:
    matchLabels:
      app: power-manager
  template:
    metadata:
      labels:
        app: power-manager
    spec:
      hostPID: true
      hostNetwork: true
      tolerations:
        - operator: Exists
      containers:
        - name: power-manager
          image: intel/power-manager:v1.3.0
          securityContext:
            privileged: true
          volumeMounts:
            - name: sys
              mountPath: /sys
            - name: dev
              mountPath: /dev
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
      volumes:
        - name: sys
          hostPath:
            path: /sys
        - name: dev
          hostPath:
            path: /dev
EOF

  # VPA for automatic right-sizing (energy efficiency)
  cat <<'EOF' | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: global-vpa-recommender
  namespace: kube-system
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-recommender
  updatePolicy:
    updateMode: "Off"  # Recommendation only
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 10m
          memory: 10Mi
        maxAllowed:
          cpu: "8"
          memory: 16Gi
        controlledResources:
          - cpu
          - memory
EOF

  # Cluster autoscaler with scale-down optimized for energy
  cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-config
  namespace: kube-system
data:
  cluster-autoscaler.yaml: |
    scale-down-enabled: true
    scale-down-delay-after-add: 10m
    scale-down-unneeded-time: 10m
    scale-down-utilization-threshold: 0.5
    scale-down-gpu-utilization-threshold: 0.5
    skip-nodes-with-local-storage: false
    skip-nodes-with-system-pods: true
    max-node-provision-time: 15m
    # Prefer spot instances (cheaper + drives efficient usage)
    expander: priority
EOF

  log "✓ Power efficiency policies configured"
}

# ─── Main ──────────────────────────────────────────────────────────────────────

main() {
  log "Starting Green Computing setup..."
  setup_kepler
  setup_carbon_aware_scheduler
  setup_power_efficiency

  log ""
  log "╔══════════════════════════════════════════════════════╗"
  log "║        Green Computing Summary                        ║"
  log "╠══════════════════════════════════════════════════════╣"
  log "║  Energy Profiling: Kepler (eBPF, RAPL)               ║"
  log "║  Carbon API:       WattTime real-time CO2 intensity  ║"
  log "║  Scheduler:        Carbon-aware batch deferral       ║"
  log "║  Right-sizing:     VPA automatic recommendations     ║"
  log "║  Scale-down:       Cluster Autoscaler (spot-first)   ║"
  log "║  Power Capping:    Intel RAPL via DaemonSet          ║"
  log "╚══════════════════════════════════════════════════════╝"
}

main "$@"
```

---

## สรุป Part 58

| ขั้นตอน | หัวข้อ | เทคโนโลยีหลัก |
|---------|--------|---------------|
| 581 | Real-Time Analytics | Kafka Streams, ClickHouse Kafka Engine, Superset |
| 582 | Event Sourcing / CQRS | EventStoreDB, CQRS Pattern, Kafka Bridge |
| 583 | Digital Twin | TimescaleDB, WebSocket, MQTT, Anomaly Detection |
| 584 | Green Computing | Kepler, WattTime API, Carbon-Aware Scheduler, VPA |

### แนวคิดสำคัญ

1. **Kafka Streams Session Windows** — ใช้ inactivity gap แทน fixed window เพื่อ capture user sessions แบบ natural
2. **ClickHouse Kafka Engine** — consumer built-in สามารถ ingest โดยตรงจาก Kafka ไม่ต้องใช้ connector แยก
3. **Event Sourcing** — Append-only events เป็น source of truth, Read Model สร้างจาก projections
4. **Digital Twin** — Z-score > 3.5σ เป็น threshold anomaly, TimescaleDB continuous aggregates ลด query latency
5. **Carbon-Aware Scheduling** — Suspend batch jobs เมื่อ grid carbon intensity สูง, reschedule อัตโนมัติ

---

ขั้นตอนต่อไป: **Part 59** — Blockchain Infrastructure, Service Mesh Advanced Patterns และ Platform Security
