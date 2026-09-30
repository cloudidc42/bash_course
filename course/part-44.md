# Part 44: ML/AI Infrastructure Engineering

## Module 4: Professional Level — Machine Learning Platform

### ขั้นตอนที่ 530: MLflow Model Management Platform

**MLflow** สำหรับ experiment tracking, model registry, และ deployment lifecycle

```bash
#!/bin/bash
# mlflow-platform-manager.sh - MLflow ML Platform Management

set -euo pipefail

MLFLOW_NAMESPACE="${MLFLOW_NAMESPACE:-mlflow}"
MLFLOW_RELEASE="${MLFLOW_RELEASE:-mlflow}"
MLFLOW_ARTIFACT_BUCKET="${MLFLOW_ARTIFACT_BUCKET:-mlflow-artifacts}"
MLFLOW_DB_URI="${MLFLOW_DB_URI:-postgresql://mlflow:password@postgres:5432/mlflow}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_mlflow() {
    log "Installing MLflow Tracking Server..."
    
    kubectl create namespace "${MLFLOW_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    kubectl create secret generic mlflow-secrets \
        --from-literal=db-uri="${MLFLOW_DB_URI}" \
        --from-literal=aws-access-key-id="${AWS_ACCESS_KEY_ID:-}" \
        --from-literal=aws-secret-access-key="${AWS_SECRET_ACCESS_KEY:-}" \
        -n "${MLFLOW_NAMESPACE}" \
        --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-server
  namespace: ${MLFLOW_NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mlflow-server
  template:
    metadata:
      labels:
        app: mlflow-server
    spec:
      containers:
        - name: mlflow
          image: ghcr.io/mlflow/mlflow:v2.9.2
          ports:
            - containerPort: 5000
          command:
            - mlflow
            - server
            - --host=0.0.0.0
            - --port=5000
            - --backend-store-uri=\$(MLFLOW_DB_URI)
            - --default-artifact-root=s3://${MLFLOW_ARTIFACT_BUCKET}/
            - --workers=4
            - --gunicorn-opts=--timeout=300
          env:
            - name: MLFLOW_DB_URI
              valueFrom:
                secretKeyRef:
                  name: mlflow-secrets
                  key: db-uri
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: mlflow-secrets
                  key: aws-access-key-id
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: mlflow-secrets
                  key: aws-secret-access-key
            - name: AWS_DEFAULT_REGION
              value: ap-southeast-1
            - name: MLFLOW_AUTH_CONFIG_PATH
              value: /etc/mlflow/auth.ini
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          livenessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 30
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 10
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow-server
  namespace: ${MLFLOW_NAMESPACE}
spec:
  selector:
    app: mlflow-server
  ports:
    - port: 5000
      targetPort: 5000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mlflow-ingress
  namespace: ${MLFLOW_NAMESPACE}
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: mlflow-basic-auth
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  rules:
    - host: mlflow.company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mlflow-server
                port:
                  number: 5000
  tls:
    - hosts:
        - mlflow.company.com
      secretName: mlflow-tls
EOF
    
    log "MLflow server deployed"
}

register_model() {
    local model_name="$1"
    local run_id="$2"
    local artifact_path="${3:-model}"
    local stage="${4:-Staging}"
    
    log "Registering model: ${model_name} from run: ${run_id}"
    
    python3 <<PYTHON
import mlflow
from mlflow.tracking import MlflowClient
import os

mlflow.set_tracking_uri("http://mlflow-server.mlflow:5000")
client = MlflowClient()

# Register the model
model_uri = f"runs:/{run_id}/${artifact_path}"
result = mlflow.register_model(model_uri, "${model_name}")
print(f"Model registered: version {result.version}")

# Transition to stage
client.transition_model_version_stage(
    name="${model_name}",
    version=result.version,
    stage="${stage}",
    archive_existing_versions=False
)
print(f"Model transitioned to: ${stage}")

# Add model description
client.update_model_version(
    name="${model_name}",
    version=result.version,
    description="Model registered via automation pipeline"
)

# Add tags
client.set_model_version_tag(
    name="${model_name}",
    version=result.version,
    key="deployment_environment",
    value="${stage}"
)

print(f"Model registration complete: ${model_name} v{result.version}")
PYTHON
}

deploy_model_serving() {
    local model_name="${1:-my-model}"
    local model_version="${2:-1}"
    local replicas="${3:-2}"
    
    log "Deploying model serving: ${model_name} v${model_version}"
    
    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-${model_name}-v${model_version}
  namespace: ${MLFLOW_NAMESPACE}
  labels:
    app: model-serving
    model: ${model_name}
    version: "${model_version}"
spec:
  replicas: ${replicas}
  selector:
    matchLabels:
      app: model-${model_name}
  template:
    metadata:
      labels:
        app: model-${model_name}
        model: ${model_name}
        version: "${model_version}"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: model-server
          image: ghcr.io/mlflow/mlflow:v2.9.2
          command:
            - mlflow
            - models
            - serve
            - --model-uri=models:/${model_name}/${model_version}
            - --host=0.0.0.0
            - --port=8080
            - --workers=4
            - --timeout=60
          env:
            - name: MLFLOW_TRACKING_URI
              value: http://mlflow-server.mlflow:5000
            - name: AWS_DEFAULT_REGION
              value: ap-southeast-1
          ports:
            - containerPort: 8080
              name: http
          resources:
            requests:
              cpu: "500m"
              memory: "2Gi"
            limits:
              cpu: "4"
              memory: "8Gi"
          readinessProbe:
            httpGet:
              path: /ping
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /ping
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: model-${model_name}
  namespace: ${MLFLOW_NAMESPACE}
  labels:
    app: model-serving
    model: ${model_name}
spec:
  selector:
    app: model-${model_name}
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: model-${model_name}-hpa
  namespace: ${MLFLOW_NAMESPACE}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: model-${model_name}-v${model_version}
  minReplicas: ${replicas}
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: model_inference_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
EOF
    
    log "Model serving deployed: ${model_name} v${model_version}"
}

monitor_model_performance() {
    local model_name="$1"
    
    log "Monitoring model performance: ${model_name}"
    
    python3 <<PYTHON
import mlflow
from mlflow.tracking import MlflowClient
import json

mlflow.set_tracking_uri("http://mlflow-server.mlflow:5000")
client = MlflowClient()

# Get latest model versions
versions = client.search_model_versions(f"name='{model_name}'")

print(f"\n=== Model Performance Summary: {model_name} ===")
print(f"Total versions: {len(versions)}")

for version in sorted(versions, key=lambda x: int(x.version), reverse=True)[:5]:
    print(f"\nVersion {version.version} ({version.current_stage}):")
    
    if version.run_id:
        run = client.get_run(version.run_id)
        metrics = run.data.metrics
        params = run.data.params
        
        print(f"  Metrics: {json.dumps(metrics, indent=4)}")
        print(f"  Key Params: {dict(list(params.items())[:5])}")
        print(f"  Created: {version.creation_timestamp}")
PYTHON
}

case "${1:-help}" in
    "install") install_mlflow ;;
    "register-model") register_model "$2" "$3" "${4:-model}" "${5:-Staging}" ;;
    "deploy-model") deploy_model_serving "$2" "${3:-1}" "${4:-2}" ;;
    "monitor") monitor_model_performance "$2" ;;
    *) echo "Usage: $0 {install|register-model|deploy-model|monitor}" ;;
esac
```

