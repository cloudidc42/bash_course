# Part 43: FinOps and Cloud Cost Engineering

## Module 4: Professional Level — Cloud Financial Management

### ขั้นตอนที่ 526: Cloud Cost Analysis and Reporting

**FinOps** คือการบริหารจัดการค่าใช้จ่าย Cloud แบบ professional เพื่อ optimize ต้นทุนและ ROI

```bash
#!/bin/bash
# finops-cost-analyzer.sh - Cloud Cost Analysis and FinOps Dashboard

set -euo pipefail

AWS_REGION="${AWS_REGION:-ap-southeast-1}"
REPORT_BUCKET="${REPORT_BUCKET:-finops-reports}"
COST_THRESHOLD_DAILY="${COST_THRESHOLD_DAILY:-1000}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
format_cost() { printf "%.2f" "$1"; }

get_daily_cost_breakdown() {
    local start_date="${1:-$(date -d '-7 days' '+%Y-%m-%d')}"
    local end_date="${2:-$(date '+%Y-%m-%d')}"
    
    log "Fetching cost breakdown: ${start_date} to ${end_date}"
    
    aws ce get-cost-and-usage \
        --time-period "Start=${start_date},End=${end_date}" \
        --granularity DAILY \
        --metrics "BlendedCost" "UnblendedCost" "UsageQuantity" \
        --group-by "[{\"Type\":\"DIMENSION\",\"Key\":\"SERVICE\"},{\"Type\":\"DIMENSION\",\"Key\":\"REGION\"}]" \
        --filter '{"Dimensions":{"Key":"RECORD_TYPE","Values":["Usage"]}}' \
        --output json | \
        jq -r '
            .ResultsByTime[] | 
            .TimePeriod.Start as $date |
            .Groups[] | 
            . as $group |
            [$date, $group.Keys[0], $group.Keys[1], $group.Metrics.BlendedCost.Amount, $group.Metrics.BlendedCost.Unit] |
            @tsv
        ' | \
        awk -F'\t' '{
            printf "%-12s | %-40s | %-20s | $%8.4f %s\n", $1, $2, $3, $4, $5
        }'
}

get_cost_by_tag() {
    local tag_key="${1:-Environment}"
    local start_date="${2:-$(date -d '-30 days' '+%Y-%m-%d')}"
    local end_date="${3:-$(date '+%Y-%m-%d')}"
    
    log "Cost analysis by tag: ${tag_key}"
    
    aws ce get-cost-and-usage \
        --time-period "Start=${start_date},End=${end_date}" \
        --granularity MONTHLY \
        --metrics "BlendedCost" \
        --group-by "[{\"Type\":\"TAG\",\"Key\":\"${tag_key}\"}]" \
        --output json | \
        jq -r '
            .ResultsByTime[] |
            .TimePeriod.Start as $period |
            .Groups[] |
            [$period, .Keys[0], .Metrics.BlendedCost.Amount] |
            @tsv
        ' | \
        sort -t$'\t' -k3 -rn | \
        awk -F'\t' 'BEGIN{print "Period\t\tTag Value\t\t\tCost"} {printf "%s\t%-30s\t$%.2f\n", $1, $2, $3}'
}

analyze_cost_anomalies() {
    log "Detecting cost anomalies..."
    
    aws ce get-anomalies \
        --date-interval "StartDate=$(date -d '-30 days' '+%Y-%m-%d'),EndDate=$(date '+%Y-%m-%d')" \
        --max-results 20 \
        --output json | \
        jq -r '
            .Anomalies[] |
            {
                id: .AnomalyId,
                service: (.RootCauses[0].Service // "Unknown"),
                start: .AnomalyStartDate,
                end: (.AnomalyEndDate // "Ongoing"),
                expected: .Impact.ExpectedSpend,
                actual: .Impact.TotalActualSpend,
                excess: .Impact.TotalImpact
            } |
            "\(.start) | \(.service) | Expected: $\(.expected | tonumber | floor) | Actual: $\(.actual | tonumber | floor) | Excess: $\(.excess | tonumber | floor)"
        '
}

generate_savings_recommendations() {
    log "Generating cost savings recommendations..."
    
    echo "=== EC2 Reserved Instance Recommendations ==="
    aws ce get-reservation-purchase-recommendation \
        --service "Amazon EC2" \
        --lookback-period-in-days SIXTY_DAYS \
        --term-in-years ONE_YEAR \
        --payment-option NO_UPFRONT \
        --output json | \
        jq -r '
            .Recommendations[] |
            .RecommendationDetails[] |
            "Instance: \(.InstanceDetails.EC2InstanceDetails.InstanceType) | "
            + "Region: \(.InstanceDetails.EC2InstanceDetails.Region) | "
            + "Estimated Monthly Savings: $\(.EstimatedMonthlySavingsAmount)"
        ' | head -20
    
    echo ""
    echo "=== Compute Savings Plans Recommendations ==="
    aws ce get-savings-plans-purchase-recommendation \
        --savings-plans-type COMPUTE_SP \
        --term-in-years ONE_YEAR \
        --payment-option NO_UPFRONT \
        --lookback-period-in-days SIXTY_DAYS \
        --output json | \
        jq -r '
            .SavingsPlansPurchaseRecommendation |
            "Recommended Commitment: $\(.SavingsPlansDetails.HourlyCommitment)/hr | "
            + "Estimated Monthly Savings: $\(.SavingsPlansPurchaseRecommendationSummary.EstimatedMonthlySavingsAmount)"
        '
    
    echo ""
    echo "=== Right-sizing Recommendations ==="
    aws compute-optimizer get-ec2-instance-recommendations \
        --filters name=Finding,values=OVER_PROVISIONED \
        --output json | \
        jq -r '
            .instanceRecommendations[] |
            "Instance: \(.instanceId) | Current: \(.currentInstanceType) | "
            + "Recommended: \(.recommendationOptions[0].instanceType) | "
            + "Savings: $\(.recommendationOptions[0].estimatedMonthlySavings.value | tostring)/mo"
        ' | head -20
}

identify_idle_resources() {
    log "Identifying idle/unused resources..."
    
    echo "=== Underutilized EC2 Instances ==="
    aws cloudwatch get-metric-statistics \
        --namespace AWS/EC2 \
        --metric-name CPUUtilization \
        --start-time "$(date -u -d '-7 days' '+%Y-%m-%dT%H:%M:%SZ')" \
        --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
        --period 86400 \
        --statistics Average \
        --dimensions Name=InstanceId,Value=PLACEHOLDER \
        --query 'Datapoints[?Average<`10`].[Timestamp,Average]' \
        --output table 2>/dev/null || echo "Need specific instance ID"
    
    echo ""
    echo "=== Unattached EBS Volumes ==="
    aws ec2 describe-volumes \
        --filters Name=status,Values=available \
        --query 'Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType,Created:CreateTime,AZ:AvailabilityZone}' \
        --output table
    
    echo ""
    echo "=== Unused Elastic IPs ==="
    aws ec2 describe-addresses \
        --filters Name=domain,Values=vpc \
        --query 'Addresses[?AssociationId==null].{IP:PublicIp,AllocationId:AllocationId}' \
        --output table
    
    echo ""
    echo "=== Old EBS Snapshots (>90 days) ==="
    aws ec2 describe-snapshots \
        --owner-ids self \
        --query "Snapshots[?StartTime<='$(date -d '-90 days' '+%Y-%m-%d')'].{ID:SnapshotId,Size:VolumeSize,Date:StartTime,Desc:Description}" \
        --output table | head -30
    
    echo ""
    echo "=== Unused Load Balancers ==="
    aws elbv2 describe-load-balancers \
        --query 'LoadBalancers[*].{Name:LoadBalancerName,DNS:DNSName,State:State.Code,Created:CreatedTime}' \
        --output table
}

create_cost_budget() {
    local budget_name="${1:-monthly-budget}"
    local amount="${2:-5000}"
    local email="${3:-finops@company.com}"
    
    log "Creating cost budget: ${budget_name} = $${amount}/month"
    
    local account_id=$(aws sts get-caller-identity --query 'Account' --output text)
    
    aws budgets create-budget \
        --account-id "${account_id}" \
        --budget "{
            \"BudgetName\": \"${budget_name}\",
            \"BudgetLimit\": {
                \"Amount\": \"${amount}\",
                \"Unit\": \"USD\"
            },
            \"TimeUnit\": \"MONTHLY\",
            \"BudgetType\": \"COST\",
            \"CostFilters\": {},
            \"CostTypes\": {
                \"IncludeTax\": true,
                \"IncludeSubscription\": true,
                \"UseBlended\": false,
                \"IncludeRefund\": false,
                \"IncludeCredit\": false,
                \"IncludeUpfront\": true,
                \"IncludeRecurring\": true,
                \"IncludeOtherSubscription\": true,
                \"IncludeSupport\": true,
                \"IncludeDiscount\": true,
                \"UseAmortized\": false
            }
        }" \
        --notifications-with-subscribers "[
            {
                \"Notification\": {
                    \"NotificationType\": \"ACTUAL\",
                    \"ComparisonOperator\": \"GREATER_THAN\",
                    \"Threshold\": 80,
                    \"ThresholdType\": \"PERCENTAGE\"
                },
                \"Subscribers\": [
                    {\"SubscriptionType\": \"EMAIL\", \"Address\": \"${email}\"}
                ]
            },
            {
                \"Notification\": {
                    \"NotificationType\": \"ACTUAL\",
                    \"ComparisonOperator\": \"GREATER_THAN\",
                    \"Threshold\": 100,
                    \"ThresholdType\": \"PERCENTAGE\"
                },
                \"Subscribers\": [
                    {\"SubscriptionType\": \"EMAIL\", \"Address\": \"${email}\"}
                ]
            },
            {
                \"Notification\": {
                    \"NotificationType\": \"FORECASTED\",
                    \"ComparisonOperator\": \"GREATER_THAN\",
                    \"Threshold\": 110,
                    \"ThresholdType\": \"PERCENTAGE\"
                },
                \"Subscribers\": [
                    {\"SubscriptionType\": \"EMAIL\", \"Address\": \"${email}\"}
                ]
            }
        ]"
    
    log "Budget created: ${budget_name}"
}

generate_finops_report() {
    local output_file="${1:-finops-report-$(date '+%Y%m').json}"
    
    log "Generating FinOps report..."
    
    local start_date=$(date -d 'first day of this month' '+%Y-%m-%d' 2>/dev/null || date '+%Y-%m-01')
    local end_date=$(date '+%Y-%m-%d')
    
    local total_cost=$(aws ce get-cost-and-usage \
        --time-period "Start=${start_date},End=${end_date}" \
        --granularity MONTHLY \
        --metrics "BlendedCost" \
        --output json | \
        jq -r '.ResultsByTime[0].Total.BlendedCost.Amount // "0"')
    
    cat <<JSON > "${output_file}"
{
    "report_date": "$(date '+%Y-%m-%d')",
    "period": "${start_date} to ${end_date}",
    "summary": {
        "total_cost_mtd": ${total_cost},
        "currency": "USD"
    },
    "optimization_score": {
        "overall": "72/100",
        "reserved_instances": "45%",
        "savings_plans_coverage": "38%",
        "rightsizing": "pending_review"
    },
    "action_items": [
        {
            "priority": "HIGH",
            "category": "Reserved Instances",
            "action": "Purchase 1-year RIs for baseline EC2 usage",
            "estimated_savings": "30%"
        },
        {
            "priority": "HIGH",
            "category": "Rightsizing",
            "action": "Downsize over-provisioned instances",
            "estimated_savings": "15%"
        },
        {
            "priority": "MEDIUM",
            "category": "Storage",
            "action": "Delete unattached EBS volumes and old snapshots",
            "estimated_savings": "5%"
        },
        {
            "priority": "LOW",
            "category": "Data Transfer",
            "action": "Implement VPC endpoints for S3/DynamoDB",
            "estimated_savings": "3%"
        }
    ]
}
JSON
    
    log "FinOps report saved: ${output_file}"
    cat "${output_file}"
}

case "${1:-help}" in
    "daily-cost") get_daily_cost_breakdown "${2:-}" "${3:-}" ;;
    "cost-by-tag") get_cost_by_tag "${2:-Environment}" "${3:-}" "${4:-}" ;;
    "anomalies") analyze_cost_anomalies ;;
    "recommendations") generate_savings_recommendations ;;
    "idle-resources") identify_idle_resources ;;
    "create-budget") create_cost_budget "${2:-monthly-budget}" "${3:-5000}" "${4:-finops@company.com}" ;;
    "report") generate_finops_report "${2:-}" ;;
    *) echo "Usage: $0 {daily-cost|cost-by-tag|anomalies|recommendations|idle-resources|create-budget|report}" ;;
esac
```

