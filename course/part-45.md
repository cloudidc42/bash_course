# Part 45: Event-Driven Architecture and Advanced Messaging

## Module 4: Professional Level — Event-Driven Systems

### ขั้นตอนที่ 534: AWS EventBridge Enterprise Integration

**EventBridge** สำหรับ event-driven integration ระดับ enterprise

```bash
#!/bin/bash
# eventbridge-manager.sh - AWS EventBridge Enterprise Integration

set -euo pipefail

AWS_REGION="${AWS_REGION:-ap-southeast-1}"
EVENT_BUS_NAME="${EVENT_BUS_NAME:-company-events}"
SCHEMA_REGISTRY="${SCHEMA_REGISTRY:-company-schema-registry}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

create_custom_event_bus() {
    local bus_name="${1:-${EVENT_BUS_NAME}}"
    
    log "Creating custom event bus: ${bus_name}"
    
    aws events create-event-bus \
        --name "${bus_name}" \
        --tags "Environment=production,ManagedBy=terraform,Team=platform" \
        --region "${AWS_REGION}"
    
    local bus_arn=$(aws events describe-event-bus \
        --name "${bus_name}" \
        --query 'Arn' \
        --output text)
    
    log "Setting resource policy for cross-account access..."
    aws events put-permission \
        --event-bus-name "${bus_name}" \
        --action "events:PutEvents" \
        --principal "*" \
        --statement-id "AllowCrossAccountPutEvents" \
        --condition "{\"Type\":\"StringEquals\",\"Key\":\"aws:PrincipalOrgID\",\"Value\":\"o-xxxxxxxxxx\"}"
    
    log "Event bus created: ${bus_arn}"
    echo "${bus_arn}"
}

create_event_rules() {
    log "Creating event rules for order processing domain..."
    
    declare -A rules
    rules["order-created"]='{"source":["company.orders"],"detail-type":["OrderCreated"]}'
    rules["payment-processed"]='{"source":["company.payments"],"detail-type":["PaymentProcessed","PaymentFailed"]}'
    rules["inventory-low"]='{"source":["company.inventory"],"detail-type":["InventoryLow"],"detail":{"quantity":[{"numeric":["<",10]}]}}'
    rules["user-activity"]='{"source":["company.users"],"detail-type":["UserRegistered","UserUpdated","UserDeleted"]}'
    
    for rule_name in "${!rules[@]}"; do
        local pattern="${rules[$rule_name]}"
        
        aws events put-rule \
            --name "${rule_name}" \
            --event-bus-name "${EVENT_BUS_NAME}" \
            --event-pattern "${pattern}" \
            --state ENABLED \
            --description "Rule for ${rule_name} events" \
            --region "${AWS_REGION}"
        
        log "Rule created: ${rule_name}"
    done
}

add_event_targets() {
    local rule_name="$1"
    local target_type="${2:-lambda}"
    local target_arn="$3"
    
    log "Adding target to rule: ${rule_name} -> ${target_type}"
    
    case "${target_type}" in
        "lambda")
            aws events put-targets \
                --rule "${rule_name}" \
                --event-bus-name "${EVENT_BUS_NAME}" \
                --targets "[{
                    \"Id\": \"lambda-target-$(date +%s)\",
                    \"Arn\": \"${target_arn}\",
                    \"RetryPolicy\": {
                        \"MaximumRetryAttempts\": 3,
                        \"MaximumEventAgeInSeconds\": 3600
                    },
                    \"DeadLetterConfig\": {
                        \"Arn\": \"arn:aws:sqs:${AWS_REGION}:${AWS_ACCOUNT_ID:-}:dlq-${rule_name}\"
                    }
                }]" \
                --region "${AWS_REGION}"
            ;;
        "sqs")
            aws events put-targets \
                --rule "${rule_name}" \
                --event-bus-name "${EVENT_BUS_NAME}" \
                --targets "[{
                    \"Id\": \"sqs-target-$(date +%s)\",
                    \"Arn\": \"${target_arn}\",
                    \"SqsParameters\": {
                        \"MessageGroupId\": \"${rule_name}\"
                    }
                }]" \
                --region "${AWS_REGION}"
            ;;
        "step-functions")
            aws events put-targets \
                --rule "${rule_name}" \
                --event-bus-name "${EVENT_BUS_NAME}" \
                --targets "[{
                    \"Id\": \"sfn-target-$(date +%s)\",
                    \"Arn\": \"${target_arn}\",
                    \"RoleArn\": \"arn:aws:iam::${AWS_ACCOUNT_ID:-}:role/EventBridgeStepFunctionsRole\",
                    \"InputTransformer\": {
                        \"InputPathsMap\": {
                            \"orderId\": \"$.detail.orderId\",
                            \"userId\": \"$.detail.userId\",
                            \"time\": \"$.time\"
                        },
                        \"InputTemplate\": \"{\\\"orderId\\\": <orderId>, \\\"userId\\\": <userId>, \\\"triggeredAt\\\": <time>}\"
                    }
                }]" \
                --region "${AWS_REGION}"
            ;;
    esac
    
    log "Target added to rule: ${rule_name}"
}

publish_event() {
    local source="$1"
    local detail_type="$2"
    local detail="$3"
    local event_bus="${4:-${EVENT_BUS_NAME}}"
    
    log "Publishing event: ${detail_type} from ${source}"
    
    local result=$(aws events put-events \
        --entries "[{
            \"Source\": \"${source}\",
            \"DetailType\": \"${detail_type}\",
            \"Detail\": $(echo "${detail}" | jq -c .),
            \"EventBusName\": \"${event_bus}\",
            \"Time\": \"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\"
        }]" \
        --region "${AWS_REGION}")
    
    local failed=$(echo "${result}" | jq -r '.FailedEntryCount')
    if [[ "${failed}" -gt 0 ]]; then
        log "ERROR: Event publishing failed"
        echo "${result}" | jq '.Entries[]'
        return 1
    fi
    
    log "Event published successfully"
    echo "${result}" | jq -r '.Entries[0].EventId'
}

create_event_archive() {
    local archive_name="${1:-${EVENT_BUS_NAME}-archive}"
    local retention_days="${2:-90}"
    
    log "Creating event archive: ${archive_name}"
    
    aws events create-archive \
        --archive-name "${archive_name}" \
        --event-source-arn "arn:aws:events:${AWS_REGION}:$(aws sts get-caller-identity --query Account --output text):event-bus/${EVENT_BUS_NAME}" \
        --event-pattern '{}' \
        --retention-days "${retention_days}" \
        --description "Archive of all ${EVENT_BUS_NAME} events" \
        --region "${AWS_REGION}"
    
    log "Event archive created: ${archive_name} (retention: ${retention_days} days)"
}

replay_events() {
    local archive_name="${1:-${EVENT_BUS_NAME}-archive}"
    local start_time="${2:-$(date -u -d '-1 hour' '+%Y-%m-%dT%H:%M:%SZ')}"
    local end_time="${3:-$(date -u '+%Y-%m-%dT%H:%M:%SZ')}"
    local replay_name="${4:-replay-$(date +%Y%m%d-%H%M%S)}"
    
    log "Replaying events from archive: ${archive_name}"
    
    aws events start-replay \
        --replay-name "${replay_name}" \
        --source-arn "arn:aws:events:${AWS_REGION}:$(aws sts get-caller-identity --query Account --output text):archive/${archive_name}" \
        --event-start-time "${start_time}" \
        --event-end-time "${end_time}" \
        --destination "{
            \"Arn\": \"arn:aws:events:${AWS_REGION}:$(aws sts get-caller-identity --query Account --output text):event-bus/${EVENT_BUS_NAME}\",
            \"FilterArns\": []
        }" \
        --region "${AWS_REGION}"
    
    log "Event replay started: ${replay_name}"
}

create_event_schemas() {
    log "Creating event schemas in Schema Registry..."
    
    aws schemas create-registry \
        --registry-name "${SCHEMA_REGISTRY}" \
        --description "Company event schemas" \
        --region "${AWS_REGION}" 2>/dev/null || true
    
    local order_schema='{
        "$schema": "http://json-schema.org/draft-04/schema#",
        "title": "OrderCreated",
        "description": "Event published when an order is created",
        "type": "object",
        "properties": {
            "orderId": {"type": "string", "description": "Unique order identifier"},
            "userId": {"type": "string", "description": "Customer user ID"},
            "totalAmount": {"type": "number", "description": "Order total in USD"},
            "currency": {"type": "string", "enum": ["USD", "EUR", "THB"]},
            "items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "productId": {"type": "string"},
                        "quantity": {"type": "integer"},
                        "price": {"type": "number"}
                    }
                }
            },
            "status": {"type": "string", "enum": ["pending", "confirmed", "cancelled"]},
            "createdAt": {"type": "string", "format": "date-time"}
        },
        "required": ["orderId", "userId", "totalAmount", "currency", "items", "status", "createdAt"]
    }'
    
    aws schemas create-schema \
        --registry-name "${SCHEMA_REGISTRY}" \
        --schema-name "company.orders@OrderCreated" \
        --type JSONSchemaDraft4 \
        --content "${order_schema}" \
        --description "Order created event schema" \
        --region "${AWS_REGION}"
    
    log "Event schemas created in registry: ${SCHEMA_REGISTRY}"
}

monitor_event_bus() {
    log "=== EventBridge Monitoring ==="
    
    local bus_arn=$(aws events describe-event-bus \
        --name "${EVENT_BUS_NAME}" \
        --query 'Arn' \
        --output text 2>/dev/null || echo "N/A")
    
    echo "Event Bus: ${EVENT_BUS_NAME}"
    echo "ARN: ${bus_arn}"
    echo ""
    
    echo "=== Rules ==="
    aws events list-rules \
        --event-bus-name "${EVENT_BUS_NAME}" \
        --query 'Rules[*].[Name,State,EventPattern]' \
        --output table
    
    echo ""
    echo "=== Event Metrics (last 1 hour) ==="
    local start_time=$(date -u -d '-1 hour' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ')
    local end_time=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
    
    aws cloudwatch get-metric-statistics \
        --namespace AWS/Events \
        --metric-name MatchedEvents \
        --dimensions "Name=EventBusName,Value=${EVENT_BUS_NAME}" \
        --start-time "${start_time}" \
        --end-time "${end_time}" \
        --period 3600 \
        --statistics Sum \
        --query 'Datapoints[*].[Timestamp,Sum]' \
        --output table
}

case "${1:-help}" in
    "create-bus") create_custom_event_bus "${2:-${EVENT_BUS_NAME}}" ;;
    "create-rules") create_event_rules ;;
    "add-target") add_event_targets "$2" "${3:-lambda}" "$4" ;;
    "publish") publish_event "$2" "$3" "$4" "${5:-${EVENT_BUS_NAME}}" ;;
    "create-archive") create_event_archive "${2:-}" "${3:-90}" ;;
    "replay") replay_events "$2" "${3:-}" "${4:-}" "${5:-}" ;;
    "create-schemas") create_event_schemas ;;
    "monitor") monitor_event_bus ;;
    *) echo "Usage: $0 {create-bus|create-rules|add-target|publish|create-archive|replay|create-schemas|monitor}" ;;
esac
```

