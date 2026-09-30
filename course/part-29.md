# Part 29: Infrastructure as Code

## Module 3: Advanced Level — การทำงานระดับสูง

---

## ขั้นตอนที่ 473: Terraform Integration ด้วย Bash

Infrastructure as Code (IaC) ช่วยให้เราจัดการ infrastructure ด้วยโค้ดที่ version control ได้

```bash
#!/bin/bash
# terraform_manager.sh - จัดการ Terraform workflows

# Configuration
TF_DIR="${TF_DIR:-./terraform}"
TF_VARS_FILE="${TF_VARS_FILE:-terraform.tfvars}"
TF_STATE_BACKEND="${TF_STATE_BACKEND:-s3}"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

log_tf() {
    local level="$1"; shift
    local ts=$(date '+%H:%M:%S')
    case "$level" in
        INFO)  echo -e "${BLUE}[${ts}] $*${NC}" ;;
        OK)    echo -e "${GREEN}[${ts}] ✓ $*${NC}" ;;
        WARN)  echo -e "${YELLOW}[${ts}] ⚠ $*${NC}" ;;
        ERROR) echo -e "${RED}[${ts}] ✗ $*${NC}" ;;
    esac
}

# ตรวจสอบ Terraform installation
check_terraform() {
    if ! command -v terraform &>/dev/null; then
        log_tf ERROR "Terraform ไม่ได้ติดตั้ง"
        echo "ติดตั้ง: https://terraform.io/downloads"
        return 1
    fi
    
    local tf_version
    tf_version=$(terraform version -json 2>/dev/null | python3 -c "import json,sys; print(json.load(sys.stdin).get('terraform_version','unknown'))" 2>/dev/null || terraform version | head -1)
    
    log_tf OK "Terraform: ${tf_version}"
}

# Init workspace
tf_init() {
    local env="${1:-default}"
    local workspace="${2:-default}"
    
    log_tf INFO "Initializing Terraform (env: ${env}, workspace: ${workspace})"
    
    cd "$TF_DIR" || return 1
    
    # Init
    terraform init \
        -backend-config="key=${env}/terraform.tfstate" \
        -reconfigure \
        -input=false 2>&1
    
    # เลือก/สร้าง workspace
    if terraform workspace list | grep -q "^[* ] ${workspace}$"; then
        terraform workspace select "$workspace"
        log_tf OK "Switched to workspace: ${workspace}"
    else
        terraform workspace new "$workspace"
        log_tf OK "Created workspace: ${workspace}"
    fi
}

# Validate configuration
tf_validate() {
    log_tf INFO "Validating Terraform configuration..."
    
    cd "$TF_DIR" || return 1
    
    if terraform validate 2>&1; then
        log_tf OK "Validation passed"
        return 0
    else
        log_tf ERROR "Validation failed"
        return 1
    fi
}

# Format check
tf_fmt_check() {
    log_tf INFO "Checking Terraform formatting..."
    
    if terraform fmt -check -recursive "$TF_DIR" 2>&1; then
        log_tf OK "Format check passed"
        return 0
    else
        log_tf WARN "Files not properly formatted"
        
        # Auto-fix
        read -rp "Auto-format files? (y/N): " answer
        if [[ "$answer" =~ ^[Yy]$ ]]; then
            terraform fmt -recursive "$TF_DIR"
            log_tf OK "Files formatted"
        fi
        return 1
    fi
}

# Plan
tf_plan() {
    local env="${1:-default}"
    local plan_file="${2:-/tmp/tfplan-${env}-$$}"
    
    log_tf INFO "Creating Terraform plan (env: ${env})..."
    
    cd "$TF_DIR" || return 1
    
    local var_file="vars/${env}.tfvars"
    local vars_args=()
    
    [[ -f "$var_file" ]] && vars_args+=(-var-file="$var_file")
    [[ -f "$TF_VARS_FILE" ]] && vars_args+=(-var-file="$TF_VARS_FILE")
    
    terraform plan \
        "${vars_args[@]}" \
        -out="$plan_file" \
        -detailed-exitcode \
        -input=false 2>&1
    
    local exit_code=$?
    
    case $exit_code in
        0) log_tf OK "No changes needed" ;;
        1) log_tf ERROR "Plan failed" ;;
        2) log_tf INFO "Plan: changes detected" ;;
    esac
    
    echo "$plan_file"
    return $exit_code
}

# Apply
tf_apply() {
    local plan_file="$1"
    local auto_approve="${2:-false}"
    
    log_tf INFO "Applying Terraform plan..."
    
    cd "$TF_DIR" || return 1
    
    local apply_args=()
    [[ "$auto_approve" == "true" ]] && apply_args+=(-auto-approve)
    
    if terraform apply "${apply_args[@]}" "$plan_file" 2>&1; then
        log_tf OK "Apply completed successfully"
        return 0
    else
        log_tf ERROR "Apply failed"
        return 1
    fi
}

# Destroy
tf_destroy() {
    local env="${1:-default}"
    local auto_approve="${2:-false}"
    
    log_tf WARN "DESTROYING infrastructure (env: ${env})"
    
    if [[ "$auto_approve" != "true" ]]; then
        echo -e "${RED}คุณแน่ใจหรือไม่? (พิมพ์ 'yes' เพื่อยืนยัน)${NC}"
        read -r confirm
        [[ "$confirm" != "yes" ]] && { log_tf INFO "Destroy cancelled"; return 0; }
    fi
    
    cd "$TF_DIR" || return 1
    
    local var_file="vars/${env}.tfvars"
    local vars_args=()
    [[ -f "$var_file" ]] && vars_args+=(-var-file="$var_file")
    
    terraform destroy \
        "${vars_args[@]}" \
        -auto-approve \
        -input=false 2>&1
}

# แสดง outputs
tf_output() {
    local output_name="${1:-}"
    
    cd "$TF_DIR" || return 1
    
    if [[ -n "$output_name" ]]; then
        terraform output -raw "$output_name" 2>/dev/null
    else
        terraform output -json 2>/dev/null | python3 -c "
import json, sys
outputs = json.load(sys.stdin)
print('=== Terraform Outputs ===')
for key, val in outputs.items():
    value = val.get('value', 'N/A')
    sensitive = val.get('sensitive', False)
    display = '(sensitive)' if sensitive else str(value)
    print(f'{key}: {display}')
" 2>/dev/null
    fi
}

# State management
tf_state() {
    local action="$1"; shift
    
    cd "$TF_DIR" || return 1
    
    case "$action" in
        list)
            log_tf INFO "Listing Terraform state resources..."
            terraform state list 2>&1
            ;;
        show)
            local resource="$1"
            log_tf INFO "Showing resource: ${resource}"
            terraform state show "$resource" 2>&1
            ;;
        move)
            local src="$1" dst="$2"
            log_tf INFO "Moving resource: ${src} → ${dst}"
            terraform state mv "$src" "$dst" 2>&1
            ;;
        remove)
            local resource="$1"
            log_tf WARN "Removing from state: ${resource}"
            terraform state rm "$resource" 2>&1
            ;;
        pull)
            terraform state pull 2>&1
            ;;
        push)
            local state_file="$1"
            terraform state push "$state_file" 2>&1
            ;;
    esac
}

# Import existing resource
tf_import() {
    local resource_address="$1"
    local resource_id="$2"
    
    log_tf INFO "Importing resource: ${resource_address} (ID: ${resource_id})"
    
    cd "$TF_DIR" || return 1
    terraform import "$resource_address" "$resource_id" 2>&1
}

# Drift detection
tf_drift_check() {
    local env="${1:-default}"
    
    log_tf INFO "Checking for infrastructure drift (env: ${env})..."
    
    local plan_output
    plan_output=$(tf_plan "$env" "/tmp/tfplan-drift-$$" 2>&1)
    local exit_code=$?
    
    case $exit_code in
        0) log_tf OK "No drift detected" ;;
        1) log_tf ERROR "Plan failed - cannot check drift" ;;
        2)
            log_tf WARN "DRIFT DETECTED! Infrastructure has changed"
            echo ""
            echo "Changes detected:"
            echo "$plan_output" | grep -E "^  [+~-]" | head -20
            ;;
    esac
    
    return $exit_code
}

# สร้าง Terraform files สำหรับ AWS
generate_aws_terraform() {
    local project_name="$1"
    local output_dir="${2:-./terraform}"
    
    mkdir -p "${output_dir}/modules/vpc" "${output_dir}/modules/eks" \
             "${output_dir}/vars" "${output_dir}/envs"
    
    # main.tf
    cat > "${output_dir}/main.tf" << 'HCL'
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
  }
  
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "default/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "terraform"
    }
  }
}
HCL
    
    # variables.tf
    cat > "${output_dir}/variables.tf" << HCL
variable "project_name" {
  description = "ชื่อโปรเจค"
  type        = string
  default     = "${project_name}"
}

variable "environment" {
  description = "Environment (development/staging/production)"
  type        = string
  default     = "development"
  
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment ต้องเป็น development, staging, หรือ production"
  }
}

variable "aws_region" {
  description = "AWS Region"
  type        = string
  default     = "ap-southeast-1"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
  default     = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
}

variable "eks_cluster_version" {
  description = "EKS Kubernetes version"
  type        = string
  default     = "1.28"
}

variable "eks_node_groups" {
  description = "EKS node group configurations"
  type = map(object({
    instance_types = list(string)
    min_size       = number
    max_size       = number
    desired_size   = number
  }))
  default = {
    general = {
      instance_types = ["t3.medium"]
      min_size       = 1
      max_size       = 5
      desired_size   = 2
    }
  }
}
HCL
    
    # vpc module
    cat > "${output_dir}/modules/vpc/main.tf" << 'HCL'
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${var.project_name}-${var.environment}-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.project_name}-${var.environment}-igw"
  }
}

resource "aws_subnet" "public" {
  count = length(var.availability_zones)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.project_name}-${var.environment}-public-${count.index + 1}"
    "kubernetes.io/role/elb" = "1"
  }
}

resource "aws_subnet" "private" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "${var.project_name}-${var.environment}-private-${count.index + 1}"
    "kubernetes.io/role/internal-elb" = "1"
  }
}

resource "aws_nat_gateway" "main" {
  count         = length(var.availability_zones)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  depends_on = [aws_internet_gateway.main]
  
  tags = {
    Name = "${var.project_name}-${var.environment}-nat-${count.index + 1}"
  }
}

resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"
  
  tags = {
    Name = "${var.project_name}-${var.environment}-nat-eip-${count.index + 1}"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "${var.project_name}-${var.environment}-public-rt"
  }
}

resource "aws_route_table" "private" {
  count  = length(var.availability_zones)
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  
  tags = {
    Name = "${var.project_name}-${var.environment}-private-rt-${count.index + 1}"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
HCL
    
    # outputs.tf
    cat > "${output_dir}/outputs.tf" << 'HCL'
output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "eks_cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = module.eks.cluster_endpoint
  sensitive   = false
}

output "eks_cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}
HCL
    
    log_tf OK "สร้าง Terraform files สำหรับ: ${project_name}"
    log_tf OK "Output directory: ${output_dir}"
}

# Main
case "${1:-}" in
    check)    check_terraform ;;
    init)     tf_init "${2:-default}" "${3:-default}" ;;
    validate) tf_validate ;;
    fmt)      tf_fmt_check ;;
    plan)     tf_plan "${2:-default}" "${3:-/tmp/tfplan-$$}" ;;
    apply)    tf_apply "${2}" "${3:-false}" ;;
    destroy)  tf_destroy "${2:-default}" "${3:-false}" ;;
    output)   tf_output "${2:-}" ;;
    state)    tf_state "${2}" "${@:3}" ;;
    import)   tf_import "${2}" "${3}" ;;
    drift)    tf_drift_check "${2:-default}" ;;
    generate) generate_aws_terraform "${2:-myproject}" "${3:-./terraform}" ;;
    *)
        echo "Usage: $0 {check|init|validate|fmt|plan|apply|destroy|output|state|import|drift|generate} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 474: Pulumi Infrastructure as Code

```bash
#!/bin/bash
# pulumi_manager.sh - จัดการ Pulumi

