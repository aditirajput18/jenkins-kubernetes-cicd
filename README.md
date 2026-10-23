# Jenkins–Kubernetes CI/CD Pipeline

## Overview

This project demonstrates an automated CI/CD pipeline for a Python Flask application using Jenkins, Docker, Trivy, and Kubernetes.

A GitHub push webhook triggers Jenkins to execute the pipeline automatically. The pipeline validates the configuration, checks the required tools and deployment target, installs dependencies, runs automated tests, builds a Docker image, scans it for HIGH and CRITICAL vulnerabilities using Trivy, and records build metadata with a SHA-256 checksum.

The pipeline supports two image-delivery modes:

- **KIND:** Loads the Docker image directly into the existing kind Kubernetes cluster.
- **REGISTRY:** Pushes the image to a configured container registry for deployment.

Finally, Jenkins deploys the image to Kubernetes, verifies the rollout, collects diagnostic information if deployment fails, and attempts to roll back to the previous Deployment revision. Automated rollback is implemented but has not yet been validated through an end-to-end failed deployment test.

## Architecture

**GitHub → Webhook → Jenkins → Checkout → Tests → Docker Build → Trivy Scan → Image Delivery → Kubernetes Deployment → Rollout Verification**

The application exposes:

- `/` — application response
- `/health` — health-check endpoint

Kubernetes startup, readiness, and liveness probes help monitor application startup and health.

## Technology Stack

- Python and Flask
- pytest
- Git and GitHub
- Jenkins Declarative Pipeline and Groovy
- Docker
- Trivy vulnerability scanner
- Kubernetes
- kind
- kubectl
- GitHub webhooks
- SHA-256 build artifact checksums

## Repository Structure

```text
.
├── app/
│   └── app.py
├── tests/
│   └── test_app.py
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── docs/
│   └── SETUP_AND_INTEGRATION.md
├── Dockerfile
├── .dockerignore
├── Jenkinsfile
├── requirements.txt
└── README.md
```

## Pipeline Stages

1. **Validate Configuration:** Checks required parameters and validates the selected image-delivery configuration.
2. **Checkout:** Retrieves the configured Git repository and branch.
3. **Validate Tools and Target:** Checks required tools, the Trivy executable, the Dockerfile, the Kubernetes context, the namespace, and the target Deployment.
4. **Install Dependencies:** Creates a Python virtual environment and installs project dependencies.
5. **Run Tests:** Executes the automated test suite using pytest.
6. **Build Docker Image:** Builds the application image using a Jenkins build-number tag.
7. **Scan Docker Image:** Runs Trivy and fails the pipeline when qualifying HIGH or CRITICAL vulnerabilities are found.
8. **Record Build Metadata:** Records build and commit information, the image tag, build URL, and a SHA-256 checksum in an archived artifact.
9. **Deliver Image:** Loads the image into the selected kind cluster or pushes it to the configured container registry.
10. **Deploy and Verify:** Updates the existing Kubernetes Deployment and waits for rollout completion. If deployment fails, the pipeline gathers diagnostics and attempts an automatic rollback.

## Prerequisites

Install and configure:

- Git
- Python 3
- Docker
- Jenkins with Pipeline support
- kind
- kubectl
- Trivy

The Jenkins service account needs appropriate permissions to run Docker, access the configured Kubernetes context, and execute Trivy.

For registry delivery, configure the registry hostname and a Jenkins username/password credential. The Kubernetes cluster must be able to pull the published image; private registries may require an appropriate `imagePullSecret`.

## Jenkins Configuration

The pipeline accepts configurable parameters, including:

| Parameter | Purpose |
|---|---|
| `GIT_URL` | Git repository URL |
| `GIT_BRANCH` | Branch to build |
| `IMAGE_NAME` | Application image name |
| `DOCKERFILE_PATH` | Dockerfile location |
| `IMAGE_DELIVERY` | `KIND` or `REGISTRY` |
| `K8S_CLUSTER` | kind cluster name for KIND delivery |
| `K8S_CONTEXT` | Existing Kubernetes context |
| `K8S_NAMESPACE` | Target namespace |
| `DEPLOYMENT_NAME` | Existing Kubernetes Deployment |
| `CONTAINER_NAME` | Container to update |
| `REGISTRY_URL` | Container registry hostname |
| `REGISTRY_CREDENTIALS_ID` | Jenkins credential ID for registry authentication |
| `TRIVY_BIN` | Absolute path to the Trivy executable |

Configure GitHub to send push events to the Jenkins webhook endpoint:

```text
https://YOUR-TUNNEL-HOST/github-webhook/
```

