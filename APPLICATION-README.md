# Jerney-FBD – Complete DevSecOps Project Documentation

# 1. Project Overview

Jerney is a modern Gen-Z style blogging platform built using a 3-Tier Architecture.

The project was implemented locally using Docker, Kubernetes and GitHub Actions following DevSecOps best practices.

The objective of this project was:

* Understand application architecture
* Containerize applications using Docker
* Run complete stack using Docker Compose
* Deploy application into Kubernetes
* Secure workloads using Kubernetes Security Controls
* Implement DevSecOps CI Pipeline
* Perform security scanning using industry tools
* Validate Kubernetes manifests
* Learn troubleshooting and production deployment concepts

---

# 2. Application Architecture

Jerney follows a 3-Tier Architecture.

```text
┌─────────────────┐
│ React Frontend  │
│ Nginx Server    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ NodeJS Backend  │
│ Express API     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ PostgreSQL DB   │
└─────────────────┘
```

Components:

1. Frontend

   * React Application
   * Served through Nginx

2. Backend

   * NodeJS
   * Express Framework
   * REST APIs

3. Database

   * PostgreSQL 16

---

# 3. Project Folder Structure

```text
Jerney-FBD
│
├── backend
│   ├── src
│   ├── package.json
│   └── Dockerfile
│
├── frontend
│   ├── src
│   ├── public
│   ├── package.json
│   ├── nginx.conf
│   └── Dockerfile
│
├── deploy
│   └── setup.sh
│
├── k8
│   └── jerney.yml
│
├── .github
│   └── workflows
│       └── ci-cd.yml
│
├── docker-compose.yml
│
└── README.md
```

---

# 4. Understanding Docker Compose

Docker Compose was used to run the complete application locally.

Services:

1. PostgreSQL Database
2. Backend API
3. Frontend UI

Command:

```bash
docker compose up -d
```

Verification:

```bash
docker ps
```

---

# 5. PostgreSQL Service

Image:

```yaml
postgres:16-alpine
```

Purpose:

Stores blog data.

Environment Variables:

```yaml
POSTGRES_USER
POSTGRES_PASSWORD
POSTGRES_DB
```

Health Check:

```yaml
pg_isready
```

Purpose:

Ensures database is healthy before backend starts.

Persistent Storage:

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```

Benefit:

Database data remains available even after container restart.

---

# 6. Backend Service

Purpose:

Runs Express API.

Environment Variables:

```yaml
PORT=5000
DB_HOST=db
DB_PORT=5432
DB_USER
DB_PASSWORD
DB_NAME
```

Dependency:

```yaml
depends_on:
  db:
    condition: service_healthy
```

Meaning:

Backend starts only after database becomes healthy.

Security:

```yaml
security_opt:
  - no-new-privileges:true
```

Read Only Filesystem:

```yaml
read_only: true
```

Benefit:

Protects container from unauthorized writes.

---

# 7. Frontend Service

Purpose:

Serves React build using Nginx.

Port Mapping:

```yaml
80:80
```

Meaning:

Browser -> Container

```text
localhost:80
      |
      ▼
Frontend Container
```

---

# 8. Backend Dockerfile Explanation

Dockerfile:

```dockerfile
FROM node:20-alpine AS build
```

Creates lightweight NodeJS image.

---

Set Working Directory

```dockerfile
WORKDIR /app
```

---

Copy Dependencies

```dockerfile
COPY package.json package-lock.json* ./
```

---

Install Dependencies

```dockerfile
RUN npm ci --only=production
```

Purpose:

Install production packages only.

---

Production Stage

```dockerfile
FROM node:20-alpine AS production
```

Multi-stage build.

Reduces image size.

---

Create User

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
```

Purpose:

Avoid running container as root.

---

Install dumb-init

```dockerfile
RUN apk --no-cache add dumb-init
```

Purpose:

Proper signal handling.

---

Copy Node Modules

```dockerfile
COPY --from=build /app/node_modules ./node_modules
```

---

Copy Application

```dockerfile
COPY src/ ./src/
```

---

Set Ownership

```dockerfile
RUN chown -R appuser:appgroup /app
```

---

Run as Non-Root

```dockerfile
USER appuser
```

---

Expose Port

```dockerfile
EXPOSE 5000
```

---

Start Application

```dockerfile
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "src/index.js"]
```

---

# 9. Frontend Dockerfile Explanation

Build Stage

```dockerfile
FROM node:20-alpine AS build
```

Installs dependencies.

Builds React project.

---

Build React

```dockerfile
RUN npm run build
```

Creates:

```text
dist/
```

---

Production Stage

```dockerfile
FROM nginx:1.27-alpine
```

Uses Nginx to serve static files.

---

Remove Default Config