PULUMI_ACCESS_TOKEN="${PULUMI_ACCESS_TOKEN:-}"
PULUMI_STACK="${PULUMI_STACK:-dev}"

# ตรวจสอบ Pulumi
check_pulumi() {
    if ! command -v pulumi &>/dev/null; then
        echo "ERROR: Pulumi ไม่ได้ติดตั้ง"
        echo "ติดตั้ง: curl -fsSL https://get.pulumi.com | sh"
        return 1
    fi
    
    pulumi version
}

# สร้าง Pulumi project (TypeScript)
generate_pulumi_project() {
    local project_name="$1"
    local output_dir="${2:-./${project_name}}"
    
    mkdir -p "$output_dir"
    
    # Pulumi.yaml
    cat > "${output_dir}/Pulumi.yaml" << YAML
name: ${project_name}
runtime: nodejs
description: Infrastructure for ${project_name}
config:
  pulumi:tags:
    value:
      pulumi:template: aws-typescript
YAML
    
    # package.json
    cat > "${output_dir}/package.json" << JSON
{
    "name": "${project_name}",
    "version": "1.0.0",
    "main": "index.ts",
    "scripts": {
        "build": "tsc",
        "preview": "pulumi preview",
        "up": "pulumi up",
        "destroy": "pulumi destroy"
    },
    "dependencies": {
        "@pulumi/aws": "^6.0.0",
        "@pulumi/pulumi": "^3.0.0",
        "@pulumi/awsx": "^2.0.0"
    },
    "devDependencies": {
        "typescript": "^5.0.0",
        "@types/node": "^20.0.0",
        "ts-node": "^10.0.0"
    }
}
JSON
    
    # tsconfig.json
    cat > "${output_dir}/tsconfig.json" << JSON
{
    "compilerOptions": {
        "target": "ES2020",
        "module": "commonjs",
        "strict": true,
        "esModuleInterop": true,
        "outDir": "./bin",
        "rootDir": "."
    }
}
JSON
    
    # index.ts - Main infrastructure
    cat > "${output_dir}/index.ts" << 'TYPESCRIPT'
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";
import * as pulumi from "@pulumi/pulumi";

// Configuration
const config = new pulumi.Config();
const environment = config.require("environment");
const projectName = pulumi.getProject();
const stackName = pulumi.getStack();

// Tags ที่ใช้ทั่วไป
const commonTags: aws.Tags = {
    Environment: environment,
    Project: projectName,
    Stack: stackName,
    ManagedBy: "pulumi",
};

// VPC
const vpc = new awsx.ec2.Vpc(`${projectName}-vpc`, {
    cidrBlock: "10.0.0.0/16",
    numberOfAvailabilityZones: 3,
    natGateways: {
        strategy: environment === "production" ? "OnePerAz" : "Single",
    },
    tags: commonTags,
});

// EKS Cluster
const eksCluster = new aws.eks.Cluster(`${projectName}-eks`, {
    roleArn: eksRole.arn,
    version: "1.28",
    vpcConfig: {
        subnetIds: vpc.privateSubnetIds,
        endpointPrivateAccess: true,
        endpointPublicAccess: environment !== "production",
    },
    enabledClusterLogTypes: [
        "api",
        "audit",
        "authenticator",
        "controllerManager",
        "scheduler",
    ],
    tags: commonTags,
});

// EKS Node Group
const nodeGroup = new aws.eks.NodeGroup(`${projectName}-nodes`, {
    clusterName: eksCluster.name,
    nodeRoleArn: nodeRole.arn,
    subnetIds: vpc.privateSubnetIds,
    scalingConfig: {
        desiredSize: environment === "production" ? 3 : 1,
        minSize: 1,
        maxSize: environment === "production" ? 10 : 3,
    },
    instanceTypes: [
        environment === "production" ? "t3.large" : "t3.medium",
    ],
    labels: {
        environment: environment,
        "node-type": "general",
    },
    tags: commonTags,
});

// RDS Database
const dbSubnetGroup = new aws.rds.SubnetGroup(`${projectName}-db-subnet`, {
    subnetIds: vpc.privateSubnetIds,
    tags: commonTags,
});

const database = new aws.rds.Instance(`${projectName}-db`, {
    engine: "postgres",
    engineVersion: "15.4",
    instanceClass: environment === "production" 
        ? aws.rds.InstanceType.T3_Medium 
        : aws.rds.InstanceType.T3_Micro,
    allocatedStorage: environment === "production" ? 100 : 20,
    dbSubnetGroupName: dbSubnetGroup.name,
    multiAz: environment === "production",
    backupRetentionPeriod: environment === "production" ? 7 : 1,
    deletionProtection: environment === "production",
    skipFinalSnapshot: environment !== "production",
    tags: commonTags,
});

// S3 Buckets
const appBucket = new aws.s3.Bucket(`${projectName}-app-${stackName}`, {
    acl: "private",
    versioning: {
        enabled: true,
    },
    serverSideEncryptionConfiguration: {
        rule: {
            applyServerSideEncryptionByDefault: {
                sseAlgorithm: "AES256",
            },
        },
    },
    lifecycleRules: [
        {
            enabled: true,
            transitions: [
                {
                    days: 30,
                    storageClass: "STANDARD_IA",
                },
                {
                    days: 90,
                    storageClass: "GLACIER",
                },
            ],
        },
    ],
    tags: commonTags,
});

// CloudFront Distribution
const cdn = new aws.cloudfront.Distribution(`${projectName}-cdn`, {
    enabled: true,
    origins: [
        {
            originId: "s3-origin",
            domainName: appBucket.bucketRegionalDomainName,
            s3OriginConfig: {
                originAccessIdentity: "",
            },
        },
    ],
    defaultCacheBehavior: {
        targetOriginId: "s3-origin",
        viewerProtocolPolicy: "redirect-to-https",
        cachePolicyId: "658327ea-f89d-4fab-a63d-7e88639e58f6",
        compress: true,
        allowedMethods: ["GET", "HEAD"],
        cachedMethods: ["GET", "HEAD"],
    },
    priceClass: "PriceClass_100",
    restrictions: {
        geoRestriction: {
            restrictionType: "none",
        },
    },
    viewerCertificate: {
        cloudfrontDefaultCertificate: true,
    },
    tags: commonTags,
});

// Outputs
export const vpcId = vpc.vpcId;
export const privateSubnetIds = vpc.privateSubnetIds;
export const publicSubnetIds = vpc.publicSubnetIds;
export const eksClusterName = eksCluster.name;
export const eksClusterEndpoint = eksCluster.endpoint;
export const databaseEndpoint = database.endpoint;
export const appBucketName = appBucket.bucket;
export const cdnDomain = cdn.domainName;
TYPESCRIPT
    
    echo "✓ สร้าง Pulumi project: ${project_name}"
    echo "  Next steps:"
    echo "  1. cd ${output_dir}"
    echo "  2. npm install"
    echo "  3. pulumi stack init dev"
    echo "  4. pulumi config set environment dev"
    echo "  5. pulumi up"
}

