# Part 60: Machine Learning Platform, MLOps Pipeline และ Feature Store

## Module 5: World-Class Level (ต่อ)

---

## ขั้นตอนที่ 589: Kubeflow MLOps Platform

### MLOps Architecture

```
MLOps Platform Architecture:
┌─────────────────────────────────────────────────────────────┐
│                    Data Layer                                 │
│  Feature Store (Feast) │ Data Lake (Iceberg) │ Label Store   │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                 Training Platform                             │
│  Kubeflow Pipelines │ Ray Cluster │ Spark MLlib               │
│  HPO (Katib) │ Distributed Training (PyTorchJob)             │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Model Registry                               │
│  MLflow Tracking │ Model Versioning │ Approval Workflow       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Serving Platform                             │
│  KServe (InferenceService) │ vLLM │ TensorRT-LLM             │
│  A/B Testing │ Canary │ Shadow Mode                          │
└─────────────────────────────────────────────────────────────┘
```

### `kubeflow-mlops.sh`

```bash
#!/usr/bin/env bash
# kubeflow-mlops.sh — Full MLOps Platform with Kubeflow
set -euo pipefail

LOG_FILE="/var/log/kubeflow-mlops.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Kubeflow Installation ─────────────────────────────────────────────────

install_kubeflow() {
  log "=== Installing Kubeflow ==="

  # Install using manifests
  KUBEFLOW_VERSION="v1.8.0"
  kubectl apply -k "github.com/kubeflow/manifests/common/cert-manager/cert-manager/base?ref=${KUBEFLOW_VERSION}"
  kubectl wait --for=condition=Available \
    deployment/cert-manager-webhook \
    -n cert-manager --timeout=120s

  # Install Kubeflow Pipelines
  kubectl apply -k "github.com/kubeflow/manifests/apps/pipeline/upstream/env/platform-agnostic-multi-user?ref=${KUBEFLOW_VERSION}"

  # Install Training Operator (PyTorch, TF, MPI)
  kubectl apply -k "github.com/kubeflow/manifests/apps/training-operator/upstream/overlays/standalone?ref=${KUBEFLOW_VERSION}"

  # Install Katib (HPO)
  kubectl apply -k "github.com/kubeflow/manifests/apps/katib/upstream/installs/katib-standalone?ref=${KUBEFLOW_VERSION}"

  # Install KServe
  kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.13.0/kserve.yaml
  kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.13.0/kserve-cluster-resources.yaml

  log "✓ Kubeflow installed"
}

# ─── 2. MLflow Model Registry ─────────────────────────────────────────────────

setup_mlflow() {
  log "=== Setting up MLflow Model Registry ==="

  helm repo add community-charts https://community-charts.github.io/helm-charts
  helm upgrade --install mlflow community-charts/mlflow \
    --namespace mlops \
    --create-namespace \
    --set backendStore.postgres.enabled=true \
    --set backendStore.postgres.host=postgresql \
    --set backendStore.postgres.database=mlflow \
    --set backendStore.postgres.user=mlflow \
    --set backendStore.postgres.password="${MLFLOW_DB_PASSWORD}" \
    --set artifactRoot.s3.enabled=true \
    --set artifactRoot.s3.bucket=mlflow-artifacts \
    --set artifactRoot.s3.awsAccessKeyId="${AWS_ACCESS_KEY_ID}" \
    --set artifactRoot.s3.awsSecretAccessKey="${AWS_SECRET_ACCESS_KEY}" \
    --set serviceMonitor.enabled=true

  log "✓ MLflow Model Registry configured"
}

# ─── 3. Feature Store (Feast) ─────────────────────────────────────────────────

setup_feast() {
  log "=== Setting up Feast Feature Store ==="

  # Feast feature_store.yaml
  cat <<'EOF' > /tmp/feature_store.yaml
project: production_ml
registry: s3://ml-platform/feast/registry.db
provider: aws
online_store:
  type: redis
  connection_string: "redis://redis-cluster:6379"
offline_store:
  type: redshift
  cluster_id: ml-redshift
  region: us-east-1
  user: feast_user
  database: feast_db
  s3_staging_location: s3://ml-platform/feast/staging
entity_key_serialization_version: 2
feature_server:
  enabled: true
  mode: k8s
  transformation_service_endpoint: http://feast-transformation:6566
EOF

  # Feast feature definitions
  cat <<'PYTHON' > /tmp/customer_features.py
"""Customer features for ML models."""
from datetime import timedelta
from feast import (
    Entity, Feature, FeatureView, FileSource, ValueType,
    FeatureStore, Field, PushSource, StreamFeatureView,
    KafkaSource,
)
from feast.types import Float32, Float64, Int64, String

# ── Entities ──────────────────────────────────────────────────────────────────

customer = Entity(
    name="customer_id",
    value_type=ValueType.INT64,
    description="Customer identifier",
    tags={"owner": "ml-team", "domain": "customer"},
)

# ── Offline Sources ───────────────────────────────────────────────────────────

customer_stats_source = FileSource(
    path="s3://ml-platform/features/customer_stats/*.parquet",
    event_timestamp_column="event_timestamp",
    created_timestamp_column="created_timestamp",
)

# ── Feature Views ─────────────────────────────────────────────────────────────

customer_stats_fv = FeatureView(
    name="customer_stats",
    entities=[customer],
    ttl=timedelta(days=30),
    schema=[
        Field(name="total_orders", dtype=Int64),
        Field(name="lifetime_value", dtype=Float64),
        Field(name="avg_order_value", dtype=Float64),
        Field(name="days_since_last_order", dtype=Int64),
        Field(name="churn_risk_score", dtype=Float32),
        Field(name="preferred_category", dtype=String),
    ],
    source=customer_stats_source,
    tags={"owner": "ml-team", "status": "production"},
)

# ── Real-time Streaming Feature View ─────────────────────────────────────────

customer_activity_kafka = KafkaSource(
    name="customer_activity_kafka",
    kafka_bootstrap_servers="kafka-cluster:9092",
    topic="customer-activity",
    event_timestamp_column="timestamp",
    batch_source=customer_stats_source,
)

customer_realtime_fv = StreamFeatureView(
    name="customer_realtime_activity",
    entities=[customer],
    ttl=timedelta(hours=1),
    schema=[
        Field(name="clicks_last_1h", dtype=Int64),
        Field(name="page_views_last_1h", dtype=Int64),
        Field(name="cart_value_current", dtype=Float64),
        Field(name="session_duration_minutes", dtype=Float32),
    ],
    source=customer_activity_kafka,
    tags={"latency": "realtime"},
)

# ── On-Demand Features (computed) ────────────────────────────────────────────

from feast import on_demand_feature_view
import pandas as pd

@on_demand_feature_view(
    sources=[customer_stats_fv, customer_realtime_fv],
    schema=[
        Field(name="engagement_score", dtype=Float32),
        Field(name="purchase_probability", dtype=Float32),
    ],
)
def compute_engagement(inputs: pd.DataFrame) -> pd.DataFrame:
    df = pd.DataFrame()
    df["engagement_score"] = (
        inputs["clicks_last_1h"] * 0.3 +
        inputs["page_views_last_1h"] * 0.2 +
        inputs["session_duration_minutes"] * 0.5
    ).clip(0, 100)

    df["purchase_probability"] = (
        1 / (1 + pd.np.exp(-(
            inputs["churn_risk_score"] * -2.5 +
            df["engagement_score"] * 0.03 +
            inputs["cart_value_current"] * 0.001 - 2
        )))
    )

    return df
PYTHON

  # Feast Feature Server deployment
  cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: feast-feature-server
  namespace: mlops
spec:
  replicas: 3
  selector:
    matchLabels:
      app: feast-feature-server
  template:
    metadata:
      labels:
        app: feast-feature-server
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "6566"
    spec:
      containers:
        - name: feature-server
          image: feastdev/feature-server:0.38.0
          command:
            - feast
            - serve
            - --host=0.0.0.0
            - --port=6566
            - --workers=4
          ports:
            - containerPort: 6566
          env:
            - name: FEAST_USAGE
              value: "false"
          resources:
            requests:
              cpu: "1"
              memory: 1Gi
            limits:
              cpu: "4"
              memory: 4Gi
          readinessProbe:
            httpGet:
              path: /health
              port: 6566
            initialDelaySeconds: 15
            periodSeconds: 10
          volumeMounts:
            - name: feature-store-config
              mountPath: /feature_store
      volumes:
        - name: feature-store-config
          configMap:
            name: feast-config
EOF

  log "✓ Feast Feature Store configured"
}

# ─── 4. Kubeflow Pipeline (End-to-End ML) ─────────────────────────────────────

create_ml_pipeline() {
  log "=== Creating ML Training Pipeline ==="

  cat <<'PYTHON' > /tmp/churn_prediction_pipeline.py
"""End-to-end churn prediction ML pipeline using Kubeflow Pipelines."""
import kfp
from kfp import dsl
from kfp.components import create_component_from_func
from typing import NamedTuple

# ── Pipeline Components ───────────────────────────────────────────────────────

@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["feast==0.38.0", "pandas", "pyarrow", "boto3"],
)
def fetch_features(
    entity_df_path: str,
    feature_store_config: str,
    output_path: dsl.Output[dsl.Dataset],
):
    """Fetch features from Feast Feature Store."""
    import pandas as pd
    from feast import FeatureStore
    import json

    config = json.loads(feature_store_config)
    store = FeatureStore(repo_path="/feature_store")

    entity_df = pd.read_parquet(entity_df_path)

    feature_vector = store.get_historical_features(
        entity_df=entity_df,
        features=[
            "customer_stats:total_orders",
            "customer_stats:lifetime_value",
            "customer_stats:avg_order_value",
            "customer_stats:days_since_last_order",
            "customer_stats:churn_risk_score",
        ],
    ).to_df()

    feature_vector.to_parquet(output_path.path)
    print(f"Fetched {len(feature_vector)} feature vectors")

@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "pandas", "scikit-learn", "xgboost",
        "shap", "mlflow", "boto3",
    ],
)
def train_model(
    features_path: dsl.Input[dsl.Dataset],
    model_output: dsl.Output[dsl.Model],
    metrics_output: dsl.Output[dsl.Metrics],
    experiment_name: str = "churn-prediction",
    learning_rate: float = 0.1,
    n_estimators: int = 300,
    max_depth: int = 6,
) -> NamedTuple("TrainOutput", [("model_uri", str), ("auc_roc", float)]):
    import pandas as pd
    import xgboost as xgb
    import mlflow
    import mlflow.xgboost
    import shap
    from sklearn.model_selection import train_test_split, StratifiedKFold, cross_val_score
    from sklearn.metrics import roc_auc_score, classification_report
    import numpy as np

    df = pd.read_parquet(features_path.path)

    X = df.drop(columns=["customer_id", "churn_label", "event_timestamp"])
    y = df["churn_label"]

    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42, stratify=y
    )

    mlflow.set_tracking_uri("http://mlflow:5000")
    mlflow.set_experiment(experiment_name)

    with mlflow.start_run() as run:
        model = xgb.XGBClassifier(
            n_estimators=n_estimators,
            max_depth=max_depth,
            learning_rate=learning_rate,
            min_child_weight=3,
            subsample=0.8,
            colsample_bytree=0.8,
            gamma=0.1,
            reg_alpha=0.1,
            reg_lambda=1.0,
            scale_pos_weight=y.value_counts()[0] / y.value_counts()[1],
            use_label_encoder=False,
            eval_metric="auc",
            early_stopping_rounds=30,
        )

        model.fit(
            X_train, y_train,
            eval_set=[(X_test, y_test)],
            verbose=50,
        )

        # Evaluate
        y_pred_proba = model.predict_proba(X_test)[:, 1]
        auc = roc_auc_score(y_test, y_pred_proba)

        # Cross-validation
        cv_scores = cross_val_score(
            model, X, y, cv=StratifiedKFold(n_splits=5),
            scoring="roc_auc", n_jobs=-1
        )

        # SHAP feature importance
        explainer = shap.TreeExplainer(model)
        shap_values = explainer.shap_values(X_test[:100])
        feature_importance = dict(zip(X.columns, np.abs(shap_values).mean(0)))

        # Log to MLflow
        mlflow.log_params({
            "n_estimators": n_estimators,
            "max_depth": max_depth,
            "learning_rate": learning_rate,
        })
        mlflow.log_metrics({
            "test_auc": auc,
            "cv_auc_mean": cv_scores.mean(),
            "cv_auc_std": cv_scores.std(),
        })
        mlflow.log_dict(feature_importance, "feature_importance.json")
        mlflow.xgboost.log_model(model, "model", registered_model_name="churn-prediction")

        # Write metrics output
        metrics_output.log_metric("auc_roc", auc)
        metrics_output.log_metric("cv_auc_mean", cv_scores.mean())

        # Save model
        model.save_model(model_output.path)

        model_uri = f"runs:/{run.info.run_id}/model"
        print(f"Model trained: AUC={auc:.4f}, CV={cv_scores.mean():.4f}±{cv_scores.std():.4f}")

        from collections import namedtuple
        TrainOutput = namedtuple("TrainOutput", ["model_uri", "auc_roc"])
        return TrainOutput(model_uri=model_uri, auc_roc=auc)

@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["mlflow", "boto3"],
)
def evaluate_and_promote(
    model_uri: str,
    auc_roc: float,
    min_auc_threshold: float = 0.80,
) -> bool:
    """Evaluate model and promote to Production if above threshold."""
    import mlflow

    mlflow.set_tracking_uri("http://mlflow:5000")
    client = mlflow.MlflowClient()

    if auc_roc < min_auc_threshold:
        print(f"Model failed quality gate: AUC {auc_roc:.4f} < {min_auc_threshold}")
        return False

    # Get model version from URI
    run_id = model_uri.split("/")[1]
    model_name = "churn-prediction"

    versions = client.search_model_versions(f"name='{model_name}' and run_id='{run_id}'")
    if not versions:
        print("No model version found")
        return False

    version = versions[0].version

    # Transition to Production
    client.transition_model_version_stage(
        name=model_name,
        version=version,
        stage="Production",
        archive_existing_versions=True,
    )

    # Add tags
    client.set_model_version_tag(model_name, version, "promoted_by", "kubeflow-pipeline")
    client.set_model_version_tag(model_name, version, "auc_roc", str(auc_roc))

    print(f"✓ Model {model_name} v{version} promoted to Production (AUC={auc_roc:.4f})")
    return True

# ── Pipeline Definition ───────────────────────────────────────────────────────

@dsl.pipeline(
    name="churn-prediction-pipeline",
    description="End-to-end churn prediction ML pipeline",
)
def churn_pipeline(
    entity_df_path: str = "s3://ml-platform/entities/customers.parquet",
    experiment_name: str = "churn-prediction",
    learning_rate: float = 0.1,
    n_estimators: int = 300,
    max_depth: int = 6,
    min_auc_threshold: float = 0.80,
):
    fetch_task = fetch_features(
        entity_df_path=entity_df_path,
        feature_store_config='{"type":"s3","bucket":"ml-platform"}',
    )

    train_task = train_model(
        features_path=fetch_task.outputs["output_path"],
        experiment_name=experiment_name,
        learning_rate=learning_rate,
        n_estimators=n_estimators,
        max_depth=max_depth,
    )
    train_task.set_cpu_request("4").set_memory_request("8Gi")
    train_task.set_gpu_request("1")

    evaluate_task = evaluate_and_promote(
        model_uri=train_task.outputs["model_uri"],
        auc_roc=train_task.outputs["auc_roc"],
        min_auc_threshold=min_auc_threshold,
    )

if __name__ == "__main__":
    kfp.compiler.Compiler().compile(
        pipeline_func=churn_pipeline,
        package_path="churn_prediction_pipeline.yaml",
    )
    print("Pipeline compiled successfully")
PYTHON

  log "✓ ML training pipeline created"
}

main() {
  log "Starting MLOps Platform setup..."
  install_kubeflow
  setup_mlflow
  setup_feast
  create_ml_pipeline
  log "✓ Kubeflow MLOps Platform complete"
}
main "$@"
```