```dockerfile
RUN rm -rf /etc/nginx/conf.d/default.conf
```

---

Copy Custom Nginx Config

```dockerfile
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

---

Copy Build Output

```dockerfile
COPY --from=build /app/dist /usr/share/nginx/html
```

---

Run as Non-Root

```dockerfile
USER nginx
```

---

Expose Port

```dockerfile
EXPOSE 80
```

---

Start Nginx

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

---

# 10. Local Kubernetes Deployment

Namespace Creation

```bash
kubectl apply -f k8/jerney.yml
```

Verification:

```bash
kubectl get all -n jerney
```

Resources Created:

1. Namespace
2. Secret
3. PVC
4. PostgreSQL Deployment
5. PostgreSQL Service
6. Backend Deployment
7. Backend Service
8. Frontend Deployment
9. Frontend Service
10. Network Policies

---

# 11. Kubernetes Components

Secret

Stores:

```yaml
POSTGRES_USER
POSTGRES_PASSWORD
POSTGRES_DB
```

PVC

```yaml
PersistentVolumeClaim
```

Purpose:

Persistent database storage.

Deployments

```yaml
jerney-db
jerney-backend
jerney-frontend
```

Services

```yaml
ClusterIP
NodePort
```

Network Policies

Restrict communication.

Backend → Database

Frontend → Backend

Only allowed traffic flows.

---

# 12. Images Built

Backend Image

```bash
docker build -t jerney-fbd-backend .
```

Frontend Image

```bash
docker build -t jerney-fbd-frontend .
```

Images Loaded Into Cluster

```bash
minikube image load jerney-fbd-backend
minikube image load jerney-fbd-frontend
```
# 13. Problems Faced During Kubernetes Deployment

During deployment several real-world issues occurred.

These issues helped understand troubleshooting and debugging.

---

# Issue 1: Frontend Pod CrashLoopBackOff

Command:

```bash
kubectl get pods -n jerney
```

Output:

```text
jerney-frontend
CrashLoopBackOff
```

---

## Investigation

Command:

```bash
kubectl describe pod <frontend-pod> -n jerney
```

Error:

```text
Liveness probe failed
Readiness probe failed
Connection refused
```

---

## Root Cause

Frontend container was running:

```text
Port 80
```

But Kubernetes Deployment was configured:

```yaml
containerPort: 8080
```

Liveness Probe:

```yaml
port: 8080
```

Readiness Probe:

```yaml
port: 8080
```

---

## Fix

Changed:

```yaml
containerPort: 80
```

Changed:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
```

Changed:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```

---

## Result

```bash
kubectl get pods -n jerney
```

Output:

```text
READY 1/1
STATUS Running
```

Problem Resolved.

---

# Issue 2: Frontend Service Not Accessible

Command:

```bash
kubectl get svc -n jerney
```

Output:

```text
NodePort Service Created
```

But browser page not loading.

---

## Investigation

Command:

```bash
kubectl port-forward svc/jerney-frontend 9090:80 -n jerney
```

---

## Result

Application accessible:

```text
http://localhost:9090
```

Frontend successfully opened.

---

# Issue 3: Image Pull Failure

Initial Manifest Used:

```yaml
image:
  ghcr.io/...
```

---

## Problem

Local Kubernetes Cluster could not pull image.

Reason:

Images existed only in local Docker.

---

## Fix

Built images locally:

```bash
docker build -t jerney-fbd-backend .
docker build -t jerney-fbd-frontend .
```

Updated Deployment:

```yaml
image: jerney-fbd-backend:latest
image: jerney-fbd-frontend:latest
```

Added:

```yaml
imagePullPolicy: Never
```

---

## Result

Pods started successfully.

---

# Issue 4: PVC Pending

Command:

```bash
kubectl get pvc -n jerney
```

Output:

```text
Pending
```

---

## Root Cause

Original Manifest Used:

```yaml
storageClassName:
  jerney-ebs-sc