# Pulumi workflow wrapper
pulumi_workflow() {
    local action="$1"
    local stack="${2:-$PULUMI_STACK}"
    
    export PULUMI_ACCESS_TOKEN
    
    case "$action" in
        preview)
            pulumi preview --stack "$stack" --diff
            ;;
        up)
            pulumi up --stack "$stack" --yes
            ;;
        destroy)
            echo "WARNING: กำลังจะลบ infrastructure ทั้งหมด"
            read -rp "พิมพ์ 'yes' เพื่อยืนยัน: " confirm
            [[ "$confirm" == "yes" ]] && pulumi destroy --stack "$stack" --yes
            ;;
        output)
            pulumi stack output --stack "$stack" --json
            ;;
        refresh)
            pulumi refresh --stack "$stack" --yes
            ;;
    esac
}

# Main
case "${1:-}" in
    check)     check_pulumi ;;
    generate)  generate_pulumi_project "${2}" "${3:-}" ;;
    preview)   pulumi_workflow "preview" "${2:-}" ;;
    up)        pulumi_workflow "up" "${2:-}" ;;
    destroy)   pulumi_workflow "destroy" "${2:-}" ;;
    output)    pulumi_workflow "output" "${2:-}" ;;
    refresh)   pulumi_workflow "refresh" "${2:-}" ;;
    *)
        echo "Usage: $0 {check|generate|preview|up|destroy|output|refresh} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 475: Ansible Automation ด้วย Bash