---

## ขั้นตอนที่ 590: Distributed Training with Ray และ PyTorch

### `distributed-training.sh`

```bash
#!/usr/bin/env bash
# distributed-training.sh — Ray Cluster + PyTorch Distributed Training
set -euo pipefail

LOG_FILE="/var/log/distributed-training.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Ray Cluster on Kubernetes ────────────────────────────────────────────

setup_ray_cluster() {
  log "=== Setting up Ray Cluster ==="

  helm repo add kuberay https://ray-project.github.io/kuberay-helm/
  helm upgrade --install kuberay-operator kuberay/kuberay-operator \
    --namespace ray-system \
    --create-namespace \
    --version 1.1.0

  cat <<'EOF' | kubectl apply -f -
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  name: ml-training-cluster
  namespace: mlops
spec:
  rayVersion: '2.9.0'
  enableInTreeAutoscaling: true
  autoscalerOptions:
    upscalingMode: Default
    idleTimeoutSeconds: 60
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
  headGroupSpec:
    rayStartParams:
      dashboard-host: '0.0.0.0'
      num-cpus: "0"  # Head node: no tasks
    template:
      spec:
        containers:
          - name: ray-head
            image: rayproject/ray-ml:2.9.0-py311-gpu
            ports:
              - containerPort: 6379
                name: gcs-server
              - containerPort: 8265
                name: dashboard
              - containerPort: 10001
                name: client
            resources:
              requests:
                cpu: "2"
                memory: "8Gi"
              limits:
                cpu: "4"
                memory: "16Gi"
            env:
              - name: RAY_GRAFANA_IFRAME_HOST
                value: "http://grafana:3000"
              - name: RAY_GRAFANA_HOST
                value: "http://grafana:3000"
              - name: RAY_PROMETHEUS_HOST
                value: "http://kube-prometheus-stack-prometheus:9090"
  workerGroupSpecs:
    - groupName: gpu-workers
      replicas: 4
      minReplicas: 2
      maxReplicas: 16
      rayStartParams:
        num-gpus: "1"
      template:
        metadata:
          labels:
            node-type: gpu-worker
        spec:
          tolerations:
            - key: nvidia.com/gpu
              operator: Exists
              effect: NoSchedule
          containers:
            - name: ray-worker
              image: rayproject/ray-ml:2.9.0-py311-gpu
              resources:
                requests:
                  cpu: "8"
                  memory: "32Gi"
                  nvidia.com/gpu: "1"
                limits:
                  cpu: "16"
                  memory: "64Gi"
                  nvidia.com/gpu: "1"
          nodeSelector:
            accelerator: nvidia-a100
    - groupName: cpu-workers
      replicas: 8
      minReplicas: 4
      maxReplicas: 32
      template:
        spec:
          containers:
            - name: ray-worker
              image: rayproject/ray-ml:2.9.0-py311
              resources:
                requests:
                  cpu: "8"
                  memory: "16Gi"
                limits:
                  cpu: "16"
                  memory: "32Gi"
EOF

  log "✓ Ray Cluster deployed"
}

# ─── 2. PyTorch Distributed Training (DDP) ───────────────────────────────────

create_pytorch_training_job() {
  log "=== Creating PyTorch Distributed Training Job ==="

  # PyTorch training code
  cat <<'PYTHON' > /tmp/train_transformer.py
"""Distributed transformer training with PyTorch DDP + Ray Train."""
import os
import ray
import ray.train as train
from ray.train import ScalingConfig, RunConfig, CheckpointConfig
from ray.train.torch import TorchTrainer
from ray.train.integrations.mlflow import MLflowLoggerCallback
import torch
import torch.nn as nn
import torch.distributed as dist
from torch.utils.data import DataLoader, DistributedSampler
from transformers import (
    AutoModelForSequenceClassification,
    AutoTokenizer,
    get_linear_schedule_with_warmup,
)
import mlflow

def train_func(config: dict):
    """Training function — runs on each worker."""
    # Initialize process group
    train.torch.setup_distributed_training()

    rank = train.get_context().get_world_rank()
    world_size = train.get_context().get_world_size()
    device = train.torch.get_device()

    # Model + Tokenizer
    model_name = config.get("model_name", "bert-base-uncased")
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    model = AutoModelForSequenceClassification.from_pretrained(
        model_name, num_labels=config["num_labels"]
    )
    model = train.torch.prepare_model(model)

    # Optimizer
    no_decay = ["bias", "LayerNorm.weight"]
    optimizer_grouped_parameters = [
        {
            "params": [p for n, p in model.named_parameters() if not any(nd in n for nd in no_decay)],
            "weight_decay": config.get("weight_decay", 0.01),
        },
        {
            "params": [p for n, p in model.named_parameters() if any(nd in n for nd in no_decay)],
            "weight_decay": 0.0,
        },
    ]
    optimizer = torch.optim.AdamW(
        optimizer_grouped_parameters,
        lr=config.get("learning_rate", 2e-5),
    )

    num_epochs = config.get("num_epochs", 3)
    num_training_steps = config.get("steps_per_epoch", 1000) * num_epochs

    scheduler = get_linear_schedule_with_warmup(
        optimizer,
        num_warmup_steps=int(0.1 * num_training_steps),
        num_training_steps=num_training_steps,
    )

    # Training loop
    model.train()
    for epoch in range(num_epochs):
        total_loss = 0.0
        num_batches = 0

        # Simulate training steps
        for step in range(config.get("steps_per_epoch", 100)):
            # In production: load real batches from DataLoader
            input_ids = torch.randint(0, 30522, (config["batch_size"], 128)).to(device)
            attention_mask = torch.ones_like(input_ids).to(device)
            labels = torch.randint(0, config["num_labels"], (config["batch_size"],)).to(device)

            outputs = model(input_ids=input_ids, attention_mask=attention_mask, labels=labels)
            loss = outputs.loss

            optimizer.zero_grad()
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
            optimizer.step()
            scheduler.step()

            total_loss += loss.item()
            num_batches += 1

        avg_loss = total_loss / num_batches

        # Report metrics
        train.report({
            "epoch": epoch + 1,
            "train_loss": avg_loss,
            "learning_rate": scheduler.get_last_lr()[0],
        })

        if rank == 0:
            print(f"Epoch {epoch+1}/{num_epochs}: loss={avg_loss:.4f}")

    # Save checkpoint (rank 0 only)
    if rank == 0:
        model_state = {
            "model": model.module.state_dict() if hasattr(model, "module") else model.state_dict(),
            "config": config,
        }
        train.report({"epoch": num_epochs}, checkpoint=train.Checkpoint.from_dict(model_state))

def run_distributed_training():
    ray.init(address="auto")

    trainer = TorchTrainer(
        train_loop_per_worker=train_func,
        train_loop_config={
            "model_name": "bert-base-uncased",
            "num_labels": 2,
            "batch_size": 32,
            "learning_rate": 2e-5,
            "num_epochs": 3,
            "steps_per_epoch": 1000,
            "weight_decay": 0.01,
        },
        scaling_config=ScalingConfig(
            num_workers=8,
            use_gpu=True,
            resources_per_worker={"GPU": 1, "CPU": 4},
            trainer_resources={"CPU": 1},
        ),
        run_config=RunConfig(
            name="bert-text-classification",
            storage_path="s3://ml-platform/ray-results",
            checkpoint_config=CheckpointConfig(
                num_to_keep=3,
                checkpoint_score_attribute="train_loss",
                checkpoint_score_order="min",
            ),
            callbacks=[
                MLflowLoggerCallback(
                    tracking_uri="http://mlflow:5000",
                    experiment_name="bert-training",
                    save_artifact=True,
                )
            ],
        ),
    )

    result = trainer.fit()
    print(f"Training complete: {result.metrics}")
    return result

if __name__ == "__main__":
    run_distributed_training()
PYTHON

  # Kubernetes Job for distributed training
  cat <<'EOF' | kubectl apply -f -
apiVersion: ray.io/v1
kind: RayJob
metadata:
  name: bert-training-job
  namespace: mlops
spec:
  submissionMode: K8sJobMode
  entrypoint: python train_transformer.py
  runtimeEnvYAML: |
    working_dir: s3://ml-platform/code/
    pip:
      - torch==2.1.0
      - transformers==4.36.0
      - ray[train]==2.9.0
      - mlflow==2.9.0
    env_vars:
      MLFLOW_TRACKING_URI: "http://mlflow:5000"
  rayClusterSpec:
    rayVersion: '2.9.0'
    headGroupSpec:
      template:
        spec:
          containers:
            - name: ray-head
              image: rayproject/ray-ml:2.9.0-py311-gpu
              resources:
                requests:
                  cpu: "2"
                  memory: "8Gi"
    workerGroupSpecs:
      - groupName: gpu-workers
        replicas: 8
        template:
          spec:
            tolerations:
              - key: nvidia.com/gpu
                operator: Exists
                effect: NoSchedule
            containers:
              - name: ray-worker
                image: rayproject/ray-ml:2.9.0-py311-gpu
                resources:
                  requests:
                    cpu: "8"
                    memory: "32Gi"
                    nvidia.com/gpu: "1"
                  limits:
                    nvidia.com/gpu: "1"
EOF

  log "✓ PyTorch distributed training job created"
}

main() {
  log "Starting Distributed Training setup..."
  setup_ray_cluster
  create_pytorch_training_job
  log "✓ Distributed Training Platform complete"
}
main "$@"
```

