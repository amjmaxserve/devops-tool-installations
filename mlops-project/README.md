# End-to-End Production-Style MLOps Platform on Kubernetes

## Overview

This document explains how to build a complete production-style MLOps platform from scratch on an on-premise Ubuntu server using:

* Docker
* Kubernetes (k3s)
* MLflow
* MinIO
* PostgreSQL
* FastAPI
* Prometheus
* Grafana
* Prometheus Operator
* ServiceMonitor

The platform includes:

* Model training
* Experiment tracking
* Artifact storage
* Model serving
* Kubernetes deployment
* Monitoring
* Metrics visualization
* Real-time observability

---

# Final Architecture

```text
                    ┌──────────────────────┐
                    │   Client / Browser   │
                    └──────────┬───────────┘
                               │
                     Prediction Requests
                               │
                               ▼
                    ┌──────────────────────┐
                    │     FastAPI API      │
                    │   iris-api service   │
                    └──────────┬───────────┘
                               │
                               │ Loads Model
                               ▼
                    ┌──────────────────────┐
                    │        MinIO         │
                    │ Artifact/Object Store│
                    └──────────┬───────────┘
                               │
                               │ MLflow Artifacts
                               ▼
                    ┌──────────────────────┐
                    │       MLflow         │
                    │ Experiment Tracking  │
                    └──────────┬───────────┘
                               │
                               │ Metadata
                               ▼
                    ┌──────────────────────┐
                    │      PostgreSQL      │
                    │ Tracking Backend DB  │
                    └──────────────────────┘


                    ┌──────────────────────┐
                    │     Prometheus       │
                    │ Metrics Collection   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │       Grafana        │
                    │ Visualization Layer  │
                    └──────────────────────┘
```

---

# PHASE 1 — Ubuntu Server Preparation

## System Requirements

Recommended:

* Ubuntu 24.04 LTS
* 8 GB RAM minimum
* 4 vCPU minimum
* 50+ GB storage

---

## Update System

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Install Required Packages

```bash
sudo apt install -y \
curl \
wget \
git \
vim \
net-tools \
apt-transport-https \
ca-certificates \
gnupg \
lsb-release
```

---

# PHASE 2 — Install Docker

## Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
```

---

## Add User To Docker Group

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## Verify Docker

```bash
docker version
```

---

# PHASE 3 — Create Project Structure

## Create Project Directory

```bash
mkdir -p ~/mlops-platform
cd ~/mlops-platform
```

---

## Create Folder Structure

```bash
mkdir -p \
app \
training \
models \
k8s/api \
k8s/trainer \
k8s/mlflow \
k8s/minio \
k8s/postgres \
k8s/monitoring
```

---

# PHASE 4 — Build ML Training Pipeline

## Create Training Requirements

Create:

```text
training/requirements.txt
```

```txt
mlflow==2.11.3
scikit-learn
pandas
numpy
boto3
joblib
psycopg2-binary
```

---

## Create Training Script

Create:

```text
training/train.py
```

```python
import os
import joblib
import mlflow
import mlflow.sklearn

from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score


MLFLOW_TRACKING_URI = os.getenv("MLFLOW_TRACKING_URI")

mlflow.set_tracking_uri(MLFLOW_TRACKING_URI)
mlflow.set_experiment("iris-classification")


iris = load_iris()
X = iris.data
y = iris.target

X_train, X_test, y_train, y_test = train_test_split(
    X,
    y,
    test_size=0.2,
    random_state=42
)

model = RandomForestClassifier()
model.fit(X_train, y_train)

predictions = model.predict(X_test)
accuracy = accuracy_score(y_test, predictions)

os.makedirs("/models", exist_ok=True)
joblib.dump(model, "/models/model.pkl")

with mlflow.start_run():
    mlflow.log_metric("accuracy", accuracy)
    mlflow.sklearn.log_model(model, "model")

print("Training completed.")
print(f"Accuracy: {accuracy}")
```

---

## Create Training Dockerfile

Create:

```text
training/Dockerfile
```

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY train.py .

CMD ["python", "train.py"]
```

---

# PHASE 5 — Build FastAPI Inference Service

