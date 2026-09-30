# Part 64: Real-Time Data Streaming, Event-Driven Architecture 2.0 และ Stream Processing at Scale

## ภาพรวม
Part นี้ครอบคลุม advanced streaming architectures ด้วย Kafka, Apache Flink, Pulsar และ event-driven patterns ระดับ World-Class

---

## Step 605: Apache Kafka — Advanced Patterns + Exactly-Once Semantics

```bash
cat > kafka-advanced.sh << 'SCRIPT'
#!/bin/bash
# Advanced Kafka: Transactions, EOS, Schema Registry, Tiered Storage

set -euo pipefail

echo "=== Apache Kafka Advanced Patterns ==="

# ─── 1. Kafka Cluster with KRaft (ZooKeeper-free) ─────────────────────────
helm repo add bitnami https://charts.bitnami.com/bitnami

helm upgrade --install kafka bitnami/kafka \
  --namespace kafka \
  --create-namespace \
  --values - << 'EOF'
kraft:
  enabled: true    # KRaft mode (no ZooKeeper)

controller:
  replicaCount: 3
  persistence:
    size: 200Gi
    storageClass: gp3-io2

broker:
  replicaCount: 6
  heapOpts: "-Xmx8G -Xms8G"
  persistence:
    size: 2Ti
    storageClass: gp3-io2
  config: |
    num.network.threads=8
    num.io.threads=16
    socket.send.buffer.bytes=102400
    socket.receive.buffer.bytes=102400
    socket.request.max.bytes=104857600
    num.partitions=12
    default.replication.factor=3
    min.insync.replicas=2
    log.retention.hours=168
    log.segment.bytes=1073741824
    log.retention.check.interval.ms=300000
    # Tiered storage (S3)
    remote.log.storage.system.enable=true
    remote.log.manager.task.interval.ms=30000

externalAccess:
  enabled: true
  service:
    type: LoadBalancer

metrics:
  kafka:
    enabled: true
  jmx:
    enabled: true
EOF

# ─── 2. Schema Registry ────────────────────────────────────────────────────
helm upgrade --install schema-registry bitnami/schema-registry \
  --namespace kafka \
  --set kafka.bootstrapServers="kafka:9092" \
  --set replicaCount=3

# Register Avro schemas
cat > payment-event-schema.json << 'EOF'
{
  "type": "record",
  "name": "PaymentEvent",
  "namespace": "com.example.payments",
  "fields": [
    {"name": "event_id", "type": "string"},
    {"name": "event_type", "type": {
      "type": "enum",
      "name": "EventType",
      "symbols": ["PAYMENT_INITIATED", "PAYMENT_AUTHORIZED", "PAYMENT_COMPLETED", "PAYMENT_FAILED", "PAYMENT_REFUNDED"]
    }},
    {"name": "payment_id", "type": "string"},
    {"name": "user_id", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string"},
    {"name": "timestamp", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "metadata", "type": {"type": "map", "values": "string"}, "default": {}}
  ]
}
EOF

curl -X POST http://schema-registry:8081/subjects/payment-events-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d "{\"schema\": $(cat payment-event-schema.json | jq -Rs .)}" 2>/dev/null || true

# ─── 3. Exactly-Once Producer (Python + confluent-kafka) ───────────────────
cat > exactly_once_producer.py << 'PYEOF'
#!/usr/bin/env python3
"""
Exactly-Once Semantics (EOS) Kafka Producer.
Idempotent producer + transactions to ensure each message delivered exactly once.
"""

import json
import uuid
import time
import logging
from datetime import datetime
from typing import Dict, Any, Optional
from dataclasses import dataclass

from confluent_kafka import Producer, KafkaException
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer
from confluent_kafka.serialization import SerializationContext, MessageField

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

KAFKA_CONFIG = {
    "bootstrap.servers": "kafka:9092",
    # Idempotent producer (prevents duplicates from retries)
    "enable.idempotence": True,
    # Transactional ID (enables transactions)
    "transactional.id": f"payment-producer-{uuid.uuid4()}",
    # EOS settings
    "acks": "all",
    "retries": 2147483647,    # Max retries
    "max.in.flight.requests.per.connection": 5,
    # Performance
    "linger.ms": 5,
    "batch.size": 131072,       # 128KB batches
    "compression.type": "snappy",
}

SCHEMA_REGISTRY_CONFIG = {
    "url": "http://schema-registry:8081",
}

@dataclass
class PaymentEvent:
    event_id: str
    event_type: str
    payment_id: str
    user_id: str
    amount: float
    currency: str
    timestamp: int
    metadata: Dict[str, str]

class ExactlyOncePaymentProducer:
    def __init__(self):
        self.producer = Producer(KAFKA_CONFIG)
        self.producer.init_transactions()
        
        schema_registry = SchemaRegistryClient(SCHEMA_REGISTRY_CONFIG)
        
        with open("payment-event-schema.json") as f:
            schema_str = f.read()
        
        self.serializer = AvroSerializer(
            schema_registry,
            schema_str,
            lambda event, ctx: {
                "event_id": event.event_id,
                "event_type": event.event_type,
                "payment_id": event.payment_id,
                "user_id": event.user_id,
                "amount": event.amount,
                "currency": event.currency,
                "timestamp": event.timestamp,
                "metadata": event.metadata,
            }
        )

    def publish_payment_saga(
        self,
        payment_id: str,
        user_id: str,
        amount: float,
        currency: str = "USD"
    ) -> bool:
        """
        Publish payment saga events atomically.
        All events in one transaction — either all succeed or none.
        """
        try:
            self.producer.begin_transaction()
            
            base_time = int(datetime.utcnow().timestamp() * 1000)
            
            # Event 1: Payment Initiated
            initiated = PaymentEvent(
                event_id=str(uuid.uuid4()),
                event_type="PAYMENT_INITIATED",
                payment_id=payment_id,
                user_id=user_id,
                amount=amount,
                currency=currency,
                timestamp=base_time,
                metadata={"source": "api-gateway", "version": "2.0"},
            )
            
            # Event 2: Payment Authorized
            authorized = PaymentEvent(
                event_id=str(uuid.uuid4()),
                event_type="PAYMENT_AUTHORIZED",
                payment_id=payment_id,
                user_id=user_id,
                amount=amount,
                currency=currency,
                timestamp=base_time + 100,
                metadata={"auth_code": f"AUTH-{uuid.uuid4().hex[:8].upper()}"},
            )
            
            ctx = SerializationContext("payment-events", MessageField.VALUE)
            
            # Produce both events atomically
            for event in [initiated, authorized]:
                self.producer.produce(
                    topic="payment-events",
                    key=payment_id.encode(),
                    value=self.serializer(event, ctx),
                    on_delivery=self._delivery_callback,
                )
            
            # Also update the read model topic atomically
            self.producer.produce(
                topic="payment-state",
                key=payment_id.encode(),
                value=json.dumps({
                    "payment_id": payment_id,
                    "status": "authorized",
                    "amount": amount,
                    "updated_at": base_time + 100,
                }).encode(),
            )
            
            self.producer.commit_transaction()
            logger.info(f"Transaction committed for payment {payment_id}")
            return True
            
        except KafkaException as e:
            logger.error(f"Transaction failed: {e}")
            self.producer.abort_transaction()
            return False

    def _delivery_callback(self, err, msg):
        if err:
            logger.error(f"Delivery failed: {err}")
        else:
            logger.debug(
                f"Delivered to {msg.topic()} "
                f"[{msg.partition()}] @ offset {msg.offset()}"
            )

    def close(self):
        self.producer.flush(timeout=30)

# Demo (without actual Kafka connection)
print("=== Exactly-Once Producer (EOS) Demo ===")
print("Configuration:")
print(f"  enable.idempotence: True (prevents duplicate messages from retries)")
print(f"  transactional.id: payment-producer-xxx (enables ACID transactions)")
print(f"  acks: all (requires all ISR replicas to acknowledge)")
print(f"  max.in.flight: 5 (allows pipelining while maintaining order)")
print()
print("Transaction pattern: begin → produce events → commit (or abort)")
print("Payment saga: INITIATED + AUTHORIZED events committed atomically")
print("Read model update: payment-state topic updated in same transaction")
PYEOF

python3 exactly_once_producer.py

# ─── 4. Kafka Streams: Complex Event Processing ────────────────────────────
cat > fraud-detection-streams.java << 'JAVAEOF'
// Kafka Streams: Real-time fraud detection with complex event processing
package com.example.fraud;

import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.streams.state.*;
import org.apache.kafka.common.serialization.Serdes;
import java.time.Duration;
import java.util.Properties;

public class FraudDetectionTopology {

    public static Topology buildTopology() {
        StreamsBuilder builder = new StreamsBuilder();

        // ── Input: payment events stream ─────────────────────────────────
        KStream<String, PaymentEvent> payments = builder
            .stream("payment-events",
                Consumed.with(Serdes.String(), PaymentEventSerde.instance()));

        // ── Pattern 1: Velocity Check (>5 payments in 1 minute) ──────────
        KTable<Windowed<String>, Long> velocityCount = payments
            .groupByKey()
            .windowedBy(SlidingWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(1)))
            .count(Materialized.as("velocity-store"));

        KStream<String, FraudAlert> velocityAlerts = velocityCount
            .toStream()
            .filter((k, v) -> v > 5)
            .map((k, v) -> KeyValue.pair(
                k.key(),
                FraudAlert.of(k.key(), "VELOCITY", "More than 5 payments in 1 minute: " + v)
            ));

        // ── Pattern 2: Amount Anomaly (>3σ from 30-day mean) ─────────────
        // Compute rolling stats per user
        KTable<String, RollingStats> userStats = payments
            .groupByKey()
            .aggregate(
                RollingStats::new,
                (userId, event, stats) -> stats.update(event.getAmount()),
                Materialized.<String, RollingStats, KeyValueStore<Bytes, byte[]>>as("user-stats")
                    .withValueSerde(RollingStatsSerde.instance())
            );

        KStream<String, FraudAlert> amountAlerts = payments
            .join(userStats,
                (event, stats) -> {
                    if (stats.count() > 30 && stats.isAnomaly(event.getAmount(), 3.0)) {
                        return FraudAlert.of(
                            event.getUserId(),
                            "AMOUNT_ANOMALY",
                            String.format("Amount %.2f is %.1fσ from mean %.2f",
                                event.getAmount(),
                                stats.zScore(event.getAmount()),
                                stats.getMean())
                        );
                    }
                    return null;
                },
                Joined.with(Serdes.String(), PaymentEventSerde.instance(), RollingStatsSerde.instance())
            )
            .filter((k, v) -> v != null);

        // ── Pattern 3: Geo-impossible travel (>900km/hour) ───────────────
        KStream<String, FraudAlert> geoAlerts = payments
            .groupByKey()
            .aggregate(
                LastLocation::new,
                (userId, event, last) -> {
                    if (last.hasLocation()) {
                        double speed = last.computeSpeed(event.getLocation(), event.getTimestamp());
                        if (speed > 900) {  // km/h (faster than plane)
                            last.setAlert(FraudAlert.of(
                                userId,
                                "GEO_IMPOSSIBLE",
                                String.format("Travel speed %.0f km/h is impossible", speed)
                            ));
                        }
                    }
                    last.update(event.getLocation(), event.getTimestamp());
                    return last;
                },
                Materialized.as("geo-store")
            )
            .toStream()
            .filter((k, v) -> v.hasAlert())
            .map((k, v) -> KeyValue.pair(k, v.getAlert()));

        // ── Merge all fraud alerts ────────────────────────────────────────
        KStream<String, FraudAlert> allAlerts = velocityAlerts
            .merge(amountAlerts)
            .merge(geoAlerts);

        // ── Deduplicate alerts (same user, same type, 5-min window) ───────
        allAlerts
            .groupByKey()
            .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(5)))
            .reduce((a, b) -> a.getMerged(b))
            .toStream()
            .map((k, v) -> KeyValue.pair(k.key(), v))
            .to("fraud-alerts", Produced.with(Serdes.String(), FraudAlertSerde.instance()));

        return builder.build();
    }

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "fraud-detection");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG,
            StreamsConfig.EXACTLY_ONCE_V2);   // EOS for streams
        props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 4);
        props.put(StreamsConfig.REPLICATION_FACTOR_CONFIG, 3);
        props.put(StreamsConfig.STATE_DIR_CONFIG, "/var/kafka-streams-state");

        KafkaStreams streams = new KafkaStreams(buildTopology(), props);
        
        // Clean shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
        
        streams.setUncaughtExceptionHandler((exception) -> 
            StreamsUncaughtExceptionHandler.StreamThreadExceptionResponse.REPLACE_THREAD
        );
        
        streams.start();
    }
}
JAVAEOF

echo "Kafka Streams topology defined (Java)"

echo "=== Step 605 Complete: Advanced Kafka with EOS ==="
SCRIPT
chmod +x kafka-advanced.sh
echo "Script created: kafka-advanced.sh"
```