---

## ขั้นตอนที่ 591: KServe Model Serving Platform

### `model-serving.sh`

```bash
#!/usr/bin/env bash
# model-serving.sh — KServe Enterprise Model Serving
set -euo pipefail

LOG_FILE="/var/log/model-serving.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. KServe InferenceService ───────────────────────────────────────────────

deploy_inference_services() {
  log "=== Deploying KServe Inference Services ==="

  # Churn Prediction (XGBoost)
  cat <<'EOF' | kubectl apply -f -
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: churn-predictor
  namespace: mlops
  annotations:
    serving.kserve.io/deploymentMode: RawDeployment
spec:
  predictor:
    model:
      modelFormat:
        name: mlflow
      storageUri: s3://ml-platform/models/churn-prediction/Production
      protocolVersion: v2
    minReplicas: 2
    maxReplicas: 20
    scaleTarget: 100  # requests per second per replica
    scaleMetric: rps
    resources:
      requests:
        cpu: "1"
        memory: 2Gi
      limits:
        cpu: "4"
        memory: 8Gi
    logger:
      mode: all
      url: http://inference-logger:9000
  transformer:
    containers:
      - name: feature-transformer
        image: ghcr.io/example/feature-transformer:v1.0
        env:
          - name: FEAST_ENDPOINT
            value: http://feast-feature-server:6566
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
    minReplicas: 2
    maxReplicas: 10
  explainer:
    containers:
      - name: shap-explainer
        image: ghcr.io/example/shap-explainer:v1.0
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
---
# LLM Inference (vLLM on A100)
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llm-llama3
  namespace: mlops
spec:
  predictor:
    model:
      modelFormat:
        name: huggingface
      storageUri: s3://ml-platform/models/llama3-8b
    minReplicas: 1
    maxReplicas: 4
    resources:
      requests:
        cpu: "4"
        memory: 16Gi
        nvidia.com/gpu: "1"
      limits:
        cpu: "8"
        memory: 40Gi
        nvidia.com/gpu: "1"
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
    nodeSelector:
      accelerator: nvidia-a100
    containers:
      - name: vllm
        image: vllm/vllm-openai:v0.3.0
        args:
          - --model=/mnt/models
          - --tensor-parallel-size=1
          - --max-num-seqs=32
          - --gpu-memory-utilization=0.90
          - --dtype=bfloat16
          - --enable-chunked-prefill
EOF

  log "✓ KServe Inference Services deployed"
}

# ─── 2. A/B Testing and Canary Deployment ─────────────────────────────────────

setup_model_ab_testing() {
  log "=== Setting up Model A/B Testing ==="

  # Traffic split: 90% model-v1, 10% model-v2
  cat <<'EOF' | kubectl apply -f -
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: churn-predictor-ab
  namespace: mlops
spec:
  predictor:
    canaryTrafficPercent: 10
    model:
      modelFormat:
        name: mlflow
      storageUri: s3://ml-platform/models/churn-prediction/v2
    canary:
      model:
        modelFormat:
          name: mlflow
        storageUri: s3://ml-platform/models/churn-prediction/v3
EOF

  # Model Monitoring with Evidently
  cat <<'PYTHON' > /tmp/model_monitor.py
"""Production model drift monitoring with Evidently."""
import pandas as pd
import numpy as np
from evidently import ColumnMapping
from evidently.report import Report
from evidently.metric_suite import MetricSuite
from evidently.metrics import (
    DataDriftTable,
    DatasetDriftMetric,
    DatasetMissingValuesMetric,
    ClassificationQualityMetric,
    ClassificationClassBalance,
)
from datetime import datetime, timedelta
import boto3
import json
import httpx

class ModelMonitor:
    def __init__(self, model_name: str):
        self.model_name = model_name
        self.s3 = boto3.client('s3')
        self.bucket = "ml-platform"

    def load_reference_data(self) -> pd.DataFrame:
        """Load training data as reference."""
        df = pd.read_parquet(
            f"s3://{self.bucket}/features/reference/{self.model_name}.parquet"
        )
        return df

    def load_production_data(self, hours: int = 24) -> pd.DataFrame:
        """Load recent prediction requests from inference logger."""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours)

        # Load from inference logger (Kafka consumer or S3)
        df = pd.read_parquet(
            f"s3://{self.bucket}/inference-logs/{self.model_name}/"
            f"{start_time.strftime('%Y/%m/%d')}.parquet"
        )
        return df[df['timestamp'] >= start_time.isoformat()]

    def run_drift_detection(self) -> dict:
        reference_data = self.load_reference_data()
        production_data = self.load_production_data()

        if len(production_data) < 100:
            return {"status": "insufficient_data", "samples": len(production_data)}

        column_mapping = ColumnMapping(
            target="churn_label",
            prediction="prediction",
            numerical_features=[
                "total_orders", "lifetime_value",
                "avg_order_value", "days_since_last_order",
            ],
            categorical_features=["preferred_category"],
        )

        report = Report(metrics=[
            DatasetDriftMetric(),
            DataDriftTable(),
            DatasetMissingValuesMetric(),
            ClassificationQualityMetric(),
        ])

        report.run(
            reference_data=reference_data,
            current_data=production_data,
            column_mapping=column_mapping,
        )

        result = report.as_dict()
        drift_detected = result['metrics'][0]['result']['dataset_drift']

        # Alert if drift detected
        if drift_detected:
            self._send_drift_alert(result)

        # Save report to S3
        report_path = f"s3://{self.bucket}/drift-reports/{self.model_name}/{datetime.utcnow().isoformat()}.json"
        self.s3.put_object(
            Bucket=self.bucket,
            Key=f"drift-reports/{self.model_name}/{datetime.utcnow().strftime('%Y%m%d_%H%M')}.json",
            Body=json.dumps(result),
        )

        return {
            "drift_detected": drift_detected,
            "samples_analyzed": len(production_data),
            "report_saved": report_path,
        }

    def _send_drift_alert(self, report: dict):
        """Send Slack alert when model drift is detected."""
        webhook_url = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
        message = {
            "text": f":warning: *Model Drift Alert*: `{self.model_name}`",
            "blocks": [
                {
                    "type": "section",
                    "text": {
                        "type": "mrkdwn",
                        "text": f"*Dataset drift detected* in `{self.model_name}`\n"
                                f"Consider retraining the model.",
                    }
                }
            ]
        }
        httpx.post(webhook_url, json=message, timeout=5.0)

if __name__ == "__main__":
    monitor = ModelMonitor("churn-prediction")
    result = monitor.run_drift_detection()
    print(json.dumps(result, indent=2))
PYTHON

  # Kubernetes CronJob: hourly drift monitoring
  cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: model-drift-monitor
  namespace: mlops
spec:
  schedule: "0 * * * *"  # hourly
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: monitor
              image: ghcr.io/example/model-monitor:latest
              command: [python, model_monitor.py]
              env:
                - name: MODEL_NAME
                  value: churn-prediction
              resources:
                requests:
                  cpu: 500m
                  memory: 2Gi
          restartPolicy: OnFailure
EOF

  log "✓ Model A/B testing and monitoring configured"
}

main() {
  log "Starting Model Serving Platform..."
  deploy_inference_services
  setup_model_ab_testing
  log "✓ Model Serving Platform complete"
}
main "$@"
```

