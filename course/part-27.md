# Part 27: Cloud Provider Integration

## Module 3: Advanced Level
### ขั้นตอนที่ 461-472: AWS, GCP, Azure

---

## ขั้นตอนที่ 461: AWS CLI Automation

```bash
#!/usr/bin/env bash
# aws_automation.sh

# ===== AWS CLI Integration =====
AWS_REGION="${AWS_REGION:-ap-southeast-1}"
AWS_PROFILE="${AWS_PROFILE:-default}"

# AWS command wrapper
aws_cmd() {
    aws --region "$AWS_REGION" --profile "$AWS_PROFILE" "$@"
}

# Check AWS CLI
check_aws() {
    if ! command -v aws &>/dev/null; then
        echo "AWS CLI not found" >&2
        echo "Install: https://aws.amazon.com/cli/"
        return 1
    fi
    
    local identity
    identity=$(aws_cmd sts get-caller-identity 2>/dev/null)
    if [[ -z "$identity" ]]; then
        echo "AWS not authenticated" >&2
        return 1
    fi
    
    local account
    account=$(echo "$identity" | grep -oP '"Account": "\K[^"]+')
    local user_arn
    user_arn=$(echo "$identity" | grep -oP '"Arn": "\K[^"]+')
    
    echo "AWS Account: $account"
    echo "User: $user_arn"
    echo "Region: $AWS_REGION"
}

# ===== EC2 Operations =====
ec2_list_instances() {
    local state="${1:-running}"
    local name_filter="${2:-}"
    
    local filters="Name=instance-state-name,Values=$state"
    [[ -n "$name_filter" ]] && filters+=" Name=tag:Name,Values=*${name_filter}*"
    
    aws_cmd ec2 describe-instances \
        --filters $filters \
        --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,State.Name,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
        --output table 2>/dev/null
}

ec2_start() {
    local instance_id="$1"
    echo "Starting: $instance_id"
    aws_cmd ec2 start-instances --instance-ids "$instance_id"
    aws_cmd ec2 wait instance-running --instance-ids "$instance_id"
    echo "Started: $instance_id"
}

ec2_stop() {
    local instance_id="$1"
    echo "Stopping: $instance_id"
    aws_cmd ec2 stop-instances --instance-ids "$instance_id"
    aws_cmd ec2 wait instance-stopped --instance-ids "$instance_id"
    echo "Stopped: $instance_id"
}

ec2_get_ip() {
    local instance_id="$1"
    aws_cmd ec2 describe-instances \
        --instance-ids "$instance_id" \
        --query 'Reservations[0].Instances[0].PublicIpAddress' \
        --output text 2>/dev/null
}

ec2_create_ami() {
    local instance_id="$1"
    local name="$2"
    local description="${3:-Created by automation}"
    
    local ami_id
    ami_id=$(aws_cmd ec2 create-image \
        --instance-id "$instance_id" \
        --name "$name" \
        --description "$description" \
        --query 'ImageId' \
        --output text 2>/dev/null)
    
    echo "Creating AMI: $ami_id"
    aws_cmd ec2 wait image-available --image-ids "$ami_id"
    echo "AMI ready: $ami_id"
    echo "$ami_id"
}

# ===== S3 Operations =====
s3_upload() {
    local local_path="$1"
    local s3_bucket="$2"
    local s3_path="${3:-}"
    local options="${4:-}"
    
    local destination="s3://$s3_bucket"
    [[ -n "$s3_path" ]] && destination+="/$s3_path"
    
    echo "Uploading to $destination..."
    
    if [[ -d "$local_path" ]]; then
        aws_cmd s3 sync "$local_path" "$destination" \
            --delete $options 2>/dev/null
    else
        aws_cmd s3 cp "$local_path" "$destination" $options 2>/dev/null
    fi
    
    echo "Upload complete"
}

s3_download() {
    local s3_bucket="$1"
    local s3_path="$2"
    local local_path="$3"
    
    local source="s3://$s3_bucket/$s3_path"
    echo "Downloading from $source..."
    
    aws_cmd s3 cp "$source" "$local_path" 2>/dev/null
    echo "Downloaded: $local_path"
}

s3_list() {
    local bucket="$1"
    local prefix="${2:-}"
    
    aws_cmd s3 ls "s3://$bucket/$prefix" 2>/dev/null
}

s3_delete() {
    local bucket="$1"
    local path="$2"
    local recursive="${3:-false}"
    
    local args=()
    [[ "$recursive" == "true" ]] && args+=(--recursive)
    
    aws_cmd s3 rm "s3://$bucket/$path" "${args[@]}" 2>/dev/null
}

s3_presigned_url() {
    local bucket="$1"
    local key="$2"
    local expires="${3:-3600}"
    
    aws_cmd s3 presign "s3://$bucket/$key" \
        --expires-in "$expires" 2>/dev/null
}

# ===== RDS Operations =====
rds_list_instances() {
    aws_cmd rds describe-db-instances \
        --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceClass,DBInstanceStatus,Endpoint.Address]' \
        --output table 2>/dev/null
}

rds_create_snapshot() {
    local instance_id="$1"
    local snapshot_id="${2:-${instance_id}-$(date +%Y%m%d%H%M%S)}"
    
    echo "Creating snapshot: $snapshot_id"
    aws_cmd rds create-db-snapshot \
        --db-instance-identifier "$instance_id" \
        --db-snapshot-identifier "$snapshot_id" 2>/dev/null
    
    echo "Waiting for snapshot..."
    aws_cmd rds wait db-snapshot-available \
        --db-snapshot-identifier "$snapshot_id" 2>/dev/null
    
    echo "Snapshot ready: $snapshot_id"
}

rds_restore_from_snapshot() {
    local snapshot_id="$1"
    local new_instance_id="$2"
    local instance_class="${3:-db.t3.micro}"
    
    aws_cmd rds restore-db-instance-from-db-snapshot \
        --db-instance-identifier "$new_instance_id" \
        --db-snapshot-identifier "$snapshot_id" \
        --db-instance-class "$instance_class" 2>/dev/null
    
    echo "Restoring: $new_instance_id from $snapshot_id"
}

# ===== Lambda Operations =====
lambda_invoke() {
    local function_name="$1"
    local payload="${2:-{}}"
    local output_file="${3:-/tmp/lambda_output_$$.json}"
    
    aws_cmd lambda invoke \
        --function-name "$function_name" \
        --payload "$(echo "$payload" | base64)" \
        --cli-binary-format raw-in-base64-out \
        "$output_file" 2>/dev/null
    
    cat "$output_file"
    rm -f "$output_file"
}

lambda_update_code() {
    local function_name="$1"
    local zip_file="$2"
    
    echo "Updating Lambda: $function_name"
    aws_cmd lambda update-function-code \
        --function-name "$function_name" \
        --zip-file "fileb://$zip_file" 2>/dev/null
    
    aws_cmd lambda wait function-updated \
        --function-name "$function_name" 2>/dev/null
    
    echo "Updated: $function_name"
}

# ===== CloudWatch =====
cw_get_metric() {
    local namespace="$1"
    local metric_name="$2"
    local dimension_name="$3"
    local dimension_value="$4"
    local period="${5:-300}"  # 5 minutes
    local stat="${6:-Average}"
    local hours_back="${7:-1}"
    
    local start_time
    start_time=$(date -u -d "-${hours_back} hour" '+%Y-%m-%dT%H:%M:%S' 2>/dev/null || \
                 date -u -v-${hours_back}H '+%Y-%m-%dT%H:%M:%S')
    local end_time
    end_time=$(date -u '+%Y-%m-%dT%H:%M:%S')
    
    aws_cmd cloudwatch get-metric-statistics \
        --namespace "$namespace" \
        --metric-name "$metric_name" \
        --dimensions "Name=$dimension_name,Value=$dimension_value" \
        --start-time "$start_time" \
        --end-time "$end_time" \
        --period "$period" \
        --statistics "$stat" \
        --query 'Datapoints[*].[Timestamp,Average]' \
        --output table 2>/dev/null
}

cw_put_metric() {
    local namespace="$1"
    local metric_name="$2"
    local value="$3"
    local unit="${4:-Count}"
    
    aws_cmd cloudwatch put-metric-data \
        --namespace "$namespace" \
        --metric-name "$metric_name" \
        --value "$value" \
        --unit "$unit" 2>/dev/null
    
    echo "Metric published: $metric_name=$value"
}

# Demo
if command -v aws &>/dev/null; then
    echo "=== AWS Demo ==="
    check_aws 2>/dev/null || echo "AWS not configured (demo only)"
else
    echo "AWS CLI not available"
    echo ""
    echo "Example usage:"
    echo "  ec2_list_instances running myapp"
    echo "  s3_upload ./dist my-bucket static/v1.0/"
    echo "  rds_create_snapshot my-database"
    echo "  lambda_invoke my-function '{\"key\":\"value\"}'"
fi
```

