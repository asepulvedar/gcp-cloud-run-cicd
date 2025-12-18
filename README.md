# Cloud Run CICD with Cloud Build

This repository contains a simple Flask application and the infrastructure-as-code required to build a container, push it to Artifact Registry, and deploy it to Cloud Run via Cloud Build. Once configured, every push to your default branch in GitHub will trigger Cloud Build to deploy the latest container to Cloud Run.

## Prerequisites
- A Google Cloud project with billing enabled.
- `gcloud` CLI authenticated (`gcloud auth login`) and set to the target project (`gcloud config set project <PROJECT_ID>`).
- APIs enabled (one-time):
  ```sh
  gcloud services enable run.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com
  ```

## Local Development
1. Use Python 3.12+.
2. Create a virtual environment and install dependencies.
   ```sh
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```
3. Run locally:
   ```sh
   python app.py
   ```
4. Visit `http://localhost:8080`.

## GitHub Repository
Initialize the repo locally (this directory already contains a `.gitignore` suitable for Python projects):
```sh
git init
git add .
git commit -m "Initial commit"
```
Create an empty repository in GitHub (e.g., `gcp-cloud-run-cicd`) and connect it:
```sh
git remote add origin git@github.com:<ORG_OR_USER>/<REPO>.git
git branch -M main
git push -u origin main
```

## Cloud Build Pipeline

### Artifact Registry
Create a repository for your container images (only once per project/region). Customize the region, repository name, and description as needed.
```sh
gcloud artifacts repositories create cloud-run-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker repo for Cloud Run deployments"
```

### Cloud Run Service
Create (or allow Cloud Build to create) the Cloud Run service. To provision ahead of time:
```sh
gcloud run deploy hello-cloud-run \
  --source . \
  --region=us-central1 \
  --allow-unauthenticated
```
Cloud Build will later redeploy this service using the container image it builds.

### Cloud Build Configuration
`cloudbuild.yaml` defines three steps:
1. Build: `gcloud builds` uses the Docker builder to create an image from `Dockerfile`.
2. Push: The image is pushed to Artifact Registry.
3. Deploy: `gcloud run deploy` updates your Cloud Run service with the pushed image.

Substitution variables make the pipeline reusable:
- `_REGION` — Artifact Registry & Cloud Run region (e.g., `us-central1`).
- `_REPOSITORY` — Artifact Registry repository (e.g., `cloud-run-repo`).
- `_SERVICE_NAME` — Cloud Run service name (e.g., `hello-cloud-run`).

### Trigger
1. In the Cloud Console, open **Cloud Build → Triggers**.
2. Connect the GitHub repository created earlier and authorize Cloud Build to read it.
3. Create a trigger:
   - Event: Push to a branch.
   - Branch: `^main$`.
   - Configuration: `cloudbuild.yaml`.
   - Substitution variables: `_REGION`, `_REPOSITORY`, `_SERVICE_NAME` with your values.
4. Save the trigger.

Every push to `main` will now:
1. Build and tag the image as `$_REGION-docker.pkg.dev/$PROJECT_ID/$_REPOSITORY/$_SERVICE_NAME:$COMMIT_SHA`.
2. Push the image to Artifact Registry.
3. Deploy the image to Cloud Run in the specified region.

### Permissions
Ensure the Cloud Build service account can deploy to Cloud Run and write to Artifact Registry:
```sh
PROJECT=$(gcloud config get-value project)
SA="${PROJECT}@cloudbuild.gserviceaccount.com"
gcloud projects add-iam-policy-binding "$PROJECT" \
  --member="serviceAccount:${SA}" \
  --role="roles/run.admin"
gcloud projects add-iam-policy-binding "$PROJECT" \
  --member="serviceAccount:${SA}" \
  --role="roles/iam.serviceAccountUser"
gcloud projects add-iam-policy-binding "$PROJECT" \
  --member="serviceAccount:${SA}" \
  --role="roles/artifactregistry.writer"
```

## Manual Execution
You can manually invoke the pipeline for testing:
```sh
gcloud builds submit \
  --config cloudbuild.yaml \
  --substitutions=_REGION=us-central1,_REPOSITORY=cloud-run-repo,_SERVICE_NAME=hello-cloud-run
```
This uses the same steps the trigger executes.

## Next Steps
- Wire up environment-specific configuration if you have staging/prod services.
- Add automated tests before the build step and fail the build if they don't pass.