### ขั้นตอนที่ 535: RabbitMQ Enterprise Messaging

**RabbitMQ** สำหรับ message queuing ที่ซับซ้อน

```bash
#!/bin/bash
# rabbitmq-enterprise-manager.sh - RabbitMQ Enterprise Messaging Platform

set -euo pipefail

RABBITMQ_NAMESPACE="${RABBITMQ_NAMESPACE:-rabbitmq}"
RABBITMQ_CLUSTER="${RABBITMQ_CLUSTER:-production-rabbitmq}"
RABBITMQ_OPERATOR_VERSION="${RABBITMQ_OPERATOR_VERSION:-2.7.0}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_rabbitmq_operator() {
    log "Installing RabbitMQ Cluster Operator..."
    
    kubectl create namespace "${RABBITMQ_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    kubectl apply -f "https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml"
    
    kubectl rollout status deployment/rabbitmq-cluster-operator \
        -n rabbitmq-system --timeout=120s
    
    log "RabbitMQ operator installed"
}

create_rabbitmq_cluster() {
    local replicas="${1:-3}"
    local storage_size="${2:-50Gi}"
    
    log "Creating RabbitMQ cluster: ${RABBITMQ_CLUSTER} (${replicas} replicas)"
    
    cat <<EOF | kubectl apply -f -
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: ${RABBITMQ_CLUSTER}
  namespace: ${RABBITMQ_NAMESPACE}
spec:
  replicas: ${replicas}
  image: rabbitmq:3.13-management
  
  rabbitmq:
    additionalPlugins:
      - rabbitmq_shovel
      - rabbitmq_shovel_management
      - rabbitmq_federation
      - rabbitmq_federation_management
      - rabbitmq_prometheus
      - rabbitmq_tracing
      - rabbitmq_delayed_message_exchange
    additionalConfig: |
      cluster_partition_handling = autoheal
      vm_memory_high_watermark.relative = 0.8
      disk_free_limit.relative = 2.0
      heartbeat = 60
      consumer_timeout = 3600000
      default_vhost = /
      default_user = admin
      default_permissions.configure = .*
      default_permissions.read = .*
      default_permissions.write = .*
      tcp_listen_options.backlog = 4096
      tcp_listen_options.keepalive = true
      channel_max = 2047
      max_message_size = 134217728
    envConfig: |
      RABBITMQ_ERLANG_COOKIE=secure-cookie-$(openssl rand -hex 16)
  
  service:
    type: ClusterIP
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "15692"
  
  persistence:
    storageClassName: fast-ssd
    storage: ${storage_size}
  
  resources:
    requests:
      cpu: "1"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "8Gi"
  
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values: [${RABBITMQ_CLUSTER}]
          topologyKey: kubernetes.io/hostname
  
  override:
    statefulSet:
      spec:
        template:
          spec:
            containers:
              - name: rabbitmq
                livenessProbe:
                  exec:
                    command: [rabbitmq-diagnostics, -q, ping]
                  initialDelaySeconds: 60
                  periodSeconds: 60
                  timeoutSeconds: 15
                readinessProbe:
                  exec:
                    command: [rabbitmq-diagnostics, -q, check_port_connectivity]
                  initialDelaySeconds: 20
                  periodSeconds: 60
                  timeoutSeconds: 10
EOF
    
    log "Waiting for RabbitMQ cluster..."
    kubectl wait rabbitmqcluster/${RABBITMQ_CLUSTER} \
        --for=condition=AllReplicasReady \
        --timeout=600s \
        -n "${RABBITMQ_NAMESPACE}"
    
    log "RabbitMQ cluster created: ${RABBITMQ_CLUSTER}"
}

configure_vhost_and_exchanges() {
    local vhost="${1:-production}"
    
    log "Configuring vhost and exchanges: ${vhost}"
    
    local mgmt_url="http://$(kubectl get service -n "${RABBITMQ_NAMESPACE}" \
        "${RABBITMQ_CLUSTER}" -o jsonpath='{.spec.clusterIP}'):15672"
    local auth=$(kubectl get secret -n "${RABBITMQ_NAMESPACE}" \
        "${RABBITMQ_CLUSTER}-default-user" \
        -o jsonpath='{.data.username}' | base64 -d):$(kubectl get secret -n "${RABBITMQ_NAMESPACE}" \
        "${RABBITMQ_CLUSTER}-default-user" \
        -o jsonpath='{.data.password}' | base64 -d)
    
    kubectl exec -n "${RABBITMQ_NAMESPACE}" "${RABBITMQ_CLUSTER}-server-0" -- \
        rabbitmqadmin -u "$(echo ${auth} | cut -d: -f1)" -p "$(echo ${auth} | cut -d: -f2)" \
        declare vhost name="${vhost}" 2>/dev/null || true
    
    local exchanges=(
        "orders|direct|true"
        "payments|topic|true"
        "notifications|fanout|true"
        "events.delayed|x-delayed-message|true"
        "dlx.exchange|direct|true"
    )
    
    for exchange_config in "${exchanges[@]}"; do
        IFS='|' read -r name type durable <<< "${exchange_config}"
        
        kubectl exec -n "${RABBITMQ_NAMESPACE}" "${RABBITMQ_CLUSTER}-server-0" -- \
            rabbitmqadmin \
            -u "$(echo ${auth} | cut -d: -f1)" \
            -p "$(echo ${auth} | cut -d: -f2)" \
            -V "${vhost}" \
            declare exchange \
            name="${name}" \
            type="${type}" \
            durable="${durable}" 2>/dev/null || true
        
        log "Exchange created: ${name} (${type})"
    done
    
    local queues=(
        "orders.created:3:false"
        "orders.fulfilled:3:false"
        "payments.processed:3:false"
        "notifications.email:10:false"
        "notifications.sms:10:false"
        "dlq.orders:3:true"
    )
    
    for queue_config in "${queues[@]}"; do
        IFS=':' read -r name ttl_hours is_dlq <<< "${queue_config}"
        local ttl_ms=$(( ttl_hours * 3600000 ))
        
        kubectl exec -n "${RABBITMQ_NAMESPACE}" "${RABBITMQ_CLUSTER}-server-0" -- \
            rabbitmqadmin \
            -u "$(echo ${auth} | cut -d: -f1)" \
            -p "$(echo ${auth} | cut -d: -f2)" \
            -V "${vhost}" \
            declare queue \
            name="${name}" \
            durable=true \
            auto_delete=false \
            arguments="{\"x-message-ttl\": ${ttl_ms}, \"x-dead-letter-exchange\": \"dlx.exchange\", \"x-queue-type\": \"quorum\"}" 2>/dev/null || true
        
        log "Queue created: ${name}"
    done
    
    log "VHost ${vhost} configured with exchanges and queues"
}

setup_federation() {
    local upstream_uri="${1:-amqp://user:pass@upstream-rabbitmq:5672}"
    local federation_name="${2:-upstream-cluster}"
    
    log "Setting up RabbitMQ federation: ${federation_name}"
    
    kubectl exec -n "${RABBITMQ_NAMESPACE}" "${RABBITMQ_CLUSTER}-server-0" -- \
        rabbitmqctl set_parameter federation-upstream "${federation_name}" \
        "{\"uri\": \"${upstream_uri}\", \"ack-mode\": \"on-confirm\", \"trust-user-id\": false, \"max-hops\": 1}"
    
    kubectl exec -n "${RABBITMQ_NAMESPACE}" "${RABBITMQ_CLUSTER}-server-0" -- \
        rabbitmqctl set_policy federated-exchanges \
        "^federated\." \
        '{"federation-upstream-set":"all"}' \
        --apply-to exchanges
    
    log "Federation configured: ${federation_name}"
}

monitor_rabbitmq() {
    log "=== RabbitMQ Cluster Status ==="
    
    kubectl exec -n "${RABBITMQ_NAMESPACE}" "${RABBITMQ_CLUSTER}-server-0" -- \
        rabbitmq-diagnostics -q cluster_status 2>/dev/null || \
        kubectl get pods -n "${RABBITMQ_NAMESPACE}" -l "app.kubernetes.io/name=${RABBITMQ_CLUSTER}"
    
    log "=== Queue Statistics ==="
    kubectl exec -n "${RABBITMQ_NAMESPACE}" "${RABBITMQ_CLUSTER}-server-0" -- \
        rabbitmqadmin list queues \
        name,messages,messages_ready,messages_unacknowledged,consumers \
        2>/dev/null | head -30
}

case "${1:-help}" in
    "install") install_rabbitmq_operator ;;
    "create-cluster") create_rabbitmq_cluster "${2:-3}" "${3:-50Gi}" ;;
    "configure-vhost") configure_vhost_and_exchanges "${2:-production}" ;;
    "setup-federation") setup_federation "$2" "${3:-upstream}" ;;
    "monitor") monitor_rabbitmq ;;
    *) echo "Usage: $0 {install|create-cluster|configure-vhost|setup-federation|monitor}" ;;
esac
```