---

## ขั้นตอนที่ 462: AWS Infrastructure Automation

```bash
#!/usr/bin/env bash
# aws_infra.sh

# ===== AWS Infrastructure Scripts =====

# ===== CloudFormation =====
cf_deploy() {
    local stack_name="$1"
    local template_file="$2"
    local parameters="${3:-}"
    
    local args=(
        deploy
        --stack-name "$stack_name"
        --template-file "$template_file"
        --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
    )
    
    [[ -n "$parameters" ]] && args+=(--parameter-overrides $parameters)
    
    echo "Deploying CloudFormation stack: $stack_name"
    aws cloudformation "${args[@]}"
}

cf_delete() {
    local stack_name="$1"
    
    echo "Deleting stack: $stack_name"
    aws cloudformation delete-stack --stack-name "$stack_name"
    aws cloudformation wait stack-delete-complete --stack-name "$stack_name"
    echo "Deleted: $stack_name"
}

cf_get_output() {
    local stack_name="$1"
    local output_key="$2"
    
    aws cloudformation describe-stacks \
        --stack-name "$stack_name" \
        --query "Stacks[0].Outputs[?OutputKey=='$output_key'].OutputValue" \
        --output text 2>/dev/null
}

# Generate basic CloudFormation template
generate_cf_template() {
    local app_name="$1"
    local instance_type="${2:-t3.micro}"
    local ami_id="${3:-ami-0abcdef1234567890}"
    
    cat << EOF
AWSTemplateFormatVersion: '2010-09-09'
Description: '$app_name Infrastructure'

Parameters:
  InstanceType:
    Type: String
    Default: $instance_type
  AMIId:
    Type: AWS::EC2::Image::Id
    Default: $ami_id

Resources:
  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: $app_name Security Group
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !Ref AMIId
      SecurityGroups:
        - !Ref SecurityGroup
      Tags:
        - Key: Name
          Value: $app_name
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd

  ElasticIP:
    Type: AWS::EC2::EIP
    Properties:
      InstanceId: !Ref EC2Instance

Outputs:
  PublicIP:
    Description: Public IP Address
    Value: !Ref ElasticIP
    Export:
      Name: !Sub '\${AWS::StackName}-PublicIP'
  InstanceId:
    Description: EC2 Instance ID
    Value: !Ref EC2Instance
EOF
}

# ===== VPC Setup =====
create_vpc() {
    local name="$1"
    local cidr="${2:-10.0.0.0/16}"
    
    echo "Creating VPC: $name ($cidr)"
    
    # Create VPC
    local vpc_id
    vpc_id=$(aws ec2 create-vpc \
        --cidr-block "$cidr" \
        --query 'Vpc.VpcId' \
        --output text 2>/dev/null)
    
    # Tag VPC
    aws ec2 create-tags \
        --resources "$vpc_id" \
        --tags "Key=Name,Value=$name" 2>/dev/null
    
    # Enable DNS
    aws ec2 modify-vpc-attribute \
        --vpc-id "$vpc_id" \
        --enable-dns-hostnames '{"Value":true}' 2>/dev/null
    
    echo "VPC created: $vpc_id"
    
    # Create subnets
    local azs
    azs=$(aws ec2 describe-availability-zones \
        --query 'AvailabilityZones[*].ZoneName' \
        --output text 2>/dev/null | tr '\t' '\n' | head -2)
    
    local subnet_ids=()
    local i=0
    while IFS= read -r az; do
        local subnet_cidr="10.0.${i}.0/24"
        local subnet_id
        subnet_id=$(aws ec2 create-subnet \
            --vpc-id "$vpc_id" \
            --cidr-block "$subnet_cidr" \
            --availability-zone "$az" \
            --query 'Subnet.SubnetId' \
            --output text 2>/dev/null)
        
        aws ec2 create-tags \
            --resources "$subnet_id" \
            --tags "Key=Name,Value=${name}-subnet-${i}" 2>/dev/null
        
        subnet_ids+=("$subnet_id")
        ((i++))
    done <<< "$azs"
    
    echo "Subnets: ${subnet_ids[*]}"
    echo "$vpc_id"
}

# ===== Auto Scaling =====
create_asg() {
    local name="$1"
    local launch_template="$2"
    local min="$3"
    local max="$4"
    local desired="${5:-$min}"
    local subnet_ids="$6"
    
    aws autoscaling create-auto-scaling-group \
        --auto-scaling-group-name "$name" \
        --launch-template "LaunchTemplateName=$launch_template" \
        --min-size "$min" \
        --max-size "$max" \
        --desired-capacity "$desired" \
        --vpc-zone-identifier "$subnet_ids" \
        --health-check-type ELB \
        --health-check-grace-period 300 2>/dev/null
    
    echo "ASG created: $name"
}

# ===== Cost Optimization =====
find_unused_eips() {
    echo "=== Unattached Elastic IPs ==="
    aws ec2 describe-addresses \
        --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]' \
        --output table 2>/dev/null
}

find_stopped_instances() {
    echo "=== Stopped Instances ==="
    aws ec2 describe-instances \
        --filters "Name=instance-state-name,Values=stopped" \
        --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,Tags[?Key==`Name`].Value|[0],LaunchTime]' \
        --output table 2>/dev/null
}

find_unused_volumes() {
    echo "=== Unattached EBS Volumes ==="
    aws ec2 describe-volumes \
        --filters "Name=status,Values=available" \
        --query 'Volumes[*].[VolumeId,Size,VolumeType,CreateTime]' \
        --output table 2>/dev/null
}

estimate_monthly_cost() {
    echo "=== Cost Estimate ==="
    aws ce get-cost-and-usage \
        --time-period "Start=$(date -d '-30 days' +%Y-%m-%d 2>/dev/null || date -v-30d +%Y-%m-%d),End=$(date +%Y-%m-%d)" \
        --granularity MONTHLY \
        --metrics "BlendedCost" \
        --query 'ResultsByTime[0].Total.BlendedCost.[Amount,Unit]' \
        --output text 2>/dev/null || echo "Cost Explorer not available"
}

# Demo
echo "=== AWS Infrastructure Demo ==="
echo ""
echo "CloudFormation template:"
generate_cf_template "myapp" "t3.small"
echo ""
echo "Cost optimization checks require AWS credentials"
```