### ขั้นตอนที่ 531: Kubeflow ML Pipeline Platform

**Kubeflow** สำหรับ ML pipelines บน Kubernetes

```bash
#!/bin/bash
# kubeflow-pipeline-manager.sh - Kubeflow Pipeline Management

set -euo pipefail

KF_NAMESPACE="${KF_NAMESPACE:-kubeflow}"
KFP_HOST="${KFP_HOST:-http://ml-pipeline.kubeflow:8888}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_kubeflow_pipelines() {
    log "Installing Kubeflow Pipelines..."
    
    kubectl create namespace "${KF_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    local kfp_version="2.0.3"
    
    kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=${kfp_version}"
    kubectl wait --for condition=established --timeout=60s crd/applications.app.k8s.io
    kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/env/platform-agnostic-pns?ref=${kfp_version}"
    
    log "Waiting for Kubeflow Pipelines components..."
    kubectl wait --for=condition=ready pod -l app=ml-pipeline -n "${KF_NAMESPACE}" --timeout=300s
    
    log "Kubeflow Pipelines installed"
}

create_ml_pipeline() {
    local pipeline_name="${1:-training-pipeline}"
    local output_file="${2:-/tmp/${pipeline_name}.yaml}"
    
    log "Creating ML Pipeline: ${pipeline_name}"
    
    cat <<PYTHON > "/tmp/create_${pipeline_name}.py"
import kfp
from kfp import dsl
from kfp.components import create_component_from_func
from typing import NamedTuple
import os

@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas", "scikit-learn", "boto3", "pyarrow"]
)
def load_data(
    s3_bucket: str,
    s3_prefix: str,
    output_dataset: dsl.Output[dsl.Dataset]
):
    """Load training data from S3"""
    import pandas as pd
    import boto3
    from io import StringIO
    
    s3 = boto3.client('s3')
    
    paginator = s3.get_paginator('list_objects_v2')
    files = []
    
    for page in paginator.paginate(Bucket=s3_bucket, Prefix=s3_prefix):
        for obj in page.get('Contents', []):
            if obj['Key'].endswith('.parquet'):
                files.append(obj['Key'])
    
    dfs = []
    for f in files[:100]:
        response = s3.get_object(Bucket=s3_bucket, Key=f)
        df = pd.read_parquet(response['Body'])
        dfs.append(df)
    
    combined = pd.concat(dfs, ignore_index=True)
    combined.to_parquet(output_dataset.path)
    print(f"Loaded {len(combined)} records from {len(files)} files")


@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas", "scikit-learn", "pyarrow"]
)
def preprocess_data(
    input_dataset: dsl.Input[dsl.Dataset],
    output_train: dsl.Output[dsl.Dataset],
    output_test: dsl.Output[dsl.Dataset],
    test_size: float = 0.2,
    target_column: str = "target"
) -> NamedTuple('Outputs', [('train_size', int), ('test_size', int)]):
    """Preprocess and split dataset"""
    import pandas as pd
    from sklearn.model_selection import train_test_split
    from sklearn.preprocessing import StandardScaler, LabelEncoder
    from collections import namedtuple
    
    df = pd.read_parquet(input_dataset.path)
    
    # Handle missing values
    df = df.fillna(df.median(numeric_only=True))
    for col in df.select_dtypes(include=['object']).columns:
        df[col] = df[col].fillna(df[col].mode()[0])
    
    # Encode categoricals
    le = LabelEncoder()
    for col in df.select_dtypes(include=['object']).columns:
        if col != target_column:
            df[col] = le.fit_transform(df[col].astype(str))
    
    # Split
    X = df.drop(columns=[target_column])
    y = df[target_column]
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=test_size, random_state=42, stratify=y
    )
    
    train_df = X_train.copy()
    train_df[target_column] = y_train.values
    test_df = X_test.copy()
    test_df[target_column] = y_test.values
    
    train_df.to_parquet(output_train.path)
    test_df.to_parquet(output_test.path)
    
    Outputs = namedtuple('Outputs', ['train_size', 'test_size'])
    return Outputs(train_size=len(train_df), test_size=len(test_df))


@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas", "scikit-learn", "mlflow", "boto3", "pyarrow"]
)
def train_model(
    train_dataset: dsl.Input[dsl.Dataset],
    model_artifact: dsl.Output[dsl.Model],
    mlflow_tracking_uri: str,
    experiment_name: str,
    algorithm: str = "random_forest",
    n_estimators: int = 100,
    target_column: str = "target"
) -> NamedTuple('Metrics', [('accuracy', float), ('f1_score', float), ('run_id', str)]):
    """Train ML model with MLflow tracking"""
    import pandas as pd
    import mlflow
    import mlflow.sklearn
    from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
    from sklearn.linear_model import LogisticRegression
    from sklearn.metrics import accuracy_score, f1_score, classification_report
    import joblib
    from collections import namedtuple
    
    mlflow.set_tracking_uri(mlflow_tracking_uri)
    mlflow.set_experiment(experiment_name)
    
    df = pd.read_parquet(train_dataset.path)
    X = df.drop(columns=[target_column])
    y = df[target_column]
    
    with mlflow.start_run() as run:
        mlflow.set_tags({
            "algorithm": algorithm,
            "pipeline": "kubeflow",
            "environment": "production"
        })
        
        if algorithm == "random_forest":
            model = RandomForestClassifier(
                n_estimators=n_estimators,
                random_state=42,
                n_jobs=-1
            )
        elif algorithm == "gradient_boosting":
            model = GradientBoostingClassifier(
                n_estimators=n_estimators,
                random_state=42
            )
        else:
            model = LogisticRegression(random_state=42, max_iter=1000)
        
        mlflow.log_params({
            "algorithm": algorithm,
            "n_estimators": n_estimators,
            "n_features": X.shape[1],
            "n_samples": X.shape[0]
        })
        
        model.fit(X, y)
        y_pred = model.predict(X)
        
        acc = accuracy_score(y, y_pred)
        f1 = f1_score(y, y_pred, average='weighted')
        
        mlflow.log_metrics({
            "train_accuracy": acc,
            "train_f1": f1
        })
        
        mlflow.sklearn.log_model(model, "model")
        
        joblib.dump(model, model_artifact.path + ".pkl")
        model_artifact.metadata["run_id"] = run.info.run_id
        
        print(f"Training complete: accuracy={acc:.4f}, f1={f1:.4f}")
        
        Metrics = namedtuple('Metrics', ['accuracy', 'f1_score', 'run_id'])
        return Metrics(accuracy=acc, f1_score=f1, run_id=run.info.run_id)


@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas", "scikit-learn", "mlflow", "boto3", "pyarrow"]
)
def evaluate_model(
    test_dataset: dsl.Input[dsl.Dataset],
    model_artifact: dsl.Input[dsl.Model],
    mlflow_tracking_uri: str,
    run_id: str,
    accuracy_threshold: float = 0.85,
    target_column: str = "target"
) -> NamedTuple('Evaluation', [('test_accuracy', float), ('passed', bool)]):
    """Evaluate model on test set"""
    import pandas as pd
    import mlflow
    from sklearn.metrics import accuracy_score, f1_score, confusion_matrix
    import joblib
    from collections import namedtuple
    
    mlflow.set_tracking_uri(mlflow_tracking_uri)
    
    df = pd.read_parquet(test_dataset.path)
    X = df.drop(columns=[target_column])
    y = df[target_column]
    
    model = joblib.load(model_artifact.path + ".pkl")
    y_pred = model.predict(X)
    
    acc = accuracy_score(y, y_pred)
    f1 = f1_score(y, y_pred, average='weighted')
    
    with mlflow.start_run(run_id=run_id):
        mlflow.log_metrics({
            "test_accuracy": acc,
            "test_f1": f1
        })
    
    passed = acc >= accuracy_threshold
    print(f"Evaluation: accuracy={acc:.4f}, threshold={accuracy_threshold}, passed={passed}")
    
    Evaluation = namedtuple('Evaluation', ['test_accuracy', 'passed'])
    return Evaluation(test_accuracy=acc, passed=passed)


@dsl.pipeline(
    name="${pipeline_name}",
    description="End-to-end ML training pipeline"
)
def ml_training_pipeline(
    s3_bucket: str = "ml-data",
    s3_prefix: str = "training/",
    experiment_name: str = "${pipeline_name}",
    algorithm: str = "random_forest",
    n_estimators: int = 100,
    test_size: float = 0.2,
    accuracy_threshold: float = 0.85,
    target_column: str = "target",
    mlflow_tracking_uri: str = "http://mlflow-server.mlflow:5000"
):
    load_task = load_data(
        s3_bucket=s3_bucket,
        s3_prefix=s3_prefix
    )
    load_task.set_caching_options(False)
    
    preprocess_task = preprocess_data(
        input_dataset=load_task.outputs['output_dataset'],
        test_size=test_size,
        target_column=target_column
    )
    
    train_task = train_model(
        train_dataset=preprocess_task.outputs['output_train'],
        mlflow_tracking_uri=mlflow_tracking_uri,
        experiment_name=experiment_name,
        algorithm=algorithm,
        n_estimators=n_estimators,
        target_column=target_column
    )
    train_task.set_memory_request("4G")
    train_task.set_cpu_request("2")
    
    evaluate_task = evaluate_model(
        test_dataset=preprocess_task.outputs['output_test'],
        model_artifact=train_task.outputs['model_artifact'],
        mlflow_tracking_uri=mlflow_tracking_uri,
        run_id=train_task.outputs['run_id'],
        accuracy_threshold=accuracy_threshold,
        target_column=target_column
    )


if __name__ == '__main__':
    import kfp.compiler as compiler
    compiler.Compiler().compile(
        pipeline_func=ml_training_pipeline,
        package_path="${output_file}"
    )
    print(f"Pipeline compiled: ${output_file}")
PYTHON
    
    python3 "/tmp/create_${pipeline_name}.py" 2>/dev/null || \
        log "Note: kfp package required. Install with: pip install kfp"
    
    log "Pipeline definition created"
}

submit_pipeline_run() {
    local pipeline_file="${1:-/tmp/training-pipeline.yaml}"
    local experiment_name="${2:-default}"
    local run_name="${3:-run-$(date '+%Y%m%d-%H%M%S')}"
    
    log "Submitting pipeline run: ${run_name}"
    
    python3 <<PYTHON
import kfp
import json

client = kfp.Client(host="${KFP_HOST}")

# Get or create experiment
try:
    experiment = client.get_experiment(experiment_name="${experiment_name}")
except:
    experiment = client.create_experiment("${experiment_name}")

# Submit run
run = client.create_run_from_pipeline_package(
    pipeline_file="${pipeline_file}",
    arguments={
        "s3_bucket": "ml-data",
        "s3_prefix": "training/",
        "algorithm": "random_forest",
        "n_estimators": 200,
        "accuracy_threshold": 0.90
    },
    run_name="${run_name}",
    experiment_id=experiment.id
)

print(f"Pipeline run submitted: {run.run_id}")
print(f"View at: ${KFP_HOST}/#/runs/details/{run.run_id}")
PYTHON
}

monitor_pipeline_runs() {
    log "=== Pipeline Run Status ==="
    
    python3 <<PYTHON
import kfp

client = kfp.Client(host="${KFP_HOST}")

runs = client.list_runs(page_size=20)
print(f"{'Run ID':<40} {'Name':<40} {'Status':<15} {'Created':<25}")
print("-" * 120)

for run in runs.runs or []:
    print(f"{run.id:<40} {run.name[:38]:<40} {run.status:<15} {str(run.created_at)[:24]:<25}")
PYTHON
}

case "${1:-help}" in
    "install") install_kubeflow_pipelines ;;
    "create-pipeline") create_ml_pipeline "${2:-training-pipeline}" "${3:-/tmp/pipeline.yaml}" ;;
    "submit-run") submit_pipeline_run "${2}" "${3:-default}" "${4:-}" ;;
    "monitor") monitor_pipeline_runs ;;
    *) echo "Usage: $0 {install|create-pipeline|submit-run|monitor}" ;;
esac
```

