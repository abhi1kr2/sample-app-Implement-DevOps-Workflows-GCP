# Implement DevOps Workflows in Google Cloud – Challenge Lab Notes

## Overview

This project demonstrates a complete CI/CD workflow using Google Cloud Platform services.

The lab covers:

* Google Kubernetes Engine (GKE)
* Cloud Build CI/CD pipelines
* Artifact Registry
* GitHub integration
* Kubernetes deployments
* Production and Development environments
* Application version upgrades
* Rollback deployment strategy

---

# Architecture

GitHub Repository  →  Cloud Build Trigger  →  Artifact Registry  →  GKE Deployment

Two environments were created:

* Development (dev namespace)
* Production (prod namespace)

---

# Google Cloud Services Used

| Service                        | Purpose                       |
| ------------------------------ | ----------------------------- |
| Google Kubernetes Engine (GKE) | Container orchestration       |
| Cloud Build                    | CI/CD automation              |
| Artifact Registry              | Store Docker container images |
| GitHub                         | Source code repository        |
| Kubernetes                     | Deployment and scaling        |
| LoadBalancer Service           | Expose application publicly   |

---

# Task 1 – Create Lab Resources

## Enable Required APIs

```bash
gcloud services enable container.googleapis.com \
cloudbuild.googleapis.com \
artifactregistry.googleapis.com
```

## Set Project ID

```bash
export PROJECT_ID=$(gcloud config get-value project)
```

## Add IAM Role

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:$(gcloud projects describe $PROJECT_ID \
--format="value(projectNumber)")@cloudbuild.gserviceaccount.com \
--role="roles/container.developer"
```

---

# Configure GitHub in Cloud Shell

```bash
curl -sS https://webi.sh/gh | sh
source ~/.bashrc

gh auth login
```

## Configure Git

```bash
export GITHUB_USERNAME=$(gh api user -q ".login")

git config --global user.name "$GITHUB_USERNAME"
git config --global user.email "your-email@example.com"
```

---

# Create Artifact Registry Repository

```bash
gcloud artifacts repositories create my-repository \
--repository-format=docker \
--location=us-central1 \
--description="Docker repository"
```

---

# Create GKE Cluster

```bash
gcloud container clusters create hello-cluster \
--zone=us-central1-a \
--release-channel=regular \
--num-nodes=3 \
--enable-autoscaling \
--min-nodes=2 \
--max-nodes=6
```

## Connect Cluster

```bash
gcloud container clusters get-credentials hello-cluster \
--zone=us-central1-a
```

---

# Create Namespaces

```bash
kubectl create namespace dev
kubectl create namespace prod
```

---

# Task 2 – Create GitHub Repository

## Clone Repository

```bash
git clone https://github.com/USERNAME/sample-app.git
cd sample-app
```

## Copy Sample Code

```bash
gsutil cp -r gs://spls/gsp330/sample-app/* sample-app
```

## Replace Region and Zone

```bash
export REGION="us-central1"
export ZONE="us-central1-a"

for file in sample-app/cloudbuild-dev.yaml sample-app/cloudbuild.yaml; do
    sed -i "s/<your-region>/${REGION}/g" "$file"
    sed -i "s/<your-zone>/${ZONE}/g" "$file"
done
```

---

# Push Code to GitHub

## Master Branch

```bash
git add .
git commit -m "Initial commit"
git branch -M master
git push -u origin master
```

## Dev Branch

```bash
git checkout -b dev

echo "dev branch" >> README.md