```bash
#!/bin/bash
# ansible_manager.sh - จัดการ Ansible automation

ANSIBLE_DIR="${ANSIBLE_DIR:-./ansible}"
INVENTORY_FILE="${INVENTORY_FILE:-inventory/hosts}"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# ตรวจสอบ Ansible
check_ansible() {
    if ! command -v ansible &>/dev/null; then
        echo "ERROR: Ansible ไม่ได้ติดตั้ง"
        echo "ติดตั้ง: pip install ansible"
        return 1
    fi
    
    ansible --version | head -3
}

# รัน playbook
run_playbook() {
    local playbook="$1"
    local inventory="${2:-$INVENTORY_FILE}"
    local extra_vars="${3:-}"
    local tags="${4:-}"
    local limit="${5:-}"
    
    echo -e "${BLUE}=== Running Playbook: ${playbook} ===${NC}"
    
    local cmd=(ansible-playbook "$playbook" -i "$inventory")
    
    [[ -n "$extra_vars" ]] && cmd+=(--extra-vars "$extra_vars")
    [[ -n "$tags" ]] && cmd+=(--tags "$tags")
    [[ -n "$limit" ]] && cmd+=(--limit "$limit")
    
    # Verbose ถ้าต้องการ
    [[ "${ANSIBLE_VERBOSE:-}" == "true" ]] && cmd+=(-v)
    
    "${cmd[@]}"
}

# Check connectivity
ping_hosts() {
    local inventory="${1:-$INVENTORY_FILE}"
    local group="${2:-all}"
    
    echo "=== Pinging hosts (group: ${group}) ==="
    ansible "$group" -i "$inventory" -m ping
}

# รัน ad-hoc command
run_adhoc() {
    local hosts="$1"
    local module="$2"
    local args="${3:-}"
    local inventory="${4:-$INVENTORY_FILE}"
    
    echo "=== Ad-hoc: ${module} on ${hosts} ==="
    
    local cmd=(ansible "$hosts" -i "$inventory" -m "$module")
    [[ -n "$args" ]] && cmd+=(-a "$args")
    
    "${cmd[@]}"
}

# Gather facts
gather_facts() {
    local hosts="${1:-all}"
    local inventory="${2:-$INVENTORY_FILE}"
    local filter="${3:-ansible_*}"
    
    echo "=== Gathering facts for: ${hosts} ==="
    ansible "$hosts" -i "$inventory" -m setup -a "filter=${filter}"
}

# สร้าง Ansible project structure
generate_ansible_project() {
    local project_name="$1"
    local output_dir="${2:-./${project_name}}"
    
    # สร้าง directory structure
    mkdir -p "${output_dir}/"{inventory/{group_vars,host_vars},roles,playbooks,files,templates,vars}
    
    # ansible.cfg
    cat > "${output_dir}/ansible.cfg" << INI
[defaults]
inventory = inventory/hosts
remote_user = ubuntu
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
stdout_callback = yaml
callbacks_enabled = profile_tasks
roles_path = roles
retry_files_enabled = False
timeout = 30
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible-facts
fact_caching_timeout = 3600

[privilege_escalation]
become = True
become_method = sudo
become_user = root

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
pipelining = True
INI
    
    # Inventory
    cat > "${output_dir}/inventory/hosts" << INI
[all:vars]
ansible_python_interpreter=/usr/bin/python3

[web]
web-01 ansible_host=10.0.1.10
web-02 ansible_host=10.0.1.11
web-03 ansible_host=10.0.1.12

[db]
db-primary ansible_host=10.0.2.10
db-replica ansible_host=10.0.2.11

[lb]
lb-01 ansible_host=10.0.0.10

[monitoring]
monitor-01 ansible_host=10.0.3.10

[production:children]
web
db
lb

[staging:children]
web
INI
    
    # group_vars
    cat > "${output_dir}/inventory/group_vars/all.yml" << YAML
---
# ตัวแปรที่ใช้ทั่วไป
project_name: ${project_name}
environment: "{{ lookup('env', 'ENVIRONMENT') | default('staging') }}"

# Package management
apt_packages:
  - vim
  - curl
  - wget
  - htop
  - git
  - python3-pip
  - fail2ban
  - ufw

# Security settings
allowed_ssh_users:
  - ubuntu
  - deploy

firewall_allowed_ports:
  - "22/tcp"
  - "80/tcp"
  - "443/tcp"

# NTP settings
ntp_servers:
  - 0.th.pool.ntp.org
  - 1.th.pool.ntp.org
YAML
    
    cat > "${output_dir}/inventory/group_vars/web.yml" << YAML
---
# Web server configuration
nginx_worker_processes: auto
nginx_worker_connections: 1024

app_port: 8080
app_workers: 4

# SSL settings
ssl_enabled: true
ssl_cert_path: /etc/ssl/certs/app.crt
ssl_key_path: /etc/ssl/private/app.key
YAML
    
    # สร้าง site.yml
    cat > "${output_dir}/playbooks/site.yml" << YAML
---
- name: Configure all servers
  hosts: all
  roles:
    - common
    - security

- name: Configure web servers
  hosts: web
  roles:
    - nginx
    - app

- name: Configure database servers
  hosts: db
  roles:
    - postgresql

- name: Configure load balancers
  hosts: lb
  roles:
    - haproxy

- name: Configure monitoring
  hosts: monitoring
  roles:
    - prometheus
    - grafana
YAML
    
    # สร้าง common role
    mkdir -p "${output_dir}/roles/common/"{tasks,handlers,templates,files,defaults,vars,meta}
    
    cat > "${output_dir}/roles/common/tasks/main.yml" << YAML
---
- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600
  when: ansible_os_family == "Debian"

- name: Install common packages
  apt:
    name: "{{ apt_packages }}"
    state: present
  when: ansible_os_family == "Debian"

- name: Set timezone
  timezone:
    name: Asia/Bangkok

- name: Configure NTP
  include_tasks: ntp.yml

- name: Set hostname
  hostname:
    name: "{{ inventory_hostname }}"

- name: Configure /etc/hosts
  template:
    src: hosts.j2
    dest: /etc/hosts
    owner: root
    group: root
    mode: '0644'

- name: Create deploy user
  user:
    name: deploy
    shell: /bin/bash
    groups: sudo
    append: yes
    create_home: yes
    state: present

- name: Configure sudoers for deploy
  copy:
    content: "deploy ALL=(ALL) NOPASSWD:ALL\n"
    dest: /etc/sudoers.d/deploy
    mode: '0440'
    validate: /usr/sbin/visudo -cf %s

- name: Set kernel parameters
  sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    state: present
    reload: yes
  loop:
    - { name: 'net.core.somaxconn', value: '65535' }
    - { name: 'net.ipv4.tcp_max_syn_backlog', value: '65535' }
    - { name: 'vm.swappiness', value: '10' }
    - { name: 'fs.file-max', value: '2097152' }
YAML
    
    cat > "${output_dir}/roles/common/tasks/ntp.yml" << YAML
---
- name: Install chrony
  apt:
    name: chrony
    state: present

- name: Configure chrony
  template:
    src: chrony.conf.j2
    dest: /etc/chrony/chrony.conf
    owner: root
    group: root
    mode: '0644'
  notify: restart chrony

- name: Enable chrony service
  service:
    name: chrony
    state: started
    enabled: yes
YAML
    
    cat > "${output_dir}/roles/common/handlers/main.yml" << YAML
---
- name: restart chrony
  service:
    name: chrony
    state: restarted

- name: restart nginx
  service:
    name: nginx
    state: restarted

- name: reload nginx
  service:
    name: nginx
    state: reloaded
YAML
    
    # Security role
    mkdir -p "${output_dir}/roles/security/"{tasks,templates}
    
    cat > "${output_dir}/roles/security/tasks/main.yml" << YAML
---
- name: Configure UFW firewall
  include_tasks: firewall.yml

- name: Configure fail2ban
  include_tasks: fail2ban.yml

- name: Configure SSH hardening
  include_tasks: ssh_hardening.yml

- name: Configure auditd
  include_tasks: auditd.yml
YAML
    
    cat > "${output_dir}/roles/security/tasks/firewall.yml" << YAML
---
- name: Install UFW
  apt:
    name: ufw
    state: present

- name: Set default policies
  ufw:
    default: deny
    direction: incoming

- name: Allow outgoing
  ufw:
    default: allow
    direction: outgoing

- name: Allow specified ports
  ufw:
    rule: allow
    port: "{{ item.split('/')[0] }}"
    proto: "{{ item.split('/')[1] }}"
  loop: "{{ firewall_allowed_ports }}"

- name: Enable UFW
  ufw:
    state: enabled
    logging: low
YAML
    
    cat > "${output_dir}/roles/security/tasks/ssh_hardening.yml" << YAML
---
- name: Configure SSH
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
    state: present
  loop:
    - { regexp: '^#?PermitRootLogin', line: 'PermitRootLogin no' }
    - { regexp: '^#?PasswordAuthentication', line: 'PasswordAuthentication no' }
    - { regexp: '^#?X11Forwarding', line: 'X11Forwarding no' }
    - { regexp: '^#?MaxAuthTries', line: 'MaxAuthTries 3' }
    - { regexp: '^#?LoginGraceTime', line: 'LoginGraceTime 30' }
    - { regexp: '^#?AllowUsers', line: "AllowUsers {{ allowed_ssh_users | join(' ') }}" }
  notify: restart sshd
YAML
    
    # deploy.sh helper script
    cat > "${output_dir}/deploy.sh" << 'BASH'
#!/bin/bash
# deploy.sh - Ansible deployment helper

set -euo pipefail

ANSIBLE_DIR="$(cd "$(dirname "$0")" && pwd)"
ENVIRONMENT="${ENVIRONMENT:-staging}"
PLAYBOOK="${1:-playbooks/site.yml}"
LIMIT="${LIMIT:-}"
TAGS="${TAGS:-}"

# Export environment
export ANSIBLE_FORCE_COLOR=1

echo "=== Deploying with Ansible ==="
echo "Environment: ${ENVIRONMENT}"
echo "Playbook:    ${PLAYBOOK}"
[[ -n "$LIMIT" ]] && echo "Limit:       ${LIMIT}"
[[ -n "$TAGS" ]] && echo "Tags:        ${TAGS}"
echo ""

# Build command
CMD=(ansible-playbook "${PLAYBOOK}" 
    -i "inventory/hosts"
    --extra-vars "environment=${ENVIRONMENT}")

[[ -n "$LIMIT" ]] && CMD+=(--limit "$LIMIT")
[[ -n "$TAGS" ]] && CMD+=(--tags "$TAGS")

# Run
cd "$ANSIBLE_DIR"
"${CMD[@]}"

echo ""
echo "✓ Deployment completed"
BASH
    
    chmod +x "${output_dir}/deploy.sh"
    
    echo "✓ สร้าง Ansible project: ${project_name}"
    echo ""
    echo "Structure:"
    find "$output_dir" -type f | sort
}

# Main
case "${1:-}" in
    check)     check_ansible ;;
    ping)      ping_hosts "${2:-inventory/hosts}" "${3:-all}" ;;
    run)       run_playbook "${2}" "${3:-}" "${4:-}" "${5:-}" "${6:-}" ;;
    adhoc)     run_adhoc "${2}" "${3}" "${4:-}" "${5:-}" ;;
    facts)     gather_facts "${2:-all}" "${3:-}" "${4:-}" ;;
    generate)  generate_ansible_project "${2}" "${3:-}" ;;
    *)
        echo "Usage: $0 {check|ping|run|adhoc|facts|generate} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 476: Packer Image Building

```bash
#!/bin/bash
# packer_builder.sh - สร้าง Machine Images ด้วย Packer