---

## ขั้นตอนที่ 592: Hyperparameter Optimization (Katib)

### `hyperparameter-optimization.sh`

```bash
#!/usr/bin/env bash
# hyperparameter-optimization.sh — Katib HPO with Bayesian Optimization
set -euo pipefail

LOG_FILE="/var/log/hpo.log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# ─── 1. Katib Experiment ─────────────────────────────────────────────────────

create_katib_experiment() {
  log "=== Creating Katib HPO Experiment ==="

  cat <<'EOF' | kubectl apply -f -
apiVersion: kubeflow.org/v1beta1
kind: Experiment
metadata:
  name: churn-model-hpo
  namespace: mlops
spec:
  objective:
    type: maximize
    goal: 0.90
    objectiveMetricName: val_auc
    additionalMetricNames:
      - train_loss
      - val_precision
      - val_recall
  algorithm:
    algorithmName: bayesianoptimization
    algorithmSettings:
      - name: "random_state"
        value: "42"
      - name: "acq_func"
        value: "gp_hedge"
  parallelTrialCount: 4
  maxTrialCount: 30
  maxFailedTrialCount: 5
  parameters:
    - name: learning_rate
      parameterType: double
      feasibleSpace:
        min: "0.0001"
        max: "0.5"
        step: "0.0001"
    - name: n_estimators
      parameterType: int
      feasibleSpace:
        min: "100"
        max: "1000"
        step: "50"
    - name: max_depth
      parameterType: int
      feasibleSpace:
        min: "3"
        max: "12"
    - name: min_child_weight
      parameterType: int
      feasibleSpace:
        min: "1"
        max: "10"
    - name: subsample
      parameterType: double
      feasibleSpace:
        min: "0.5"
        max: "1.0"
        step: "0.05"
    - name: colsample_bytree
      parameterType: double
      feasibleSpace:
        min: "0.5"
        max: "1.0"
        step: "0.05"
  trialTemplate:
    primaryContainerName: training-container
    trialParameters:
      - name: learningRate
        description: Learning rate
        reference: learning_rate
      - name: nEstimators
        description: Number of trees
        reference: n_estimators
      - name: maxDepth
        description: Max tree depth
        reference: max_depth
      - name: minChildWeight
        reference: min_child_weight
      - name: subsample
        reference: subsample
      - name: colsampleBytree
        reference: colsample_bytree
    trialSpec:
      apiVersion: batch/v1
      kind: Job
      spec:
        template:
          spec:
            containers:
              - name: training-container
                image: ghcr.io/example/xgb-trainer:latest
                command:
                  - python
                  - train_with_katib.py
                  - --learning-rate=${trialParameters.learningRate}
                  - --n-estimators=${trialParameters.nEstimators}
                  - --max-depth=${trialParameters.maxDepth}
                  - --min-child-weight=${trialParameters.minChildWeight}
                  - --subsample=${trialParameters.subsample}
                  - --colsample-bytree=${trialParameters.colsampleBytree}
                resources:
                  requests:
                    cpu: "2"
                    memory: 4Gi
                  limits:
                    cpu: "4"
                    memory: 8Gi
            restartPolicy: Never
EOF

  # Training script for Katib (reports metrics via stdout)
  cat <<'PYTHON' > /tmp/train_with_katib.py
"""XGBoost trainer compatible with Katib metrics collection."""
import argparse
import xgboost as xgb
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score, precision_score, recall_score

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--learning-rate", type=float, default=0.1)
    parser.add_argument("--n-estimators", type=int, default=300)
    parser.add_argument("--max-depth", type=int, default=6)
    parser.add_argument("--min-child-weight", type=int, default=3)
    parser.add_argument("--subsample", type=float, default=0.8)
    parser.add_argument("--colsample-bytree", type=float, default=0.8)
    args = parser.parse_args()

    # Load data
    df = pd.read_parquet("s3://ml-platform/features/train.parquet")
    X = df.drop(columns=["customer_id", "churn_label", "event_timestamp"])
    y = df["churn_label"]

    X_train, X_val, y_train, y_val = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)

    model = xgb.XGBClassifier(
        n_estimators=args.n_estimators,
        max_depth=args.max_depth,
        learning_rate=args.learning_rate,
        min_child_weight=args.min_child_weight,
        subsample=args.subsample,
        colsample_bytree=args.colsample_bytree,
        use_label_encoder=False,
        eval_metric="logloss",
    )

    model.fit(
        X_train, y_train,
        eval_set=[(X_val, y_val)],
        verbose=False,
    )

    # Get metrics
    evals_result = model.evals_result()
    train_loss = evals_result['validation_0']['logloss'][-1]

    y_pred_proba = model.predict_proba(X_val)[:, 1]
    y_pred = (y_pred_proba > 0.5).astype(int)
    val_auc = roc_auc_score(y_val, y_pred_proba)
    val_precision = precision_score(y_val, y_pred)
    val_recall = recall_score(y_val, y_pred)

    # Katib reads metrics from stdout in format: "metric_name=value"
    print(f"train_loss={train_loss:.6f}")
    print(f"val_auc={val_auc:.6f}")
    print(f"val_precision={val_precision:.6f}")
    print(f"val_recall={val_recall:.6f}")

if __name__ == "__main__":
    main()
PYTHON

  log "✓ Katib HPO experiment created"
}

main() {
  log "Starting Hyperparameter Optimization..."
  create_katib_experiment
  log "✓ HPO complete"
}
main "$@"
```

