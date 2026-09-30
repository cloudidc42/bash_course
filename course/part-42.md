# Part 42: Advanced Data Platform Engineering

## Module 4: Professional Level — Data Infrastructure at Scale

### ขั้นตอนที่ 521: Apache Kafka Enterprise Platform Management

**Kafka** คือ distributed streaming platform ระดับ enterprise ที่ใช้สำหรับ real-time data pipelines

```bash
#!/bin/bash
# kafka-platform-manager.sh - Enterprise Kafka Platform Management

set -euo pipefail

KAFKA_NAMESPACE="${KAFKA_NAMESPACE:-kafka}"
KAFKA_CLUSTER_NAME="${KAFKA_CLUSTER_NAME:-production-kafka}"
STRIMZI_VERSION="${STRIMZI_VERSION:-0.38.0}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_strimzi_operator() {
    log "Installing Strimzi Kafka Operator v${STRIMZI_VERSION}..."
    
    kubectl create namespace "${KAFKA_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    kubectl apply -f "https://strimzi.io/install/latest?namespace=${KAFKA_NAMESPACE}" -n "${KAFKA_NAMESPACE}"
    
    log "Waiting for Strimzi operator..."
    kubectl rollout status deployment/strimzi-cluster-operator -n "${KAFKA_NAMESPACE}" --timeout=300s
    
    log "Strimzi operator installed successfully"
}

create_kafka_cluster() {
    local replicas="${1:-3}"
    log "Creating Kafka cluster: ${KAFKA_CLUSTER_NAME} with ${replicas} replicas..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: ${KAFKA_CLUSTER_NAME}
  namespace: ${KAFKA_NAMESPACE}
spec:
  kafka:
    version: 3.6.0
    replicas: ${replicas}
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
      - name: external
        port: 9094
        type: loadbalancer
        tls: true
        authentication:
          type: tls
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.6"
      log.retention.hours: 168
      log.retention.bytes: 107374182400
      compression.type: snappy
      message.max.bytes: 10485760
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 500Gi
          deleteClaim: false
          class: fast-ssd
    resources:
      requests:
        memory: 8Gi
        cpu: "2"
      limits:
        memory: 16Gi
        cpu: "4"
    jvmOptions:
      -Xms: 4096m
      -Xmx: 8192m
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: kafka-metrics-config.yml
    rack:
      topologyKey: topology.kubernetes.io/zone
    template:
      pod:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchExpressions:
                    - key: strimzi.io/name
                      operator: In
                      values:
                        - ${KAFKA_CLUSTER_NAME}-kafka
                topologyKey: kubernetes.io/hostname
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 100Gi
      deleteClaim: false
      class: fast-ssd
    resources:
      requests:
        memory: 2Gi
        cpu: "500m"
      limits:
        memory: 4Gi
        cpu: "1"
  entityOperator:
    topicOperator:
      resources:
        requests:
          memory: 512Mi
          cpu: "200m"
        limits:
          memory: 1Gi
          cpu: "500m"
    userOperator:
      resources:
        requests:
          memory: 512Mi
          cpu: "200m"
        limits:
          memory: 1Gi
          cpu: "500m"
EOF
    
    log "Waiting for Kafka cluster to be ready..."
    kubectl wait kafka/${KAFKA_CLUSTER_NAME} --for=condition=Ready --timeout=600s -n "${KAFKA_NAMESPACE}"
    log "Kafka cluster created successfully"
}

create_kafka_topic() {
    local topic_name="$1"
    local partitions="${2:-12}"
    local replicas="${3:-3}"
    local retention_ms="${4:-604800000}"
    
    log "Creating Kafka topic: ${topic_name}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: ${topic_name}
  namespace: ${KAFKA_NAMESPACE}
  labels:
    strimzi.io/cluster: ${KAFKA_CLUSTER_NAME}
spec:
  partitions: ${partitions}
  replicas: ${replicas}
  config:
    retention.ms: "${retention_ms}"
    retention.bytes: "5368709120"
    cleanup.policy: delete
    compression.type: snappy
    min.insync.replicas: "2"
    message.timestamp.type: CreateTime
EOF
    
    log "Topic ${topic_name} created: partitions=${partitions}, replicas=${replicas}"
}

create_kafka_user() {
    local username="$1"
    local role="${2:-consumer}"
    local topic_pattern="${3:-.*}"
    
    log "Creating Kafka user: ${username} with role: ${role}"
    
    local acls=""
    case "${role}" in
        "producer")
            acls='
    - resource:
        type: topic
        name: '"${topic_pattern}"'
        patternType: prefix
      operations:
        - Write
        - Describe
        - Create'
            ;;
        "consumer")
            acls='
    - resource:
        type: topic
        name: '"${topic_pattern}"'
        patternType: prefix
      operations:
        - Read
        - Describe
    - resource:
        type: group
        name: '"${username}-group"'
        patternType: literal
      operations:
        - Read'
            ;;
        "admin")
            acls='
    - resource:
        type: topic
        name: "*"
        patternType: literal
      operations:
        - All
    - resource:
        type: group
        name: "*"
        patternType: literal
      operations:
        - All'
            ;;
    esac
    
    cat <<EOF | kubectl apply -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: ${username}
  namespace: ${KAFKA_NAMESPACE}
  labels:
    strimzi.io/cluster: ${KAFKA_CLUSTER_NAME}
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:${acls}
EOF
    
    log "Kafka user ${username} created with ${role} permissions"
}

deploy_kafka_connect() {
    local connect_name="${1:-kafka-connect}"
    local connector_image="${2:-confluentinc/cp-kafka-connect:7.5.0}"
    
    log "Deploying Kafka Connect cluster: ${connect_name}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: ${connect_name}
  namespace: ${KAFKA_NAMESPACE}
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  version: 3.6.0
  replicas: 3
  bootstrapServers: ${KAFKA_CLUSTER_NAME}-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: ${KAFKA_CLUSTER_NAME}-cluster-ca-cert
        certificate: ca.crt
  authentication:
    type: tls
    certificateAndKey:
      secretName: connect-user
      certificate: user.crt
      key: user.key
  config:
    group.id: ${connect_name}-cluster
    offset.storage.topic: ${connect_name}-offsets
    config.storage.topic: ${connect_name}-configs
    status.storage.topic: ${connect_name}-status
    config.storage.replication.factor: 3
    offset.storage.replication.factor: 3
    status.storage.replication.factor: 3
    key.converter: org.apache.kafka.connect.json.JsonConverter
    value.converter: org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable: "false"
    value.converter.schemas.enable: "false"
  resources:
    requests:
      memory: 2Gi
      cpu: "500m"
    limits:
      memory: 4Gi
      cpu: "2"
  build:
    output:
      type: docker
      image: my-registry/kafka-connect-custom:latest
    plugins:
      - name: debezium-mysql-connector
        artifacts:
          - type: maven
            group: io.debezium
            artifact: debezium-connector-mysql
            version: 2.4.0.Final
      - name: confluentinc-kafka-connect-s3
        artifacts:
          - type: maven
            group: io.confluent
            artifact: kafka-connect-s3
            version: 10.7.0
EOF
    
    log "Kafka Connect cluster deployed"
}

monitor_kafka_lag() {
    log "Monitoring Kafka consumer group lag..."
    
    local bootstrap_server="${KAFKA_CLUSTER_NAME}-kafka-bootstrap:9092"
    
    kubectl exec -it "${KAFKA_CLUSTER_NAME}-kafka-0" -n "${KAFKA_NAMESPACE}" -- \
        /opt/kafka/bin/kafka-consumer-groups.sh \
        --bootstrap-server "${bootstrap_server}" \
        --list | while read group; do
        echo "=== Group: ${group} ==="
        kubectl exec -it "${KAFKA_CLUSTER_NAME}-kafka-0" -n "${KAFKA_NAMESPACE}" -- \
            /opt/kafka/bin/kafka-consumer-groups.sh \
            --bootstrap-server "${bootstrap_server}" \
            --group "${group}" \
            --describe 2>/dev/null | \
            awk 'NR>1 {lag+=$6} END {print "Total Lag:", lag}'
    done
}

kafka_topic_management() {
    log "Creating standard topic set for data platform..."
    
    local topics=(
        "events.user-actions:24:3:2592000000"
        "events.transactions:48:3:7776000000"
        "events.system-logs:12:3:604800000"
        "stream.processed-events:12:3:259200000"
        "dead-letter.queue:6:3:2592000000"
        "analytics.aggregates:12:3:31536000000"
    )
    
    for topic_config in "${topics[@]}"; do
        IFS=':' read -r name partitions replicas retention <<< "${topic_config}"
        create_kafka_topic "${name}" "${partitions}" "${replicas}" "${retention}"
    done
    
    log "Standard topic set created"
}

kafka_performance_test() {
    local topic="${1:-test-performance}"
    local num_records="${2:-1000000}"
    local record_size="${3:-1024}"
    
    log "Running Kafka performance test on topic: ${topic}"
    
    kubectl exec -it "${KAFKA_CLUSTER_NAME}-kafka-0" -n "${KAFKA_NAMESPACE}" -- \
        /opt/kafka/bin/kafka-producer-perf-test.sh \
        --topic "${topic}" \
        --num-records "${num_records}" \
        --record-size "${record_size}" \
        --throughput -1 \
        --producer-props bootstrap.servers="${KAFKA_CLUSTER_NAME}-kafka-bootstrap:9092" \
            acks=all \
            compression.type=snappy \
            batch.size=65536 \
            linger.ms=5
    
    log "Performance test completed"
}

case "${1:-help}" in
    "install-operator") install_strimzi_operator ;;
    "create-cluster") create_kafka_cluster "${2:-3}" ;;
    "create-topic") create_kafka_topic "$2" "${3:-12}" "${4:-3}" "${5:-604800000}" ;;
    "create-user") create_kafka_user "$2" "${3:-consumer}" "${4:-}" ;;
    "deploy-connect") deploy_kafka_connect "${2:-kafka-connect}" ;;
    "monitor-lag") monitor_kafka_lag ;;
    "setup-topics") kafka_topic_management ;;
    "perf-test") kafka_performance_test "${2:-test}" "${3:-1000000}" "${4:-1024}" ;;
    *) echo "Usage: $0 {install-operator|create-cluster|create-topic|create-user|deploy-connect|monitor-lag|setup-topics|perf-test}" ;;
esac
```

