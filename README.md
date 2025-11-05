# Django Notes App - Production-Grade Kubernetes Deployment with Kind

A comprehensive hands-on project demonstrating Kubernetes fundamentals through a real-world Django Notes application deployed on Kind (Kubernetes in Docker), featuring persistent storage, ingress routing, automated jobs, and advanced troubleshooting patterns.

## 🎯 Project Overview

This project showcases a complete Kubernetes learning journey—from basic deployments to advanced orchestration concepts. The Django Notes app serves as a practical vehicle for understanding production-grade Kubernetes patterns including service communication, persistent data management, ingress configuration, and automated job scheduling.

**Key Achievement**: Successfully deployed a multi-tier application on Kubernetes with proper networking, storage, and traffic management, complete with real-world troubleshooting experience resolving 503 Service Unavailable errors.

## ✨ Key Features

- **Kind-Based Kubernetes Cluster**: Local production-like Kubernetes environment for learning without cloud costs
- **Multi-Container Orchestration**: Coordinated Django app and NGINX reverse proxy deployments
- **Persistent Data Storage**: PersistentVolumes and PersistentVolumeClaims for reliable data across pod restarts
- **Ingress Controller**: NGINX Ingress for intelligent HTTP routing and load balancing
- **Automated Tasks**: Kubernetes Jobs and CronJobs for background processing and maintenance
- **Service Networking**: Internal DNS-based service discovery and communication
- **Advanced Troubleshooting**: Real-world debugging of 503 errors and Ingress misconfiguration
- **Port Forwarding**: Exposed services for local testing and validation with custom ports

## 🛠️ Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Container Orchestration** | Kubernetes (Kind) | Local Kubernetes cluster simulation |
| **Application** | Django 3.x | Python web framework for notes |
| **Database** | MongoDB | NoSQL document storage |
| **Reverse Proxy** | NGINX | Load balancing and HTTP routing |
| **Container Runtime** | Docker | Application containerization |
| **Configuration** | YAML | Declarative infrastructure management |
| **CLI** | kubectl | Kubernetes command-line interface |

## 📋 Prerequisites

Install the following tools before starting:

| Tool | Version | Installation |
|------|---------|--------------|
| **Docker** | 20.0+ | [Download](https://www.docker.com/products/docker-desktop) |
| **Kind** | 0.17+ | [Install Guide](https://kind.sigs.k8s.io/docs/user/quick-start/) |
| **kubectl** | 1.27+ | [Download](https://kubernetes.io/docs/tasks/tools/) |
| **Git** | 2.0+ | [Download](https://git-scm.com/) |

### Quick Installation Script (Ubuntu/macOS)

#!/bin/bash

Install all prerequisites
Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

Verify installations
kind version
kubectl version --client

## 🚀 Quick Start Guide

### Step 1: Clone Repository

git clone https://github.com/Krishna7031/django-notes-app.git
cd django-notes-app

### Step 2: Create Kind Cluster

Create a local Kubernetes cluster
kind create cluster --name notes-cluster

Verify cluster is running
kubectl cluster-info --context kind-notes-cluster

Check available nodes
kubectl get nodes

### Step 3: Build and Load Docker Image

Build Django Notes app Docker image
docker build -t django-notes-app:latest .

Load image into Kind cluster (makes it available to Kubernetes)
kind load docker-image django-notes-app:latest --name notes-cluster

Verify image is loaded
docker exec notes-cluster-control-plane crictl images | grep django-notes-app

### Step 4: Create Kubernetes Namespace

Create dedicated namespace for the application
kubectl create namespace notes-app

Set as default namespace for easier commands
kubectl config set-context --current --namespace=notes-app

### Step 5: Deploy MongoDB with Persistent Storage

Create PersistentVolume and PersistentVolumeClaim for data persistence
kubectl apply -f k8s/mongodb-pvc.yaml

Deploy MongoDB
kubectl apply -f k8s/mongodb-deployment.yaml

Wait for MongoDB to be ready
kubectl wait --for=condition=ready pod -l app=mongodb --timeout=300s

Verify MongoDB is running
kubectl get pods -n notes-app
kubectl get pvc -n notes-app

### Step 6: Deploy Django Application

Create Deployment and Service for Django notes app
kubectl apply -f k8s/django-deployment.yaml
kubectl apply -f k8s/django-service.yaml

Wait for Django pods to be ready
kubectl wait --for=condition=ready pod -l app=django-notes-app --timeout=300s

Check deployment status
kubectl get deployments -n notes-app
kubectl get pods -n notes-app

### Step 7: Deploy NGINX Reverse Proxy

Deploy NGINX as reverse proxy service
kubectl apply -f k8s/nginx-deployment.yaml
kubectl apply -f k8s/nginx-service.yaml

Verify NGINX pods are running
kubectl get pods -l app=nginx -n notes-app

### Step 8: Install and Configure NGINX Ingress Controller

Install NGINX Ingress Controller for Kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/kind/deploy.yaml

Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx
--for=condition=ready pod
--selector=app.kubernetes.io/component=controller
--timeout=90s

Apply Ingress rules
kubectl apply -f k8s/ingress.yaml

Verify Ingress is created
kubectl get ingress -n notes-app
kubectl describe ingress notes-ingress -n notes-app

### Step 9: Access the Application

Port-forward NGINX service to localhost
kubectl port-forward service/nginx-service 8081:80 -n notes-app

Open browser: http://localhost:8081

**Alternative: Using Ingress directly**

Port-forward ingress controller
kubectl port-forward service/ingress-nginx-controller -n ingress-nginx 8080:80 --address=0.0.0.0

Access: http://localhost:8080

### Step 10: Deploy Automated Background Tasks

Create database migration job
kubectl apply -f k8s/job-migration.yaml

Create daily cleanup CronJob
kubectl apply -f k8s/cronjob-cleanup.yaml

Monitor jobs
kubectl get jobs -n notes-app
kubectl get cronjobs -n notes-app

View job logs
kubectl logs job/notes-migration -n notes-app

## 🐛 Troubleshooting: Resolving 503 Service Unavailable

### Problem Description

Application returned **503 Service Unavailable** when accessed through the Ingress.

curl http://localhost:8080

Returns: 503 Service Unavailable

### Root Cause Analysis

Check Ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller | grep -i 503

Check backend service endpoints
kubectl get endpoints django-service -n notes-app
kubectl describe service django-service -n notes-app

Test if Django pods are responding
kubectl exec -it <django-pod> -n notes-app -- curl localhost:8000

**Root Cause Found**: Ingress path configuration was too strict and annotations were incorrect, causing traffic routing to fail.

### Solution Implemented

**Before (Incorrect Configuration):**
paths:

path: /api
pathType: Exact # Too strict, only matches /api exactly
backend:
service:
name: django-service

**After (Correct Configuration):**
annotations:
nginx.ingress.kubernetes.io/rewrite-target: / # Added rewrite rule

paths:

path: /
pathType: Prefix # Matches all paths under /
backend:
service:
name: django-service
port:
number: 8000

### Verification

After applying fix, test connectivity
kubectl apply -f k8s/ingress.yaml

Wait for Ingress to update
sleep 10

Test with curl
curl http://localhost:8080

Monitor real-time logs
kubectl logs -f -n ingress-nginx deployment/ingress-nginx-controller

### Key Lessons from Debugging

✅ Always check **Ingress controller logs** first when 503 occurs  
✅ Verify **service endpoints** are available: `kubectl get endpoints`  
✅ Use **pathType: Prefix** for catch-all routing  
✅ Include **rewrite-target annotation** for URL rewriting  
✅ Test backend service **directly** before testing through Ingress  

## 📊 Essential kubectl Commands

Cluster Management
kubectl cluster-info
kubectl get nodes
kubectl get namespace
kubectl config get-contexts

Deployments & Pods
kubectl get deployments -n notes-app
kubectl get pods -n notes-app
kubectl describe pod <pod-name> -n notes-app
kubectl logs <pod-name> -n notes-app
kubectl logs -f <pod-name> -n notes-app # Follow logs

Services & Networking
kubectl get svc -n notes-app
kubectl get endpoints -n notes-app
kubectl describe service django-service -n notes-app

Ingress & Load Balancing
kubectl get ingress -n notes-app
kubectl describe ingress notes-ingress -n notes-app

Storage
kubectl get pvc -n notes-app
kubectl get pv

Jobs & CronJobs
kubectl get jobs -n notes-app
kubectl get cronjobs -n notes-app
kubectl describe job <job-name> -n notes-app

Port Forwarding
kubectl port-forward pod/<pod-name> 8080:8000 -n notes-app
kubectl port-forward svc/<service-name> 8080:80 -n notes-app

Debugging
kubectl exec -it <pod-name> -n notes-app -- /bin/bash
kubectl events -n notes-app --watch
kubectl top pods -n notes-app

Cleanup
kubectl delete namespace notes-app
kind delete cluster --name notes-cluster

## 🎓 Key Learnings

Through this project, I mastered:

✅ **Kubernetes Architecture**: Pods, Deployments, Services, and control plane  
✅ **Declarative Configuration**: Writing YAML manifests for infrastructure  
✅ **Persistent Storage**: PersistentVolumes and PersistentVolumeClaims for data persistence  
✅ **Service Networking**: ClusterIP, DNS service discovery, internal communication  
✅ **Ingress Routing**: HTTP routing, path rewriting, traffic management  
✅ **Container Orchestration**: Replica sets, rolling updates, self-healing  
✅ **Automated Tasks**: Jobs and CronJobs for scheduled operations  
✅ **Port Forwarding**: Exposing internal services for external access  
✅ **Debugging Techniques**: Systematic troubleshooting of Kubernetes issues  
✅ **Production Patterns**: Resource requests/limits, health checks, liveness probes  

## 📈 Advanced Concepts Demonstrated

- **Rolling Updates**: Zero-downtime deployments with replica management
- **Resource Quotas**: CPU and memory requests/limits for efficient resource usage
- **Health Checks**: Liveness and readiness probes for pod reliability
- **Service Discovery**: DNS-based internal service communication
- **Network Isolation**: Namespace segregation for security
- **Persistent Storage**: Stateful data management in containerized environments
- **Automated Scaling**: Self-healing and replica management

## 🚀 Next Steps & Production Considerations

To scale this project to production:

Multi-node Kubernetes cluster
Use AWS EKS, GCP GKE, or Azure AKS

Deploy across multiple availability zones

Advanced Storage
Use managed storage services (EBS, PD, Azure Disk)

Implement backup and disaster recovery

Security Hardening
Network policies for traffic control

RBAC (Role-Based Access Control)

Pod security policies

Secrets management (HashiCorp Vault)

Monitoring & Observability
Prometheus for metrics collection

Grafana for visualization

ELK/Loki for centralized logging

Jaeger for distributed tracing

CI/CD Integration
GitHub Actions for automated builds

GitOps with ArgoCD for deployments

Automated testing in pipeline

High Availability
Multiple replicas across nodes

Pod Disruption Budgets

Horizontal Pod Autoscaling

Resource quotas and limits


## 📄 License

Apache 2.0 - See [LICENSE](./LICENSE) for details



**Project Duration**: April - May 2025  
**Repository**: [GitHub](https://github.com/Krishna7031/django-notes-app)  
**Skills Demonstrated**: Kubernetes · Container Orchestration · Django · MongoDB · YAML · kubectl · Linux · Distributed Systems · Troubleshooting · Networking
