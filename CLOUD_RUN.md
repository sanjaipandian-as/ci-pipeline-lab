# Cloud Run Mapping Note

This document describes how the current local Docker Hub + Docker Compose CI/CD pipeline maps to a production Google Cloud Run deployment workflow.

---

## Architectural Mapping Matrix

| Stage / Component | Current Pipeline (Docker Hub + Docker Compose) | Google Cloud Run Architecture |
| :--- | :--- | :--- |
| **Authentication** | `docker/login-action@v3` with `DOCKERHUB_USER` & `DOCKERHUB_TOKEN` | `google-github-actions/auth@v2` using a GCP Service Account Key (`GCP_SA_KEY`) or Workload Identity Provider |
| **Image Registry & Push** | Push image to Docker Hub (`<user>/app:${{ github.sha }}`) | Push image to Google Artifact Registry (`<region>-docker.pkg.dev/<project-id>/<repository>/app:${{ github.sha }}`) |
| **Deployment Target** | `docker compose up -d` on runner runtime | `gcloud run deploy` (or `google-github-actions/deploy-cloudrun@v2`) targeting Google Cloud Run serverless container environment |
| **Ingress & Port Handling** | Static port binding (`8080:8080`) | Cloud Run auto-scaling managed HTTPS endpoint forwarding traffic to container `$PORT` (8080) |

---

## Conceptual GCP GitHub Actions Workflow Example

> **Note:** Documentation only — do not run or execute.

```yaml
name: Deploy to Google Cloud Run

on:
  push:
    branches:
      - main

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  REGION: us-central1
  GAR_REPOSITORY: my-app-repo
  SERVICE_NAME: ci-pipeline-app

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      # 1. AUTHENTICATION: Authenticate using Service Account Key / Workload Identity
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker for Artifact Registry
        run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev

      # 2. PUSH TO ARTIFACT REGISTRY: Tag and push container image
      - name: Build and Push Image to Artifact Registry
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.GAR_REPOSITORY }}/app:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      # 3. DEPLOYMENT: Deploy service using gcloud run deploy
      - name: Deploy to Cloud Run
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: ${{ env.SERVICE_NAME }}
          region: ${{ env.REGION }}
          image: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.GAR_REPOSITORY }}/app:${{ github.sha }}
          flags: '--allow-unauthenticated --port=8080'
```
