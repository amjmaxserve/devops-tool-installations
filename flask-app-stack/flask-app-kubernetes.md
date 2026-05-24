# Flask App Kubernetes Deployment Guide

This document describes the full Kubernetes deployment workflow for a Flask application backed by PostgreSQL using KIND, Secrets, ConfigMaps, PVCs, and NodePort access. It also includes a Helm chart creation section for packaging the same stack.

## Overview

The final Kubernetes architecture contains the following resources inside the `flask-app` namespace:

- `Namespace`: flask-app
- `Secret`: postgres-secret
- `ConfigMap`: postgres-config
- `PersistentVolumeClaim`: postgres-pvc
- `Deployment`: postgres
- `Service`: postgres-service
- `Deployment`: flask-app
- `Service`: flask-service

## Prerequisites

Install the following tools on your Ubuntu host before beginning:

- Docker
- KIND
- kubectl
- Helm

## Phase 1 — Create the Kubernetes Namespace

1. Create the namespace:

```bash
kubectl create namespace flask-app
```

2. Verify the namespace:

```bash
kubectl get namespaces
```

## Phase 2 — Create a Secret for PostgreSQL Credentials

Create `postgres-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: flask-app
type: Opaque
stringData:
  POSTGRES_USER: flaskuser
  POSTGRES_PASSWORD: flaskpass
  POSTGRES_DB: flaskdb
```

Apply the secret:

```bash
kubectl apply -f postgres-secret.yaml
```

## Phase 3 — Create a ConfigMap for PostgreSQL Connection Data

Create `postgres-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: flask-app
data:
  POSTGRES_HOST: postgres-service
```

Apply the ConfigMap:

```bash
kubectl apply -f postgres-configmap.yaml
```

## Phase 4 — Create a Persistent Volume Claim for PostgreSQL

Create `postgres-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: flask-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply the PVC:

```bash
kubectl apply -f postgres-pvc.yaml
```

## Phase 5 — Deploy PostgreSQL

Create `postgres-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: flask-app
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
          image: postgres:16
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-secret
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-storage
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
```

Apply the PostgreSQL deployment:

```bash
kubectl apply -f postgres-deployment.yaml
```

## Phase 6 — Expose PostgreSQL with a ClusterIP Service

Create `postgres-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: flask-app
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
  type: ClusterIP
```

Apply the service:

```bash
kubectl apply -f postgres-service.yaml
```

## Phase 7 — Build and Load the Flask Docker Image into KIND

From your project directory containing `Dockerfile`, build the image:

```bash
docker build -t flask-postgres-app:v1 .
```

Load the image into the KIND cluster:

```bash
kind load docker-image flask-postgres-app:v1 --name monitoring-cluster
```

> Note: replace `monitoring-cluster` with your cluster name if different.

## Phase 8 — Deploy the Flask Application

Create `flask-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  namespace: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask-app
          image: flask-postgres-app:v1
          imagePullPolicy: Never
          ports:
            - containerPort: 5000
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_DB
            - name: POSTGRES_HOST
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: POSTGRES_HOST
```

Apply the Flask deployment:

```bash
kubectl apply -f flask-deployment.yaml
```

## Phase 9 — Expose the Flask Application with NodePort

Create `flask-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
  namespace: flask-app
spec:
  selector:
    app: flask-app
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 30010
      protocol: TCP
  type: NodePort
```

Apply the service:

```bash
kubectl apply -f flask-service.yaml
```

## Phase 10 — Verify the Deployment

Check pods and services:

```bash
kubectl get pods -n flask-app
kubectl get svc -n flask-app
```

Confirm the application is reachable on the NodePort:

```text
http://<NODE_IP>:30010
```

If using KIND on the local host, `<NODE_IP>` is usually `localhost`.

## Phase 11 — Clean Up

Delete the namespace and all resources:

```bash
kubectl delete namespace flask-app
```

Remove unused Docker images and stopped containers:

```bash
docker image prune -a
docker ps -a
```

## Kubernetes Concepts Covered

| Concept      | Purpose                            |
|--------------|------------------------------------|
| Namespace    | Resource isolation                 |
| Secret       | Secure credential storage          |
| ConfigMap    | Non-sensitive configuration        |
| PVC          | Persistent storage for PostgreSQL  |
| Deployment   | Manage replicated workloads        |
| Service      | Networking and service discovery   |
| NodePort     | External access to cluster service |
| KIND image   | Local Kubernetes image management  |

## Phase 12 — Create a Helm Chart for the Same Stack

This section shows how to package the Kubernetes stack as a Helm chart.

### 1) Create the Helm chart skeleton

```bash
cd ~
helm create flask-app-chart
```

### 2) Remove default chart templates

```bash
cd flask-app-chart/templates
rm -f *
```

### 3) Create Chart metadata

Edit `Chart.yaml` and replace the contents with:

```yaml
apiVersion: v2
name: flask-app-chart
description: A Helm chart for Flask + PostgreSQL deployment on Kubernetes
type: application
version: 0.1.0
appVersion: "1.0"
```

### 4) Create `values.yaml`

Replace the default `values.yaml` with the following:

```yaml
image:
  repository: flask-postgres-app
  tag: v1
  pullPolicy: IfNotPresent

postgres:
  image: postgres:16
  storage:
    size: 1Gi

service:
  flask:
    nodePort: 30010
```

### 5) Add Helm templates

Create the template files under `flask-app-chart/templates`:

- `postgres-secret.yaml`
- `postgres-configmap.yaml`
- `postgres-pvc.yaml`
- `postgres-deployment.yaml`
- `postgres-service.yaml`
- `flask-deployment.yaml`
- `flask-service.yaml`

### 6) Example Helm template: `postgres-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: {{ .Release.Namespace }}
type: Opaque
stringData:
  POSTGRES_USER: flaskuser
  POSTGRES_PASSWORD: flaskpass
  POSTGRES_DB: flaskdb
```

### 7) Example Helm template: `postgres-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: {{ .Release.Namespace }}
data:
  POSTGRES_HOST: postgres-service
```

### 8) Example Helm template: `postgres-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: {{ .Release.Namespace }}
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.postgres.storage.size }}
```

### 9) Example Helm template: `postgres-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: {{ .Release.Namespace }}
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
          image: {{ .Values.postgres.image }}
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-secret
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-storage
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
```

### 10) Example Helm template: `postgres-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
  type: ClusterIP
```

### 11) Example Helm template: `flask-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  namespace: {{ .Release.Namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask-app
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 5000
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_DB
            - name: POSTGRES_HOST
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: POSTGRES_HOST
```

### 12) Example Helm template: `flask-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: flask-app
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: {{ .Values.service.flask.nodePort }}
      protocol: TCP
  type: NodePort
```

### 13) Install the Helm chart

```bash
helm install flask-app-chart ./flask-app-chart -n flask-app
```

### 14) Verify Helm release

```bash
helm list -n flask-app
kubectl get all -n flask-app
```

## Conclusion

This document rewrites the full Kubernetes deployment process for the Flask application and includes an optional Helm chart packaging path. The guide now contains:

- Namespace creation and resource isolation
- Secret and ConfigMap configuration
- Persistent volume claim for PostgreSQL storage
- PostgreSQL deployment and service
- Flask deployment and external access via NodePort
- Docker image loading into KIND
- Helm chart packaging and deployment concepts

Use this document as the canonical Kubernetes deployment guide for the Flask/PostgreSQL project.