### ขั้นตอนที่ 527: Kubernetes Cost Optimization with Kubecost

**Kubecost** สำหรับ cost allocation และ optimization ระดับ Kubernetes

```bash
#!/bin/bash
# kubecost-manager.sh - Kubernetes Cost Management with Kubecost

set -euo pipefail

KUBECOST_NAMESPACE="${KUBECOST_NAMESPACE:-kubecost}"
KUBECOST_VERSION="${KUBECOST_VERSION:-1.108.0}"
KUBECOST_TOKEN="${KUBECOST_TOKEN:-}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_kubecost() {
    log "Installing Kubecost ${KUBECOST_VERSION}..."
    
    kubectl create namespace "${KUBECOST_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF > /tmp/kubecost-values.yaml
global:
  prometheus:
    fqdn: http://prometheus-operated.monitoring:9090
  grafana:
    enabled: false
    proxy: false
    fqdn: http://grafana.monitoring

kubecostToken: "${KUBECOST_TOKEN}"

kubecostProductConfigs:
  currencyCode: "USD"
  clusterName: "production"
  labelMappingConfigs:
    enabled: true
    owner_label: "team"
    team_label: "team"
    department_label: "department"
    product_label: "product"
    environment_label: "environment"

serviceMonitor:
  enabled: true
  namespace: monitoring

persistentVolume:
  enabled: true
  size: 50Gi
  storageClass: fast-ssd

resources:
  limits:
    cpu: "1"
    memory: 1Gi
  requests:
    cpu: 200m
    memory: 256Mi

extraArgs:
  - --enable-request-right-sizing-v2

networkCosts:
  enabled: true
  logLevel: info

clusterController:
  enabled: true
  actionConfigs:
    enabled: true

reporting:
  logCollection: true
  productAnalytics: false

federatedETL:
  enabled: false
EOF
    
    helm repo add kubecost https://kubecost.github.io/cost-analyzer/
    helm repo update
    
    helm upgrade --install kubecost kubecost/cost-analyzer \
        --namespace "${KUBECOST_NAMESPACE}" \
        -f /tmp/kubecost-values.yaml \
        --timeout 300s \
        --wait
    
    log "Kubecost installed"
}

query_kubecost_api() {
    local endpoint="$1"
    local params="${2:-}"
    
    local kubecost_svc=$(kubectl get service -n "${KUBECOST_NAMESPACE}" \
        -l "app=cost-analyzer" \
        -o jsonpath='{.items[0].metadata.name}')
    
    kubectl port-forward -n "${KUBECOST_NAMESPACE}" "svc/${kubecost_svc}" 9090:9090 &
    local pf_pid=$!
    sleep 3
    
    local result=$(curl -s "http://localhost:9090/model/${endpoint}?${params}")
    
    kill ${pf_pid} 2>/dev/null
    echo "${result}"
}

get_namespace_costs() {
    local window="${1:-month}"
    
    log "Getting namespace costs for window: ${window}"
    
    query_kubecost_api "allocation" \
        "window=${window}&aggregate=namespace&accumulate=true&shareIdle=true" | \
        jq -r '
            .data[0] | 
            to_entries[] |
            {
                namespace: .key,
                total: (.value.totalCost | . * 100 | round | . / 100),
                cpu: (.value.cpuCost | . * 100 | round | . / 100),
                memory: (.value.ramCost | . * 100 | round | . / 100),
                network: (.value.networkCost | . * 100 | round | . / 100),
                storage: (.value.pvCost | . * 100 | round | . / 100),
                efficiency: (.value.totalEfficiency * 100 | round | tostring + "%")
            }
        ' | \
        jq -r '[.namespace, .total, .cpu, .memory, .storage, .efficiency] | @tsv' | \
        sort -t$'\t' -k2 -rn | \
        awk -F'\t' 'BEGIN{printf "%-30s %10s %10s %10s %10s %10s\n", "Namespace", "Total", "CPU", "Memory", "Storage", "Efficiency"}
                    {printf "%-30s $%9.2f $%9.2f $%9.2f $%9.2f %10s\n", $1, $2, $3, $4, $5, $6}'
}

get_team_costs() {
    local window="${1:-month}"
    local label="${2:-team}"
    
    log "Getting team cost allocation for window: ${window}"
    
    query_kubecost_api "allocation" \
        "window=${window}&aggregate=label:${label}&accumulate=true" | \
        jq -r '
            .data[0] |
            to_entries[] |
            [.key, (.value.totalCost * 100 | round | . / 100)] |
            @tsv
        ' | \
        sort -t$'\t' -k2 -rn | \
        awk -F'\t' 'BEGIN{printf "%-25s %12s\n", "Team", "Monthly Cost"} {printf "%-25s $%11.2f\n", $1, $2}'
}

get_rightsizing_recommendations() {
    log "Getting rightsizing recommendations..."
    
    query_kubecost_api "savings/requestSizing" \
        "window=3d&filterNamespaces=&threshold=0.15" | \
        jq -r '
            .recommendations[] |
            {
                namespace: .namespace,
                controller: .controllerName,
                container: .containerName,
                current_cpu: .currentCPUReq,
                recommended_cpu: .recommendedCPUReq,
                current_mem: .currentRAMReq,
                recommended_mem: .recommendedRAMReq,
                monthly_savings: .monthlySavings
            }
        ' | \
        jq -r '[.namespace, .controller, .container, .current_cpu, .recommended_cpu, .monthly_savings] | @tsv' | \
        sort -t$'\t' -k6 -rn | head -30 | \
        awk -F'\t' 'BEGIN{printf "%-20s %-30s %-20s %-15s %-15s %12s\n", "Namespace", "Controller", "Container", "Current CPU", "Rec CPU", "Monthly Save"}
                    {printf "%-20s %-30s %-20s %-15s %-15s $%11.2f\n", $1, $2, $3, $4, $5, $6}'
}

generate_cost_report() {
    local output="${1:-cost-report-$(date '+%Y%m%d').md}"
    
    log "Generating comprehensive cost report..."
    
    cat <<EOF > "${output}"
# Kubernetes Cost Report - $(date '+%Y-%m-%d')

## Executive Summary

Generated: $(date '+%Y-%m-%d %H:%M:%S')
Cluster: production
Tool: Kubecost

## Monthly Cost by Namespace

$(get_namespace_costs month 2>/dev/null || echo "Run with Kubecost access")

## Cost by Team

$(get_team_costs month team 2>/dev/null || echo "Run with Kubecost access")

## Top Rightsizing Opportunities

$(get_rightsizing_recommendations 2>/dev/null || echo "Run with Kubecost access")

## Action Items

1. **Review over-provisioned workloads** - Apply recommended CPU/memory limits
2. **Enable spot instances** for non-critical batch workloads
3. **Implement namespace budgets** for cost accountability
4. **Review idle resources** - Remove unused PVCs and deployments

EOF
    
    log "Cost report saved: ${output}"
}

create_namespace_budget_alert() {
    local namespace="$1"
    local monthly_budget="$2"
    local alert_email="${3:-finops@company.com}"
    
    log "Creating budget alert for namespace ${namespace}: \$${monthly_budget}/month"
    
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: budget-alert-${namespace}
  namespace: ${KUBECOST_NAMESPACE}
  labels:
    kubecost/budget: "true"
data:
  namespace: "${namespace}"
  monthly_budget: "${monthly_budget}"
  alert_email: "${alert_email}"
  alert_threshold_80: "$(echo "${monthly_budget} * 0.8" | bc)"
  alert_threshold_100: "${monthly_budget}"
EOF
    
    log "Budget alert configured for ${namespace}"
}

case "${1:-help}" in
    "install") install_kubecost ;;
    "namespace-costs") get_namespace_costs "${2:-month}" ;;
    "team-costs") get_team_costs "${2:-month}" "${3:-team}" ;;
    "rightsizing") get_rightsizing_recommendations ;;
    "report") generate_cost_report "${2:-}" ;;
    "budget-alert") create_namespace_budget_alert "$2" "$3" "${4:-finops@company.com}" ;;
    *) echo "Usage: $0 {install|namespace-costs|team-costs|rightsizing|report|budget-alert}" ;;
esac
```