PACKER_DIR="${PACKER_DIR:-./packer}"

# ตรวจสอบ Packer
check_packer() {
    if ! command -v packer &>/dev/null; then
        echo "ERROR: Packer ไม่ได้ติดตั้ง"
        echo "ติดตั้ง: https://packer.io/downloads"
        return 1
    fi
    packer version
}

# สร้าง Packer template สำหรับ AWS AMI
generate_aws_ami_template() {
    local app_name="$1"
    local base_ami="${2:-ami-0c55b159cbfafe1f0}"  # Amazon Linux 2
    local output_dir="${3:-${PACKER_DIR}}"
    
    mkdir -p "$output_dir"
    
    # HCL2 template
    cat > "${output_dir}/${app_name}.pkr.hcl" << HCL
packer {
  required_version = ">= 1.9.0"
  required_plugins {
    amazon = {
      version = ">= 1.3.0"
      source  = "github.com/hashicorp/amazon"
    }
    ansible = {
      version = ">= 1.1.0"
      source  = "github.com/hashicorp/ansible"
    }
  }
}

# Variables
variable "app_name" {
  type    = string
  default = "${app_name}"
}

variable "aws_region" {
  type    = string
  default = "ap-southeast-1"
}

variable "instance_type" {
  type    = string
  default = "t3.medium"
}

variable "base_ami" {
  type    = string
  default = "${base_ami}"
}

variable "app_version" {
  type    = string
  default = "latest"
}

# Data source - ค้นหา latest AMI
data "amazon-ami" "ubuntu" {
  filters = {
    name                = "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
    root-device-type    = "ebs"
    virtualization-type = "hvm"
  }
  most_recent = true
  owners      = ["099720109477"]
  region      = var.aws_region
}

# Build definition
source "amazon-ebs" "app" {
  ami_name             = "\${var.app_name}-\${var.app_version}-\${formatdate("YYYYMMDD-HHmmss", timestamp())}"
  instance_type        = var.instance_type
  region               = var.aws_region
  source_ami           = data.amazon-ami.ubuntu.id
  ssh_username         = "ubuntu"
  
  # Tags
  ami_description = "Custom AMI for \${var.app_name}"
  tags = {
    Name        = var.app_name
    Version     = var.app_version
    BuildDate   = formatdate("YYYY-MM-DD", timestamp())
    ManagedBy   = "packer"
  }
  
  # Root volume
  launch_block_device_mappings {
    device_name           = "/dev/sda1"
    volume_size           = 30
    volume_type           = "gp3"
    delete_on_termination = true
    encrypted             = true
  }
  
  # Build VPC settings
  vpc_filter {
    filters = {
      "tag:Name" = "build-vpc"
      "isDefault" = "false"
    }
  }
  
  subnet_filter {
    filters = {
      "tag:Name" = "build-subnet"
    }
    most_free = true
    random    = false
  }
}

# Build
build {
  name = var.app_name
  
  sources = ["source.amazon-ebs.app"]
  
  # Step 1: ติดตั้ง packages พื้นฐาน
  provisioner "shell" {
    inline = [
      "sudo apt-get update -y",
      "sudo apt-get upgrade -y",
      "sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release",
      "sudo apt-get install -y vim htop tmux jq awscli",
      "sudo apt-get install -y fail2ban ufw",
    ]
  }
  
  # Step 2: ติดตั้ง Docker
  provisioner "shell" {
    script = "scripts/install_docker.sh"
  }
  
  # Step 3: ติดตั้ง Kubernetes tools
  provisioner "shell" {
    script = "scripts/install_k8s_tools.sh"
  }
  
  # Step 4: Config ด้วย Ansible
  provisioner "ansible" {
    playbook_file = "ansible/configure.yml"
    extra_arguments = [
      "--extra-vars", "app_name=\${var.app_name}",
    ]
  }
  
  # Step 5: Copy files
  provisioner "file" {
    source      = "files/"
    destination = "/tmp/"
  }
  
  # Step 6: Final configuration
  provisioner "shell" {
    inline = [
      # Cleanup
      "sudo apt-get autoremove -y",
      "sudo apt-get autoclean -y",
      "sudo rm -rf /tmp/* /var/tmp/*",
      "sudo rm -rf /var/log/*.gz /var/log/**/*.gz",
      
      # Clear history
      "history -c",
      "cat /dev/null > ~/.bash_history",
      
      # Setup SSH
      "sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config",
      
      # Enable UFW
      "sudo ufw default deny incoming",
      "sudo ufw default allow outgoing",
      "sudo ufw allow 22/tcp",
      "sudo ufw --force enable",
    ]
  }
  
  # Post-processor: สร้าง manifest
  post-processor "manifest" {
    output     = "builds/manifest.json"
    strip_path = true
  }
}
HCL
    
    # สร้าง scripts
    mkdir -p "${output_dir}/scripts"
    
    cat > "${output_dir}/scripts/install_docker.sh" << 'BASH'
#!/bin/bash
set -euo pipefail

# ติดตั้ง Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
    https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# เพิ่ม ubuntu user เข้า docker group
sudo usermod -aG docker ubuntu

# เปิด Docker service
sudo systemctl enable docker
sudo systemctl start docker

echo "✓ Docker installed: $(docker --version)"
BASH
    
    cat > "${output_dir}/scripts/install_k8s_tools.sh" << 'BASH'
#!/bin/bash
set -euo pipefail

K8S_VERSION="v1.28"

# ติดตั้ง kubectl
curl -LO "https://dl.k8s.io/release/${K8S_VERSION}.0/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# ติดตั้ง Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# ติดตั้ง k9s
K9S_VERSION="v0.27.4"
curl -L "https://github.com/derailed/k9s/releases/download/${K9S_VERSION}/k9s_Linux_amd64.tar.gz" | \
    sudo tar -xz -C /usr/local/bin k9s

echo "✓ kubectl: $(kubectl version --client --short)"
echo "✓ Helm: $(helm version --short)"
BASH
    
    chmod +x "${output_dir}/scripts/"*.sh
    
    echo "✓ สร้าง Packer template: ${app_name}"
    echo "Usage:"
    echo "  packer init ${output_dir}/${app_name}.pkr.hcl"
    echo "  packer build -var 'app_version=1.0.0' ${output_dir}/${app_name}.pkr.hcl"
}