### ขั้นตอนที่ 532: GPU Cluster Management for ML Workloads

**GPU management** สำหรับ training และ inference workloads

```bash
#!/bin/bash
# gpu-cluster-manager.sh - GPU Cluster Management for ML

set -euo pipefail

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

install_gpu_operator() {
    log "Installing NVIDIA GPU Operator..."
    
    helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
    helm repo update
    
    kubectl create namespace gpu-operator --dry-run=client -o yaml | kubectl apply -f -
    
    helm upgrade --install gpu-operator nvidia/gpu-operator \
        --namespace gpu-operator \
        --set driver.enabled=true \
        --set toolkit.enabled=true \
        --set devicePlugin.enabled=true \
        --set dcgmExporter.enabled=true \
        --set dcgmExporter.serviceMonitor.enabled=true \
        --set mig.strategy=mixed \
        --set operator.defaultRuntime=containerd \
        --wait
    
    log "GPU operator installed"
}

configure_gpu_node_pool() {
    local cluster_name="${1:-production}"
    local gpu_type="${2:-p3.2xlarge}"
    
    log "Configuring GPU node pool for cluster: ${cluster_name}"
    
    cat <<EOF > /tmp/gpu-nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${cluster_name}
  region: ap-southeast-1

nodeGroups:
  - name: gpu-workers
    instanceType: ${gpu_type}
    minSize: 0
    maxSize: 10
    desiredCapacity: 0
    volumeSize: 200
    labels:
      role: gpu-worker
      accelerator: nvidia
    taints:
      - key: nvidia.com/gpu
        value: "true"
        effect: NoSchedule
    preBootstrapCommands:
      - yum install -y amazon-linux-extras
      - amazon-linux-extras install -y nvidia
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
    tags:
      nodegroup-role: gpu-worker
      accelerator: nvidia
      cost-tier: gpu
EOF
    
    log "GPU node group configuration: /tmp/gpu-nodegroup.yaml"
}

deploy_pytorch_training_job() {
    local job_name="${1:-pytorch-training}"
    local namespace="${2:-ml-training}"
    local image="${3:-pytorch/pytorch:2.1.0-cuda11.8-cudnn8-runtime}"
    local gpu_count="${4:-1}"
    
    log "Deploying PyTorch training job: ${job_name}"
    
    kubectl create namespace "${namespace}" --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: ${job_name}
  namespace: ${namespace}
  labels:
    app: ml-training
    job-type: pytorch
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  activeDeadlineSeconds: 86400
  template:
    metadata:
      labels:
        app: ml-training
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      restartPolicy: OnFailure
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - name: trainer
          image: ${image}
          command: ["python3", "/training/train.py"]
          args:
            - --epochs=100
            - --batch-size=64
            - --lr=0.001
            - --data-path=s3://ml-data/training/
            - --output-path=s3://ml-models/$(date '+%Y%m%d')/
            - --mlflow-uri=http://mlflow-server.mlflow:5000
          resources:
            requests:
              cpu: "4"
              memory: "16Gi"
              nvidia.com/gpu: "${gpu_count}"
            limits:
              cpu: "8"
              memory: "32Gi"
              nvidia.com/gpu: "${gpu_count}"
          env:
            - name: CUDA_VISIBLE_DEVICES
              value: "0"
            - name: TORCH_DISTRIBUTED_DEBUG
              value: "INFO"
            - name: AWS_DEFAULT_REGION
              value: ap-southeast-1
          volumeMounts:
            - mountPath: /training
              name: training-scripts
            - mountPath: /dev/shm
              name: dshm
      volumes:
        - name: training-scripts
          configMap:
            name: training-scripts
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: "8Gi"
EOF
    
    log "PyTorch training job submitted: ${job_name}"
}

deploy_distributed_training() {
    local job_name="${1:-distributed-training}"
    local namespace="${2:-ml-training}"
    local worker_count="${3:-4}"
    local gpus_per_worker="${4:-2}"
    
    log "Deploying distributed PyTorch training: ${job_name} (${worker_count} workers, ${gpus_per_worker} GPUs each)"
    
    cat <<EOF | kubectl apply -f -
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: ${job_name}
  namespace: ${namespace}
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          tolerations:
            - key: nvidia.com/gpu
              operator: Exists
              effect: NoSchedule
          containers:
            - name: pytorch
              image: pytorch/pytorch:2.1.0-cuda11.8-cudnn8-runtime
              command: ["torchrun"]
              args:
                - --nproc_per_node=${gpus_per_worker}
                - --nnodes=\$((\${WORLD_SIZE}))
                - --node_rank=\$(RANK)
                - --master_addr=\$(MASTER_ADDR)
                - --master_port=\$(MASTER_PORT)
                - /training/distributed_train.py
              resources:
                requests:
                  cpu: "8"
                  memory: "32Gi"
                  nvidia.com/gpu: "${gpus_per_worker}"
                limits:
                  cpu: "16"
                  memory: "64Gi"
                  nvidia.com/gpu: "${gpus_per_worker}"
              env:
                - name: NCCL_DEBUG
                  value: INFO
                - name: NCCL_IB_DISABLE
                  value: "0"
    
    Worker:
      replicas: ${worker_count}
      restartPolicy: OnFailure
      template:
        spec:
          tolerations:
            - key: nvidia.com/gpu
              operator: Exists
              effect: NoSchedule
          containers:
            - name: pytorch
              image: pytorch/pytorch:2.1.0-cuda11.8-cudnn8-runtime
              command: ["torchrun"]
              args:
                - --nproc_per_node=${gpus_per_worker}
                - --nnodes=\$((\${WORLD_SIZE}))
                - --node_rank=\$(RANK)
                - --master_addr=\$(MASTER_ADDR)
                - --master_port=\$(MASTER_PORT)
                - /training/distributed_train.py
              resources:
                requests:
                  cpu: "8"
                  memory: "32Gi"
                  nvidia.com/gpu: "${gpus_per_worker}"
                limits:
                  cpu: "16"
                  memory: "64Gi"
                  nvidia.com/gpu: "${gpus_per_worker}"
EOF
    
    log "Distributed training job submitted: ${job_name}"
}

monitor_gpu_utilization() {
    log "=== GPU Utilization Monitoring ==="
    
    echo "--- GPU Node Status ---"
    kubectl get nodes -l accelerator=nvidia \
        -o custom-columns="NAME:.metadata.name,STATUS:.status.conditions[-1].type,GPU:.status.capacity.nvidia\.com/gpu"
    
    echo ""
    echo "--- GPU Pod Resource Usage ---"
    kubectl get pods -A \
        -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,GPU_REQUEST:.spec.containers[0].resources.requests.nvidia\.com/gpu,GPU_LIMIT:.spec.containers[0].resources.limits.nvidia\.com/gpu" | \
        grep -v "<none>" | head -20
    
    echo ""
    echo "--- DCGM Metrics (if available) ---"
    kubectl exec -n gpu-operator \
        "$(kubectl get pod -n gpu-operator -l app=nvidia-dcgm-exporter -o name | head -1)" -- \
        nvidia-smi 2>/dev/null || echo "nvidia-smi not accessible directly"
}

case "${1:-help}" in
    "install-operator") install_gpu_operator ;;
    "configure-nodepool") configure_gpu_node_pool "${2:-production}" "${3:-p3.2xlarge}" ;;
    "train-single") deploy_pytorch_training_job "$2" "${3:-ml-training}" "${4}" "${5:-1}" ;;
    "train-distributed") deploy_distributed_training "$2" "${3:-ml-training}" "${4:-4}" "${5:-2}" ;;
    "monitor") monitor_gpu_utilization ;;
    *) echo "Usage: $0 {install-operator|configure-nodepool|train-single|train-distributed|monitor}" ;;
esac
```