### ขั้นตอนที่ 522: Apache Flink Stream Processing Platform

**Apache Flink** สำหรับ stateful stream processing ขนาด enterprise

```bash
#!/bin/bash
# flink-platform-manager.sh - Apache Flink Stream Processing Platform

set -euo pipefail

FLINK_NAMESPACE="${FLINK_NAMESPACE:-flink}"
FLINK_OPERATOR_VERSION="${FLINK_OPERATOR_VERSION:-1.7.0}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_flink_operator() {
    log "Installing Flink Kubernetes Operator..."
    
    kubectl create namespace "${FLINK_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-${FLINK_OPERATOR_VERSION}/
    helm repo update
    
    helm upgrade --install flink-kubernetes-operator flink-operator-repo/flink-kubernetes-operator \
        --namespace "${FLINK_NAMESPACE}" \
        --set webhook.create=true \
        --set operatorConfiguration.flink-conf.yaml="kubernetes.operator.metrics.reporter.slf4j.factory.class: org.apache.flink.metrics.slf4j.Slf4jReporterFactory" \
        --wait
    
    log "Flink operator installed"
}

deploy_flink_session_cluster() {
    local cluster_name="${1:-flink-session}"
    local tm_replicas="${2:-3}"
    
    log "Deploying Flink Session Cluster: ${cluster_name}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: ${cluster_name}
  namespace: ${FLINK_NAMESPACE}
spec:
  image: flink:1.18-scala_2.12-java11
  flinkVersion: v1_18
  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "4"
    state.backend: rocksdb
    state.backend.rocksdb.use-bloom-filter: "true"
    state.checkpoints.dir: s3://flink-checkpoints/checkpoints
    state.savepoints.dir: s3://flink-checkpoints/savepoints
    execution.checkpointing.interval: "60000"
    execution.checkpointing.mode: EXACTLY_ONCE
    execution.checkpointing.min-pause: "30000"
    execution.checkpointing.timeout: "300000"
    restart-strategy: failure-rate
    restart-strategy.failure-rate.max-failures-per-interval: "3"
    restart-strategy.failure-rate.failure-rate-interval: "5 min"
    restart-strategy.failure-rate.delay: "30 s"
    metrics.reporter.prom.factory.class: org.apache.flink.metrics.prometheus.PrometheusReporterFactory
    metrics.reporter.prom.port: "9249"
    table.exec.mini-batch.enabled: "true"
    table.exec.mini-batch.allow-latency: "5 s"
    table.exec.mini-batch.size: "5000"
  serviceAccount: flink-sa
  jobManager:
    resource:
      memory: "4096m"
      cpu: 1
    replicas: 1
  taskManager:
    resource:
      memory: "8192m"
      cpu: 2
    replicas: ${tm_replicas}
  podTemplate:
    spec:
      containers:
        - name: flink-main-container
          env:
            - name: AWS_REGION
              value: ap-southeast-1
          volumeMounts:
            - mountPath: /opt/flink/conf/log4j.properties
              name: flink-config-volume
              subPath: log4j.properties
      volumes:
        - name: flink-config-volume
          configMap:
            name: flink-config
EOF
    
    log "Flink Session Cluster deployed: ${cluster_name}"
}

submit_flink_sql_job() {
    local job_name="${1:-sql-job}"
    local sql_statements="${2}"
    
    log "Submitting Flink SQL job: ${job_name}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: ${job_name}
  namespace: ${FLINK_NAMESPACE}
spec:
  image: flink:1.18-scala_2.12-java11
  flinkVersion: v1_18
  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "2"
    state.backend: rocksdb
    state.checkpoints.dir: s3://flink-checkpoints/${job_name}/checkpoints
    execution.checkpointing.interval: "30000"
  serviceAccount: flink-sa
  jobManager:
    resource:
      memory: "2048m"
      cpu: 0.5
  taskManager:
    resource:
      memory: "4096m"
      cpu: 1
  job:
    jarURI: local:///opt/flink/opt/flink-table-store-flink-1.18.jar
    entryClass: org.apache.flink.table.gateway.SqlGatewayEndpointFactoryUtils
    args: []
    parallelism: 4
    upgradeMode: stateful
    state: running
EOF
    
    log "Flink SQL job submitted: ${job_name}"
}

create_flink_streaming_job() {
    local job_name="${1:-stream-processor}"
    local jar_uri="${2:-local:///opt/flink/examples/streaming/StateMachineExample.jar}"
    local parallelism="${3:-4}"
    
    log "Creating Flink streaming job: ${job_name}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: ${job_name}
  namespace: ${FLINK_NAMESPACE}
spec:
  image: my-flink-app:latest
  flinkVersion: v1_18
  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "4"
    state.backend: rocksdb
    state.checkpoints.dir: s3://flink-checkpoints/${job_name}
    execution.checkpointing.interval: "60000"
    execution.checkpointing.mode: EXACTLY_ONCE
    metrics.reporter.prom.factory.class: org.apache.flink.metrics.prometheus.PrometheusReporterFactory
    metrics.reporter.prom.port: "9249"
  serviceAccount: flink-sa
  jobManager:
    resource:
      memory: "2048m"
      cpu: 1
  taskManager:
    resource:
      memory: "8192m"
      cpu: 2
  job:
    jarURI: ${jar_uri}
    entryClass: com.company.streaming.StreamProcessor
    args:
      - --kafka-brokers
      - production-kafka-kafka-bootstrap.kafka:9092
      - --input-topic
      - events.user-actions
      - --output-topic
      - stream.processed-events
      - --parallelism
      - "${parallelism}"
    parallelism: ${parallelism}
    upgradeMode: stateful
    state: running
    savepointTriggerNonce: 0
EOF
    
    log "Flink streaming job created: ${job_name}"
}

manage_flink_savepoints() {
    local job_name="$1"
    local action="${2:-trigger}"
    
    log "Managing savepoint for job: ${job_name}, action: ${action}"
    
    case "${action}" in
        "trigger")
            kubectl patch flinkdeployment "${job_name}" -n "${FLINK_NAMESPACE}" \
                --type merge \
                -p '{"spec":{"job":{"savepointTriggerNonce":'"$(date +%s)"'}}}'
            log "Savepoint triggered for ${job_name}"
            ;;
        "restore")
            local savepoint_path="${3:-}"
            if [[ -z "${savepoint_path}" ]]; then
                log "ERROR: Savepoint path required for restore"
                exit 1
            fi
            kubectl patch flinkdeployment "${job_name}" -n "${FLINK_NAMESPACE}" \
                --type merge \
                -p "{\"spec\":{\"job\":{\"initialSavepointPath\":\"${savepoint_path}\",\"upgradeMode\":\"savepoint\"}}}"
            log "Job ${job_name} will restore from: ${savepoint_path}"
            ;;
        "stop")
            kubectl patch flinkdeployment "${job_name}" -n "${FLINK_NAMESPACE}" \
                --type merge \
                -p '{"spec":{"job":{"state":"suspended","upgradeMode":"savepoint"}}}'
            log "Job ${job_name} stopped with savepoint"
            ;;
    esac
}

monitor_flink_jobs() {
    log "=== Flink Jobs Status ==="
    kubectl get flinkdeployment -n "${FLINK_NAMESPACE}" -o custom-columns=\
"NAME:.metadata.name,STATE:.status.jobStatus.state,UPTIME:.status.jobStatus.startTime,RESTARTS:.status.jobStatus.jobRestartCount"
}

case "${1:-help}" in
    "install") install_flink_operator ;;
    "session-cluster") deploy_flink_session_cluster "${2:-flink-session}" "${3:-3}" ;;
    "submit-job") create_flink_streaming_job "$2" "${3}" "${4:-4}" ;;
    "savepoint") manage_flink_savepoints "$2" "${3:-trigger}" "${4:-}" ;;
    "monitor") monitor_flink_jobs ;;
    *) echo "Usage: $0 {install|session-cluster|submit-job|savepoint|monitor}" ;;
esac
```

