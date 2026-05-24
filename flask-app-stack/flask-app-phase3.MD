# Phase 3 — Deploy the Flask Application to Kubernetes with KIND

This document contains the Phase 3 deployment steps for the Flask + PostgreSQL application.

## 1) Install required tools

Install kubectl, KIND, and Helm if they are not already present:

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# KIND
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify the installed tools:

```bash
kubectl version --client
kind version
helm version
```

## 2) Create a KIND cluster

Create a local working folder and a KIND cluster config file:

```bash
mkdir -p ~/kind-flask-app
cd ~/kind-flask-app
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
YAML
```

Create the cluster:

```bash
kind create cluster --name flask-monitoring --config kind-config.yaml
```

Verify the cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

## 3) Create the Kubernetes namespace

```bash
kubectl create namespace flask-app
```

## 4) Create Kubernetes manifests

Create a `k8s` directory and add the cluster resources:

```bash
mkdir -p k8s
```

### 4.1 ConfigMap for database configuration

```bash
cat > k8s/flask-configmap.yaml <<'YAML'
apiVersion: v1
kind: ConfigMap
metadata:
  name: flask-config
  namespace: flask-app
data:
  POSTGRES_HOST: postgres-db
  POSTGRES_DB: flaskdb
YAML
```

### 4.2 Secret for database credentials

```bash
cat > k8s/flask-secret.yaml <<'YAML'
apiVersion: v1
kind: Secret
metadata:
  name: flask-secret
  namespace: flask-app
type: Opaque
data:
  POSTGRES_USER: $(echo -n "flaskuser" | base64)
  POSTGRES_PASSWORD: $(echo -n "flaskpass" | base64)
YAML
```

### 4.3 PostgreSQL Deployment and Service

```bash
cat > k8s/postgres-deployment.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-db
  namespace: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-db
  template:
    metadata:
      labels:
        app: postgres-db
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: flaskdb
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: flask-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: flask-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-data
      volumes:
        - name: postgres-data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: flask-app
spec:
  selector:
    app: postgres-db
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
  type: ClusterIP
YAML
```

### 4.4 Flask Deployment and Service

```bash
cat > k8s/flask-deployment.yaml <<'YAML'
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
          image: python:3.12-slim
          command: ["python", "app.py"]
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: flask-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: flask-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_HOST
              valueFrom:
                configMapKeyRef:
                  name: flask-config
                  key: POSTGRES_HOST
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: flask-config
                  key: POSTGRES_DB
          ports:
            - containerPort: 5000
          volumeMounts:
            - mountPath: /app
              name: app-code
      volumes:
        - name: app-code
          hostPath:
            path: /workspace/flask-app-stack
            type: Directory
---
apiVersion: v1
kind: Service
metadata:
  name: flask-app
  namespace: flask-app
spec:
  selector:
    app: flask-app
  ports:
    - port: 5000
      targetPort: 5000
      protocol: TCP
  type: NodePort
YAML
```

> Note: The `hostPath` mount is convenient for local KIND development only. Replace it with a proper container image in production.

## 5) Apply Kubernetes resources

```bash
kubectl apply -f k8s/flask-configmap.yaml
kubectl apply -f k8s/flask-secret.yaml
kubectl apply -f k8s/postgres-deployment.yaml
kubectl apply -f k8s/flask-deployment.yaml
```

## 6) Verify the deployment

```bash
kubectl get namespaces
kubectl get all -n flask-app
```

## 7) Access the Flask application

Find the assigned NodePort:

```bash
kubectl get svc -n flask-app flask-app
```

Then open the application in your browser at `http://localhost:<nodePort>`.

## 8) (Optional) Add monitoring support

If you want to extend Phase 3 with observability, use Helm to install `kube-prometheus-stack` in the same namespace.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack -n flask-app
```

## Notes

- This document contains the Phase 3 Kubernetes deployment process only.
- For local development, continue using the Docker Compose stack defined in `flask-app-create.MD`.
- Replace `hostPath` with image-based deployment in production.
-------------------------------------------------------------------------------------

## Sealed Secrets for Secure GitOps

Storing raw credentials in Helm values or plain YAML is not secure. Bitnami Sealed Secrets enables a GitOps-friendly workflow where secrets are encrypted before they enter Git.

### Why use Sealed Secrets?

- Kubernetes-native secret encryption
- Git-friendly encrypted manifests
- No plaintext secrets in version control
- Production-ready secret management

### Architecture

```text
Developer
   │
kubeseal
   │
Encrypted Secret YAML
   │
Git Repository
   │
Helm Chart / Manifests
   │
Kubernetes Cluster
   │
Sealed Secrets Controller
   │
Kubernetes Secret
   │
Flask Application
```

## Phase 13 — Install the Sealed Secrets Controller

Add the Bitnami Sealed Secrets Helm repo and install the controller:

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
kubectl create namespace sealed-secrets
helm install sealed-secrets-controller sealed-secrets/sealed-secrets -n sealed-secrets
```

Verify the controller:

```bash
kubectl get pods -n sealed-secrets
```

## Phase 14 — Install the kubeseal CLI

Download and install the CLI tool:

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.3/kubeseal-0.27.3-linux-amd64.tar.gz
tar -xvzf kubeseal-0.27.3-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

Verify installation:

```bash
kubeseal --version
```

## Phase 15 — Create and Seal the PostgreSQL Secret

Create a plain secret manifest locally named `postgres-secret.yaml`:

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

Seal the secret using the cluster controller public key:

```bash
kubeseal --format yaml < postgres-secret.yaml > postgres-sealedsecret.yaml
```

> Keep `postgres-secret.yaml` out of Git. Commit only `postgres-sealedsecret.yaml`.

## Phase 16 — Apply the Sealed Secret

```bash
kubectl apply -f postgres-sealedsecret.yaml
```

Verify the sealed secret and generated Kubernetes Secret:

```bash
kubectl get sealedsecrets -n flask-app
kubectl get secrets -n flask-app
```

## Phase 17 — Use Sealed Secrets in the Deployment

The Flask deployment continues to reference the same secret name `postgres-secret`.

Example deployment env section remains:

```yaml
- name: POSTGRES_USER
  valueFrom:
    secretKeyRef:
      name: postgres-secret
      key: POSTGRES_USER
```

## Phase 18 — Optional Cleanup of Plain Secret Templates

If you are using a Helm chart, remove any plain secret template that creates `postgres-secret` directly and keep only the sealed secret manifest.

```bash
rm -f templates/postgres-secret.yaml
```

## Phase 19 — Install or Upgrade the Helm Chart

From the chart directory:

```bash
cd ~/flask-app-chart
helm upgrade --install flask-app . -n flask-app
```

Verify the chart deployment:

```bash
kubectl get pods -n flask-app
kubectl get svc -n flask-app
```

## Summary

This section replaces the insecure secret workflow with Sealed Secrets. The encrypted manifest can be safely stored in Git, while the Kubernetes cluster decrypts it only after the sealed secrets controller validates it.