**สิ่งที่เรียนรู้:**
- KRaft mode (ZooKeeper-free) Kafka cluster 6 brokers, tiered storage to S3
- Schema Registry + Avro serialization สำหรับ schema evolution
- Exactly-Once Semantics: `enable.idempotence=true` + `transactional.id` + `begin_transaction()`/`commit_transaction()`
- Kafka Streams: sliding window velocity check (>5 TXN/min), rolling-stats Z-score anomaly (>3σ), geo-impossible travel
- Session Windows deduplication (5-min inactivity gap)
- `EXACTLY_ONCE_V2` processing guarantee in Streams

---

## Step 606: Apache Flink — Stateful Stream Processing at Scale

```bash
cat > flink-advanced.sh << 'SCRIPT'
#!/bin/bash
# Apache Flink Advanced: CEP, Table API, Checkpointing

set -euo pipefail

echo "=== Apache Flink Advanced Stream Processing ==="

# ─── 1. Flink Cluster on Kubernetes ───────────────────────────────────────
kubectl apply -f - << 'EOF'
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: fraud-detection-flink
  namespace: flink
spec:
  image: flink:1.19-scala_2.12-java17
  flinkVersion: v1_19
  flinkConfiguration:
    # High availability (ZooKeeper or K8s native)
    high-availability: kubernetes
    high-availability.storageDir: s3a://flink-state/ha
    # Checkpointing (exactly-once)
    execution.checkpointing.mode: EXACTLY_ONCE
    execution.checkpointing.interval: "30s"
    execution.checkpointing.timeout: "5m"
    execution.checkpointing.min-pause: "10s"
    execution.checkpointing.max-concurrent-checkpoints: "1"
    # State backend: RocksDB for large state
    state.backend: rocksdb
    state.backend.rocksdb.memory.managed: "true"
    state.backend.rocksdb.memory.fixed-per-slot: "256mb"
    state.backend.incremental: "true"
    state.checkpoints.dir: s3a://flink-state/checkpoints
    state.savepoints.dir: s3a://flink-state/savepoints
    # S3 access
    fs.s3a.endpoint: https://s3.amazonaws.com
    fs.s3a.aws.credentials.provider: com.amazonaws.auth.WebIdentityTokenCredentialsProvider
    # Parallelism
    parallelism.default: "16"
    taskmanager.numberOfTaskSlots: "4"
    # Web UI
    rest.port: "8081"
    
  serviceAccount: flink
  
  jobManager:
    resource:
      memory: "4096m"
      cpu: 2
    replicas: 2    # HA: 2 JM, only 1 active
    
  taskManager:
    resource:
      memory: "8192m"
      cpu: 4
    replicas: 10
EOF

# ─── 2. Flink Table API + SQL for real-time analytics ─────────────────────
cat > flink-table-analytics.py << 'PYEOF'
#!/usr/bin/env python3
"""
Flink Table API + SQL: Real-time analytics pipeline.
Using PyFlink (pyflink) for Python-based stream processing.
"""

# pip install apache-flink

from pyflink.datastream import StreamExecutionEnvironment, CheckpointingMode
from pyflink.table import (
    StreamTableEnvironment, EnvironmentSettings,
    Schema, DataTypes
)
from pyflink.table.expressions import col, lit, call
from pyflink.table.window import Tumble, Slide, Session

def create_fraud_analytics_pipeline():
    # ── Environment setup ─────────────────────────────────────────────────
    env = StreamExecutionEnvironment.get_execution_environment()
    env.enable_checkpointing(30_000)  # 30 seconds
    env.get_checkpoint_config().set_checkpointing_mode(CheckpointingMode.EXACTLY_ONCE)
    env.get_checkpoint_config().set_min_pause_between_checkpoints(10_000)
    env.get_checkpoint_config().set_checkpoint_timeout(300_000)
    env.set_parallelism(16)
    
    settings = EnvironmentSettings.new_instance().in_streaming_mode().build()
    tenv = StreamTableEnvironment.create(env, environment_settings=settings)
    
    # ── Source: Kafka payment events ──────────────────────────────────────
    tenv.execute_sql("""
        CREATE TABLE payment_events (
            event_id STRING,
            event_type STRING,
            payment_id STRING,
            user_id STRING,
            amount DOUBLE,
            currency STRING,
            country STRING,
            device_id STRING,
            event_time TIMESTAMP(3),
            proc_time AS PROCTIME(),
            WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
        ) WITH (
            'connector' = 'kafka',
            'topic' = 'payment-events',
            'properties.bootstrap.servers' = 'kafka:9092',
            'properties.group.id' = 'flink-analytics',
            'scan.startup.mode' = 'earliest-offset',
            'value.format' = 'avro-confluent',
            'value.avro-confluent.schema-registry.url' = 'http://schema-registry:8081'
        )
    """)
    
    # ── Sink: Fraud alerts to Kafka ───────────────────────────────────────
    tenv.execute_sql("""
        CREATE TABLE fraud_alerts (
            user_id STRING,
            alert_type STRING,
            severity STRING,
            detail STRING,
            window_start TIMESTAMP(3),
            window_end TIMESTAMP(3),
            created_at TIMESTAMP(3)
        ) WITH (
            'connector' = 'kafka',
            'topic' = 'fraud-alerts',
            'properties.bootstrap.servers' = 'kafka:9092',
            'key.format' = 'raw',
            'key.fields' = 'user_id',
            'value.format' = 'json',
            'sink.partitioner' = 'fixed'
        )
    """)
    
    # ── Sink: Aggregates to ClickHouse ────────────────────────────────────
    tenv.execute_sql("""
        CREATE TABLE payment_aggregates (
            window_start TIMESTAMP(3),
            window_end TIMESTAMP(3),
            currency STRING,
            country STRING,
            total_amount DOUBLE,
            transaction_count BIGINT,
            unique_users BIGINT,
            avg_amount DOUBLE,
            p99_amount DOUBLE,
            failure_rate DOUBLE,
            PRIMARY KEY (window_start, currency, country) NOT ENFORCED
        ) WITH (
            'connector' = 'jdbc',
            'url' = 'jdbc:clickhouse://clickhouse:8123/analytics',
            'table-name' = 'payment_aggregates',
            'driver' = 'com.clickhouse.jdbc.ClickHouseDriver'
        )
    """)
    
    # ── Analytics Query 1: 1-minute tumbling window aggregation ──────────
    tenv.execute_sql("""
        INSERT INTO payment_aggregates
        SELECT
            TUMBLE_START(event_time, INTERVAL '1' MINUTE) AS window_start,
            TUMBLE_END(event_time, INTERVAL '1' MINUTE) AS window_end,
            currency,
            country,
            SUM(amount) AS total_amount,
            COUNT(*) AS transaction_count,
            COUNT(DISTINCT user_id) AS unique_users,
            AVG(amount) AS avg_amount,
            PERCENTILE_DISC(0.99) WITHIN GROUP (ORDER BY amount) AS p99_amount,
            CAST(SUM(CASE WHEN event_type = 'PAYMENT_FAILED' THEN 1 ELSE 0 END) AS DOUBLE)
                / COUNT(*) AS failure_rate
        FROM payment_events
        WHERE event_type IN ('PAYMENT_COMPLETED', 'PAYMENT_FAILED')
        GROUP BY
            TUMBLE(event_time, INTERVAL '1' MINUTE),
            currency,
            country
    """)
    
    # ── Analytics Query 2: Velocity fraud detection ───────────────────────
    tenv.execute_sql("""
        INSERT INTO fraud_alerts
        SELECT
            user_id,
            'VELOCITY' AS alert_type,
            CASE WHEN payment_count > 20 THEN 'CRITICAL'
                 WHEN payment_count > 10 THEN 'HIGH'
                 ELSE 'MEDIUM' END AS severity,
            CONCAT('User made ', CAST(payment_count AS STRING), ' payments in 5 min') AS detail,
            window_start,
            window_end,
            CURRENT_TIMESTAMP AS created_at
        FROM (
            SELECT
                user_id,
                COUNT(*) AS payment_count,
                TUMBLE_START(event_time, INTERVAL '5' MINUTE) AS window_start,
                TUMBLE_END(event_time, INTERVAL '5' MINUTE) AS window_end
            FROM payment_events
            GROUP BY
                user_id,
                TUMBLE(event_time, INTERVAL '5' MINUTE)
        )
        WHERE payment_count > 5
    """)
    
    # ── Analytics Query 3: Sliding window for trend detection ─────────────
    tenv.execute_sql("""
        CREATE VIEW hourly_trend AS
        SELECT
            currency,
            HOP_START(event_time, INTERVAL '5' MINUTE, INTERVAL '1' HOUR) AS window_start,
            HOP_END(event_time, INTERVAL '5' MINUTE, INTERVAL '1' HOUR) AS window_end,
            SUM(amount) AS total_amount,
            COUNT(*) AS txn_count,
            AVG(amount) AS avg_amount
        FROM payment_events
        WHERE event_type = 'PAYMENT_COMPLETED'
        GROUP BY
            currency,
            HOP(event_time, INTERVAL '5' MINUTE, INTERVAL '1' HOUR)
    """)
    
    print("Flink Table API pipeline configured")
    print("Checkpointing: EXACTLY_ONCE, 30s interval, RocksDB state backend")
    print("Queries: 1-min tumbling aggregation, 5-min velocity check, 1-hour sliding trend")

create_fraud_analytics_pipeline()
PYEOF

echo "Flink Table API pipeline created (Python)"

# ─── 3. Complex Event Processing (CEP) with Flink ─────────────────────────
cat > flink-cep.java << 'JAVAEOF'
// Flink CEP: Detect multi-step fraud patterns across events
package com.example.flink.cep;

import org.apache.flink.cep.*;
import org.apache.flink.cep.pattern.*;
import org.apache.flink.cep.pattern.conditions.*;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import java.util.List;
import java.util.Map;

public class FraudCEPTopology {
    
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        DataStream<PaymentEvent> payments = env.addSource(
            new KafkaPaymentSource("payment-events")
        ).assignTimestampsAndWatermarks(
            WatermarkStrategy
                .<PaymentEvent>forBoundedOutOfOrderness(Duration.ofSeconds(5))
                .withTimestampAssigner((e, ts) -> e.getTimestamp())
        );
        
        // ── Pattern: Account Takeover (login from new device + password change + large transfer) ──
        Pattern<PaymentEvent, ?> accountTakeoverPattern = Pattern
            .<PaymentEvent>begin("new-device-login")
                .where(new SimpleCondition<PaymentEvent>() {
                    @Override
                    public boolean filter(PaymentEvent e) {
                        return "LOGIN".equals(e.getEventType()) && e.isNewDevice();
                    }
                })
            .next("password-change")
                .where(new SimpleCondition<PaymentEvent>() {
                    @Override
                    public boolean filter(PaymentEvent e) {
                        return "PASSWORD_CHANGE".equals(e.getEventType());
                    }
                })
            .next("large-transfer")
                .where(new SimpleCondition<PaymentEvent>() {
                    @Override
                    public boolean filter(PaymentEvent e) {
                        return "PAYMENT_INITIATED".equals(e.getEventType()) 
                            && e.getAmount() > 10000;
                    }
                })
            .within(Time.minutes(10));  // All 3 events within 10 minutes
        
        // ── Pattern: Card Testing (multiple small amounts, then large) ───
        Pattern<PaymentEvent, ?> cardTestingPattern = Pattern
            .<PaymentEvent>begin("small-test")
                .where(e -> e.getAmount() < 1.0 && "PAYMENT_COMPLETED".equals(e.getEventType()))
                .times(3, 10)         // 3 to 10 small transactions
            .followedBy("large-charge")
                .where(e -> e.getAmount() > 500 && "PAYMENT_INITIATED".equals(e.getEventType()))
            .within(Time.minutes(30));
        
        // ── Apply patterns ────────────────────────────────────────────────
        PatternStream<PaymentEvent> accountTakeoverStream = CEP.pattern(
            payments.keyBy(PaymentEvent::getUserId),
            accountTakeoverPattern
        );
        
        PatternStream<PaymentEvent> cardTestingStream = CEP.pattern(
            payments.keyBy(PaymentEvent::getCardId),
            cardTestingPattern
        );
        
        // ── Extract matches and emit alerts ───────────────────────────────
        DataStream<FraudAlert> accountTakeoverAlerts = accountTakeoverStream.select(
            (PatternSelectFunction<PaymentEvent, FraudAlert>) matchedEvents -> {
                PaymentEvent login = matchedEvents.get("new-device-login").get(0);
                PaymentEvent transfer = matchedEvents.get("large-transfer").get(0);
                return FraudAlert.builder()
                    .userId(login.getUserId())
                    .type("ACCOUNT_TAKEOVER")
                    .severity("CRITICAL")
                    .detail(String.format(
                        "New device login → password change → $%.2f transfer in 10min",
                        transfer.getAmount()))
                    .build();
            }
        );
        
        DataStream<FraudAlert> cardTestingAlerts = cardTestingStream.select(
            matchedEvents -> {
                List<PaymentEvent> tests = matchedEvents.get("small-test");
                PaymentEvent charge = matchedEvents.get("large-charge").get(0);
                return FraudAlert.builder()
                    .cardId(charge.getCardId())
                    .type("CARD_TESTING")
                    .severity("HIGH")
                    .detail(String.format(
                        "%d small test charges followed by $%.2f charge",
                        tests.size(), charge.getAmount()))
                    .build();
            }
        );
        
        // Merge and sink to Kafka
        accountTakeoverAlerts
            .union(cardTestingAlerts)
            .addSink(new KafkaFraudAlertSink("fraud-alerts-cep"));
        
        env.execute("Fraud Detection CEP");
    }
}
JAVAEOF

echo "Flink CEP topology defined (Java)"
echo "=== Step 606 Complete: Apache Flink Advanced ==="
SCRIPT
chmod +x flink-advanced.sh
echo "Script created: flink-advanced.sh"
```