### ขั้นตอนที่ 523: Data Lake Architecture with Delta Lake

**Delta Lake** สำหรับ ACID transactions บน data lake ขนาด enterprise

```bash
#!/bin/bash
# delta-lake-manager.sh - Delta Lake Data Platform Management

set -euo pipefail

SPARK_NAMESPACE="${SPARK_NAMESPACE:-spark}"
S3_BUCKET="${S3_BUCKET:-my-data-lake}"
DELTA_LOG_DIR="${DELTA_LOG_DIR:-s3://${S3_BUCKET}/delta}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_spark_operator() {
    log "Installing Spark on Kubernetes Operator..."
    
    kubectl create namespace "${SPARK_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator
    helm repo update
    
    helm upgrade --install spark-operator spark-operator/spark-operator \
        --namespace "${SPARK_NAMESPACE}" \
        --set sparkJobNamespace="${SPARK_NAMESPACE}" \
        --set enableWebhook=true \
        --set enableMetrics=true \
        --set metrics.enable=true \
        --set metrics.port=10254 \
        --wait
    
    log "Spark operator installed"
}

create_delta_table() {
    local table_name="$1"
    local schema="$2"
    local partition_by="${3:-}"
    
    log "Creating Delta table: ${table_name}"
    
    local partition_clause=""
    if [[ -n "${partition_by}" ]]; then
        partition_clause="PARTITIONED BY (${partition_by})"
    fi
    
    cat <<EOF > /tmp/create_delta_table.py
from delta import *
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("DeltaTableCreation") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .config("spark.hadoop.fs.s3a.aws.credentials.provider", "com.amazonaws.auth.WebIdentityTokenCredentialsProvider") \
    .getOrCreate()

spark.sql("""
    CREATE TABLE IF NOT EXISTS delta.\`${DELTA_LOG_DIR}/${table_name}\`
    (${schema})
    USING DELTA
    ${partition_clause}
    TBLPROPERTIES (
        'delta.autoOptimize.optimizeWrite' = 'true',
        'delta.autoOptimize.autoCompact' = 'true',
        'delta.enableChangeDataFeed' = 'true',
        'delta.columnMapping.mode' = 'name',
        'delta.minReaderVersion' = '2',
        'delta.minWriterVersion' = '5'
    )
""")

print(f"Delta table created: ${table_name}")
spark.stop()
EOF
    
    kubectl create configmap "create-table-${table_name}" \
        --from-file=script.py=/tmp/create_delta_table.py \
        -n "${SPARK_NAMESPACE}" \
        --dry-run=client -o yaml | kubectl apply -f -
    
    log "Delta table creation script prepared for: ${table_name}"
}

submit_delta_spark_job() {
    local job_name="$1"
    local python_script="$2"
    local driver_memory="${3:-2g}"
    local executor_memory="${4:-4g}"
    local executor_count="${5:-4}"
    
    log "Submitting Spark Delta job: ${job_name}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: ${job_name}
  namespace: ${SPARK_NAMESPACE}
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: my-spark-delta:3.5.0-delta-3.0.0
  imagePullPolicy: IfNotPresent
  mainApplicationFile: local:///opt/spark/work-dir/app.py
  sparkVersion: "3.5.0"
  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
    onFailureRetryInterval: 10
    onSubmissionFailureRetries: 5
    onSubmissionFailureRetryInterval: 20
  hadoopConf:
    fs.s3a.impl: org.apache.hadoop.fs.s3a.S3AFileSystem
    fs.s3a.aws.credentials.provider: com.amazonaws.auth.WebIdentityTokenCredentialsProvider
    fs.s3a.path.style.access: "true"
    fs.s3a.connection.ssl.enabled: "true"
    fs.s3a.fast.upload: "true"
    fs.s3a.multipart.size: "128M"
  sparkConf:
    spark.sql.extensions: io.delta.sql.DeltaSparkSessionExtension
    spark.sql.catalog.spark_catalog: org.apache.spark.sql.delta.catalog.DeltaCatalog
    spark.delta.logStore.class: io.delta.storage.S3SingleDriverLogStore
    spark.sql.adaptive.enabled: "true"
    spark.sql.adaptive.coalescePartitions.enabled: "true"
    spark.sql.adaptive.skewJoin.enabled: "true"
    spark.serializer: org.apache.spark.serializer.KryoSerializer
    spark.dynamicAllocation.enabled: "true"
    spark.dynamicAllocation.minExecutors: "1"
    spark.dynamicAllocation.maxExecutors: "20"
    spark.shuffle.service.enabled: "true"
    spark.metrics.conf.*.sink.prometheusServlet.class: org.apache.spark.metrics.sink.PrometheusServlet
    spark.metrics.conf.*.sink.prometheusServlet.path: /metrics/prometheus
  driver:
    cores: 1
    coreLimit: "1200m"
    memory: ${driver_memory}
    labels:
      version: 3.5.0
    serviceAccount: spark-sa
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/spark-role
  executor:
    cores: 2
    instances: ${executor_count}
    memory: ${executor_memory}
    labels:
      version: 3.5.0
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/spark-role
EOF
    
    log "Spark Delta job submitted: ${job_name}"
}

delta_vacuum_tables() {
    local table_path="${1:-${DELTA_LOG_DIR}}"
    local retention_hours="${2:-168}"
    
    log "Running VACUUM on Delta tables at: ${table_path}"
    
    cat <<EOF > /tmp/vacuum_delta.py
from delta import *
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("DeltaVacuum") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .config("spark.databricks.delta.retentionDurationCheck.enabled", "false") \
    .getOrCreate()

from delta.tables import DeltaTable

dt = DeltaTable.forPath(spark, "${table_path}")
dt.vacuum(retentionHours=${retention_hours})

print(f"VACUUM completed for ${table_path}, retention: ${retention_hours} hours")
spark.stop()
EOF
    
    log "Delta VACUUM script created"
}

delta_optimize_tables() {
    local table_path="${1:-${DELTA_LOG_DIR}/events}"
    local z_order_cols="${2:-}"
    
    log "Running OPTIMIZE on Delta table: ${table_path}"
    
    cat <<EOF > /tmp/optimize_delta.py
from delta import *
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("DeltaOptimize") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

from delta.tables import DeltaTable

dt = DeltaTable.forPath(spark, "${table_path}")

# Run OPTIMIZE with Z-ORDER if columns specified
if "${z_order_cols}":
    z_order_list = ["${z_order_cols}".replace(",", "\",\"")]
    dt.optimize().executeZOrderBy(*z_order_list)
    print(f"OPTIMIZE with ZORDER({z_order_list}) completed")
else:
    dt.optimize().executeCompaction()
    print(f"OPTIMIZE (compaction) completed")

# Show table history
history = dt.history(10)
history.show(truncate=False)

spark.stop()
EOF
    
    log "Delta OPTIMIZE script created for: ${table_path}"
}

generate_delta_lake_schema() {
    log "Generating standard Delta Lake table schemas for data platform..."
    
    declare -A tables
    tables["events.user_events"]="event_id STRING NOT NULL, user_id STRING NOT NULL, event_type STRING NOT NULL, event_timestamp TIMESTAMP NOT NULL, properties MAP<STRING,STRING>, session_id STRING, ip_address STRING, user_agent STRING, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()"
    tables["events.transactions"]="transaction_id STRING NOT NULL, user_id STRING NOT NULL, amount DECIMAL(18,4) NOT NULL, currency STRING NOT NULL, status STRING NOT NULL, merchant_id STRING, metadata MAP<STRING,STRING>, created_at TIMESTAMP NOT NULL, updated_at TIMESTAMP"
    tables["dimension.users"]="user_id STRING NOT NULL, email STRING, name STRING, tier STRING DEFAULT 'free', country STRING, created_at TIMESTAMP, updated_at TIMESTAMP, is_active BOOLEAN DEFAULT true, metadata MAP<STRING,STRING>"
    tables["fact.daily_metrics"]="metric_date DATE NOT NULL, metric_name STRING NOT NULL, dimension_key STRING, value DOUBLE NOT NULL, unit STRING, tags MAP<STRING,STRING>, computed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()"
    
    for table_name in "${!tables[@]}"; do
        local schema="${tables[$table_name]}"
        local safe_name="${table_name//\//_}"
        create_delta_table "${safe_name}" "${schema}" "event_date DATE"
    done
    
    log "Standard Delta Lake schemas created"
}

monitor_delta_jobs() {
    log "=== Spark Application Status ==="
    kubectl get sparkapplication -n "${SPARK_NAMESPACE}" \
        -o custom-columns="NAME:.metadata.name,STATE:.status.applicationState.state,DRIVER:.status.driverInfo.podName,EXECUTORS:.status.executorState"
}

case "${1:-help}" in
    "install") install_spark_operator ;;
    "create-table") create_delta_table "$2" "$3" "${4:-}" ;;
    "submit-job") submit_delta_spark_job "$2" "$3" "${4:-2g}" "${5:-4g}" "${6:-4}" ;;
    "vacuum") delta_vacuum_tables "${2:-${DELTA_LOG_DIR}}" "${3:-168}" ;;
    "optimize") delta_optimize_tables "${2:-${DELTA_LOG_DIR}/events}" "${3:-}" ;;
    "setup-schema") generate_delta_lake_schema ;;
    "monitor") monitor_delta_jobs ;;
    *) echo "Usage: $0 {install|create-table|submit-job|vacuum|optimize|setup-schema|monitor}" ;;
esac
```