# Build image
build_image() {
    local template="$1"
    local vars="${@:2}"
    
    echo "=== Building AMI: ${template} ==="
    
    # Validate
    packer validate "$template"
    
    # Build
    local cmd=(packer build)
    
    for var in $vars; do
        cmd+=(-var "$var")
    done
    
    cmd+=("$template")
    
    "${cmd[@]}"
}

# Main
case "${1:-}" in
    check)    check_packer ;;
    generate) generate_aws_ami_template "${2}" "${3:-}" "${4:-}" ;;
    build)    build_image "${@:2}" ;;
    *)
        echo "Usage: $0 {check|generate|build} [args]"
        ;;
esac
```

---

## ขั้นตอนที่ 477: Infrastructure Testing ด้วย Bash

```bash
#!/bin/bash
# infra_testing.sh - ทดสอบ Infrastructure

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# Test results
TESTS_PASSED=0
TESTS_FAILED=0
declare -a FAILED_TESTS=()

# Assert functions
assert_equal() {
    local desc="$1"
    local expected="$2"
    local actual="$3"
    
    if [[ "$expected" == "$actual" ]]; then
        echo -e "${GREEN}  ✓ ${desc}${NC}"
        ((TESTS_PASSED++))
    else
        echo -e "${RED}  ✗ ${desc}${NC}"
        echo -e "${RED}    Expected: ${expected}${NC}"
        echo -e "${RED}    Actual:   ${actual}${NC}"
        ((TESTS_FAILED++))
        FAILED_TESTS+=("$desc")
    fi
}

assert_true() {
    local desc="$1"
    local condition="$2"
    
    if eval "$condition"; then
        echo -e "${GREEN}  ✓ ${desc}${NC}"
        ((TESTS_PASSED++))
    else
        echo -e "${RED}  ✗ ${desc}${NC}"
        ((TESTS_FAILED++))
        FAILED_TESTS+=("$desc")
    fi
}

assert_command_success() {
    local desc="$1"
    local command="$2"
    
    if eval "$command" &>/dev/null; then
        echo -e "${GREEN}  ✓ ${desc}${NC}"
        ((TESTS_PASSED++))
    else
        echo -e "${RED}  ✗ ${desc}${NC}"
        ((TESTS_FAILED++))
        FAILED_TESTS+=("$desc")
    fi
}

# === Infrastructure Tests ===

test_kubernetes_cluster() {
    echo -e "\n${BLUE}=== Testing Kubernetes Cluster ===${NC}"
    
    # ตรวจสอบ cluster connection
    assert_command_success "kubectl cluster accessible" \
        "kubectl cluster-info --request-timeout=5s"
    
    # ตรวจสอบ nodes
    local node_count
    node_count=$(kubectl get nodes --no-headers 2>/dev/null | wc -l || echo "0")
    
    assert_true "Cluster has nodes" "[[ $node_count -gt 0 ]]"
    
    # ตรวจสอบ node readiness
    local not_ready
    not_ready=$(kubectl get nodes --no-headers 2>/dev/null | grep -v "Ready" | wc -l || echo "0")
    
    assert_equal "All nodes are Ready" "0" "$not_ready"
    
    # ตรวจสอบ system namespaces
    for ns in kube-system kube-public default; do
        assert_command_success "Namespace '${ns}' exists" \
            "kubectl get namespace ${ns}"
    done
    
    # ตรวจสอบ core services
    for svc in kubernetes; do
        assert_command_success "Service '${svc}' exists in default" \
            "kubectl get service ${svc} -n default"
    done
}

test_deployments() {
    local namespace="${1:-default}"
    
    echo -e "\n${BLUE}=== Testing Deployments (namespace: ${namespace}) ===${NC}"
    
    # รับรายการ deployments
    local deployments
    deployments=$(kubectl get deployments -n "$namespace" --no-headers -o name 2>/dev/null)
    
    if [[ -z "$deployments" ]]; then
        echo -e "${YELLOW}  No deployments found in ${namespace}${NC}"
        return
    fi
    
    while IFS= read -r deployment; do
        local name="${deployment#deployment.apps/}"
        
        # ตรวจสอบ desired vs available
        local desired available
        desired=$(kubectl get deployment "$name" -n "$namespace" \
            -o jsonpath='{.spec.replicas}' 2>/dev/null || echo "0")
        available=$(kubectl get deployment "$name" -n "$namespace" \
            -o jsonpath='{.status.availableReplicas}' 2>/dev/null || echo "0")
        
        assert_equal "Deployment '${name}' fully available" \
            "$desired" "${available:-0}"
    done <<< "$deployments"
}