**สิ่งที่เรียนรู้:**
- Flink on Kubernetes: HA mode (K8s native), RocksDB state backend (incremental checkpoints), S3 checkpoints/savepoints
- PyFlink Table API: DDL-based Kafka source/sink, JDBC sink to ClickHouse
- Tumbling Window (1-min), Sliding/HOP Window (5-min slide, 1-hr window), Session Window
- Flink CEP: account takeover pattern (login + password change + large transfer in 10 min), card testing pattern
- `PatternSelectFunction` extraction of matched events across time

---

## Step 607: Apache Pulsar — Multi-Tenancy + Geo-Replication

```bash
cat > pulsar-platform.sh << 'SCRIPT'
#!/bin/bash
# Apache Pulsar: Multi-Tenancy, Geo-Replication, Functions

set -euo pipefail

echo "=== Apache Pulsar Platform ==="

helm repo add apache https://pulsar.apache.org/charts

# ─── 1. Pulsar Cluster with Geo-Replication ────────────────────────────────
helm upgrade --install pulsar apache/pulsar \
  --namespace pulsar \
  --create-namespace \
  --values - << 'EOF'
global:
  clusterName: ap-southeast-1
  persistence: true

bookkeeper:
  replicaCount: 3
  configData:
    # Tiered storage to S3
    managedLedgerOffloadDriver: aws-s3
    s3ManagedLedgerOffloadBucket: pulsar-offload
    s3ManagedLedgerOffloadRegion: ap-southeast-1
    managedLedgerOffloadDeletionLagMs: "14400000"  # 4 hours
    managedLedgerOffloadAutoTriggerSizeThresholdBytes: "1073741824"  # 1 GB

broker:
  replicaCount: 3
  configData:
    # Geo-replication
    brokerDeduplicationEnabled: "true"
    # Transaction support
    transactionCoordinatorEnabled: "true"
    # Functions
    functionsWorkerEnabled: "true"

proxy:
  replicaCount: 2
  service:
    type: LoadBalancer

zookeeper:
  replicaCount: 3

autorecovery:
  enabled: true

monitoring:
  prometheus: true
  grafana: true
EOF

# ─── 2. Multi-tenant Configuration ────────────────────────────────────────
setup_multitenancy() {
    local PULSAR_ADMIN="kubectl exec -n pulsar deployment/pulsar-broker -- bin/pulsar-admin"
    
    # Tenant: payments
    $PULSAR_ADMIN tenants create payments \
        --admin-roles payments-admin \
        --allowed-clusters ap-southeast-1,us-east-1
    
    # Tenant: analytics
    $PULSAR_ADMIN tenants create analytics \
        --admin-roles analytics-admin \
        --allowed-clusters ap-southeast-1
    
    # Namespaces
    $PULSAR_ADMIN namespaces create payments/events \
        --replication-clusters ap-southeast-1,us-east-1 \
        --message-ttl 604800 \  # 7 days
        --retention-size 100G \
        --retention-time 30d
    
    $PULSAR_ADMIN namespaces create payments/commands \
        --replication-clusters ap-southeast-1 \
        --message-ttl 3600     # 1 hour
    
    # Schema compatibility mode
    $PULSAR_ADMIN namespaces set-schema-compatibility-strategy \
        --compatibility FULL_TRANSITIVE \
        payments/events
    
    # Resource quotas
    $PULSAR_ADMIN resourcequotas set \
        --msgRateIn 10000 \
        --msgRateOut 50000 \
        --bandwidthIn 100M \
        --bandwidthOut 500M \
        payments/events
    
    echo "Multi-tenancy configured"
}

# ─── 3. Pulsar Functions ───────────────────────────────────────────────────
cat > payment-enrichment-function.py << 'PYEOF'
#!/usr/bin/env python3
"""
Pulsar Function: Enrich payment events with user profile and risk score.
Runs serverless inside Pulsar cluster.
"""

import json
import pulsar
from pulsar import Function

class PaymentEnrichmentFunction(Function):
    def __init__(self):
        self.user_cache = {}  # In-memory cache (state backed by BookKeeper)
    
    def process(self, input: bytes, context: pulsar.Context) -> bytes:
        try:
            event = json.loads(input)
            user_id = event.get("user_id")
            
            # Get user profile (cached)
            user_profile = self._get_user_profile(user_id, context)
            
            # Calculate risk score
            risk_score = self._calculate_risk(event, user_profile)
            
            # Enrich the event
            enriched = {
                **event,
                "user_tier": user_profile.get("tier", "standard"),
                "user_country": user_profile.get("country", "unknown"),
                "account_age_days": user_profile.get("account_age_days", 0),
                "risk_score": risk_score,
                "risk_level": self._risk_level(risk_score),
                "enriched_at": context.get_partition_key(),
            }
            
            # Route to different topics based on risk
            if risk_score > 0.8:
                context.publish(
                    "payments/events/high-risk-payments",
                    json.dumps(enriched).encode()
                )
            
            # Track metrics
            context.record_metric("processed_events", 1)
            context.record_metric("high_risk_events", 1 if risk_score > 0.8 else 0)
            
            return json.dumps(enriched).encode()
            
        except Exception as e:
            context.get_logger().error(f"Enrichment failed: {e}")
            return input  # Pass through unchanged on error

    def _get_user_profile(self, user_id: str, context) -> dict:
        # Use Pulsar state store (backed by BookKeeper)
        cached = context.get_state(f"user:{user_id}")
        if cached:
            return json.loads(cached)
        
        # Fetch from API (fallback)
        # In production: call internal user service
        profile = {
            "tier": "premium",
            "country": "TH",
            "account_age_days": 730,
        }
        
        # Cache for 5 minutes
        context.put_state(f"user:{user_id}", json.dumps(profile))
        return profile

    def _calculate_risk(self, event: dict, profile: dict) -> float:
        risk = 0.0
        
        # Amount-based risk
        amount = event.get("amount", 0)
        if amount > 10000:
            risk += 0.3
        elif amount > 1000:
            risk += 0.1
        
        # New account risk
        age = profile.get("account_age_days", 0)
        if age < 7:
            risk += 0.4
        elif age < 30:
            risk += 0.2
        
        # Cross-border risk
        if event.get("currency") != "THB" and profile.get("country") == "TH":
            risk += 0.2
        
        return min(risk, 1.0)

    def _risk_level(self, score: float) -> str:
        if score > 0.8:
            return "HIGH"
        elif score > 0.5:
            return "MEDIUM"
        return "LOW"
PYEOF

# Deploy function
kubectl exec -n pulsar deployment/pulsar-broker -- \
    bin/pulsar-admin functions create \
    --name payment-enrichment \
    --py /tmp/payment-enrichment-function.py \
    --classname payment_enrichment_function.PaymentEnrichmentFunction \
    --inputs payments/events/raw-payments \
    --output payments/events/enriched-payments \
    --log-topic payments/events/function-logs \
    --parallelism 4 \
    --cpu 0.5 \
    --ram 512M 2>/dev/null || echo "Function deployment (Pulsar required)"

python3 payment-enrichment-function.py 2>/dev/null || \
    echo "Pulsar Function code ready (Pulsar runtime required)"

echo "=== Step 607 Complete: Apache Pulsar Platform ==="
SCRIPT
chmod +x pulsar-platform.sh
echo "Script created: pulsar-platform.sh"
```