### ขั้นตอนที่ 524: ClickHouse Analytics Database Platform

**ClickHouse** สำหรับ OLAP analytics ที่มีประสิทธิภาพสูงสุดใน enterprise

```bash
#!/bin/bash
# clickhouse-platform-manager.sh - ClickHouse Analytics Platform

set -euo pipefail

CH_NAMESPACE="${CH_NAMESPACE:-clickhouse}"
CH_CLUSTER_NAME="${CH_CLUSTER_NAME:-production}"
CH_OPERATOR_VERSION="${CH_OPERATOR_VERSION:-0.23.0}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_clickhouse_operator() {
    log "Installing ClickHouse Operator..."
    
    kubectl create namespace "${CH_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    kubectl apply -f "https://raw.githubusercontent.com/Altinity/clickhouse-operator/master/deploy/operator/clickhouse-operator-install-bundle.yaml" \
        -n "${CH_NAMESPACE}"
    
    kubectl rollout status deployment/clickhouse-operator -n "${CH_NAMESPACE}" --timeout=120s
    
    log "ClickHouse operator installed"
}

create_clickhouse_cluster() {
    local shards="${1:-2}"
    local replicas="${2:-2}"
    
    log "Creating ClickHouse cluster: ${CH_CLUSTER_NAME} (${shards} shards x ${replicas} replicas)"
    
    cat <<EOF | kubectl apply -f -
apiVersion: clickhouse.altinity.com/v1
kind: ClickHouseInstallation
metadata:
  name: ${CH_CLUSTER_NAME}
  namespace: ${CH_NAMESPACE}
spec:
  defaults:
    replicasUseFQDN: no
    distributedDDL:
      profile: default
    templates:
      podTemplate: pod-template
      dataVolumeClaimTemplate: data-volume
      logVolumeClaimTemplate: log-volume
  
  configuration:
    zookeeper:
      nodes:
        - host: zookeeper.zookeeper
          port: 2181
      session_timeout_ms: 30000
      operation_timeout_ms: 10000
    
    users:
      admin/password: "$(openssl rand -base64 32)"
      admin/networks/ip: "::/0"
      admin/profile: default
      admin/quota: default
      
      readonly/password: "readonly_password"
      readonly/networks/ip: "::/0"
      readonly/profile: readonly
      readonly/quota: default
    
    profiles:
      default/max_memory_usage: 10000000000
      default/use_uncompressed_cache: 0
      default/load_balancing: random
      default/max_threads: 8
      default/max_execution_time: 300
      
      readonly/readonly: 1
      readonly/max_memory_usage: 5000000000
      readonly/max_execution_time: 60
      
      analytics/max_memory_usage: 50000000000
      analytics/max_threads: 32
      analytics/max_execution_time: 3600
      analytics/distributed_aggregation_memory_efficient: 1
    
    quotas:
      default/interval/duration: 3600
      default/interval/queries: 0
      default/interval/errors: 0
      default/interval/result_rows: 0
      default/interval/read_rows: 0
      default/interval/execution_time: 0
    
    settings:
      max_concurrent_queries: 100
      max_connections: 4096
      keep_alive_timeout: 3
      tcp_port: 9000
      http_port: 8123
      
      compression/case/method: zstd
      merge_tree/max_parts_in_total: 100000
      merge_tree/parts_to_throw_insert: 300
      merge_tree/max_delay_to_insert: 1
      
      background_pool_size: 32
      background_merges_mutations_concurrency_ratio: 2
      
      remote_servers/${CH_CLUSTER_NAME}/shard/internal_replication: true
    
    clusters:
      - name: ${CH_CLUSTER_NAME}
        layout:
          shardsCount: ${shards}
          replicasCount: ${replicas}
  
  templates:
    podTemplates:
      - name: pod-template
        spec:
          containers:
            - name: clickhouse
              image: clickhouse/clickhouse-server:23.12
              resources:
                requests:
                  memory: "16Gi"
                  cpu: "4"
                limits:
                  memory: "32Gi"
                  cpu: "8"
              env:
                - name: TZ
                  value: "Asia/Bangkok"
              ports:
                - name: http
                  containerPort: 8123
                - name: client
                  containerPort: 9000
                - name: interserver
                  containerPort: 9009
    
    volumeClaimTemplates:
      - name: data-volume
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 2Ti
          storageClassName: fast-ssd
      
      - name: log-volume
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 50Gi
          storageClassName: standard
EOF
    
    log "Waiting for ClickHouse cluster to be ready..."
    kubectl wait chi/${CH_CLUSTER_NAME} --for=jsonpath='{.status.status}'=Completed \
        --timeout=600s -n "${CH_NAMESPACE}"
    log "ClickHouse cluster created: ${CH_CLUSTER_NAME}"
}

create_analytics_tables() {
    log "Creating analytics tables in ClickHouse..."
    
    local ch_host=$(kubectl get service -n "${CH_NAMESPACE}" \
        -l "clickhouse.altinity.com/chi=${CH_CLUSTER_NAME}" \
        -o jsonpath='{.items[0].spec.clusterIP}')
    
    local queries=(
        "CREATE DATABASE IF NOT EXISTS analytics ON CLUSTER ${CH_CLUSTER_NAME}"
        
        "CREATE TABLE IF NOT EXISTS analytics.events ON CLUSTER ${CH_CLUSTER_NAME}
         (
             event_id UUID DEFAULT generateUUIDv4(),
             user_id String,
             event_type LowCardinality(String),
             event_timestamp DateTime64(3, 'UTC'),
             properties Map(String, String),
             session_id String,
             ip_address IPv4,
             country LowCardinality(String),
             date Date DEFAULT toDate(event_timestamp)
         )
         ENGINE = ReplicatedMergeTree('/clickhouse/{cluster}/analytics/events/{shard}', '{replica}')
         PARTITION BY toYYYYMM(date)
         ORDER BY (event_type, user_id, event_timestamp)
         TTL date + INTERVAL 2 YEAR
         SETTINGS index_granularity = 8192, storage_policy = 'tiered'"
        
        "CREATE TABLE IF NOT EXISTS analytics.events_distributed ON CLUSTER ${CH_CLUSTER_NAME}
         AS analytics.events
         ENGINE = Distributed(${CH_CLUSTER_NAME}, analytics, events, rand())"
        
        "CREATE TABLE IF NOT EXISTS analytics.user_metrics ON CLUSTER ${CH_CLUSTER_NAME}
         (
             metric_date Date,
             user_id String,
             metric_name LowCardinality(String),
             value Float64,
             count UInt64 DEFAULT 1
         )
         ENGINE = ReplicatedAggregatingMergeTree('/clickhouse/{cluster}/analytics/user_metrics/{shard}', '{replica}')
         PARTITION BY toYYYYMM(metric_date)
         ORDER BY (metric_date, metric_name, user_id)"
        
        "CREATE MATERIALIZED VIEW IF NOT EXISTS analytics.hourly_stats ON CLUSTER ${CH_CLUSTER_NAME}
         ENGINE = ReplicatedSummingMergeTree('/clickhouse/{cluster}/analytics/hourly_stats/{shard}', '{replica}')
         PARTITION BY toYYYYMM(hour)
         ORDER BY (hour, event_type, country)
         AS SELECT
             toStartOfHour(event_timestamp) AS hour,
             event_type,
             country,
             count() AS event_count,
             uniq(user_id) AS unique_users,
             uniq(session_id) AS unique_sessions
         FROM analytics.events
         GROUP BY hour, event_type, country"
    )
    
    for query in "${queries[@]}"; do
        echo "${query}" | kubectl exec -i -n "${CH_NAMESPACE}" \
            "chi-${CH_CLUSTER_NAME}-${CH_CLUSTER_NAME}-0-0-0" -- \
            clickhouse-client --multiquery
        log "Query executed"
    done
    
    log "Analytics tables created"
}

run_clickhouse_query() {
    local query="$1"
    local format="${2:-PrettyCompactMonoBlock}"
    
    kubectl exec -i -n "${CH_NAMESPACE}" \
        "chi-${CH_CLUSTER_NAME}-${CH_CLUSTER_NAME}-0-0-0" -- \
        clickhouse-client \
        --query "${query}" \
        --format "${format}"
}

monitor_clickhouse_performance() {
    log "=== ClickHouse Performance Metrics ==="
    
    run_clickhouse_query "
        SELECT 
            query_kind,
            count() as query_count,
            round(avg(query_duration_ms), 2) as avg_duration_ms,
            round(max(query_duration_ms), 2) as max_duration_ms,
            formatReadableSize(sum(memory_usage)) as total_memory,
            sum(read_rows) as total_read_rows
        FROM system.query_log
        WHERE event_date >= today() - 1
          AND type = 'QueryFinish'
        GROUP BY query_kind
        ORDER BY query_count DESC
        LIMIT 20
    "
    
    log "=== Active Merges ==="
    run_clickhouse_query "
        SELECT database, table, elapsed, progress, num_parts
        FROM system.merges
        ORDER BY elapsed DESC
        LIMIT 10
    "
    
    log "=== Table Sizes ==="
    run_clickhouse_query "
        SELECT 
            database,
            table,
            formatReadableSize(sum(data_compressed_bytes)) AS compressed,
            formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
            round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS compression_ratio,
            sum(rows) AS rows
        FROM system.parts
        WHERE active = 1
        GROUP BY database, table
        ORDER BY sum(data_compressed_bytes) DESC
        LIMIT 20
    "
}

case "${1:-help}" in
    "install") install_clickhouse_operator ;;
    "create-cluster") create_clickhouse_cluster "${2:-2}" "${3:-2}" ;;
    "create-tables") create_analytics_tables ;;
    "query") run_clickhouse_query "$2" "${3:-PrettyCompactMonoBlock}" ;;
    "monitor") monitor_clickhouse_performance ;;
    *) echo "Usage: $0 {install|create-cluster|create-tables|query|monitor}" ;;
esac
```