test_services() {
    local namespace="${1:-default}"
    
    echo -e "\n${BLUE}=== Testing Services (namespace: ${namespace}) ===${NC}"
    
    local services
    services=$(kubectl get services -n "$namespace" --no-headers -o name 2>/dev/null)
    
    while IFS= read -r service; do
        local name="${service#service/}"
        [[ "$name" == "kubernetes" ]] && continue
        
        # ตรวจสอบ endpoints
        local endpoints
        endpoints=$(kubectl get endpoints "$name" -n "$namespace" \
            -o jsonpath='{.subsets[0].addresses}' 2>/dev/null)
        
        assert_true "Service '${name}' has endpoints" \
            "[[ -n '$endpoints' ]]"
    done <<< "$services"
}

test_persistent_volumes() {
    echo -e "\n${BLUE}=== Testing Persistent Volumes ===${NC}"
    
    # ตรวจสอบ PVs
    local bound_pvs
    bound_pvs=$(kubectl get pv --no-headers 2>/dev/null | \
        grep "Bound" | wc -l || echo "0")
    
    local failed_pvs
    failed_pvs=$(kubectl get pv --no-headers 2>/dev/null | \
        grep -E "Failed|Released" | wc -l || echo "0")
    
    echo "  Bound PVs: ${bound_pvs}"
    assert_equal "No failed PVs" "0" "$failed_pvs"
    
    # ตรวจสอบ PVCs
    local pending_pvcs
    pending_pvcs=$(kubectl get pvc --all-namespaces --no-headers 2>/dev/null | \
        grep "Pending" | wc -l || echo "0")
    
    assert_equal "No pending PVCs" "0" "$pending_pvcs"
}

test_network_policies() {
    local namespace="${1:-default}"
    
    echo -e "\n${BLUE}=== Testing Network Policies ===${NC}"
    
    local policy_count
    policy_count=$(kubectl get networkpolicies -n "$namespace" --no-headers 2>/dev/null | wc -l || echo "0")
    
    echo "  Network policies: ${policy_count}"
    
    assert_true "Network policies exist" "[[ $policy_count -gt 0 ]]"
}

test_aws_infrastructure() {
    echo -e "\n${BLUE}=== Testing AWS Infrastructure ===${NC}"
    
    if ! command -v aws &>/dev/null; then
        echo -e "${YELLOW}  AWS CLI not available, skipping${NC}"
        return
    fi
    
    # ตรวจสอบ AWS credentials
    assert_command_success "AWS credentials valid" \
        "aws sts get-caller-identity"
    
    # ตรวจสอบ VPC
    local vpc_count
    vpc_count=$(aws ec2 describe-vpcs --query 'length(Vpcs)' --output text 2>/dev/null || echo "0")
    
    assert_true "At least one VPC exists" "[[ $vpc_count -gt 0 ]]"
    
    # ตรวจสอบ running instances
    local running_instances
    running_instances=$(aws ec2 describe-instances \
        --filters "Name=instance-state-name,Values=running" \
        --query 'length(Reservations[].Instances[])' \
        --output text 2>/dev/null || echo "0")
    
    echo "  Running EC2 instances: ${running_instances}"
}

test_http_endpoints() {
    local -a endpoints=("${@}")
    
    echo -e "\n${BLUE}=== Testing HTTP Endpoints ===${NC}"
    
    for endpoint in "${endpoints[@]}"; do
        local url="${endpoint%%:*}"
        local expected_code="${endpoint##*:}"
        [[ "$url" == "$endpoint" ]] && expected_code="200"
        
        local actual_code
        actual_code=$(curl -so /dev/null -w "%{http_code}" \
            --connect-timeout 5 \
            --max-time 10 \
            "$url" 2>/dev/null || echo "000")
        
        assert_equal "HTTP ${url} returns ${expected_code}" \
            "$expected_code" "$actual_code"
    done
}

test_ssl_certificates() {
    local -a hosts=("${@}")
    
    echo -e "\n${BLUE}=== Testing SSL Certificates ===${NC}"
    
    for host in "${hosts[@]}"; do
        # ตรวจสอบ expiry
        local expiry_date
        expiry_date=$(echo | openssl s_client -servername "$host" \
            -connect "${host}:443" 2>/dev/null | \
            openssl x509 -noout -enddate 2>/dev/null | \
            sed 's/notAfter=//')
        
        if [[ -n "$expiry_date" ]]; then
            local expiry_ts expiry_days
            expiry_ts=$(date -d "$expiry_date" +%s 2>/dev/null || echo "0")
            expiry_days=$(( (expiry_ts - $(date +%s)) / 86400 ))
            
            assert_true "SSL cert for '${host}' expires in >30 days (${expiry_days} days)" \
                "[[ $expiry_days -gt 30 ]]"
        else
            echo -e "${YELLOW}  Could not check SSL for: ${host}${NC}"
        fi
    done
}

# แสดง test summary
show_test_summary() {
    local total=$((TESTS_PASSED + TESTS_FAILED))
    
    echo ""
    echo "======================================"
    echo "       INFRASTRUCTURE TEST SUMMARY"
    echo "======================================"
    echo "Total:  ${total}"
    echo -e "${GREEN}Passed: ${TESTS_PASSED}${NC}"
    echo -e "${RED}Failed: ${TESTS_FAILED}${NC}"
    
    if [[ ${#FAILED_TESTS[@]} -gt 0 ]]; then
        echo ""
        echo "Failed tests:"
        for test in "${FAILED_TESTS[@]}"; do
            echo -e "  ${RED}✗ ${test}${NC}"
        done
    fi
    
    echo "======================================"
    
    [[ $TESTS_FAILED -eq 0 ]] && return 0 || return 1
}

# Main test runner
main() {
    echo "=== Infrastructure Testing ==="
    echo "Started: $(date)"
    
    # รัน tests
    test_kubernetes_cluster
    test_deployments "production"
    test_services "production"
    test_persistent_volumes
    test_network_policies "production"
    test_aws_infrastructure
    
    # HTTP endpoint tests
    test_http_endpoints \
        "https://api.example.com/health:200" \
        "https://example.com:200"
    
    # SSL tests
    test_ssl_certificates "example.com" "api.example.com"
    
    # แสดงสรุป
    show_test_summary
}

main "$@"
```

---

## Workshop: Complete IaC Pipeline

```bash
#!/bin/bash
# iac_pipeline.sh - Workshop: Complete Infrastructure as Code Pipeline

set -euo pipefail

# Configuration
PROJECT_NAME="${PROJECT_NAME:-myproject}"
ENVIRONMENT="${ENVIRONMENT:-staging}"
TF_DIR="./terraform"
ANSIBLE_DIR="./ansible"
PACKER_DIR="./packer"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
NC='\033[0m'

log_iac() {
    local level="$1"; shift
    local ts=$(date '+%H:%M:%S')
    case "$level" in
        INFO)  echo -e "${BLUE}[${ts}][IaC] $*${NC}" ;;
        OK)    echo -e "${GREEN}[${ts}][IaC] ✓ $*${NC}" ;;
        WARN)  echo -e "${YELLOW}[${ts}][IaC] ⚠ $*${NC}" ;;
        ERROR) echo -e "${RED}[${ts}][IaC] ✗ $*${NC}" ;;
        STAGE) echo -e "\n${CYAN}${BOLD}[${ts}] ▶ $*${NC}" ;;
    esac
}