### ขั้นตอนที่ 528: Auto-Scaling Cost Optimization

**Intelligent scaling** เพื่อ balance cost และ performance

```bash
#!/bin/bash
# cost-aware-autoscaling.sh - Cost-Optimized Auto-Scaling

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

configure_spot_node_groups() {
    local cluster_name="${1:-production}"
    local node_group_name="${2:-spot-workers}"
    
    log "Configuring spot instance node group: ${node_group_name}"
    
    cat <<EOF > /tmp/spot-nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${cluster_name}
  region: ap-southeast-1

managedNodeGroups:
  - name: ${node_group_name}
    instanceTypes:
      - m5.xlarge
      - m5a.xlarge
      - m5n.xlarge
      - m4.xlarge
      - m5d.xlarge
      - r5.xlarge
      - r5a.xlarge
    spot: true
    minSize: 0
    maxSize: 100
    desiredCapacity: 5
    volumeSize: 50
    labels:
      role: spot-worker
      cost-tier: spot
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
    tags:
      nodegroup-role: spot-worker
      cost-center: engineering
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
    asgMetricsCollection:
      - granularity: 1Minute
        metrics:
          - GroupDesiredCapacity
          - GroupInServiceCapacity
          - GroupPendingCapacity
          - GroupMinSize
          - GroupMaxSize
EOF
    
    log "Spot node group configuration created: /tmp/spot-nodegroup.yaml"
}

deploy_spot_tolerant_workload() {
    local app_name="${1:-batch-processor}"
    local namespace="${2:-default}"
    local image="${3:-my-app:latest}"
    local replicas="${4:-3}"
    
    log "Deploying spot-tolerant workload: ${app_name}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${app_name}
  namespace: ${namespace}
  labels:
    app: ${app_name}
    cost-tier: spot
spec:
  replicas: ${replicas}
  selector:
    matchLabels:
      app: ${app_name}
  template:
    metadata:
      labels:
        app: ${app_name}
        cost-tier: spot
    spec:
      tolerations:
        - key: spot
          operator: Equal
          value: "true"
          effect: NoSchedule
        - key: node.kubernetes.io/unreachable
          operator: Exists
          effect: NoExecute
          tolerationSeconds: 30
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoExecute
          tolerationSeconds: 30
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: cost-tier
                    operator: In
                    values: [spot]
            - weight: 50
              preference:
                matchExpressions:
                  - key: role
                    operator: In
                    values: [spot-worker]
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: ${app_name}
                topologyKey: kubernetes.io/hostname
      terminationGracePeriodSeconds: 60
      containers:
        - name: ${app_name}
          image: ${image}
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "2Gi"
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 30 && graceful_shutdown.sh"]
          env:
            - name: SPOT_INSTANCE
              value: "true"
            - name: DRAIN_TIMEOUT
              value: "30"
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: ${app_name}
        - maxSkew: 2
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: ${app_name}
EOF
    
    log "Spot-tolerant workload deployed: ${app_name}"
}

configure_karpenter() {
    local cluster_name="${1:-production}"
    
    log "Configuring Karpenter for cost-optimized auto-provisioning..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: cost-optimized
spec:
  template:
    metadata:
      labels:
        provisioner: karpenter
        cost-tier: spot
    spec:
      nodeClassRef:
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["m", "r", "c"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["3"]
        - key: karpenter.k8s.aws/instance-cpu
          operator: In
          values: ["2", "4", "8", "16", "32"]
      taints:
        - key: karpenter
          value: "true"
          effect: NoSchedule
  limits:
    cpu: "1000"
    memory: "4000Gi"
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    expireAfter: 720h
  weight: 10
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  role: KarpenterNodeRole-${cluster_name}
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${cluster_name}
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${cluster_name}
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        iops: 3000
        throughput: 125
        encrypted: true
  tags:
    ManagedBy: Karpenter
    Cluster: ${cluster_name}
EOF
    
    log "Karpenter NodePool configured"
}

configure_vpa_for_rightsizing() {
    local app_name="$1"
    local namespace="${2:-default}"
    
    log "Configuring VPA for ${app_name}..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: ${app_name}-vpa
  namespace: ${namespace}
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ${app_name}
  updatePolicy:
    updateMode: "Auto"
    minReplicas: 2
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: "4"
          memory: 8Gi
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits
EOF
    
    log "VPA configured for ${app_name}"
}

setup_hpa_with_custom_metrics() {
    local app_name="$1"
    local namespace="${2:-default}"
    local min_replicas="${3:-2}"
    local max_replicas="${4:-20}"
    
    log "Setting up HPA with custom metrics for ${app_name}..."
    
    cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ${app_name}-hpa
  namespace: ${namespace}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ${app_name}
  minReplicas: ${min_replicas}
  maxReplicas: ${max_replicas}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
    - type: External
      external:
        metric:
          name: sqs_messages_visible
          selector:
            matchLabels:
              queue: ${app_name}
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
        - type: Pods
          value: 2
          periodSeconds: 60
      selectPolicy: Min
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 30
        - type: Pods
          value: 5
          periodSeconds: 30
      selectPolicy: Max
EOF
    
    log "HPA configured for ${app_name}"
}

monitor_scaling_efficiency() {
    log "=== Auto-scaling Efficiency Report ==="
    
    echo "--- HPA Status ---"
    kubectl get hpa -A \
        -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,CURRENT:.status.currentReplicas,DESIRED:.status.desiredReplicas,MIN:.spec.minReplicas,MAX:.spec.maxReplicas,CPU:.status.currentMetrics[0].resource.current.averageUtilization"
    
    echo ""
    echo "--- VPA Recommendations ---"
    kubectl get vpa -A \
        -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,MODE:.spec.updatePolicy.updateMode"
    
    echo ""
    echo "--- Node Utilization ---"
    kubectl top nodes --sort-by=cpu 2>/dev/null || echo "Metrics server not available"
    
    echo ""
    echo "--- Pod Resource Requests vs Limits ---"
    kubectl get pods -A \
        -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,CPU_REQ:.spec.containers[0].resources.requests.cpu,CPU_LIM:.spec.containers[0].resources.limits.cpu,MEM_REQ:.spec.containers[0].resources.requests.memory,MEM_LIM:.spec.containers[0].resources.limits.memory" | \
        head -30
}

case "${1:-help}" in
    "spot-nodegroup") configure_spot_node_groups "${2:-production}" "${3:-spot-workers}" ;;
    "spot-workload") deploy_spot_tolerant_workload "$2" "${3:-default}" "${4:-my-app:latest}" "${5:-3}" ;;
    "karpenter") configure_karpenter "${2:-production}" ;;
    "vpa") configure_vpa_for_rightsizing "$2" "${3:-default}" ;;
    "hpa") setup_hpa_with_custom_metrics "$2" "${3:-default}" "${4:-2}" "${5:-20}" ;;
    "monitor") monitor_scaling_efficiency ;;
    *) echo "Usage: $0 {spot-nodegroup|spot-workload|karpenter|vpa|hpa|monitor}" ;;
esac
```

