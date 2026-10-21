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
        string(
            name: 'IMAGE_NAME',
            defaultValue: 'jenkins-kubernetes-cicd',
            description: 'Docker image name, without a tag'
        )
        string(
            name: 'K8S_CLUSTER',
            defaultValue: 'cicd-assessment',
            description: 'Existing kind cluster name'
        )
        string(
            name: 'K8S_CONTEXT',
            defaultValue: 'kind-cicd-assessment',
            description: 'kubectl context for deployment'
        )
        string(
            name: 'K8S_NAMESPACE',
            defaultValue: 'default',
            description: 'Kubernetes namespace'
        )
        string(
            name: 'DEPLOYMENT_NAME',
            defaultValue: 'cicd-assessment-app',
            description: 'Existing Kubernetes Deployment'
        )
        string(
            name: 'CONTAINER_NAME',
            defaultValue: 'flask-app',
            description: 'Container name inside the Deployment'
        )
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

        stage('Validate Configuration') {
            steps {
                script {
                    def checks = [
                        IMAGE_NAME: params.IMAGE_NAME,
                        K8S_CLUSTER: params.K8S_CLUSTER,
                        K8S_CONTEXT: params.K8S_CONTEXT,
                        K8S_NAMESPACE: params.K8S_NAMESPACE,
                        DEPLOYMENT_NAME: params.DEPLOYMENT_NAME,
                        CONTAINER_NAME: params.CONTAINER_NAME
                    ]

                    checks.each { key, value ->
                        if (!value?.trim()) {
                            error("${key} cannot be empty")
                        }
                    }

                    if (!(params.IMAGE_NAME ==~ /[a-z0-9][a-z0-9._\/-]*/)) {
                        error('IMAGE_NAME contains unsupported characters')
                    }

                    if (!(params.K8S_CLUSTER ==~ /[a-zA-Z0-9._-]+/)) {
                        error('Invalid kind cluster name')
                    }

                    if (!(params.K8S_CONTEXT ==~ /[a-zA-Z0-9._:-]+/)) {
                        error('Invalid Kubernetes context')
                    }

                    if (!(params.K8S_NAMESPACE ==~ /[a-z0-9]([-a-z0-9]*[a-z0-9])?/)) {
                        error('Invalid Kubernetes namespace')
                    }

                    if (!(params.DEPLOYMENT_NAME ==~ /[a-z0-9]([-a-z0-9]*[a-z0-9])?/)) {
                        error('Invalid Deployment name')
                    }

                    if (!(params.CONTAINER_NAME ==~ /[a-zA-Z0-9._-]+/)) {
                        error('Invalid container name')
                    }
                }

                sh '''
                    set -eu
                    command -v python3
                    command -v docker
                    command -v kind
                    command -v kubectl

                    kubectl --context "$K8S_CONTEXT" \
                      get namespace "$K8S_NAMESPACE"

                    kubectl --context "$K8S_CONTEXT" \
                      -n "$K8S_NAMESPACE" \
                      get deployment "$DEPLOYMENT_NAME"
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    set -eu
                    python3 -m venv venv
                    ./venv/bin/python -m pip install --upgrade pip
                    ./venv/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    set -eu
                    ./venv/bin/python -m pytest -v
                '''
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

        stage('Load Image into kind') {
            steps {
                sh '''
                    set -eu
                    kind get clusters | grep -Fx -- "$K8S_CLUSTER"

                    kind load docker-image \
                      "$IMAGE_NAME:$IMAGE_TAG" \
                      --name "$K8S_CLUSTER"
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -eu

                    kubectl --context "$K8S_CONTEXT" \
                      -n "$K8S_NAMESPACE" \
                      set image "deployment/$DEPLOYMENT_NAME" \
                      "$CONTAINER_NAME=$IMAGE_NAME:$IMAGE_TAG"

                    kubectl --context "$K8S_CONTEXT" \
                      -n "$K8S_NAMESPACE" \
                      rollout status "deployment/$DEPLOYMENT_NAME" \
                      --timeout=120s
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment succeeded: ${params.IMAGE_NAME}:${env.BUILD_NUMBER}"
        }

        failure {
            echo 'Pipeline failed. Collecting Kubernetes diagnostics.'

            sh '''
                kubectl --context "$K8S_CONTEXT" \
                  -n "$K8S_NAMESPACE" \
                  get pods -o wide || true

                kubectl --context "$K8S_CONTEXT" \
                  -n "$K8S_NAMESPACE" \
                  describe deployment "$DEPLOYMENT_NAME" || true

                kubectl --context "$K8S_CONTEXT" \
                  -n "$K8S_NAMESPACE" \
                  get events --sort-by=.metadata.creationTimestamp || true
            '''
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