# Stage 1: Validate configurations
stage_validate() {
    log_iac STAGE "VALIDATE - ตรวจสอบ configurations"
    
    local errors=0
    
    # ตรวจสอบ Terraform
    if [[ -d "$TF_DIR" ]]; then
        log_iac INFO "Validating Terraform..."
        if cd "$TF_DIR" && terraform validate 2>&1; then
            log_iac OK "Terraform valid"
        else
            log_iac ERROR "Terraform validation failed"
            ((errors++))
        fi
        cd - > /dev/null
    fi
    
    # ตรวจสอบ Ansible
    if [[ -d "$ANSIBLE_DIR" ]]; then
        log_iac INFO "Validating Ansible..."
        if ansible-playbook --syntax-check "${ANSIBLE_DIR}/site.yml" &>/dev/null; then
            log_iac OK "Ansible syntax valid"
        else
            log_iac WARN "Ansible syntax check failed (non-blocking)"
        fi
    fi
    
    [[ $errors -gt 0 ]] && return 1 || return 0
}

# Stage 2: Plan infrastructure changes
stage_plan() {
    log_iac STAGE "PLAN - วางแผนการเปลี่ยนแปลง"
    
    if [[ -d "$TF_DIR" ]]; then
        log_iac INFO "Creating Terraform plan for ${ENVIRONMENT}..."
        
        cd "$TF_DIR"
        
        # Init ถ้ายังไม่ได้ทำ
        if [[ ! -d ".terraform" ]]; then
            terraform init -input=false 2>&1
        fi
        
        # Plan
        local plan_file="/tmp/tfplan-${PROJECT_NAME}-${ENVIRONMENT}-$$"
        
        terraform plan \
            -var "environment=${ENVIRONMENT}" \
            -var "project_name=${PROJECT_NAME}" \
            -out="$plan_file" \
            -input=false 2>&1 || true
        
        export TF_PLAN_FILE="$plan_file"
        
        cd - > /dev/null
    fi
    
    log_iac OK "Planning complete"
}

# Stage 3: Security scan
stage_security_scan() {
    log_iac STAGE "SECURITY SCAN - ตรวจสอบความปลอดภัย"
    
    # Scan Terraform configs
    if command -v tfsec &>/dev/null; then
        log_iac INFO "Running tfsec..."
        tfsec "$TF_DIR" --no-color 2>&1 | tail -20
    fi
    
    # Scan for secrets
    log_iac INFO "Scanning for secrets..."
    local secret_files=0
    
    while IFS= read -r file; do
        if grep -lE "(password|secret|api_key|private_key)\s*=\s*['\"][^'\"]{8,}" \
            "$file" 2>/dev/null; then
            ((secret_files++))
        fi
    done < <(find . -name "*.tf" -o -name "*.yml" -o -name "*.yaml" 2>/dev/null | head -50)
    
    if [[ $secret_files -gt 0 ]]; then
        log_iac WARN "Potential secrets found in ${secret_files} files"
    else
        log_iac OK "No secrets detected"
    fi
}

# Stage 4: Apply infrastructure
stage_apply() {
    log_iac STAGE "APPLY - ปรับใช้ infrastructure"
    
    if [[ -d "$TF_DIR" ]] && [[ -n "${TF_PLAN_FILE:-}" ]]; then
        log_iac INFO "Applying Terraform plan..."
        
        cd "$TF_DIR"
        terraform apply -auto-approve "$TF_PLAN_FILE" 2>&1
        cd - > /dev/null
        
        log_iac OK "Terraform applied"
    fi
}

# Stage 5: Configure with Ansible
stage_configure() {
    log_iac STAGE "CONFIGURE - ตั้งค่าด้วย Ansible"
    
    if [[ -d "$ANSIBLE_DIR" ]]; then
        log_iac INFO "Running Ansible playbook..."
        
        ansible-playbook \
            "${ANSIBLE_DIR}/site.yml" \
            -i "${ANSIBLE_DIR}/inventory/hosts" \
            --extra-vars "environment=${ENVIRONMENT}" \
            2>&1 || true
        
        log_iac OK "Ansible configuration applied"
    fi
}

# Stage 6: Test infrastructure
stage_test() {
    log_iac STAGE "TEST - ทดสอบ infrastructure"
    
    # รัน infrastructure tests
    if command -v kubectl &>/dev/null; then
        log_iac INFO "Testing Kubernetes cluster..."
        kubectl get nodes 2>/dev/null | head -5 || true
    fi
    
    log_iac OK "Infrastructure tests passed"
}

# Main IaC pipeline
main() {
    echo ""
    echo -e "${BOLD}${CYAN}"
    echo "╔════════════════════════════════════╗"
    echo "║   INFRASTRUCTURE AS CODE PIPELINE  ║"
    echo "╚════════════════════════════════════╝"
    echo -e "${NC}"
    echo "Project:     ${PROJECT_NAME}"
    echo "Environment: ${ENVIRONMENT}"
    echo "Started:     $(date)"
    echo ""
    
    local pipeline_start=$(date +%s)
    
    # รัน stages
    local stages=(
        "stage_validate"
        "stage_plan"
        "stage_security_scan"
    )
    
    for stage in "${stages[@]}"; do
        if ! $stage; then
            log_iac ERROR "Pipeline failed at: ${stage}"
            exit 1
        fi
    done
    
    # Apply - require confirmation ถ้าไม่ได้ auto approve
    if [[ "${AUTO_APPLY:-false}" != "true" ]]; then
        echo ""
        echo -e "${YELLOW}Ready to apply changes to ${ENVIRONMENT}.${NC}"
        read -rp "Continue? (yes/no): " confirm
        
        if [[ "$confirm" != "yes" ]]; then
            log_iac INFO "Pipeline aborted by user"
            exit 0
        fi
    fi
    
    stage_apply
    stage_configure
    stage_test
    
    local pipeline_duration=$(($(date +%s) - pipeline_start))
    
    echo ""
    echo -e "${GREEN}${BOLD}"
    echo "╔════════════════════════════════════╗"
    echo "║      PIPELINE COMPLETED ✓          ║"
    echo "╚════════════════════════════════════╝"
    echo -e "${NC}"
    echo "Duration: ${pipeline_duration}s"
}

# Entry point
case "${1:-run}" in
    run)      main ;;
    validate) stage_validate ;;
    plan)     stage_plan ;;
    apply)    stage_apply ;;
    test)     stage_test ;;
    *)
        echo "Usage: $0 {run|validate|plan|apply|test}"
        ;;
esac
```

---

## สรุป Part 29

| หัวข้อ | เนื้อหา |
|--------|---------|
| **Terraform** | Init, plan, apply, destroy, state management, drift detection |
| **Terraform AWS** | VPC module, EKS, variables, outputs |
| **Pulumi** | TypeScript-based IaC, AWS resources |
| **Ansible** | Playbooks, roles, inventory, hardening |
| **Packer** | AMI building, provisioners, scripts |
| **Infra Testing** | Kubernetes, services, PVs, HTTP, SSL |
| **IaC Pipeline** | Complete validate→plan→scan→apply→configure→test |

**ขั้นตอนต่อไป**: Part 30 - Advanced Networking และ Service Mesh
