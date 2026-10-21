# Jenkins–Kubernetes CI/CD Pipeline

## Overview

This project demonstrates an automated CI/CD pipeline for a Python Flask application using Jenkins, Docker, Trivy, and Kubernetes (kind).

A Git commit triggers Jenkins SCM polling. The pipeline checks out the source code, validates the environment, installs dependencies, runs automated tests, builds a Docker image, scans the image for HIGH and CRITICAL vulnerabilities, records build metadata, loads the image into kind, and deploys it to Kubernetes.

## Architecture

**GitHub → Jenkins → Tests → Docker Build → Trivy Scan → kind → Kubernetes Deployment**

The application exposes:
- `/` — application response
- `/health` — health-check endpoint

Kubernetes uses startup, readiness, and liveness probes to monitor the application.

## Technology Stack

- Python and Flask
- pytest
- Git and GitHub
- Jenkins Pipeline with Groovy
- Docker
- Trivy image vulnerability scanner
- Kubernetes and kind

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
├── Dockerfile
├── .dockerignore
├── Jenkinsfile
├── requirements.txt
└── README.md
```

## Pipeline Stages

1. **Checkout:** Retrieves source code from Git.
2. **Validate Environment:** Checks required tools and Kubernetes resources.
3. **Install Dependencies:** Installs Python packages in a virtual environment.
4. **Run Tests:** Executes pytest tests.
5. **Build Docker Image:** Builds an image tagged with the Jenkins build number.
6. **Scan Docker Image:** Uses Trivy to fail the build when qualifying HIGH or CRITICAL findings are detected.
7. **Record Build Metadata:** Archives build number, commit information, image tag, build URL, and a checksum.
8. **Load Image into kind:** Makes the built image available to the local Kubernetes cluster.
9. **Deploy to Kubernetes:** Updates the Deployment image and waits for rollout completion.

## Prerequisites

Install and configure:
- Git
- Python 3
- Docker
- Jenkins with Pipeline support
- kind
- kubectl
- Trivy

The Jenkins service account must have the necessary Docker and Kubernetes access, and permission to execute Trivy.

## Local Testing

```bash
python3 -m venv venv
./venv/bin/pip install -r requirements.txt
./venv/bin/python -m pytest -v
```

## Kubernetes Verification

```bash
kubectl --context kind-cicd-assessment get nodes
kubectl --context kind-cicd-assessment get deployments
kubectl --context kind-cicd-assessment get pods
kubectl --context kind-cicd-assessment rollout status deployment/cicd-assessment-app
```

To test the service locally:

```bash
kubectl --context kind-cicd-assessment port-forward \
  service/cicd-assessment-service 8081:5000
```

In another terminal:

```bash
curl -i http://localhost:8081/health
curl -i http://localhost:8081/
```

## Security and Reliability

- Automated tests run before image deployment.
- Trivy provides a configurable vulnerability gate.
- Kubernetes health probes help detect startup and runtime failures.
- Build metadata and SHA-256 checksums improve traceability.
- Git and Jenkins build history provide an audit trail.

## Troubleshooting

**Pipeline fails during tests:** Review pytest output and application code.

**Trivy blocks the build:** Inspect the vulnerability report and update affected dependencies or the base image where appropriate.

**Image is unavailable in Kubernetes:** Verify the image tag and confirm that the image was loaded into the correct kind cluster.

**Rollout fails:** Inspect pod status, logs, events, and Deployment conditions.

```bash
kubectl --context kind-cicd-assessment describe deployment cicd-assessment-app
kubectl --context kind-cicd-assessment get events --sort-by=.lastTimestamp
```

## Limitations and Future Improvements

- The current deployment targets a local kind cluster.
- SCM polling is used instead of a webhook.
- Production deployment would require managed secrets, access controls, image registry integration, and a defined rollback strategy.
- Monitoring and alerting can be integrated with Prometheus and Grafana.

## Author

Developed as an SRE Intern assessment project to demonstrate automated testing, containerization, vulnerability scanning, and Kubernetes deployment.


Webhook trigger verification: 2026-10-21 13:49:08 UTC

Webhook verification: GitHub push triggers Jenkins automatically.
