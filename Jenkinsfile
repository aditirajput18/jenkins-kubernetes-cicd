pipeline {
    agent any

    triggers {
        pollSCM('H/2 * * * *')
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 15, unit: 'MINUTES')
    }

    parameters {
        string(name: 'IMAGE_NAME', defaultValue: 'jenkins-kubernetes-cicd', description: 'Docker image name')
        string(name: 'K8S_CLUSTER', defaultValue: 'cicd-assessment', description: 'kind cluster name')
        string(name: 'K8S_CONTEXT', defaultValue: 'kind-cicd-assessment', description: 'kubectl context')
        string(name: 'K8S_NAMESPACE', defaultValue: 'default', description: 'Kubernetes namespace')
        string(name: 'DEPLOYMENT_NAME', defaultValue: 'cicd-assessment-app', description: 'Deployment name')
        string(name: 'CONTAINER_NAME', defaultValue: 'flask-app', description: 'Container name in Deployment')
    }

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Environment') {
            steps {
                sh '''
                    set -eu
                    command -v python3
                    command -v docker
                    command -v kind
                    command -v kubectl
                    test -x /usr/local/bin/trivy
                    kubectl --context "$K8S_CONTEXT" get namespace "$K8S_NAMESPACE"
                    kubectl --context "$K8S_CONTEXT" \
                      -n "$K8S_NAMESPACE" get deployment "$DEPLOYMENT_NAME"
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    set -eu
                    python3 -m venv venv
                    ./venv/bin/pip install --upgrade pip
                    ./venv/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh './venv/bin/python -m pytest -v'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build -t "$IMAGE_NAME:$IMAGE_TAG" .
                '''
            }
        }

        stage('Scan Docker Image') {
            steps {
                sh '''
                    set -eu
                    /usr/local/bin/trivy image \
                      --exit-code 1 \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      "$IMAGE_NAME:$IMAGE_TAG"
                '''
            }
        }

        stage('Record Build Metadata') {
            steps {
                sh '''
                    set -eu
                    {
                      echo "BUILD_NUMBER=$BUILD_NUMBER"
                      echo "GIT_COMMIT=${GIT_COMMIT:-unknown}"
                      echo "IMAGE=$IMAGE_NAME:$IMAGE_TAG"
                      echo "BUILD_URL=${BUILD_URL:-unknown}"
                    } > build-info.txt

                    sha256sum build-info.txt > build-info.txt.sha256
                '''
                archiveArtifacts(
                    artifacts: 'build-info.txt,build-info.txt.sha256',
                    fingerprint: true
                )
            }
        }

        stage('Load Image into kind') {
            steps {
                sh '''
                    set -eu
                    kind load docker-image "$IMAGE_NAME:$IMAGE_TAG" \
                      --name "$K8S_CLUSTER"
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -eu
                    kubectl --context "$K8S_CONTEXT" \
                      -n "$K8S_NAMESPACE" set image \
                      "deployment/$DEPLOYMENT_NAME" \
                      "$CONTAINER_NAME=$IMAGE_NAME:$IMAGE_TAG"

                    kubectl --context "$K8S_CONTEXT" \
                      -n "$K8S_NAMESPACE" rollout status \
                      "deployment/$DEPLOYMENT_NAME" \
                      --timeout=120s
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline succeeded: security scan passed and application deployed.'
        }

        failure {
            echo 'Pipeline failed. Review the stage logs to identify the cause.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
