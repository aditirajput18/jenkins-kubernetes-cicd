# Jenkins CI/CD Setup and Integration Guide

## What the pipeline does

1. Validates parameters and required tools.
2. Checks out the selected Git repository and branch.
3. Verifies access to the configured Kubernetes context and existing Deployment.
4. Creates a Python virtual environment, installs requirements, and runs tests.
5. Builds a Docker image and scans it with Trivy.
6. Archives build metadata and its SHA-256 checksum.
7. Delivers the image using either KIND or REGISTRY mode.
8. Updates the selected Deployment and waits for rollout completion.
9. On deployment failure, gathers diagnostics and attempts a rollout rollback.

## Prerequisites on the Jenkins agent

- Git, Python 3, Docker CLI/daemon, kubectl, and Trivy.
- kind CLI when using KIND delivery.
- Permission to use Docker and access the selected Kubernetes context.
- A repository containing a Dockerfile, requirements.txt, and a compatible test command.
- An existing Kubernetes namespace and Deployment with the specified container name.
- The Deployment's application configuration, Service, probes, and other resources must already be provisioned.

The Jenkins agent must have a working kubeconfig for the target cluster. Configure credentials and access using Jenkins and Kubernetes administration practices; do not commit kubeconfig files or secrets to Git.

## Configure a Jenkins Pipeline job

1. Configure a Pipeline job whose Pipeline definition loads the Jenkinsfile from source control.
2. Configure the repository used to load the Jenkinsfile. SCM polling applies to that configured SCM source.
3. Ensure the Git plugin and required pipeline steps are available.
4. Ensure the Jenkins agent has the tools and permissions listed above.
5. Run the job once with the defaults and inspect each stage.
6. For private source repositories, create a Jenkins Git credential and set GIT_CREDENTIALS_ID.
7. Keep Jenkins credentials out of source control and build logs.

## Important parameters

| Parameter | Purpose |
| --- | --- |
| GIT_URL / GIT_BRANCH | Source repository and branch |
| GIT_CREDENTIALS_ID | Optional credential ID for private Git |
| IMAGE_NAME | Image name without tag or registry hostname |
| DOCKERFILE_PATH | Dockerfile location |
| TEST_COMMAND | Test command for the checked-out project |
| IMAGE_DELIVERY | KIND or REGISTRY |
| K8S_CLUSTER | kind cluster name in KIND mode |
| K8S_CONTEXT | Existing kubectl context |
| K8S_NAMESPACE | Target namespace |
| DEPLOYMENT_NAME | Existing Deployment to update |
| CONTAINER_NAME | Container in that Deployment |
| REGISTRY_URL | Registry hostname in REGISTRY mode |
| REGISTRY_CREDENTIALS_ID | Jenkins username/password credential for registry |
| TRIVY_BIN | Absolute path to the Trivy executable |

## Image delivery modes

### KIND

The built image is loaded into an existing local kind cluster. The Kubernetes nodes must be able to use the loaded image, and the Deployment's pull policy must be compatible with locally loaded images.

### REGISTRY

The image is tagged and pushed to REGISTRY_URL using a Jenkins username/password credential. The target cluster must be able to reach the registry, and its workloads must have appropriate image-pull credentials if the registry is private.

## Application compatibility

The pipeline makes repository, branch, image, test command, and deployment target configurable. It is not a universal build system for every application: projects using other languages, package managers, Docker build contexts, or deployment models may need corresponding build/test parameters or pipeline changes.

The target Deployment must already exist. Provisioning namespaces, Deployments, Services, secrets, and other infrastructure is intentionally separate from the image-delivery pipeline.

## Security notes

- Trivy blocks deployment when unfixed HIGH or CRITICAL image vulnerabilities are found.
- The test command runs shell code from the checked-out repository. Only run trusted repositories and branches.
- Use least-privilege Jenkins and Kubernetes credentials.
- Never pass secrets as plain-text build parameters or commit them to Git.
- Registry credentials must be stored in Jenkins Credentials.
- Validate parameters and review logs before allowing untrusted users to trigger builds.
- Pin and update base images and dependencies; a passing scan is not a guarantee of zero risk.

## Troubleshooting

- **Git checkout fails:** check URL, branch, network access, and credential ID.
- **Docker permission denied:** verify the Jenkins agent's Docker daemon access.
- **Trivy not found:** set TRIVY_BIN to the executable path available to the Jenkins agent.
- **Kubernetes context not found:** configure the agent's kubeconfig and confirm K8S_CONTEXT.
- **Deployment not found:** provision the Deployment first or correct namespace/name parameters.
- **ImagePullBackOff in REGISTRY mode:** check the image name, registry access, and imagePullSecrets.
- **Rollout timeout:** inspect the diagnostic Deployment, pod, and event output. The pipeline attempts rollback, but verify the resulting application state manually.
- **SCM polling does not trigger:** verify the job's configured SCM repository, Jenkins polling logs, and repository/branch settings.

## Verify the deployed application

Use kubectl with the configured context and namespace:

    kubectl --context kind-cicd-assessment get deployments,pods,services
    kubectl --context kind-cicd-assessment rollout status deployment/cicd-assessment-app

For an application-specific health check, use its Service and health endpoint, for example:

    kubectl --context kind-cicd-assessment port-forward service/cicd-assessment-service 8081:5000
    curl -i http://localhost:8081/health