---

## ขั้นตอนที่ 463: GCP Integration

```bash
#!/usr/bin/env bash
# gcp_integration.sh

# ===== Google Cloud Platform Integration =====
GCP_PROJECT="${GCP_PROJECT:-my-project}"
GCP_REGION="${GCP_REGION:-asia-southeast1}"
GCP_ZONE="${GCP_ZONE:-asia-southeast1-a}"

# GCP command wrapper
gcloud_cmd() {
    gcloud "$@" --project="$GCP_PROJECT" 2>/dev/null
}

# Check gcloud
check_gcloud() {
    if ! command -v gcloud &>/dev/null; then
        echo "gcloud not found" >&2
        echo "Install: https://cloud.google.com/sdk/docs/install"
        return 1
    fi
    
    local account
    account=$(gcloud config get-value account 2>/dev/null)
    echo "Account: $account"
    echo "Project: $GCP_PROJECT"
    echo "Region: $GCP_REGION"
}

# ===== Compute Engine =====
gce_list_instances() {
    gcloud_cmd compute instances list \
        --format='table(name,zone,machineType,status,networkInterfaces[0].accessConfigs[0].natIP)'
}

gce_create_instance() {
    local name="$1"
    local machine_type="${2:-e2-micro}"
    local image_family="${3:-debian-11}"
    local startup_script="${4:-}"
    
    local args=(
        compute instances create "$name"
        --zone="$GCP_ZONE"
        --machine-type="$machine_type"
        --image-family="$image_family"
        --image-project=debian-cloud
        --boot-disk-size=20GB
        --tags=http-server,https-server
    )
    
    if [[ -n "$startup_script" && -f "$startup_script" ]]; then
        args+=(--metadata-from-file="startup-script=$startup_script")
    fi
    
    gcloud_cmd "${args[@]}"
    echo "Instance created: $name"
}

gce_ssh() {
    local instance="$1"
    shift
    gcloud_cmd compute ssh "$instance" \
        --zone="$GCP_ZONE" \
        -- "$@"
}

# ===== Cloud Storage =====
gcs_upload() {
    local local_path="$1"
    local bucket="$2"
    local gcs_path="${3:-}"
    
    local dest="gs://$bucket"
    [[ -n "$gcs_path" ]] && dest+="/$gcs_path"
    
    gsutil cp -r "$local_path" "$dest" 2>/dev/null
    echo "Uploaded to $dest"
}

gcs_sync() {
    local local_dir="$1"
    local bucket="$2"
    local prefix="${3:-}"
    
    local dest="gs://$bucket"
    [[ -n "$prefix" ]] && dest+="/$prefix"
    
    gsutil -m rsync -r -d "$local_dir" "$dest" 2>/dev/null
}

gcs_signed_url() {
    local bucket="$1"
    local object="$2"
    local expires="${3:-1h}"
    
    gsutil signurl -d "$expires" "gs://$bucket/$object" 2>/dev/null
}

# ===== Cloud Run =====
cloudrun_deploy() {
    local service_name="$1"
    local image="$2"
    local ns="${3:-default}"
    local min_instances="${4:-0}"
    local max_instances="${5:-10}"
    
    gcloud_cmd run deploy "$service_name" \
        --image="$image" \
        --region="$GCP_REGION" \
        --platform=managed \
        --allow-unauthenticated \
        --min-instances="$min_instances" \
        --max-instances="$max_instances" \
        --memory=512Mi \
        --cpu=1
    
    local url
    url=$(gcloud_cmd run services describe "$service_name" \
        --region="$GCP_REGION" \
        --format='value(status.url)' 2>/dev/null)
    
    echo "Deployed: $service_name"
    echo "URL: $url"
    echo "$url"
}

cloudrun_list() {
    gcloud_cmd run services list \
        --region="$GCP_REGION" \
        --format='table(name,status.url,status.conditions[0].status)'
}

# ===== Cloud SQL =====
sql_list() {
    gcloud_cmd sql instances list \
        --format='table(name,databaseVersion,settings.tier,state,ipAddresses[0].ipAddress)'
}

sql_backup() {
    local instance="$1"
    
    gcloud_cmd sql backups create \
        --instance="$instance" \
        --description="Automated backup $(date)"
    
    echo "Backup created for: $instance"
}

sql_connect() {
    local instance="$1"
    local database="${2:-postgres}"
    
    gcloud_cmd sql connect "$instance" \
        --database="$database" \
        --user=postgres
}

# ===== Pub/Sub =====
pubsub_publish() {
    local topic="$1"
    local message="$2"
    
    gcloud_cmd pubsub topics publish "$topic" \
        --message="$message"
    
    echo "Published to $topic"
}

pubsub_pull() {
    local subscription="$1"
    local max_messages="${2:-10}"
    
    gcloud_cmd pubsub subscriptions pull "$subscription" \
        --max-messages="$max_messages" \
        --auto-ack
}

# Demo
if command -v gcloud &>/dev/null; then
    check_gcloud
else
    echo "gcloud not available"
    echo ""
    echo "Example usage:"
    echo "  gce_list_instances"
    echo "  gcs_upload ./dist my-bucket static/"
    echo "  cloudrun_deploy myservice gcr.io/myproject/myimage:latest"
fi
```