git add .
git commit -m "Create dev branch"
git push origin dev
```

---

# Task 3 – Create Cloud Build Triggers

## Production Trigger

| Field        | Value                  |
| ------------ | ---------------------- |
| Trigger Name | sample-app-prod-deploy |
| Branch       | ^master$               |
| Config File  | cloudbuild.yaml        |

## Development Trigger

| Field        | Value                 |
| ------------ | --------------------- |
| Trigger Name | sample-app-dev-deploy |
| Branch       | ^dev$                 |
| Config File  | cloudbuild-dev.yaml   |

---

# Task 4 – Deploy Version v1.0

## Update cloudbuild-dev.yaml

Replace:

```yaml
<version>
```

With:

```yaml
v1.0
```

---

# Development Deployment Image

```yaml
image: us-central1-docker.pkg.dev/PROJECT_ID/my-repository/hello-cloudbuild-dev:v1.0
```

## Push Changes

```bash
git add .
git commit -m "Deploy dev v1.0"
git push origin dev
```

---

# Expose Development Deployment

```bash
kubectl expose deployment development-deployment \
--name=dev-deployment-service \
--type=LoadBalancer \
--port=8080 \
--target-port=8080 \
-n dev
```

## Get External IP

```bash
kubectl get svc -n dev
```

---

# Production Deployment Image

```yaml
image: us-central1-docker.pkg.dev/PROJECT_ID/my-repository/hello-cloudbuild:v1.0
```

## Push Changes

```bash
git add .
git commit -m "Deploy prod v1.0"
git push origin master
```

---

# Expose Production Deployment

```bash
kubectl expose deployment production-deployment \
--name=prod-deployment-service \
--type=LoadBalancer \
--port=8080 \
--target-port=8080 \
-n prod
```

---

# Verify Application

Development:

```text
http://EXTERNAL-IP:8080/blue
```

Production:

```text
http://EXTERNAL-IP:8080/blue
```

---

# Task 5 – Deploy Version v2.0

## Update main.go

```go
func main() {
    http.HandleFunc("/blue", blueHandler)
    http.HandleFunc("/red", redHandler)
    http.ListenAndServe(":8080", nil)
}
```

## Add redHandler Function

```go
func redHandler(w http.ResponseWriter, r *http.Request) {
    img := image.NewRGBA(image.Rect(0, 0, 100, 100))
    draw.Draw(img, img.Bounds(), &image.Uniform{color.RGBA{255, 0, 0, 255}}, image.ZP, draw.Src)
    w.Header().Set("Content-Type", "image/png")
    png.Encode(w, img)
}
```

---

# Required Imports

```go
import (
    "image"
    "image/color"
    "image/draw"
    "image/png"
    "net/http"
)
```

---

# Update Image Version

Replace:

```yaml
v1.0
```

With:

```yaml
v2.0
```

---

# Development v2.0 Image

```yaml
image: us-central1-docker.pkg.dev/PROJECT_ID/my-repository/hello-cloudbuild-dev:v2.0
```

---

# Production v2.0 Image

```yaml
image: us-central1-docker.pkg.dev/PROJECT_ID/my-repository/hello-cloudbuild:v2.0
```

---

# Verify RED Endpoint

```text
http://EXTERNAL-IP:8080/red
```

Expected:

* Red color output

---

# Task 6 – Rollback Production Deployment

## Rollback Command

```bash
kubectl rollout undo deployment production-deployment -n prod
```

## Verify Rollback

```bash
kubectl describe deployment production-deployment -n prod | grep Image
```

Expected:

```text
:v1.0
```

---

# Verify Rollback Result

```text
http://EXTERNAL-IP:8080/red
```

Expected Output:

```text
404 page not found
```

---

# Common Errors and Fixes

## ImagePullBackOff

Reason:

* Wrong image name
* Wrong region
* Wrong tag version

Fix:

```bash
gcloud artifacts docker images list \
us-central1-docker.pkg.dev/$PROJECT_ID/my-repository
```

Match deployment image exactly.

---

## Git Commit Error

```text
Author identity unknown
```

Fix:

```bash
git config --global user.name "USERNAME"
git config --global user.email "EMAIL"
```

---

## Pods Not Running

Check:

```bash
kubectl get pods -n dev
kubectl get pods -n prod
```

Logs:

```bash
kubectl logs POD_NAME -n dev
```

---

# DevOps Concepts Practiced

* CI/CD Pipeline
* Docker Image Build
* Artifact Registry
* Kubernetes Deployment
* Kubernetes Namespace
* Rolling Deployment
* Rollback Strategy
* Git Branch Workflow
* Infrastructure Automation
* Production vs Development Environment

---

# Final Outcome

Successfully implemented:

* Development deployment pipeline
* Production deployment pipeline
* Automated build and deployment
* Version upgrade from v1.0 → v2.0
* Kubernetes rollback to v1.0
* Blue and Red application endpoints

---

# Author

Piyush Kumar
Cloud / DevOps Learning Project