### ขั้นตอนที่ 536: NATS JetStream for Cloud-Native Messaging

**NATS JetStream** สำหรับ high-performance cloud-native messaging

```bash
#!/bin/bash
# nats-jetstream-manager.sh - NATS JetStream Platform Management

set -euo pipefail

NATS_NAMESPACE="${NATS_NAMESPACE:-nats}"
NATS_CLUSTER_NAME="${NATS_CLUSTER_NAME:-nats}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

deploy_nats_cluster() {
    local replicas="${1:-3}"
    
    log "Deploying NATS JetStream cluster: ${replicas} nodes"
    
    helm repo add nats https://nats-io.github.io/k8s/helm/charts/
    helm repo update
    
    kubectl create namespace "${NATS_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF > /tmp/nats-values.yaml
cluster:
  enabled: true
  replicas: ${replicas}
  name: ${NATS_CLUSTER_NAME}

nats:
  image: nats:2.10-alpine
  
  jetstream:
    enabled: true
    fileStorage:
      enabled: true
      size: 100Gi
      storageClassName: fast-ssd
    memStorage:
      enabled: true
      size: 2Gi
  
  config:
    cluster:
      name: ${NATS_CLUSTER_NAME}
    jetstream:
      max_memory_store: 2Gi
      max_file_store: 100Gi
    accounts:
      A:
        jetstream: enabled
        users: [{user: app, password: apppassword}]
      SYS:
        users: [{user: sys, password: syspassword}]
    system_account: SYS
    websocket:
      port: 8080
      compression: true
    mqtt:
      port: 1883
    max_connections: 65536
    max_payload: 10MB
    max_pending: 128MB
    write_deadline: "10s"
    lame_duck_duration: "30s"
  
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"

exporter:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring

natsbox:
  enabled: true
  
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values: [nats]
        topologyKey: kubernetes.io/hostname
EOF
    
    helm upgrade --install "${NATS_CLUSTER_NAME}" nats/nats \
        --namespace "${NATS_NAMESPACE}" \
        -f /tmp/nats-values.yaml \
        --wait
    
    log "NATS JetStream cluster deployed"
}

create_jetstream_streams() {
    log "Creating JetStream streams..."
    
    local nats_server="nats://${NATS_CLUSTER_NAME}.${NATS_NAMESPACE}:4222"
    
    declare -A streams
    streams["ORDERS"]="orders.> --max-msgs=10000000 --max-bytes=10GB --max-age=7d --storage=file --replicas=3 --retention=limits --discard=old"
    streams["EVENTS"]="events.> --max-msgs=50000000 --max-bytes=50GB --max-age=30d --storage=file --replicas=3 --retention=limits"
    streams["COMMANDS"]="commands.> --max-msgs=1000000 --max-bytes=1GB --max-age=1h --storage=memory --replicas=3 --retention=workqueue"
    
    for stream_name in "${!streams[@]}"; do
        local subjects=$(echo "${streams[$stream_name]}" | awk '{print $1}')
        local args="${streams[$stream_name]}"
        
        kubectl exec -n "${NATS_NAMESPACE}" \
            "$(kubectl get pod -n "${NATS_NAMESPACE}" -l "app.kubernetes.io/name=nats" -o name | head -1 | cut -d/ -f2)" -- \
            nats stream add "${stream_name}" \
            --server "${nats_server}" \
            --subjects "${subjects}" \
            --defaults 2>/dev/null || \
            log "Stream ${stream_name} may already exist"
        
        log "Stream configured: ${stream_name}"
    done
    
    log "JetStream streams created"
}

create_consumers() {
    local stream_name="${1:-ORDERS}"
    local consumer_name="${2:-order-processor}"
    local filter_subject="${3:-orders.created}"
    
    log "Creating JetStream consumer: ${consumer_name} on ${stream_name}"
    
    kubectl exec -n "${NATS_NAMESPACE}" \
        "$(kubectl get pod -n "${NATS_NAMESPACE}" -l "app.kubernetes.io/name=nats" -o name | head -1 | cut -d/ -f2)" -- \
        nats consumer add "${stream_name}" "${consumer_name}" \
        --filter "${filter_subject}" \
        --ack=explicit \
        --deliver=all \
        --max-deliver=5 \
        --max-ack-pending=1000 \
        --wait=30s \
        --defaults 2>/dev/null || true
    
    log "Consumer created: ${consumer_name}"
}

publish_messages() {
    local subject="${1:-orders.created}"
    local count="${2:-100}"
    
    log "Publishing ${count} messages to: ${subject}"
    
    for i in $(seq 1 "${count}"); do
        local payload="{\"orderId\":\"order-${i}\",\"userId\":\"user-${RANDOM}\",\"amount\":$(( RANDOM % 1000 + 10 )).99,\"timestamp\":\"$(date -u '+%Y-%m-%dT%H:%M:%SZ')\"}"
        
        kubectl exec -n "${NATS_NAMESPACE}" \
            "$(kubectl get pod -n "${NATS_NAMESPACE}" -l "app.kubernetes.io/name=nats" -o name | head -1 | cut -d/ -f2)" -- \
            nats pub "${subject}" "${payload}" --count=1 2>/dev/null || true
    done
    
    log "Published ${count} messages to ${subject}"
}

monitor_nats() {
    log "=== NATS JetStream Status ==="
    
    kubectl exec -n "${NATS_NAMESPACE}" \
        "$(kubectl get pod -n "${NATS_NAMESPACE}" -l "app.kubernetes.io/name=nats" -o name | head -1 | cut -d/ -f2)" -- \
        nats server report jetstream 2>/dev/null || \
        kubectl get pods -n "${NATS_NAMESPACE}"
    
    log "=== Stream Information ==="
    kubectl exec -n "${NATS_NAMESPACE}" \
        "$(kubectl get pod -n "${NATS_NAMESPACE}" -l "app.kubernetes.io/name=nats" -o name | head -1 | cut -d/ -f2)" -- \
        nats stream ls 2>/dev/null || true
}

case "${1:-help}" in
    "deploy") deploy_nats_cluster "${2:-3}" ;;
    "create-streams") create_jetstream_streams ;;
    "create-consumer") create_consumers "${2:-ORDERS}" "${3:-processor}" "${4:-orders.created}" ;;
    "publish") publish_messages "${2:-orders.created}" "${3:-100}" ;;
    "monitor") monitor_nats ;;
    *) echo "Usage: $0 {deploy|create-streams|create-consumer|publish|monitor}" ;;
esac
```