---

## สรุป Part 60

| ขั้นตอน | หัวข้อ | เทคโนโลยีหลัก |
|---------|--------|---------------|
| 589 | Kubeflow MLOps Platform | Kubeflow Pipelines, MLflow, Feast Feature Store |
| 590 | Distributed Training | Ray Cluster, PyTorch DDP, RayJob on K8s |
| 591 | Model Serving | KServe InferenceService, vLLM, Evidently drift detection |
| 592 | Hyperparameter Optimization | Katib Bayesian Optimization, XGBoost, metric collection |

### สิ่งที่ทำให้ระบบ ML World-class

1. **Feature Store (Feast)** — Online/offline consistency, streaming features, on-demand computed features
2. **Kubeflow Pipelines** — Reproducible ML workflows, artifact tracking, versioned components
3. **Ray Distributed** — Elastic compute สำหรับ training, dynamic scaling บน GPU workers
4. **KServe** — Kubernetes-native model serving พร้อม A/B testing, canary, shadow mode
5. **Katib Bayesian HPO** — ค้นหา hyperparameters ด้วย Gaussian Process แทน grid search

---

### Module 5 Progress (Steps 569 → 592)

| Part | Steps | Topics |
|------|-------|--------|
| 55 | 569-572 | WASM, Knative, FinTech, LLM Infrastructure |
| 56 | 573-576 | Quantum Security, eBPF, Confidential Computing, API Marketplace |
| 57 | 577-580 | Multi-Cloud, Edge Computing, Custom Operators, Data Engineering |
| 58 | 581-584 | Real-Time Analytics, Event Sourcing, Digital Twin, Green Computing |
| 59 | 585-588 | Blockchain, Advanced Service Mesh, Zero Trust, Advanced Networking |
| 60 | 589-592 | MLOps, Distributed Training, Model Serving, HPO |

**Steps remaining to 1000: 408 steps across Parts 61-100+**

ขั้นตอนต่อไป: **Part 61** — Advanced Observability, SLO Engineering และ Capacity Intelligence
