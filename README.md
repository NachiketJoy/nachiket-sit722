# DevOps Deployment Guide

This document provides step-by-step instructions for deploying the application infrastructure and services on **Azure Kubernetes Service (AKS)** using **OpenTofu**. It also covers building and pushing images to **Azure Container Registry (ACR)**, applying **Kubernetes manifests**, and setting up **monitoring with Prometheus and Grafana**.

---

## Project Structure

```
.
├── workflows/          # GitHub Actions or CI/CD workflows
├── backend/            # Backend microservices (product, order, customer)
├── frontend/           # Frontend application
├── k8s/                # Kubernetes manifests
│   ├── configmaps.yaml
│   ├── secrets.yaml
│   ├── product-db.yaml
│   ├── order-db.yaml
│   ├── customer-db.yaml
│   ├── rabbitmq.yaml
│   ├── product-service.yaml
│   ├── order-service.yaml
│   ├── customer-service.yaml
│   └── frontend.yaml
└── opentofu/           # OpenTofu IaC configuration files
```

---

## 1. Authenticate with Azure
```bash
az login
```

---

## 2. Provision Infrastructure with OpenTofu
Navigate to the `opentofu/` directory:
```bash
cd opentofu/
```

Initialize and validate:
```bash
tofu init
tofu validate
```

Create and apply execution plan:
```bash
tofu plan -out=tfplan
tofu apply "tfplan"
```

This will provision the following resources in Azure:
- Resource Group
- AKS Cluster
- Azure Container Registry (ACR)
- Storage Account

---

## 3. Build and Push Docker Images
Login to your ACR:
```bash
az acr login --name sit722w10acr
```

Build images:
```bash
docker build -t sit722w10acr.azurecr.io/frontend:latest ./frontend
docker build -t sit722w10acr.azurecr.io/product_service:latest ./backend/product_service
docker build -t sit722w10acr.azurecr.io/order_service:latest ./backend/order_service
docker build -t sit722w10acr.azurecr.io/customer_service:latest ./backend/customer_service
```

Push images:
```bash
docker push sit722w10acr.azurecr.io/frontend:latest
docker push sit722w10acr.azurecr.io/product_service:latest
docker push sit722w10acr.azurecr.io/order_service:latest
docker push sit722w10acr.azurecr.io/customer_service:latest
```

---

## 4. Update Kubernetes Manifests
Update the image references in your Kubernetes YAML files (`k8s/`) to point to ACR images:

- **frontend.yaml**
  ```yaml
  image: sit722w10acr.azurecr.io/frontend:latest
  ```
- **product-service.yaml**
  ```yaml
  image: sit722w10acr.azurecr.io/product_service:latest
  ```
- **order-service.yaml**
  ```yaml
  image: sit722w10acr.azurecr.io/order_service:latest
  ```
- **customer-service.yaml**
  ```yaml
  image: sit722w10acr.azurecr.io/customer_service:latest
  ```

---

## 5. Deploy Kubernetes Resources
Apply manifests in order:

```bash
kubectl apply -f k8s/configmaps.yaml
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/product-db.yaml
kubectl apply -f k8s/order-db.yaml
kubectl apply -f k8s/customer-db.yaml
kubectl apply -f k8s/rabbitmq.yaml
kubectl apply -f k8s/product-service.yaml
kubectl apply -f k8s/order-service.yaml
kubectl apply -f k8s/customer-service.yaml
kubectl apply -f k8s/frontend.yaml
```

Or apply all at once:
```bash
kubectl apply -f k8s/
```

Verify deployments:
```bash
kubectl get pods -w
kubectl get svc frontend-w10
```

---

## 6. Monitoring Setup (Prometheus & Grafana)

### Add Helm Repositories
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Create Namespace
```bash
kubectl create namespace monitoring
```

### Install kube-prometheus-stack
```bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring
```

Verify pods:
```bash
kubectl get pods -n monitoring
```

### Access Grafana
Port-forward Grafana service:
```bash
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

Then open [http://localhost:3000](http://localhost:3000) in your browser.  
Default credentials:  
- **Username:** `admin`  
- **Password:** `admin`

---

## 📊 Architecture Overview

<img width="441" height="520" alt="image" src="https://github.com/user-attachments/assets/6e9548a7-ddd9-415d-ada8-384f1ceac834" />
```