### ขั้นตอนที่ 529: Cloud Resource Tagging Strategy

**Tagging strategy** สำหรับ cost allocation และ governance

```bash
#!/bin/bash
# tagging-governance.sh - Cloud Resource Tagging and Cost Governance

set -euo pipefail

AWS_REGION="${AWS_REGION:-ap-southeast-1}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

define_tagging_policy() {
    log "Defining mandatory tagging policy..."
    
    cat <<'EOF' > /tmp/tagging-policy.json
{
    "tags": {
        "mandatory": [
            {
                "key": "Environment",
                "allowed_values": ["production", "staging", "development", "testing"],
                "description": "Deployment environment"
            },
            {
                "key": "Team",
                "allowed_values": "any",
                "description": "Owning team name"
            },
            {
                "key": "Product",
                "allowed_values": "any",
                "description": "Product or service name"
            },
            {
                "key": "CostCenter",
                "allowed_values": "any",
                "description": "Finance cost center code"
            },
            {
                "key": "ManagedBy",
                "allowed_values": ["terraform", "helm", "manual", "cloudformation"],
                "description": "IaC tool managing resource"
            }
        ],
        "recommended": [
            {"key": "DataClassification", "allowed_values": ["public", "internal", "confidential", "restricted"]},
            {"key": "BackupEnabled", "allowed_values": ["true", "false"]},
            {"key": "AutoShutdown", "allowed_values": ["true", "false"]},
            {"key": "ExpiryDate", "allowed_values": "date-format"}
        ],
        "automated": [
            {"key": "CreatedAt", "auto_populate": "timestamp"},
            {"key": "CreatedBy", "auto_populate": "iam-principal"},
            {"key": "Region", "auto_populate": "aws-region"}
        ]
    }
}
EOF
    
    log "Tagging policy defined: /tmp/tagging-policy.json"
}

audit_untagged_resources() {
    log "Auditing untagged AWS resources..."
    
    local mandatory_tags=("Environment" "Team" "Product" "CostCenter")
    
    echo "=== EC2 Instances Without Required Tags ==="
    aws ec2 describe-instances \
        --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags]' \
        --output json | \
        python3 -c "
import json, sys
data = json.load(sys.stdin)
required = ['Environment', 'Team', 'Product', 'CostCenter']
for reservation in data:
    for instance in reservation:
        instance_id = instance[0]
        state = instance[1]
        tags = {t['Key']: t['Value'] for t in (instance[2] or [])}
        missing = [t for t in required if t not in tags]
        if missing and state == 'running':
            print(f'{instance_id} | {state} | Missing: {missing}')
" 2>/dev/null | head -20
    
    echo ""
    echo "=== S3 Buckets Without Required Tags ==="
    aws s3api list-buckets --query 'Buckets[*].Name' --output text | \
        tr '\t' '\n' | while read bucket; do
        tags=$(aws s3api get-bucket-tagging --bucket "${bucket}" 2>/dev/null | \
               jq -r '.TagSet[].Key' 2>/dev/null | tr '\n' ',')
        for tag in "${mandatory_tags[@]}"; do
            if [[ "${tags}" != *"${tag}"* ]]; then
                echo "${bucket} | Missing: ${tag}"
                break
            fi
        done
    done | head -20
    
    echo ""
    echo "=== RDS Instances Without Required Tags ==="
    aws rds describe-db-instances \
        --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,TagList]' \
        --output json | \
        python3 -c "
import json, sys
data = json.load(sys.stdin)
required = ['Environment', 'Team', 'Product']
for db in data:
    db_id = db[0]
    status = db[1]
    tags = {t['Key']: t['Value'] for t in (db[2] or [])}
    missing = [t for t in required if t not in tags]
    if missing:
        print(f'{db_id} | {status} | Missing: {missing}')
" 2>/dev/null | head -20
}

enforce_tagging_with_scp() {
    log "Creating SCP to enforce mandatory tags on EC2..."
    
    cat <<'JSON' > /tmp/require-tags-scp.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "RequireEC2Tags",
            "Effect": "Deny",
            "Action": [
                "ec2:RunInstances"
            ],
            "Resource": [
                "arn:aws:ec2:*:*:instance/*"
            ],
            "Condition": {
                "Null": {
                    "aws:RequestTag/Environment": "true"
                }
            }
        },
        {
            "Sid": "RequireTeamTag",
            "Effect": "Deny",
            "Action": [
                "ec2:RunInstances",
                "rds:CreateDBInstance",
                "elasticloadbalancing:CreateLoadBalancer"
            ],
            "Resource": "*",
            "Condition": {
                "Null": {
                    "aws:RequestTag/Team": "true"
                }
            }
        },
        {
            "Sid": "RequireCostCenter",
            "Effect": "Deny",
            "Action": [
                "ec2:RunInstances",
                "rds:CreateDBInstance"
            ],
            "Resource": "*",
            "Condition": {
                "Null": {
                    "aws:RequestTag/CostCenter": "true"
                }
            }
        }
    ]
}
JSON
    
    log "SCP policy created: /tmp/require-tags-scp.json"
    log "Apply with: aws organizations create-policy --type SERVICE_CONTROL_POLICY --content file:///tmp/require-tags-scp.json --name RequireMandatoryTags --description 'Enforce mandatory cost allocation tags'"
}

auto_tag_resources() {
    local resource_type="${1:-ec2}"
    local tag_key="${2:-AutoTagged}"
    local tag_value="${3:-true}"
    
    log "Auto-tagging ${resource_type} resources..."
    
    case "${resource_type}" in
        "ec2")
            aws ec2 describe-instances \
                --filters "Name=tag:${tag_key},Values=" \
                --query 'Reservations[*].Instances[*].InstanceId' \
                --output text | tr '\t' '\n' | while read instance_id; do
                aws ec2 create-tags \
                    --resources "${instance_id}" \
                    --tags "Key=${tag_key},Value=${tag_value}" \
                            "Key=AutoTaggedAt,Value=$(date '+%Y-%m-%d')"
                log "Tagged instance: ${instance_id}"
            done
            ;;
        "ebs")
            aws ec2 describe-volumes \
                --filters "Name=status,Values=available" \
                --query 'Volumes[*].VolumeId' \
                --output text | tr '\t' '\n' | while read vol_id; do
                aws ec2 create-tags \
                    --resources "${vol_id}" \
                    --tags "Key=${tag_key},Value=${tag_value}" \
                            "Key=Status,Value=unattached"
                log "Tagged volume: ${vol_id}"
            done
            ;;
    esac
}

generate_tagging_report() {
    local output="${1:-tagging-report-$(date '+%Y%m%d').csv}"
    
    log "Generating tagging compliance report..."
    
    echo "ResourceType,ResourceId,Environment,Team,Product,CostCenter,Compliant" > "${output}"
    
    aws ec2 describe-instances \
        --query 'Reservations[*].Instances[*].[InstanceId,Tags]' \
        --output json | \
        python3 -c "
import json, sys, csv

data = json.load(sys.stdin)
required = ['Environment', 'Team', 'Product', 'CostCenter']
writer = csv.writer(sys.stdout)

for reservation in data:
    for instance in reservation:
        instance_id = instance[0]
        tags = {t['Key']: t['Value'] for t in (instance[1] or [])}
        compliant = all(t in tags for t in required)
        writer.writerow([
            'EC2',
            instance_id,
            tags.get('Environment', 'MISSING'),
            tags.get('Team', 'MISSING'),
            tags.get('Product', 'MISSING'),
            tags.get('CostCenter', 'MISSING'),
            'YES' if compliant else 'NO'
        ])
" >> "${output}" 2>/dev/null
    
    local total=$(tail -n +2 "${output}" | wc -l)
    local compliant=$(tail -n +2 "${output}" | grep ",YES$" | wc -l)
    local non_compliant=$(tail -n +2 "${output}" | grep ",NO$" | wc -l)
    
    echo ""
    echo "=== Tagging Compliance Summary ==="
    echo "Total Resources: ${total}"
    echo "Compliant: ${compliant} ($(( compliant * 100 / (total + 1) ))%)"
    echo "Non-Compliant: ${non_compliant}"
    echo "Report saved: ${output}"
}

case "${1:-help}" in
    "define-policy") define_tagging_policy ;;
    "audit") audit_untagged_resources ;;
    "enforce-scp") enforce_tagging_with_scp ;;
    "auto-tag") auto_tag_resources "${2:-ec2}" "${3:-AutoTagged}" "${4:-true}" ;;
    "report") generate_tagging_report "${2:-}" ;;
    *) echo "Usage: $0 {define-policy|audit|enforce-scp|auto-tag|report}" ;;
esac
```

---

## สรุป Part 43

ในส่วนนี้เราได้เรียนรู้:

| ขั้นตอน | หัวข้อ | เครื่องมือหลัก |
|---------|--------|----------------|
| 526 | Cloud Cost Analysis | AWS Cost Explorer, Anomaly Detection, Savings Recommendations |
| 527 | Kubernetes Cost Management | Kubecost, Namespace Costs, Team Allocation, Rightsizing |
| 528 | Cost-Optimized Auto-Scaling | Spot Instances, Karpenter, VPA, HPA Custom Metrics |
| 529 | Resource Tagging Governance | SCP Enforcement, Auto-tagging, Compliance Reporting |

### ขั้นตอนต่อไป: Part 44 - ML/AI Infrastructure Engineering