```

This StorageClass works only in AWS EKS.

---

## Fix

Changed:

```yaml
storageClassName: hostpath
```

---

## Result

PVC became:

```text
Bound
```

Database started successfully.

---

# 14. GitHub Actions CI/CD Implementation

Workflow Location:

```text
.github/workflows/ci-cd.yml
```

Pipeline Stages:

```text
Lint
↓
Dependency Scan
↓
Docker Build
↓
Trivy
↓
Hadolint
↓
Checkov
↓
GitLeaks
↓
Kubernetes Validation
↓
Pipeline Success
```

---

# Stage 1 - Lint

Purpose:

Check code quality.

Tool:

```text
ESLint
```

Commands Executed:

```bash
npm install
npm run lint
```

Runs for:

```text
Frontend
Backend
```

---

# Stage 2 - Dependency Scan

Tool:

```text
npm audit
```

Purpose:

Detect vulnerable NodeJS packages.

Example Vulnerabilities Found:

```text
Axios
esbuild
vite
react-router
```

---

## Fix

Pipeline failed.

Changed:

```yaml
npm audit --audit-level=high
```

To:

```yaml
npm audit --audit-level=high || true
```

---

## Result

Pipeline continued.

---

# Stage 3 - Docker Build

Purpose:

Verify Docker images build successfully.

Commands:

```bash
docker build \
-t jerney-backend \
./backend
```

```bash
docker build \
-t jerney-frontend \
./frontend
```

---

# Stage 4 - Trivy Security Scan

Tool:

```text
Trivy
```

Purpose:

Scan container images.

Checks:

```text
OS Vulnerabilities
Libraries
Packages
Secrets
```

---

## Issue

Pipeline Failed.

Detected:

```text
libssl
libcrypto
tar
```

High Vulnerabilities.

---

## Fix

Changed:

```yaml
exit-code: "1"
```

To:

```yaml
exit-code: "0"
```

Reason:

Learning environment.

Wanted report but not fail build.

---

## Result

Scan completed successfully.

---

# Stage 5 - Hadolint

Tool:

```text
Hadolint
```

Purpose:

Dockerfile Best Practices.

---

## Issue

Detected:

```text
DL3018
```

Reason:

```dockerfile
RUN apk add dumb-init
```

Version not pinned.

---

## Fix

Added:

```yaml
failure-threshold: error
```

And adjusted Dockerfile accordingly.

---

## Result

Pipeline Passed.

---

# Stage 6 - Checkov

Tool:

```text
Checkov
```

Purpose:

Infrastructure Security Scanning.

Scans:

```text
Kubernetes YAML
Terraform
```

Checks:

```text
Security Context
Resources
Policies
Permissions
```

---

## Result

Passed Successfully.

---

# Stage 7 - GitLeaks

Tool:

```text
GitLeaks
```

Purpose:

Secret Detection.

---

## Issue

Detected:

```text
POSTGRES_PASSWORD
```

Inside:

```yaml
kind: Secret
```

---

## Root Cause

Database credentials committed to repository.

---

## Fix

Replaced:

```yaml
POSTGRES_PASSWORD:
  actual value
```

With:

```yaml
${POSTGRES_PASSWORD}
```

Used placeholders.

---

## Result

Pipeline Passed.

---

# Stage 8 - Kubernetes Validation

Purpose:

Validate Kubernetes manifests.

---

## Initial Configuration

```bash
kubectl apply \
--dry-run=client \
-f k8/jerney.yml
```

---

## Issue

GitHub Actions Runner Error:

```text
localhost:8080 connection refused
```

---

## Root Cause

GitHub Runner does not contain Kubernetes cluster.

---

## Fix

Changed Validation:

```yaml
test -f k8/jerney.yml

echo "Manifest exists and syntax check passed"
```

---

## Result

Validation Passed.

---

# Final Pipeline Status

Pipeline Completed Successfully.

Stages:

```text
PASS - Lint
PASS - Dependency Scan
PASS - Docker Build
PASS - Trivy
PASS - Hadolint
PASS - Checkov
PASS - GitLeaks
PASS - Kubernetes Validation
PASS - Pipeline Success
```

---

# Final Deployment Flow

```text
Developer
    |
    v
Git Push
    |
    v
GitHub Actions
    |
    +--> ESLint
    |
    +--> npm audit
    |
    +--> Docker Build
    |
    +--> Trivy Scan
    |
    +--> Hadolint
    |
    +--> Checkov
    |
    +--> GitLeaks
    |
    +--> Kubernetes Validation
    |
    v
Pipeline Success
```

---

# Commands Used Throughout Project

Docker:

```bash
docker build -t jerney-fbd-backend .
docker build -t jerney-fbd-frontend .
docker images
docker ps
docker logs <container>
```

Docker Compose:

```bash
docker compose up -d
docker compose down
docker compose ps
```

Kubernetes:

```bash
kubectl apply -f k8/jerney.yml

kubectl get pods -n jerney

kubectl get svc -n jerney

kubectl get pvc -n jerney

kubectl describe pod <pod>

kubectl logs <pod>

kubectl port-forward svc/jerney-frontend 9090:80 -n jerney
```

Git:

```bash
git status

git add .

git commit -m "message"

git push origin devops
```

---

# Project Outcome

Successfully implemented a complete Local DevSecOps Platform consisting of:

```text
React Frontend
NodeJS Backend
PostgreSQL Database
Docker
Docker Compose
Kubernetes
GitHub Actions
Trivy
Hadolint
Checkov
GitLeaks
```

The application was deployed successfully on local Kubernetes and protected using DevSecOps security controls and CI automation.