### ขั้นตอนที่ 525: Data Orchestration with Apache Airflow

**Apache Airflow** สำหรับ workflow orchestration ระดับ enterprise

```bash
#!/bin/bash
# airflow-platform-manager.sh - Apache Airflow Data Orchestration

set -euo pipefail

AIRFLOW_NAMESPACE="${AIRFLOW_NAMESPACE:-airflow}"
AIRFLOW_RELEASE="${AIRFLOW_RELEASE:-airflow}"
AIRFLOW_VERSION="${AIRFLOW_VERSION:-2.8.0}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_airflow() {
    log "Installing Apache Airflow ${AIRFLOW_VERSION} with KubernetesExecutor..."
    
    kubectl create namespace "${AIRFLOW_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF > /tmp/airflow-values.yaml
airflowVersion: ${AIRFLOW_VERSION}

executor: KubernetesExecutor

defaultAirflowRepository: apache/airflow
defaultAirflowTag: "${AIRFLOW_VERSION}"

images:
  airflow:
    repository: my-registry/airflow-custom
    tag: "${AIRFLOW_VERSION}"
    pullPolicy: IfNotPresent

workers:
  resources:
    limits:
      cpu: "4"
      memory: "8Gi"
    requests:
      cpu: "500m"
      memory: "1Gi"
  
  podAnnotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::ACCOUNT:role/airflow-worker-role"

webserver:
  replicas: 2
  resources:
    limits:
      cpu: "2"
      memory: "4Gi"
    requests:
      cpu: "500m"
      memory: "1Gi"
  service:
    type: ClusterIP
  
  defaultUser:
    enabled: true
    role: Admin
    username: admin
    email: admin@company.com
    firstName: Admin
    lastName: User
    password: "$(openssl rand -base64 24)"

scheduler:
  replicas: 2
  resources:
    limits:
      cpu: "2"
      memory: "4Gi"
    requests:
      cpu: "500m"
      memory: "1Gi"
  
  livenessProbe:
    initialDelaySeconds: 10
    timeoutSeconds: 20
    failureThreshold: 5
    periodSeconds: 60

triggerer:
  enabled: true
  replicas: 2
  resources:
    limits:
      cpu: "1"
      memory: "2Gi"

postgresql:
  enabled: false

externalDatabase:
  type: postgres
  host: airflow-postgres.airflow
  port: 5432
  database: airflow
  user: airflow
  passwordSecret: airflow-postgresql-secret
  passwordSecretKey: postgresql-password

redis:
  enabled: false

config:
  core:
    dags_folder: /opt/airflow/dags
    load_examples: "False"
    executor: KubernetesExecutor
    default_timezone: Asia/Bangkok
    enable_xcom_pickling: "False"
    max_active_tasks_per_dag: 16
    max_active_runs_per_dag: 5
  
  kubernetes:
    namespace: ${AIRFLOW_NAMESPACE}
    worker_container_repository: my-registry/airflow-custom
    worker_container_tag: "${AIRFLOW_VERSION}"
    delete_worker_pods: "True"
    delete_worker_pods_on_failure: "False"
    pod_template_file: /opt/airflow/pod_templates/pod_template_file.yaml
  
  kubernetes_executor:
    worker_pods_creation_batch_size: 16
    verify_ssl: "True"
  
  webserver:
    expose_config: "False"
    web_server_ssl_cert: ""
    web_server_ssl_key: ""
    base_url: "https://airflow.company.com"
  
  smtp:
    smtp_host: smtp.company.com
    smtp_starttls: "True"
    smtp_ssl: "False"
    smtp_port: "587"
    smtp_mail_from: airflow@company.com
  
  logging:
    remote_logging: "True"
    remote_base_log_folder: "s3://airflow-logs/logs"
    remote_log_conn_id: aws_default
    logging_level: INFO
  
  celery:
    worker_concurrency: "16"
  
  metrics:
    statsd_on: "True"
    statsd_host: statsd-exporter.monitoring
    statsd_port: "8125"
    statsd_prefix: airflow

dags:
  gitSync:
    enabled: true
    repo: https://github.com/company/airflow-dags.git
    branch: main
    rev: HEAD
    depth: 1
    maxFailures: 3
    subPath: dags
    credentialsSecret: airflow-git-secret
    period: 60s
    containerName: git-sync
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"

logs:
  persistence:
    enabled: true
    storageClassName: standard
    size: 100Gi

pgbouncer:
  enabled: true
  maxClientConn: 100
  poolSize: 5

ingress:
  web:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - name: airflow.company.com
        tls:
          secretName: airflow-tls
          enabled: true
EOF
    
    helm repo add apache-airflow https://airflow.apache.org
    helm repo update
    
    helm upgrade --install "${AIRFLOW_RELEASE}" apache-airflow/airflow \
        --namespace "${AIRFLOW_NAMESPACE}" \
        -f /tmp/airflow-values.yaml \
        --timeout 600s \
        --wait
    
    log "Apache Airflow installed"
}

create_airflow_connections() {
    log "Creating standard Airflow connections..."
    
    local connections=(
        "aws_default:aws::::{\"region_name\":\"ap-southeast-1\"}"
        "kafka_default:kafka::production-kafka-kafka-bootstrap.kafka::9092:{}"
        "spark_default:spark::spark://spark-master.spark::7077:{}"
        "clickhouse_default:generic::chi-production-production-0-0-0.clickhouse::8123:{\"schema\":\"analytics\"}"
    )
    
    for conn in "${connections[@]}"; do
        IFS=':' read -r conn_id conn_type host password port extra <<< "${conn}"
        
        kubectl exec -n "${AIRFLOW_NAMESPACE}" \
            deployment/airflow-webserver -- \
            airflow connections add "${conn_id}" \
                --conn-type "${conn_type}" \
                --conn-host "${host}" \
                --conn-port "${port}" \
                --conn-extra "${extra}" \
            2>/dev/null || log "Connection ${conn_id} already exists"
    done
    
    log "Airflow connections created"
}

create_dag_template() {
    local dag_name="${1:-example_dag}"
    local schedule="${2:-@daily}"
    
    log "Creating DAG template: ${dag_name}"
    
    cat <<PYTHON > "/tmp/${dag_name}.py"
"""
${dag_name} - Auto-generated DAG template
"""
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.apache.kafka.operators.produce import ProduceToTopicOperator
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from airflow.utils.dates import days_ago
from airflow.utils.task_group import TaskGroup

default_args = {
    'owner': 'data-platform',
    'depends_on_past': False,
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
    'execution_timeout': timedelta(hours=2),
    'sla': timedelta(hours=4),
}

with DAG(
    dag_id='${dag_name}',
    default_args=default_args,
    description='${dag_name} pipeline',
    schedule='${schedule}',
    start_date=days_ago(1),
    catchup=False,
    max_active_runs=1,
    tags=['data-platform', 'production'],
    doc_md="""
    ## ${dag_name}
    
    Pipeline for processing data.
    
    ### Steps
    1. Extract data from source
    2. Transform with Spark
    3. Load to ClickHouse
    4. Update metrics
    """,
) as dag:
    
    def extract_data(**context):
        """Extract data from source systems"""
        execution_date = context['execution_date']
        print(f"Extracting data for: {execution_date}")
        
        # Push XCom for downstream tasks
        context['task_instance'].xcom_push(
            key='record_count',
            value=1000
        )
        return "extraction_complete"
    
    def validate_data(**context):
        """Validate extracted data quality"""
        record_count = context['task_instance'].xcom_pull(
            task_ids='extract_data',
            key='record_count'
        )
        
        if record_count == 0:
            raise ValueError("No records extracted!")
        
        print(f"Validated {record_count} records")
    
    with TaskGroup("extraction", tooltip="Data extraction tasks") as extraction_group:
        extract_task = PythonOperator(
            task_id='extract_data',
            python_callable=extract_data,
        )
        
        validate_task = PythonOperator(
            task_id='validate_data',
            python_callable=validate_data,
        )
        
        extract_task >> validate_task
    
    transform_task = SparkSubmitOperator(
        task_id='transform_with_spark',
        application='s3://spark-jobs/${dag_name}/transform.py',
        conn_id='spark_default',
        conf={
            'spark.sql.extensions': 'io.delta.sql.DeltaSparkSessionExtension',
            'spark.kubernetes.container.image': 'my-spark-delta:3.5.0',
        },
        application_args=[
            '--execution-date', '{{ ds }}',
            '--input-path', 's3://raw-data/{{ ds }}',
            '--output-path', 's3://processed-data/{{ ds }}',
        ],
        executor_cores=2,
        executor_memory='4g',
        num_executors=4,
    )
    
    load_task = SQLExecuteQueryOperator(
        task_id='load_to_clickhouse',
        conn_id='clickhouse_default',
        sql="""
            INSERT INTO analytics.events
            SELECT * FROM s3(
                's3://processed-data/{{ ds }}/*.parquet',
                'PARQUET'
            )
        """,
    )
    
    extraction_group >> transform_task >> load_task

PYTHON
    
    log "DAG template created: /tmp/${dag_name}.py"
}

monitor_airflow_health() {
    log "=== Airflow Health Status ==="
    
    kubectl get pods -n "${AIRFLOW_NAMESPACE}" \
        -o custom-columns="POD:.metadata.name,STATUS:.status.phase,READY:.status.containerStatuses[0].ready,RESTARTS:.status.containerStatuses[0].restartCount"
    
    log "=== Recent DAG Run Status ==="
    kubectl exec -n "${AIRFLOW_NAMESPACE}" \
        deployment/airflow-webserver -- \
        airflow dags list-runs \
        --output table \
        --no-backfill \
        2>/dev/null | head -30 || true
}

case "${1:-help}" in
    "install") install_airflow ;;
    "setup-connections") create_airflow_connections ;;
    "create-dag") create_dag_template "${2:-example_dag}" "${3:-@daily}" ;;
    "monitor") monitor_airflow_health ;;
    *) echo "Usage: $0 {install|setup-connections|create-dag|monitor}" ;;
esac
```

---

## สรุป Part 42

ในส่วนนี้เราได้เรียนรู้:

| ขั้นตอน | หัวข้อ | เครื่องมือหลัก |
|---------|--------|----------------|
| 521 | Apache Kafka Enterprise Platform | Strimzi Operator, KafkaTopic, KafkaUser, Kafka Connect |
| 522 | Apache Flink Stream Processing | Flink Operator, FlinkDeployment, Savepoints |
| 523 | Delta Lake Architecture | Spark Operator, SparkApplication, ACID Transactions |
| 524 | ClickHouse Analytics Database | ClickHouse Operator, Distributed Tables, Materialized Views |
| 525 | Apache Airflow Orchestration | KubernetesExecutor, DAG Templates, GitSync |

### ขั้นตอนต่อไป: Part 43 - FinOps and Cloud Cost Engineering