---

## ขั้นตอนที่ 464: Azure Integration

```bash
#!/usr/bin/env bash
# azure_integration.sh

# ===== Azure CLI Integration =====
AZ_SUBSCRIPTION="${AZ_SUBSCRIPTION:-}"
AZ_RESOURCE_GROUP="${AZ_RESOURCE_GROUP:-myapp-rg}"
AZ_LOCATION="${AZ_LOCATION:-southeastasia}"

# Azure command wrapper
az_cmd() {
    local args=("$@")
    [[ -n "$AZ_SUBSCRIPTION" ]] && args+=(--subscription "$AZ_SUBSCRIPTION")
    az "${args[@]}" 2>/dev/null
}

# Check Azure CLI
check_azure() {
    if ! command -v az &>/dev/null; then
        echo "Azure CLI not found" >&2
        echo "Install: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli"
        return 1
    fi
    
    local account
    account=$(az account show --query '{name:name,id:id,user:user.name}' 2>/dev/null)
    echo "Azure Account: $account"
}

# ===== Resource Groups =====
create_resource_group() {
    local name="$1"
    local location="${2:-$AZ_LOCATION}"
    local tags="${3:-}"
    
    az_cmd group create \
        --name "$name" \
        --location "$location" \
        ${tags:+--tags $tags}
    
    echo "Resource group created: $name"
}

delete_resource_group() {
    local name="$1"
    
    echo "Deleting resource group: $name"
    az_cmd group delete --name "$name" --yes --no-wait
}

# ===== Virtual Machines =====
vm_list() {
    az_cmd vm list \
        --resource-group "$AZ_RESOURCE_GROUP" \
        --output table \
        --query '[*].[name,location,hardwareProfile.vmSize,powerState]' 2>/dev/null
}

vm_create() {
    local name="$1"
    local size="${2:-Standard_B1s}"
    local image="${3:-UbuntuLTS}"
    local admin_user="${4:-azureuser}"
    local rg="${5:-$AZ_RESOURCE_GROUP}"
    
    az_cmd vm create \
        --resource-group "$rg" \
        --name "$name" \
        --image "$image" \
        --size "$size" \
        --admin-username "$admin_user" \
        --generate-ssh-keys \
        --location "$AZ_LOCATION"
    
    # Open port 80 and 443
    az_cmd vm open-port --resource-group "$rg" --name "$name" --port 80
    az_cmd vm open-port --resource-group "$rg" --name "$name" --port 443
    
    echo "VM created: $name"
}

vm_start() {
    local name="$1"
    az_cmd vm start --resource-group "$AZ_RESOURCE_GROUP" --name "$name"
    echo "Started: $name"
}

vm_stop() {
    local name="$1"
    az_cmd vm deallocate --resource-group "$AZ_RESOURCE_GROUP" --name "$name"
    echo "Stopped: $name"
}

# ===== Azure Container Registry =====
acr_login() {
    local registry="$1"
    az_cmd acr login --name "$registry"
}

acr_build() {
    local registry="$1"
    local image="$2"
    local tag="${3:-latest}"
    local context="${4:-.}"
    
    az_cmd acr build \
        --registry "$registry" \
        --image "${image}:${tag}" \
        "$context"
    
    echo "Built: ${registry}.azurecr.io/${image}:${tag}"
}

acr_list_images() {
    local registry="$1"
    az_cmd acr repository list --name "$registry" --output table
}

# ===== Azure Blob Storage =====
blob_upload() {
    local file="$1"
    local account="$2"
    local container="$3"
    local blob_name="${4:-$(basename "$file")}"
    
    az_cmd storage blob upload \
        --account-name "$account" \
        --container-name "$container" \
        --file "$file" \
        --name "$blob_name"
    
    echo "Uploaded: $blob_name to $account/$container"
}

blob_download() {
    local account="$1"
    local container="$2"
    local blob_name="$3"
    local dest="${4:-.}"
    
    az_cmd storage blob download \
        --account-name "$account" \
        --container-name "$container" \
        --name "$blob_name" \
        --file "$dest/$blob_name"
}

blob_sas_url() {
    local account="$1"
    local container="$2"
    local blob="$3"
    local expiry="${4:-$(date -u -d '+1 hour' '+%Y-%m-%dT%H:%MZ' 2>/dev/null || date -u -v+1H '+%Y-%m-%dT%H:%MZ')}"
    
    az_cmd storage blob generate-sas \
        --account-name "$account" \
        --container-name "$container" \
        --name "$blob" \
        --permissions r \
        --expiry "$expiry" \
        --output tsv
}

# ===== Azure App Service =====
webapp_deploy() {
    local app_name="$1"
    local plan="${2:-${app_name}-plan}"
    local runtime="${3:-NODE|18-lts}"
    local rg="${4:-$AZ_RESOURCE_GROUP}"
    
    # Create plan
    az_cmd appservice plan create \
        --name "$plan" \
        --resource-group "$rg" \
        --sku B1
    
    # Create webapp
    az_cmd webapp create \
        --name "$app_name" \
        --resource-group "$rg" \
        --plan "$plan" \
        --runtime "$runtime"
    
    local url="${app_name}.azurewebsites.net"
    echo "WebApp created: https://$url"
    echo "$url"
}

webapp_deploy_zip() {
    local app_name="$1"
    local zip_file="$2"
    local rg="${3:-$AZ_RESOURCE_GROUP}"
    
    az_cmd webapp deployment source config-zip \
        --resource-group "$rg" \
        --name "$app_name" \
        --src "$zip_file"
    
    echo "Deployed: $app_name"
}

# ===== Azure Key Vault =====
keyvault_get_secret() {
    local vault_name="$1"
    local secret_name="$2"
    
    az_cmd keyvault secret show \
        --vault-name "$vault_name" \
        --name "$secret_name" \
        --query 'value' \
        --output tsv 2>/dev/null
}

keyvault_set_secret() {
    local vault_name="$1"
    local secret_name="$2"
    local secret_value="$3"
    
    az_cmd keyvault secret set \
        --vault-name "$vault_name" \
        --name "$secret_name" \
        --value "$secret_value" \
        --output none
    
    echo "Secret set: $secret_name"
}

# Demo
if command -v az &>/dev/null; then
    echo "=== Azure Demo ==="
    check_azure 2>/dev/null || echo "Azure not configured"
else
    echo "Azure CLI not available"
    echo ""
    echo "Example usage:"
    echo "  vm_list"
    echo "  blob_upload ./file.txt mystorage mycontainer"
    echo "  webapp_deploy myapp"
fi
```

