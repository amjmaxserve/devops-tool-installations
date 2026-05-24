# DevOps Tools Stack

This repository contains multiple observability and application deployment workflows built for Ubuntu, Docker, Docker Compose, KIND, and Kubernetes.

## Contents

- `flask-app-stack/`
  - `flask-app-create.MD` — local Flask + PostgreSQL development, Dockerfile, and Docker Compose setup
  - `flask-app-phase3.MD` — Kubernetes deployment with KIND, secret management, and Sealed Secrets workflow
  - `flask-app-kubernetes.md` — full Kubernetes deployment guide including Helm packaging
- `observatory-stack/`
  - `observatory-dockercompose.md` — observability stack deployment using Docker Compose
  - `observatory-kubernetes.md` — Kubernetes observability stack deployment using KIND and Helm
  - `Install-Observatory.md` — Prometheus and Node Exporter installation guide for a non-container environment

## Project Overview

This repository is designed to support two main use cases:

1. **Flask application delivery**
   - Build a Flask web application backed by PostgreSQL
   - Containerize with Docker
   - Deploy locally with Docker Compose
   - Deploy to Kubernetes using KIND
   - Secure secrets using Sealed Secrets
   - Package the stack with Helm

2. **Observability stack deployment**
   - Run Prometheus, Grafana, Alertmanager, Node Exporter, and cAdvisor
   - Deploy with Docker Compose for a quick local observability stack
   - Deploy to Kubernetes using KIND and Helm for a production-like setup

## Getting Started

1. Choose the stack you want to work with:
   - Flask app development: `flask-app-stack/`
   - Observability stack: `observatory-stack/`
2. Read the appropriate document to follow the end-to-end workflow.
3. Install required tools before starting:
   - `docker`, `docker-compose`
   - `kubectl`, `kind`, `helm`
   - `python3`, `pip`, `python-venv` for Flask development

## Recommended Workflow

### Flask application stack

- Start with `flask-app-stack/flask-app-create.MD` to build and run the Flask app locally.
- Move to `flask-app-stack/flask-app-phase3.MD` for Kubernetes deployment in KIND and secure secrets with Sealed Secrets.
- Use `flask-app-stack/flask-app-kubernetes.md` for a complete Kubernetes deployment and Helm packaging reference.

### Observatory stack

- Use `observatory-stack/observatory-dockercompose.md` to deploy a monitoring stack quickly with Docker Compose.
- Use `observatory-stack/observatory-kubernetes.md` to deploy the monitoring stack to Kubernetes with KIND and Helm.
- Use `observatory-stack/Install-Observatory.md` for manual Prometheus + Node Exporter installation on a Linux host.

## Notes

- The project is intended for Ubuntu environments and local Kubernetes using KIND.
- Kubernetes manifests use `NodePort` services for easy browser access in local clusters.
- The Flask stack includes both local Docker Compose and Kubernetes deployment paths.
- The repository also demonstrates secure secret management with Sealed Secrets for GitOps-friendly workflows.

## Useful Commands

```bash
# Inspect all markdown guides
ls -R *.md flask-app-stack/*.MD observatory-stack/*.md

# Start the Flask Docker Compose stack
cd flask-app-stack
docker-compose up -d --build

# Create a local KIND cluster
kind create cluster --name flask-monitoring --config kind-config.yaml

# Apply Kubernetes manifests
kubectl apply -f k8s/your-manifest.yaml -n flask-app
```

## License

Use and adapt this repository for learning and experimentation. No explicit license is included in this repository.