### ขั้นตอนที่ 537: AWS SQS/SNS Enterprise Patterns

**SQS/SNS** patterns สำหรับ microservices communication ระดับ enterprise

```bash
#!/bin/bash
# aws-messaging-patterns.sh - AWS SQS/SNS Enterprise Messaging Patterns

set -euo pipefail

AWS_REGION="${AWS_REGION:-ap-southeast-1}"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text 2>/dev/null || echo "123456789")

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

create_sns_topic() {
    local topic_name="$1"
    local fifo="${2:-false}"
    
    log "Creating SNS topic: ${topic_name}"
    
    local extra_args=""
    if [[ "${fifo}" == "true" ]]; then
        topic_name="${topic_name}.fifo"
        extra_args="--attributes ContentBasedDeduplication=true"
    fi
    
    local topic_arn=$(aws sns create-topic \
        --name "${topic_name}" \
        ${extra_args} \
        --tags "[{\"Key\":\"Environment\",\"Value\":\"production\"},{\"Key\":\"ManagedBy\",\"Value\":\"automation\"}]" \
        --query 'TopicArn' \
        --output text \
        --region "${AWS_REGION}")
    
    aws sns set-topic-attributes \
        --topic-arn "${topic_arn}" \
        --attribute-name DisplayName \
        --attribute-value "${topic_name}" \
        --region "${AWS_REGION}"
    
    log "SNS topic created: ${topic_arn}"
    echo "${topic_arn}"
}

create_sqs_queue() {
    local queue_name="$1"
    local visibility_timeout="${2:-300}"
    local retention_period="${3:-1209600}"
    local fifo="${4:-false}"
    local dlq_arn="${5:-}"
    
    log "Creating SQS queue: ${queue_name}"
    
    local extra_attrs=""
    if [[ "${fifo}" == "true" ]]; then
        queue_name="${queue_name}.fifo"
        extra_attrs='"FifoQueue":"true","ContentBasedDeduplication":"true",'
    fi
    
    local redrive_policy=""
    if [[ -n "${dlq_arn}" ]]; then
        redrive_policy=",\"RedrivePolicy\":\"{\\\"deadLetterTargetArn\\\":\\\"${dlq_arn}\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\""
    fi
    
    local queue_url=$(aws sqs create-queue \
        --queue-name "${queue_name}" \
        --attributes "{${extra_attrs}\"VisibilityTimeout\":\"${visibility_timeout}\",\"MessageRetentionPeriod\":\"${retention_period}\",\"ReceiveMessageWaitTimeSeconds\":\"20\"${redrive_policy}}" \
        --tags "Environment=production,ManagedBy=automation" \
        --query 'QueueUrl' \
        --output text \
        --region "${AWS_REGION}")
    
    local queue_arn=$(aws sqs get-queue-attributes \
        --queue-url "${queue_url}" \
        --attribute-names QueueArn \
        --query 'Attributes.QueueArn' \
        --output text \
        --region "${AWS_REGION}")
    
    log "SQS queue created: ${queue_url}"
    echo "${queue_arn}"
}

setup_fanout_pattern() {
    local topic_name="${1:-notifications}"
    local subscriber_services=("${@:2}")
    
    if [[ ${#subscriber_services[@]} -eq 0 ]]; then
        subscriber_services=("email-service" "sms-service" "push-service")
    fi
    
    log "Setting up SNS fanout pattern: ${topic_name} -> ${subscriber_services[*]}"
    
    local topic_arn=$(create_sns_topic "${topic_name}")
    
    for service in "${subscriber_services[@]}"; do
        local dlq_arn=$(create_sqs_queue "${service}-dlq" 300 1209600)
        local queue_arn=$(create_sqs_queue "${service}-queue" 300 1209600 false "${dlq_arn}")
        local queue_url="https://sqs.${AWS_REGION}.amazonaws.com/${AWS_ACCOUNT_ID}/${service}-queue"
        
        aws sqs set-queue-attributes \
            --queue-url "${queue_url}" \
            --attributes "{
                \"Policy\": \"{\\\"Version\\\":\\\"2012-10-17\\\",\\\"Statement\\\":[{\\\"Effect\\\":\\\"Allow\\\",\\\"Principal\\\":{\\\"Service\\\":\\\"sns.amazonaws.com\\\"},\\\"Action\\\":\\\"sqs:SendMessage\\\",\\\"Resource\\\":\\\"${queue_arn}\\\",\\\"Condition\\\":{\\\"ArnEquals\\\":{\\\"aws:SourceArn\\\":\\\"${topic_arn}\\\"}}}]}\"
            }" \
            --region "${AWS_REGION}" 2>/dev/null || true
        
        aws sns subscribe \
            --topic-arn "${topic_arn}" \
            --protocol sqs \
            --notification-endpoint "${queue_arn}" \
            --attributes '{"RawMessageDelivery":"true","FilterPolicy":"{\"event_type\":[\"'+${service}+'\"]}"}' \
            --region "${AWS_REGION}"
        
        log "Subscriber added: ${service} -> ${queue_arn}"
    done
    
    log "Fanout pattern configured: 1 topic, ${#subscriber_services[@]} subscribers"
}

setup_saga_pattern() {
    local saga_name="${1:-order-saga}"
    
    log "Setting up SAGA pattern for: ${saga_name}"
    
    local services=("orders" "inventory" "payments" "shipping")
    
    for service in "${services[@]}"; do
        local dlq_arn=$(create_sqs_queue "${saga_name}-${service}-dlq" 300 604800)
        local queue_arn=$(create_sqs_queue "${saga_name}-${service}" 120 86400 false "${dlq_arn}")
        log "SAGA queue created: ${saga_name}-${service}"
    done
    
    local orchestrator_topic=$(create_sns_topic "${saga_name}-orchestrator")
    log "SAGA orchestrator topic: ${orchestrator_topic}"
    
    cat <<'PYTHON' > /tmp/saga_state_machine.py
"""
SAGA Orchestrator State Machine
Manages distributed transactions with compensating transactions
"""
import json
import boto3
import time

class SagaOrchestrator:
    def __init__(self, saga_name: str, region: str = 'ap-southeast-1'):
        self.saga_name = saga_name
        self.sqs = boto3.client('sqs', region_name=region)
        self.sns = boto3.client('sns', region_name=region)
        self.dynamodb = boto3.resource('dynamodb', region_name=region)
        self.state_table = self.dynamodb.Table(f'{saga_name}-state')
    
    def start_saga(self, transaction_id: str, payload: dict) -> dict:
        """Start a new SAGA transaction"""
        state = {
            'transaction_id': transaction_id,
            'status': 'STARTED',
            'payload': payload,
            'completed_steps': [],
            'started_at': int(time.time()),
            'current_step': 'inventory_reserve'
        }
        
        # Save initial state
        self.state_table.put_item(Item=state)
        
        # Send to first step
        self._send_to_service('inventory', {
            'action': 'reserve',
            'transaction_id': transaction_id,
            'data': payload
        })
        
        return state
    
    def handle_step_result(self, step: str, transaction_id: str, success: bool, result: dict):
        """Handle result from a SAGA step"""
        state = self.state_table.get_item(
            Key={'transaction_id': transaction_id}
        )['Item']
        
        if success:
            state['completed_steps'].append(step)
            next_step = self._get_next_step(step)
            
            if next_step:
                state['current_step'] = next_step
                self._send_to_service(next_step.split('_')[0], {
                    'action': next_step.split('_')[1],
                    'transaction_id': transaction_id,
                    'data': state['payload']
                })
            else:
                state['status'] = 'COMPLETED'
        else:
            state['status'] = 'COMPENSATING'
            self._compensate(state)
        
        self.state_table.put_item(Item=state)
    
    def _get_next_step(self, current_step: str) -> str:
        steps = [
            'inventory_reserve',
            'payment_charge',
            'order_confirm',
            'shipping_schedule'
        ]
        try:
            idx = steps.index(current_step)
            return steps[idx + 1] if idx + 1 < len(steps) else None
        except ValueError:
            return None
    
    def _compensate(self, state: dict):
        """Execute compensating transactions in reverse order"""
        compensation_map = {
            'inventory_reserve': 'inventory_release',
            'payment_charge': 'payment_refund',
            'order_confirm': 'order_cancel',
        }
        
        for completed_step in reversed(state['completed_steps']):
            if completed_step in compensation_map:
                compensating_action = compensation_map[completed_step]
                service = compensating_action.split('_')[0]
                self._send_to_service(service, {
                    'action': compensating_action.split('_')[1],
                    'transaction_id': state['transaction_id'],
                    'data': state['payload'],
                    'is_compensation': True
                })
    
    def _send_to_service(self, service: str, message: dict):
        """Send message to service queue"""
        queue_url = f"https://sqs.ap-southeast-1.amazonaws.com/ACCOUNT/{self.saga_name}-{service}"
        self.sqs.send_message(
            QueueUrl=queue_url,
            MessageBody=json.dumps(message)
        )

if __name__ == '__main__':
    orchestrator = SagaOrchestrator('order-saga')
    result = orchestrator.start_saga(
        'txn-001',
        {'user_id': 'user-123', 'order_id': 'ord-456', 'amount': 99.99}
    )
    print(f"SAGA started: {result['transaction_id']}")
PYTHON
    
    log "SAGA pattern configuration complete"
    log "State machine saved: /tmp/saga_state_machine.py"
}

monitor_messaging() {
    log "=== SQS Queue Metrics ==="
    
    aws sqs list-queues \
        --queue-name-prefix "" \
        --query 'QueueUrls[]' \
        --output text \
        --region "${AWS_REGION}" | tr '\t' '\n' | while read queue_url; do
        
        local queue_name=$(echo "${queue_url}" | awk -F'/' '{print $NF}')
        local attrs=$(aws sqs get-queue-attributes \
            --queue-url "${queue_url}" \
            --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible \
            --query 'Attributes' \
            --output json \
            --region "${AWS_REGION}" 2>/dev/null)
        
        local messages=$(echo "${attrs}" | jq -r '.ApproximateNumberOfMessages // "0"')
        local in_flight=$(echo "${attrs}" | jq -r '.ApproximateNumberOfMessagesNotVisible // "0"')
        
        printf "%-50s Messages: %6s  In-Flight: %6s\n" \
            "${queue_name}" "${messages}" "${in_flight}"
    done
}

case "${1:-help}" in
    "create-topic") create_sns_topic "$2" "${3:-false}" ;;
    "create-queue") create_sqs_queue "$2" "${3:-300}" "${4:-1209600}" "${5:-false}" "${6:-}" ;;
    "fanout") setup_fanout_pattern "$2" "${@:3}" ;;
    "saga") setup_saga_pattern "${2:-order-saga}" ;;
    "monitor") monitor_messaging ;;
    *) echo "Usage: $0 {create-topic|create-queue|fanout|saga|monitor}" ;;
esac
```

---

## สรุป Part 45

ในส่วนนี้เราได้เรียนรู้:

| ขั้นตอน | หัวข้อ | เครื่องมือหลัก |
|---------|--------|----------------|
| 534 | AWS EventBridge Enterprise | Custom Event Bus, Rules, Targets, Archive, Schema Registry |
| 535 | RabbitMQ Enterprise Messaging | Cluster Operator, VHost, Exchanges, Queues, Federation |
| 536 | NATS JetStream | JetStream Streams, Consumers, Cloud-Native Messaging |
| 537 | AWS SQS/SNS Patterns | Fanout Pattern, SAGA Orchestrator, Dead Letter Queues |

### ขั้นตอนต่อไป: Part 46 - Database Engineering at Scale