---

## ขั้นตอนที่ 465: Multi-Cloud Operations

```bash
#!/usr/bin/env bash
# multi_cloud.sh

# ===== Multi-Cloud Abstraction Layer =====
CLOUD_PROVIDER="${CLOUD_PROVIDER:-aws}"  # aws, gcp, azure

# Cloud-agnostic operations
cloud_upload_file() {
    local file="$1"
    local bucket="$2"
    local path="${3:-$(basename "$file")}"
    
    case "$CLOUD_PROVIDER" in
        aws)
            aws s3 cp "$file" "s3://$bucket/$path" 2>/dev/null
            ;;
        gcp)
            gsutil cp "$file" "gs://$bucket/$path" 2>/dev/null
            ;;
        azure)
            az storage blob upload \
                --account-name "$bucket" \
                --container-name "default" \
                --file "$file" \
                --name "$path" 2>/dev/null
            ;;
    esac
    
    echo "Uploaded to $CLOUD_PROVIDER://$bucket/$path"
}

cloud_get_secret() {
    local secret_name="$1"
    
    case "$CLOUD_PROVIDER" in
        aws)
            aws secretsmanager get-secret-value \
                --secret-id "$secret_name" \
                --query 'SecretString' \
                --output text 2>/dev/null
            ;;
        gcp)
            gcloud secrets versions access latest \
                --secret="$secret_name" 2>/dev/null
            ;;
        azure)
            az keyvault secret show \
                --vault-name "$AZURE_VAULT" \
                --name "$secret_name" \
                --query 'value' \
                --output tsv 2>/dev/null
            ;;
    esac
}

cloud_get_instance_metadata() {
    # Get metadata from current instance
    case "$CLOUD_PROVIDER" in
        aws)
            curl -s "http://169.254.169.254/latest/meta-data/" 2>/dev/null
            ;;
        gcp)
            curl -s -H "Metadata-Flavor: Google" \
                "http://metadata.google.internal/computeMetadata/v1/" 2>/dev/null
            ;;
        azure)
            curl -s -H "Metadata: true" \
                "http://169.254.169.254/metadata/instance?api-version=2021-02-01" 2>/dev/null
            ;;
    esac
}

# Auto-detect cloud provider
detect_cloud_provider() {
    # Check AWS
    if curl -s --max-time 2 "http://169.254.169.254/latest/meta-data/instance-id" &>/dev/null; then
        echo "aws"
        return
    fi
    
    # Check GCP
    if curl -s --max-time 2 \
        -H "Metadata-Flavor: Google" \
        "http://metadata.google.internal/computeMetadata/v1/instance/id" &>/dev/null; then
        echo "gcp"
        return
    fi
    
    # Check Azure
    if curl -s --max-time 2 \
        -H "Metadata: true" \
        "http://169.254.169.254/metadata/instance?api-version=2021-02-01" &>/dev/null; then
        echo "azure"
        return
    fi
    
    echo "unknown"
}

# Cross-cloud DNS management
setup_dns() {
    local domain="$1"
    local ip="$2"
    
    case "$CLOUD_PROVIDER" in
        aws)
            local hosted_zone_id
            hosted_zone_id=$(aws route53 list-hosted-zones-by-name \
                --dns-name "$domain" \
                --query 'HostedZones[0].Id' \
                --output text 2>/dev/null | sed 's|/hostedzone/||')
            
            local change_batch
            change_batch=$(cat << EOF
{
    "Changes": [{
        "Action": "UPSERT",
        "ResourceRecordSet": {
            "Name": "$domain",
            "Type": "A",
            "TTL": 300,
            "ResourceRecords": [{"Value": "$ip"}]
        }
    }]
}
EOF
)
            aws route53 change-resource-record-sets \
                --hosted-zone-id "$hosted_zone_id" \
                --change-batch "$change_batch" 2>/dev/null
            ;;
        gcp)
            gcloud dns record-sets update "$domain." \
                --rrdatas="$ip" \
                --ttl=300 \
                --type=A \
                --zone="$(echo "$domain" | tr '.' '-')" 2>/dev/null
            ;;
    esac
    
    echo "DNS updated: $domain -> $ip on $CLOUD_PROVIDER"
}

# Cost comparison across clouds
compare_cloud_costs() {
    echo "=== Cloud Cost Comparison ==="
    echo ""
    echo "Small instance (2 vCPU, 4GB RAM) - monthly estimate:"
    printf "  %-10s: %s\n" "AWS" "t3.medium ~\$30/month"
    printf "  %-10s: %s\n" "GCP" "e2-medium ~\$28/month"
    printf "  %-10s: %s\n" "Azure" "B2s ~\$35/month"
    
    echo ""
    echo "Storage (100GB) - monthly estimate:"
    printf "  %-10s: %s\n" "AWS" "S3 ~\$2.30/month"
    printf "  %-10s: %s\n" "GCP" "GCS ~\$2.00/month"
    printf "  %-10s: %s\n" "Azure" "Blob ~\$2.10/month"
}

# Demo
echo "=== Multi-Cloud Demo ==="
echo ""
echo "Detecting cloud provider..."
detected=$(detect_cloud_provider)
echo "Current environment: $detected"

echo ""
compare_cloud_costs

echo ""
echo "CLOUD_PROVIDER=$CLOUD_PROVIDER"
echo "Use: CLOUD_PROVIDER=aws cloud_upload_file ./file.txt my-bucket"
```

---

## สรุป Part 27

### สิ่งที่เรียนรู้ (Steps 461-465):

| Step | หัวข้อ | Operations |
|------|--------|------------|
| 461 | AWS CLI | EC2, S3, RDS, Lambda, CloudWatch |
| 462 | AWS Infrastructure | CloudFormation, VPC, Auto Scaling |
| 463 | GCP | Compute, Storage, Cloud Run, Pub/Sub |
| 464 | Azure | VMs, ACR, Blob, App Service, Key Vault |
| 465 | Multi-Cloud | Abstraction layer, cost comparison |

### Key Concepts:
- **Infrastructure as Code**: CloudFormation, ARM Templates
- **Cloud-agnostic**: ใช้ abstraction layer
- **Security**: Key Vault, Secrets Manager, Cloud KMS
- **Cost Optimization**: ตรวจหา unused resources

**ขั้นตอนต่อไป**: Part 28 - CI/CD Pipeline Automation