**สิ่งที่เรียนรู้:**
- Pulsar multi-tenancy: tenant/namespace/topic hierarchy, admin roles, resource quotas
- Tiered storage: offload old segments to S3 (1GB threshold, 4-hour lag)
- Geo-replication: `--replication-clusters ap-southeast-1,us-east-1` ระดับ namespace
- Pulsar Functions: serverless compute inside cluster, state store (BookKeeper-backed), metric recording
- Schema compatibility: `FULL_TRANSITIVE` (backward + forward compatible)
- Risk scoring: multi-factor (amount + account age + cross-border)

---

## Step 608: Event-Driven Microservices — Saga Pattern + Outbox Pattern

```bash
cat > event-driven-patterns.sh << 'SCRIPT'
#!/bin/bash
# Advanced Event-Driven: Transactional Outbox + Saga Orchestration

set -euo pipefail

echo "=== Event-Driven Architecture: Outbox + Saga ==="

# ─── 1. Transactional Outbox Pattern ──────────────────────────────────────
cat > outbox_pattern.py << 'PYEOF'
#!/usr/bin/env python3
"""
Transactional Outbox Pattern.
Ensures events are published atomically with database changes.
"""

import json
import uuid
import asyncio
import asyncpg
import logging
from datetime import datetime
from dataclasses import dataclass, field
from typing import List, Optional, Dict

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Outbox table DDL:
OUTBOX_SCHEMA = """
CREATE TABLE IF NOT EXISTS outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'published', 'failed')),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    published_at TIMESTAMPTZ,
    retry_count INT DEFAULT 0,
    last_error TEXT
);
CREATE INDEX IF NOT EXISTS outbox_pending_idx ON outbox(status, created_at) WHERE status = 'pending';
"""

@dataclass
class OutboxEvent:
    aggregate_type: str
    aggregate_id: str
    event_type: str
    payload: Dict
    id: str = field(default_factory=lambda: str(uuid.uuid4()))

class OrderService:
    def __init__(self, pool: asyncpg.Pool):
        self.pool = pool

    async def create_order(
        self,
        customer_id: str,
        items: List[Dict],
        total_amount: float
    ) -> str:
        order_id = str(uuid.uuid4())
        
        async with self.pool.acquire() as conn:
            async with conn.transaction():
                # 1. Insert order (business operation)
                await conn.execute("""
                    INSERT INTO orders (id, customer_id, status, total_amount, created_at)
                    VALUES ($1, $2, 'pending', $3, NOW())
                """, order_id, customer_id, total_amount)
                
                # 2. Insert order items
                for item in items:
                    await conn.execute("""
                        INSERT INTO order_items (order_id, product_id, quantity, price)
                        VALUES ($1, $2, $3, $4)
                    """, order_id, item["product_id"], item["quantity"], item["price"])
                
                # 3. Write to outbox IN SAME TRANSACTION (atomic!)
                event = OutboxEvent(
                    aggregate_type="Order",
                    aggregate_id=order_id,
                    event_type="OrderCreated",
                    payload={
                        "order_id": order_id,
                        "customer_id": customer_id,
                        "items": items,
                        "total_amount": total_amount,
                        "timestamp": datetime.utcnow().isoformat(),
                    }
                )
                await conn.execute("""
                    INSERT INTO outbox (id, aggregate_type, aggregate_id, event_type, payload)
                    VALUES ($1, $2, $3, $4, $5)
                """,
                    event.id,
                    event.aggregate_type,
                    event.aggregate_id,
                    event.event_type,
                    json.dumps(event.payload)
                )
                
                # Both INSERT succeed or both rollback — no phantom events!
        
        logger.info(f"Order {order_id} created with outbox event")
        return order_id


class OutboxRelay:
    """
    Relay service: polls outbox table and publishes to Kafka.
    Can also use Debezium CDC (preferred for production).
    """
    
    def __init__(self, pool: asyncpg.Pool, kafka_producer):
        self.pool = pool
        self.producer = kafka_producer
        self.batch_size = 100
        self.poll_interval = 0.5  # 500ms

    async def run(self):
        logger.info("OutboxRelay started")
        while True:
            try:
                published = await self._publish_batch()
                if published == 0:
                    await asyncio.sleep(self.poll_interval)
            except Exception as e:
                logger.error(f"Relay error: {e}")
                await asyncio.sleep(5)

    async def _publish_batch(self) -> int:
        async with self.pool.acquire() as conn:
            # Lock rows for processing (SKIP LOCKED prevents multiple relay instances)
            rows = await conn.fetch("""
                SELECT id, aggregate_type, aggregate_id, event_type, payload
                FROM outbox
                WHERE status = 'pending'
                ORDER BY created_at ASC
                LIMIT $1
                FOR UPDATE SKIP LOCKED
            """, self.batch_size)
            
            if not rows:
                return 0
            
            published_ids = []
            failed_ids = []
            
            for row in rows:
                try:
                    topic = f"{row['aggregate_type'].lower()}-events"
                    self.producer.produce(
                        topic=topic,
                        key=row["aggregate_id"].encode(),
                        value=json.dumps({
                            "event_id": str(row["id"]),
                            "event_type": row["event_type"],
                            **row["payload"],
                        }).encode(),
                    )
                    published_ids.append(row["id"])
                except Exception as e:
                    logger.error(f"Failed to publish {row['id']}: {e}")
                    failed_ids.append((row["id"], str(e)))
            
            self.producer.flush(timeout=10)
            
            # Mark as published
            if published_ids:
                await conn.execute("""
                    UPDATE outbox
                    SET status = 'published', published_at = NOW()
                    WHERE id = ANY($1)
                """, published_ids)
            
            # Mark failures (with retry limit)
            for event_id, error in failed_ids:
                await conn.execute("""
                    UPDATE outbox
                    SET retry_count = retry_count + 1,
                        last_error = $2,
                        status = CASE WHEN retry_count >= 5 THEN 'failed' ELSE 'pending' END
                    WHERE id = $1
                """, event_id, error)
            
            return len(published_ids)

print("=== Transactional Outbox Pattern ===")
print("Schema: outbox table with status, retry_count, last_error")
print("Atomic guarantee: order INSERT + outbox INSERT in same transaction")
print("Relay: SKIP LOCKED for concurrent relay instances")
print("Retry: up to 5 attempts before marking as 'failed'")
PYEOF

python3 outbox_pattern.py

# ─── 2. Saga Orchestration Pattern ─────────────────────────────────────────
cat > saga_orchestrator.py << 'PYEOF'
#!/usr/bin/env python3
"""
Order Saga Orchestrator.
Coordinates multi-service transaction with compensating actions.
"""

import asyncio
import uuid
import logging
from datetime import datetime
from enum import Enum, auto
from dataclasses import dataclass, field
from typing import Optional, List, Callable, Awaitable

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class SagaState(Enum):
    STARTED = auto()
    INVENTORY_RESERVED = auto()
    PAYMENT_PROCESSED = auto()
    SHIPPING_SCHEDULED = auto()
    COMPLETED = auto()
    # Compensating states
    COMPENSATING = auto()
    PAYMENT_REFUNDED = auto()
    INVENTORY_RELEASED = auto()
    FAILED = auto()

@dataclass
class SagaContext:
    saga_id: str
    order_id: str
    customer_id: str
    items: List[dict]
    total_amount: float
    state: SagaState = SagaState.STARTED
    reservation_id: Optional[str] = None
    payment_id: Optional[str] = None
    shipment_id: Optional[str] = None
    error: Optional[str] = None
    created_at: datetime = field(default_factory=datetime.utcnow)

class OrderSagaOrchestrator:
    """
    Saga orchestrator for order fulfillment.
    Implements forward path + compensating transactions.
    """
    
    async def execute_order_saga(self, ctx: SagaContext) -> bool:
        logger.info(f"Starting saga {ctx.saga_id} for order {ctx.order_id}")
        
        try:
            # ── Step 1: Reserve inventory ─────────────────────────────────
            ctx.state = SagaState.STARTED
            ctx.reservation_id = await self._reserve_inventory(ctx)
            ctx.state = SagaState.INVENTORY_RESERVED
            logger.info(f"Inventory reserved: {ctx.reservation_id}")
            
            # ── Step 2: Process payment ───────────────────────────────────
            ctx.payment_id = await self._process_payment(ctx)
            ctx.state = SagaState.PAYMENT_PROCESSED
            logger.info(f"Payment processed: {ctx.payment_id}")
            
            # ── Step 3: Schedule shipping ─────────────────────────────────
            ctx.shipment_id = await self._schedule_shipping(ctx)
            ctx.state = SagaState.SHIPPING_SCHEDULED
            logger.info(f"Shipping scheduled: {ctx.shipment_id}")
            
            ctx.state = SagaState.COMPLETED
            logger.info(f"Saga {ctx.saga_id} COMPLETED successfully")
            return True
            
        except InventoryError as e:
            ctx.error = str(e)
            ctx.state = SagaState.FAILED
            logger.error(f"Saga failed at inventory step: {e}")
            # No compensations needed — nothing succeeded yet
            return False
            
        except PaymentError as e:
            ctx.error = str(e)
            logger.error(f"Saga failed at payment step: {e}")
            ctx.state = SagaState.COMPENSATING
            # Compensate: release inventory
            await self._release_inventory(ctx)
            ctx.state = SagaState.INVENTORY_RELEASED
            ctx.state = SagaState.FAILED
            return False
            
        except ShippingError as e:
            ctx.error = str(e)
            logger.error(f"Saga failed at shipping step: {e}")
            ctx.state = SagaState.COMPENSATING
            # Compensate: refund payment, release inventory
            await self._refund_payment(ctx)
            ctx.state = SagaState.PAYMENT_REFUNDED
            await self._release_inventory(ctx)
            ctx.state = SagaState.INVENTORY_RELEASED
            ctx.state = SagaState.FAILED
            return False

    async def _reserve_inventory(self, ctx: SagaContext) -> str:
        """Call inventory service to reserve items."""
        await asyncio.sleep(0.01)  # Simulate network call
        # In production: POST /api/v1/inventory/reserve
        return f"RES-{uuid.uuid4().hex[:8].upper()}"
    
    async def _process_payment(self, ctx: SagaContext) -> str:
        """Call payment service."""
        await asyncio.sleep(0.02)
        return f"PAY-{uuid.uuid4().hex[:8].upper()}"
    
    async def _schedule_shipping(self, ctx: SagaContext) -> str:
        """Call shipping service."""
        await asyncio.sleep(0.01)
        return f"SHIP-{uuid.uuid4().hex[:8].upper()}"
    
    async def _release_inventory(self, ctx: SagaContext):
        """Compensating transaction: release reserved inventory."""
        logger.info(f"Compensating: releasing inventory reservation {ctx.reservation_id}")
        await asyncio.sleep(0.01)
    
    async def _refund_payment(self, ctx: SagaContext):
        """Compensating transaction: refund payment."""
        logger.info(f"Compensating: refunding payment {ctx.payment_id}")
        await asyncio.sleep(0.02)

class InventoryError(Exception): pass
class PaymentError(Exception): pass
class ShippingError(Exception): pass

async def demo():
    orchestrator = OrderSagaOrchestrator()
    
    print("=== Order Saga Orchestration Demo ===\n")
    
    # Successful saga
    ctx1 = SagaContext(
        saga_id=str(uuid.uuid4()),
        order_id="ORD-001",
        customer_id="USER-123",
        items=[{"product_id": "P1", "quantity": 2, "price": 150.0}],
        total_amount=300.0,
    )
    success = await orchestrator.execute_order_saga(ctx1)
    print(f"Order ORD-001: {'SUCCESS' if success else 'FAILED'}")
    print(f"  State: {ctx1.state.name}")
    print(f"  Reservation: {ctx1.reservation_id}")
    print(f"  Payment: {ctx1.payment_id}")
    print(f"  Shipment: {ctx1.shipment_id}\n")

asyncio.run(demo())
PYEOF

python3 saga_orchestrator.py

echo "=== Step 608 Complete: Event-Driven Patterns ==="
SCRIPT
chmod +x event-driven-patterns.sh
bash event-driven-patterns.sh
echo "Script created: event-driven-patterns.sh"
```

**สิ่งที่เรียนรู้:**
- Transactional Outbox: INSERT order + INSERT outbox ใน single transaction → ไม่มี phantom events
- Outbox Relay: `FOR UPDATE SKIP LOCKED` รองรับ multiple relay instances, retry logic (max 5)
- Debezium CDC: alternative ที่ดีกว่า polling สำหรับ production
- Saga Orchestration: forward path (3 steps) + compensating transactions (refund + release inventory)
- Error isolation: `InventoryError` → no compensation, `PaymentError` → release inventory, `ShippingError` → full rollback

---

## สรุป Part 64

| Step | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|----------------|
| 605 | Advanced Kafka | KRaft, EOS transactions, Schema Registry, Kafka Streams CEP |
| 606 | Apache Flink | Table API, CEP patterns, RocksDB state, EXACTLY_ONCE |
| 607 | Apache Pulsar | Multi-tenancy, geo-replication, Functions (serverless) |
| 608 | Event-Driven Patterns | Transactional Outbox, Saga Orchestration, compensating transactions |

**ขั้นตอนต่อไป: Part 65 — AI/ML Infrastructure, LLMOps, Vector Databases และ RAG Architecture**