## Create API Requirements

Create:

```text
app/requirements.txt
```

```txt
fastapi
uvicorn
scikit-learn
joblib
boto3
prometheus_client
```

---

## Create FastAPI Application

Create:

```text
app/main.py
```

```python
import os
import time
import joblib
import boto3

from fastapi import FastAPI
from pydantic import BaseModel

from prometheus_client import Counter, Histogram, generate_latest
from fastapi.responses import Response


PREDICTION_COUNTER = Counter(
    "prediction_requests_total",
    "Total prediction requests"
)

PREDICTION_LATENCY = Histogram(
    "prediction_latency_seconds",
    "Prediction latency"
)


MODEL_PATH = "/tmp/model.pkl"


s3 = boto3.client(
    's3',
    endpoint_url='http://minio.mlops.svc.cluster.local:9000',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin'
)

print("Downloading model from MinIO...")

s3.download_file(
    'mlflow',
    'mlflow/1/latest/artifacts/model/model.pkl',
    MODEL_PATH
)

model = joblib.load(MODEL_PATH)

app = FastAPI()


class IrisRequest(BaseModel):
    features: list


@app.post("/predict")
def predict(data: IrisRequest):
    start = time.time()

    prediction = model.predict([data.features])[0]

    latency = time.time() - start

    PREDICTION_COUNTER.inc()
    PREDICTION_LATENCY.observe(latency)

    return {
        "prediction": int(prediction),
        "latency": latency
    }


@app.get("/metrics")
def metrics():
    return Response(
        generate_latest(),
        media_type="text/plain"
    )
```

---

## Create API Dockerfile

Create:

```text
app/Dockerfile
```

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

CMD [
  "uvicorn",
  "main:app",
  "--host",
  "0.0.0.0",
  "--port",
  "8000"
]
```

---

# PHASE 6 — Install Kubernetes (k3s)

## Install k3s

```bash
curl -sfL https://get.k3s.io | sh -
```

---

## Give kubeconfig Permissions

```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

---

## Export kubeconfig

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

---

## Verify Cluster

```bash
kubectl get nodes
```

---

# PHASE 7 — Deploy Kubernetes Resources

## Create Namespace

```bash
kubectl create namespace mlops
```

---

# PostgreSQL Deployment

## Create:

```text
k8s/postgres/postgres.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: mlops

spec:
  replicas: 1

  selector:
    matchLabels:
      app: postgres

  template:
    metadata:
      labels:
        app: postgres

    spec:
      containers:
      - name: postgres
        image: postgres:15

        env:
        - name: POSTGRES_USER
          value: mlflow

        - name: POSTGRES_PASSWORD
          value: mlflow123

        - name: POSTGRES_DB
          value: mlflow

        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: mlops

spec:
  selector:
    app: postgres

  ports:
  - port: 5432
```

---

# MinIO Deployment

## Create:

```text
k8s/minio/minio.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: mlops

spec:
  replicas: 1

  selector:
    matchLabels:
      app: minio

  template:
    metadata:
      labels:
        app: minio

    spec:
      containers:
      - name: minio
        image: minio/minio

        args:
        - server
        - /data
        - --console-address
        - ":9001"

        env:
        - name: MINIO_ROOT_USER
          value: minioadmin

        - name: MINIO_ROOT_PASSWORD
          value: minioadmin

        ports:
        - containerPort: 9000
        - containerPort: 9001
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: mlops

spec:
  type: NodePort

  selector:
    app: minio

  ports:
  - name: api
    port: 9000
    targetPort: 9000

  - name: console
    port: 9001
    targetPort: 9001
```

---

# MLflow Deployment

## Create:

```text
k8s/mlflow/mlflow.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow
  namespace: mlops

spec:
  replicas: 1

  selector:
    matchLabels:
      app: mlflow

  template:
    metadata:
      labels:
        app: mlflow

    spec:
      containers:
      - name: mlflow
        image: ghcr.io/mlflow/mlflow:v2.11.3

        command:
        - mlflow
        - server

        args:
        - --backend-store-uri
        - postgresql://mlflow:mlflow123@postgres:5432/mlflow
        - --default-artifact-root
        - s3://mlflow/
        - --host
        - 0.0.0.0
        - --port
        - "5000"

        env:
        - name: AWS_ACCESS_KEY_ID
          value: minioadmin

        - name: AWS_SECRET_ACCESS_KEY
          value: minioadmin

        - name: MLFLOW_S3_ENDPOINT_URL
          value: http://minio:9000

        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow
  namespace: mlops

spec:
  type: NodePort

  selector:
    app: mlflow

  ports:
  - port: 5000
    targetPort: 5000
```

---

# Training Job Deployment

## Build Training Image

```bash
docker build -t iris-trainer:latest training/
```

---

## Import Image To k3s

```bash
sudo k3s ctr images import iris-trainer.tar
```

---

## Create Training Job

```text
k8s/trainer/train-job.yaml
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: iris-trainer
  namespace: mlops

spec:
  template:
    spec:
      restartPolicy: Never

      containers:
      - name: trainer
        image: iris-trainer:latest
        imagePullPolicy: Never

        env:
        - name: MLFLOW_TRACKING_URI
          value: http://mlflow:5000
```

---

## Apply Training Job

```bash
kubectl apply -f k8s/trainer/
```

---

## Check Logs

```bash
kubectl logs job/iris-trainer -n mlops
```

---

# API Deployment

## Build API Image

```bash
docker build -t iris-api:latest app/
```

---

## Create API Deployment

Create:

```text
k8s/api/api.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iris-api
  namespace: mlops

spec:
  replicas: 2

  selector:
    matchLabels:
      app: iris-api

  template:
    metadata:
      labels:
        app: iris-api

    spec:
      containers:
      - name: iris-api
        image: iris-api:latest
        imagePullPolicy: Never

        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: iris-api
  namespace: mlops

  labels:
    app: iris-api

spec:
  type: NodePort

  selector:
    app: iris-api

  ports:
  - name: http
    port: 8000
    targetPort: 8000
```

---

## Deploy API

```bash
kubectl apply -f k8s/api/
```

---

# PHASE 8 — Test Inference

## Get Service Port

```bash
kubectl get svc -n mlops
```

---

## Send Prediction Request

```bash
curl -X POST http://SERVER_IP:NODEPORT/predict \
-H "Content-Type: application/json" \
-d '{"features":[5.1,3.5,1.4,0.2]}'
```

Expected:

```json
{
  "prediction": 0,
  "latency": 0.002
}
```

---

# PHASE 9 — Install Monitoring Stack

## Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

---

## Add Helm Repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## Install kube-prometheus-stack

```bash
helm install kube-prometheus-stack \
prometheus-community/kube-prometheus-stack \
-n monitoring
```

---

## Verify Monitoring Pods

```bash
kubectl get pods -n monitoring
```

---

# Expose Grafana

```bash
kubectl patch svc kube-prometheus-stack-grafana \
-n monitoring \
-p '{"spec":{"type":"NodePort"}}'
```

---

# Expose Prometheus

```bash
kubectl patch svc kube-prometheus-stack-prometheus \
-n monitoring \
-p '{"spec":{"type":"NodePort"}}'
```

---

## Get Services

```bash
kubectl get svc -n monitoring
```

---

# PHASE 10 — Configure ServiceMonitor

## Create:

```text
k8s/monitoring/iris-api-monitor.yaml
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: iris-api-monitor
  namespace: monitoring

  labels:
    release: kube-prometheus-stack

spec:
  selector:
    matchLabels:
      app: iris-api

  namespaceSelector:
    matchNames:
      - mlops

  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

---

## Apply ServiceMonitor

```bash
kubectl apply -f k8s/monitoring/
```

---

## Verify Target Discovery

Open Prometheus:

```text
Status → Target Health
```

Expected:

```text
iris-api-monitor → UP
```

---

# PHASE 11 — Create Grafana Dashboard

## Dashboard Panels

### Total Predictions

```promql
sum(prediction_requests_total)
```

---

### Predictions Per Second

```promql
sum(rate(prediction_requests_total[1m]))
```

---

### Average Prediction Latency

```promql
sum(rate(prediction_latency_seconds_sum[1m]))
/
sum(rate(prediction_latency_seconds_count[1m]))
```

---

### API CPU Usage

```promql
sum(
  rate(container_cpu_usage_seconds_total{
    pod=~"iris-api.*",
    container="iris-api"
  }[1m])
)
```

---

### API Memory Usage

```promql
sum(
  container_memory_usage_bytes{
    pod=~"iris-api.*",
    container="iris-api"
  }
) / 1024 / 1024
```

---

### Pod Restarts

```promql
sum(
  kube_pod_container_status_restarts_total{
    namespace="mlops",
    pod=~"iris-api.*"
  }
)
```

---

### Node CPU Usage

```promql
100 - (
  avg by(instance)(
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
```

---

### Node Memory Usage

```promql
(
1 - (
node_memory_MemAvailable_bytes
/
node_memory_MemTotal_bytes
)
) * 100
```

---

# PHASE 12 — Generate Real-Time Traffic

## Continuous Traffic Generator

```bash
while true

do
  curl -s -X POST http://SERVER_IP:NODEPORT/predict \
  -H "Content-Type: application/json" \
  -d '{"features":[5.1,3.5,1.4,0.2]}'

  sleep 0.2

done
```

---

## Heavy Load Test

```bash
while true

do
  for i in {1..50}
  do
    curl -s -X POST http://SERVER_IP:NODEPORT/predict \
    -H "Content-Type: application/json" \
    -d '{"features":[5.1,3.5,1.4,0.2]}' &
  done

  wait

done
```

---

# PHASE 13 — Kubernetes Self-Healing Tests

## Delete Pods

```bash
kubectl delete pod -n mlops -l app=iris-api
```

Observe:

* Kubernetes recreates pods
* Grafana restart metrics increase
* Availability recovers automatically

---

## Crash Containers

```bash
kubectl exec -it POD_NAME -n mlops -- sh
```

Then:

```bash
kill 1
```

Observe:

* Restart count increases
* Kubernetes self-healing occurs
* Grafana updates in real time

---

# Production Concepts Learned

## Kubernetes Concepts

* Deployments
* Services
* NodePort
* Jobs
* Namespaces
* ReplicaSets
* Pod self-healing
* Internal DNS
* Metrics scraping

---

## MLOps Concepts

* Experiment tracking
* Model artifact storage
* ML inference serving
* Monitoring
* Metrics collection
* Model lifecycle
* Observability

---

## Monitoring Concepts

* Prometheus Operator
* ServiceMonitor
* Grafana dashboards
* Time-series metrics
* Request throughput
* Latency monitoring
* CPU monitoring
* Memory monitoring

---

# Current Platform Status

| Component            | Status   |
| -------------------- | -------- |
| Kubernetes           | Complete |
| MLflow               | Complete |
| PostgreSQL           | Complete |
| MinIO                | Complete |
| FastAPI Inference    | Complete |
| Model Loading        | Complete |
| Prometheus           | Complete |
| Grafana              | Complete |
| ServiceMonitor       | Complete |
| Real-time Monitoring | Complete |
| Pod Restart Metrics  | Complete |
| Traffic Simulation   | Complete |

---

# Recommended Next Steps

## High Priority

* Loki + Promtail logging
* AlertManager rules
* Horizontal Pod Autoscaler
* Ingress Controller
* TLS/HTTPS

---

## Medium Priority

* ArgoCD GitOps
* CI/CD pipelines
* KServe
* Canary deployments

---

## Advanced Topics

* Drift detection
* Model version rollout
* Multi-model serving
* GPU workloads
* Distributed training
* Service mesh

---

# Final Result

You now have a complete Kubernetes-native MLOps platform capable of:

* Training ML models
* Tracking experiments
* Storing artifacts
* Serving inference APIs
* Monitoring infrastructure
* Monitoring application performance
* Visualizing metrics in Grafana
* Handling Kubernetes self-healing
* Scaling into enterprise-grade architecture

This platform closely resembles real production MLOps infrastructure used in enterprise environments.