Replace `YOUR-TUNNEL-HOST` with the active public tunnel hostname. If using a Cloudflare Quick Tunnel, keep the tunnel running and update the webhook whenever its URL changes.

### GitHub Webhook Integration

#### Initial Approach: SCM Polling

During the initial setup, SCM polling was used to check the GitHub repository periodically for new commits and trigger Jenkins builds.

#### Webhook-Based Triggering

We later configured a GitHub webhook to trigger Jenkins automatically whenever code was pushed to the repository. The integration was verified by pushing code to GitHub and confirming that Jenkins started a new pipeline build.

Since Jenkins was running locally on an Ubuntu VM, **Cloudflare Quick Tunnel** was used to expose the Jenkins webhook endpoint to GitHub.

The webhook URL follows this format:

```text
https://YOUR-TUNNEL-HOST/github-webhook/
```

Replace `YOUR-TUNNEL-HOST` with the active Cloudflare Quick Tunnel hostname.

#### Why SCM Polling Was Removed

After webhook triggering was verified, SCM polling was removed. The pipeline now uses event-driven webhook triggering instead of periodically checking GitHub for changes.

#### Limitation

Cloudflare Quick Tunnel provides a temporary public URL. If the tunnel is recreated and its hostname changes, the GitHub webhook URL must be updated accordingly. A named Cloudflare Tunnel with a stable hostname could be considered for a more permanent setup.

**Current configuration:** GitHub push webhook through Cloudflare Quick Tunnel. SCM polling was used during initial setup but is no longer enabled.

## Local Testing

Create a virtual environment and install dependencies:

```bash
python3 -m venv venv
./venv/bin/pip install -r requirements.txt
./venv/bin/python -m pytest -v
```

## Kubernetes Verification

Check the cluster, Deployment, and pods:

```bash
kubectl --context kind-cicd-assessment get nodes
kubectl --context kind-cicd-assessment get deployments
kubectl --context kind-cicd-assessment get pods
kubectl --context kind-cicd-assessment rollout status \
  deployment/cicd-assessment-app
```

To access the service locally, start port forwarding:

```bash
kubectl --context kind-cicd-assessment port-forward \
  service/cicd-assessment-service 8081:5000
```

In another terminal, test the endpoints:

```bash
curl -i http://localhost:8081/health
curl -i http://localhost:8081/
```

## Security and Reliability

- Automated tests run before image deployment.
- Trivy provides a vulnerability-scanning gate.
- Kubernetes health probes help detect startup and runtime failures.
- Build-number image tags improve build traceability.
- Archived build metadata and SHA-256 checksums support artifact verification.
- Jenkins build history and Git commit history provide an audit trail.
- Deployment failure handling collects Kubernetes diagnostics and attempts a rollback.
- Registry credentials are managed through Jenkins credentials rather than hard-coded into the pipeline.

## Troubleshooting

**Pipeline fails during tests**

Review the pytest output, application code, and dependency configuration.

**Trivy blocks the build**

Inspect the vulnerability report and update affected dependencies or the base image where appropriate. Do not bypass the security gate without understanding the risk.

**Kubernetes cannot find the image**

For `KIND` delivery, verify that the image was loaded into the correct kind cluster. For `REGISTRY` delivery, check the registry URL, image tag, cluster network access, and image-pull credentials.

**Deployment rollout fails**

Inspect the Deployment, pods, logs, and events:

```bash
kubectl --context kind-cicd-assessment \
  describe deployment cicd-assessment-app

kubectl --context kind-cicd-assessment get pods -o wide

kubectl --context kind-cicd-assessment \
  get events --sort-by=.lastTimestamp
```

**GitHub push does not trigger Jenkins**

Check that the webhook URL points to the currently reachable Jenkins endpoint, the GitHub push event is enabled, and the latest delivery succeeded under **GitHub → Settings → Webhooks → Recent Deliveries**. Also verify that the Jenkins job has **GitHub hook trigger for GITScm polling** enabled.

## Limitations and Future Improvements

- The current deployment target is a local kind cluster.
- The public webhook depends on the availability of the configured endpoint. Cloudflare Quick Tunnels are temporary and are not intended to provide production uptime guarantees.
- The `REGISTRY` delivery path and deployment against a second Kubernetes cluster require independent testing before being considered verified.
- Automatic rollback is implemented in the Jenkins pipeline but has not yet been verified through an end-to-end failed deployment test.
- Production use would require hardened credentials, restricted access controls, a reliable registry, managed infrastructure, and a documented recovery strategy.
- Prometheus and Grafana could be added for metrics, dashboards, and operational monitoring.

## Author

Developed as a Devops Intern assessment project to demonstrate CI/CD automation, automated testing, containerization, vulnerability scanning, Kubernetes deployment, and deployment recovery handling.
