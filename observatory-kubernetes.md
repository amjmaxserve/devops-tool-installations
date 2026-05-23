# Kubernetes Observability Stack Deployment Guide

## Overview

This document describes the complete deployment of a Kubernetes observability stack using:

1. KIND (Kubernetes in Docker)
2. Helm
3. kube-prometheus-stack (Prometheus Operator)
4. Prometheus
5. Grafana
6. Alertmanager
7. kube-state-metrics
8. node-exporter

The stack is deployed inside a dedicated Kubernetes namespace called `monitoring`.

## Architecture

```text
KIND Cluster
├── Control Plane
├── Worker Node
└── monitoring namespace
    ├── Prometheus Operator
    ├── Prometheus
    ├── Grafana
    ├── Alertmanager
    ├── kube-state-metrics
    └── node-exporter
```

## Environment Details

| Component           | Value                                                         |
|--------------------|---------------------------------------------------------------|
| OS                 | Ubuntu 24.04                                                  |
| Kubernetes Runtime | KIND                                                          |
| Container Runtime  | Docker                                                        |
| Package Manager    | Helm                                                          |
| Namespace          | monitoring                                                    |
| Host IP            | 192.168.29.8                                                  |

## Step 1 — Install Docker

```bash
sudo apt update -y
sudo apt install -y docker.io docker-compose
```

Add your user to the Docker group and then log out and log back in:

```bash
sudo usermod -aG docker $USER
```

## Step 2 — Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify kubectl:

```bash
kubectl version --client
```

## Step 3 — Install KIND

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/
```

Verify KIND:

```bash
kind version
```

## Step 4 — Create KIND Cluster Configuration

Create a working directory and a cluster config file:

```bash
mkdir -p kind-monitoring
cd kind-monitoring
cat > kind-config.yaml <<'YAML'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30000
        hostPort: 30000
        protocol: TCP
      - containerPort: 30001
        hostPort: 30001
        protocol: TCP
      - containerPort: 30002
        hostPort: 30002
        protocol: TCP
  - role: worker
YAML
```

## Step 5 — Create KIND Cluster

```bash
kind create cluster --name monitoring-cluster --config kind-config.yaml
```

Verify the cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

## Step 6 — Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify Helm:

```bash
helm version
```

## Step 7 — Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

## Step 8 — Add Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## Step 9 — Install kube-prometheus-stack

```bash
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
```

## Step 10 — Verify Pods

```bash
kubectl get pods -n monitoring
```

## Step 11 — Expose Services Using NodePort

### Expose Grafana

```bash
kubectl patch svc monitoring-grafana -n monitoring \
  -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":3000,"nodePort":30000}]}}'
```

### Expose Prometheus

```bash
kubectl patch svc monitoring-kube-prometheus-prometheus -n monitoring \
  -p '{"spec":{"type":"NodePort","ports":[{"port":9090,"targetPort":9090,"nodePort":30001}]}}'
```

### Expose Alertmanager

```bash
kubectl patch svc monitoring-kube-prometheus-alertmanager -n monitoring \
  -p '{"spec":{"type":"NodePort","ports":[{"port":9093,"targetPort":9093,"nodePort":30002}]}}'
```

## Step 12 — Access Services

| Service      | URL                                      |
|-------------|------------------------------------------|
| Grafana     | http://192.168.29.8:30000                |
| Prometheus  | http://192.168.29.8:30001                |
| Alertmanager| http://192.168.29.8:30002                |

## Step 13 — Get Grafana Password

```bash
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode && echo
```

## Step 14 — Configure Prometheus Data Source in Grafana

In Grafana, go to:

- `Connections` → `Add new connection` → `Prometheus`

Use the following Data Source URL:

```text
http://monitoring-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
```

Save and test.

## Step 15 — Import Dashboards

Use these dashboard IDs in Grafana:

- Kubernetes Cluster Monitoring: `315`
- Node Exporter Full: `1860`
- Kubernetes Pod Monitoring: `6417`
- Kubernetes Views Global: `15757`
- Kubernetes API Server: `15761`

## Useful Commands

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get deployments -n monitoring
kubectl get statefulsets -n monitoring
kubectl logs -n monitoring <pod-name>
kubectl describe pod -n monitoring <pod-name>
```

## Kubernetes Resources Used

| Resource    | Purpose                         |
|-------------|---------------------------------|
| Namespace   | Logical isolation               |
| Deployment  | Stateless applications          |
| StatefulSet | Stateful workloads              |
| DaemonSet   | Run pods on all nodes           |
| Service     | Pod networking and access       |
| CRD         | Custom monitoring resources     |

## Troubleshooting

Check the cluster:

```bash
kubectl cluster-info
kubectl get nodes
kubectl get all -n monitoring
```

Verify Prometheus targets in the Prometheus UI under `Status` → `Targets`. All targets should show `up`.

## Observability Workflow

The deployed stack provides:

- Metrics collection with Prometheus
- Dashboard visualization with Grafana
- Alert routing with Alertmanager
- Kubernetes state monitoring with kube-state-metrics
- Node monitoring with node-exporter

## Conclusion

A complete Kubernetes observability platform was deployed using KIND and Helm. This setup delivers:

- Cluster monitoring
- Node monitoring
- Kubernetes resource visibility
- Metrics visualization
- Alerting infrastructure
- Cloud-native observability

This workflow is aligned with real-world Kubernetes monitoring environments for DevOps, SRE, Platform Engineering, and MLOps teams.