### ขั้นตอนที่ 533: Model Serving with Triton Inference Server

**Triton Inference Server** สำหรับ high-performance ML inference

```bash
#!/bin/bash
# triton-inference-server.sh - NVIDIA Triton Inference Server Management

set -euo pipefail

TRITON_NAMESPACE="${TRITON_NAMESPACE:-triton}"
MODEL_REPOSITORY="${MODEL_REPOSITORY:-s3://ml-models/triton-repository}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

deploy_triton_server() {
    local server_name="${1:-triton-server}"
    local model_repo="${2:-${MODEL_REPOSITORY}}"
    local replicas="${3:-2}"
    local gpu_count="${4:-1}"
    
    log "Deploying Triton Inference Server: ${server_name}"
    
    kubectl create namespace "${TRITON_NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
    
    cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${server_name}
  namespace: ${TRITON_NAMESPACE}
  labels:
    app: ${server_name}
spec:
  replicas: ${replicas}
  selector:
    matchLabels:
      app: ${server_name}
  template:
    metadata:
      labels:
        app: ${server_name}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8002"
        prometheus.io/path: "/metrics"
    spec:
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - name: triton
          image: nvcr.io/nvidia/tritonserver:23.12-py3
          command:
            - tritonserver
            - --model-repository=${model_repo}
            - --model-control-mode=poll
            - --repository-poll-secs=30
            - --http-port=8000
            - --grpc-port=8001
            - --metrics-port=8002
            - --log-verbose=1
            - --allow-metrics=true
            - --allow-gpu-metrics=true
            - --backend-config=python,shm-default-byte-size=1073741824
            - --backend-config=tensorrt,plugins=/opt/tritonserver/backends/tensorrt/
          ports:
            - containerPort: 8000
              name: http
            - containerPort: 8001
              name: grpc
            - containerPort: 8002
              name: metrics
          resources:
            requests:
              cpu: "4"
              memory: "16Gi"
              nvidia.com/gpu: "${gpu_count}"
            limits:
              cpu: "8"
              memory: "32Gi"
              nvidia.com/gpu: "${gpu_count}"
          env:
            - name: AWS_DEFAULT_REGION
              value: ap-southeast-1
            - name: TRITON_AWS_MOUNT_DIRECTORY
              value: /tmp/s3mnt
          readinessProbe:
            httpGet:
              path: /v2/health/ready
              port: 8000
            initialDelaySeconds: 60
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /v2/health/live
              port: 8000
            initialDelaySeconds: 60
            periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: ${server_name}
  namespace: ${TRITON_NAMESPACE}
spec:
  selector:
    app: ${server_name}
  ports:
    - port: 8000
      targetPort: 8000
      name: http
    - port: 8001
      targetPort: 8001
      name: grpc
    - port: 8002
      targetPort: 8002
      name: metrics
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ${server_name}-hpa
  namespace: ${TRITON_NAMESPACE}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ${server_name}
  minReplicas: ${replicas}
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: nv_inference_queue_duration_us
        target:
          type: AverageValue
          averageValue: "100000"
EOF
    
    log "Triton server deployed: ${server_name}"
}

create_triton_model_config() {
    local model_name="${1:-resnet50}"
    local model_type="${2:-onnx}"
    local batch_size="${3:-32}"
    local input_shape="${4:-3,224,224}"
    local output_dir="${5:-/tmp/triton-models}"
    
    log "Creating Triton model config: ${model_name}"
    
    mkdir -p "${output_dir}/${model_name}/1"
    
    cat <<EOF > "${output_dir}/${model_name}/config.pbtxt"
name: "${model_name}"
platform: "$([ "${model_type}" = "pytorch" ] && echo 'pytorch_libtorch' || echo 'onnxruntime_onnx')"
max_batch_size: ${batch_size}

input [
  {
    name: "INPUT__0"
    data_type: TYPE_FP32
    dims: [${input_shape}]
  }
]

output [
  {
    name: "OUTPUT__0"
    data_type: TYPE_FP32
    dims: [1000]
    label_filename: "labels.txt"
  }
]

dynamic_batching {
  preferred_batch_size: [8, 16, 32]
  max_queue_delay_microseconds: 50000
}

instance_group [
  {
    count: 1
    kind: KIND_GPU
    gpus: [0]
  }
]

optimization {
  execution_accelerators {
    gpu_execution_accelerator [
      {
        name: "tensorrt"
        parameters {
          key: "precision_mode"
          value: "FP16"
        }
        parameters {
          key: "max_workspace_size_bytes"
          value: "1073741824"
        }
      }
    ]
  }
}
EOF
    
    log "Triton model config created: ${output_dir}/${model_name}/config.pbtxt"
}

test_triton_inference() {
    local server_url="${1:-http://triton-server.triton:8000}"
    local model_name="${2:-resnet50}"
    local test_requests="${3:-100}"
    
    log "Testing Triton inference: ${model_name} with ${test_requests} requests"
    
    python3 <<PYTHON
import tritonclient.http as httpclient
import numpy as np
import time

client = httpclient.InferenceServerClient(url="${server_url}")

# Check server health
if not client.is_server_ready():
    raise Exception("Server not ready")

# Check model
model_metadata = client.get_model_metadata("${model_name}")
print(f"Model: ${model_name}")
print(f"Inputs: {model_metadata['inputs']}")
print(f"Outputs: {model_metadata['outputs']}")

# Run inference
latencies = []
for i in range(${test_requests}):
    input_data = np.random.rand(1, 3, 224, 224).astype(np.float32)
    
    inputs = httpclient.InferInput("INPUT__0", input_data.shape, "FP32")
    inputs.set_data_from_numpy(input_data)
    
    outputs = httpclient.InferRequestedOutput("OUTPUT__0")
    
    start = time.perf_counter()
    response = client.infer("${model_name}", [inputs], outputs=[outputs])
    end = time.perf_counter()
    
    latencies.append((end - start) * 1000)

latencies.sort()
print(f"\n=== Inference Performance (${test_requests} requests) ===")
print(f"P50: {np.percentile(latencies, 50):.2f}ms")
print(f"P95: {np.percentile(latencies, 95):.2f}ms")
print(f"P99: {np.percentile(latencies, 99):.2f}ms")
print(f"Max: {max(latencies):.2f}ms")
print(f"Throughput: {1000 / np.mean(latencies):.0f} req/s")
PYTHON
}

monitor_triton_metrics() {
    log "=== Triton Server Metrics ==="
    
    local triton_url="${1:-http://triton-server.triton:8002}"
    
    curl -s "${triton_url}/metrics" | \
        grep -E "^nv_inference|^nv_gpu" | \
        grep -v "^#" | \
        awk '{printf "%-60s %s\n", $1, $2}' | head -30
}

case "${1:-help}" in
    "deploy") deploy_triton_server "${2:-triton-server}" "${3:-${MODEL_REPOSITORY}}" "${4:-2}" "${5:-1}" ;;
    "create-config") create_triton_model_config "$2" "${3:-onnx}" "${4:-32}" "${5:-3,224,224}" ;;
    "test") test_triton_inference "${2:-http://triton-server.triton:8000}" "${3:-resnet50}" "${4:-100}" ;;
    "metrics") monitor_triton_metrics "${2:-http://triton-server.triton:8002}" ;;
    *) echo "Usage: $0 {deploy|create-config|test|metrics}" ;;
esac
```

---

## สรุป Part 44

ในส่วนนี้เราได้เรียนรู้:

| ขั้นตอน | หัวข้อ | เครื่องมือหลัก |
|---------|--------|----------------|
| 530 | MLflow Model Management | MLflow Tracking, Model Registry, Model Serving |
| 531 | Kubeflow ML Pipelines | KFP Components, DAG Pipelines, Distributed Training |
| 532 | GPU Cluster Management | GPU Operator, PyTorchJob, Distributed Training |
| 533 | Triton Inference Server | NVIDIA Triton, Dynamic Batching, TensorRT Optimization |

### ขั้นตอนต่อไป: Part 45 - Event-Driven Architecture and Message Queuing
