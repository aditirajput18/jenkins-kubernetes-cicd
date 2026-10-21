def requireValue(String label, String value) {
    if (value == null || value.trim().isEmpty()) {
        error("Configuration error: ${label} must not be empty.")
    }
}

pipeline {
    agent any


    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
        skipDefaultCheckout(true)
    }

    parameters {
        string(name: 'GIT_URL', defaultValue: 'https://github.com/aditirajput18/jenkins-kubernetes-cicd.git', description: 'Git repository URL')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Branch to build')
        string(name: 'GIT_CREDENTIALS_ID', defaultValue: '', description: 'Optional Jenkins credential ID for a private Git repository')

        string(name: 'IMAGE_NAME', defaultValue: 'jenkins-kubernetes-cicd', description: 'Image name, without tag or registry hostname')
        string(name: 'DOCKERFILE_PATH', defaultValue: 'Dockerfile', description: 'Path to Dockerfile relative to repository root')


        choice(name: 'IMAGE_DELIVERY', choices: ['KIND', 'REGISTRY'], description: 'Image delivery method')
        string(name: 'K8S_CLUSTER', defaultValue: 'cicd-assessment', description: 'kind cluster name; used only with KIND delivery')
        string(name: 'K8S_CONTEXT', defaultValue: 'kind-cicd-assessment', description: 'Existing kubectl context')
        string(name: 'K8S_NAMESPACE', defaultValue: 'default', description: 'Target namespace')
        string(name: 'DEPLOYMENT_NAME', defaultValue: 'cicd-assessment-app', description: 'Existing Kubernetes Deployment')
        string(name: 'CONTAINER_NAME', defaultValue: 'flask-app', description: 'Container name inside the Deployment')

        string(name: 'REGISTRY_URL', defaultValue: '', description: 'Registry hostname; required for REGISTRY')
        string(name: 'REGISTRY_CREDENTIALS_ID', defaultValue: '', description: 'Jenkins username/password credential ID for registry login')
        string(name: 'TRIVY_BIN', defaultValue: '/usr/local/bin/trivy', description: 'Absolute path to the Trivy executable')
    }

    environment {
        PYTHON_VENV = 'venv'
        TRIVY_BIN = "${params.TRIVY_BIN}"
    }

    stages {
        stage('Validate Configuration') {
            steps {
                script {
                    requireValue('GIT_URL', params.GIT_URL)
                    requireValue('GIT_BRANCH', params.GIT_BRANCH)
                    requireValue('IMAGE_NAME', params.IMAGE_NAME)
                    requireValue('K8S_CONTEXT', params.K8S_CONTEXT)
                    requireValue('K8S_NAMESPACE', params.K8S_NAMESPACE)
                    requireValue('DEPLOYMENT_NAME', params.DEPLOYMENT_NAME)
                    requireValue('CONTAINER_NAME', params.CONTAINER_NAME)
                    requireValue('DOCKERFILE_PATH', params.DOCKERFILE_PATH)
                    requireValue('TRIVY_BIN', params.TRIVY_BIN)

                    if (!(params.IMAGE_NAME ==~ /[a-zA-Z0-9][a-zA-Z0-9._\/-]*/)) {
                        error('IMAGE_NAME contains unsupported characters.')
                    }
                    if (!(params.IMAGE_DELIVERY in ['KIND', 'REGISTRY'])) {
                        error('IMAGE_DELIVERY must be KIND or REGISTRY.')
                    }
                    if (params.IMAGE_DELIVERY == 'REGISTRY') {
                        requireValue('REGISTRY_URL', params.REGISTRY_URL)
                        requireValue('REGISTRY_CREDENTIALS_ID',
                            params.REGISTRY_CREDENTIALS_ID)
                    }
                }
            }
        }

        stage('Checkout') {
            steps {
                script {
                    def remote = [url: params.GIT_URL]
                    if (params.GIT_CREDENTIALS_ID?.trim()) {
                        remote.credentialsId = params.GIT_CREDENTIALS_ID.trim()
                    }
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.GIT_BRANCH}"]],
                        userRemoteConfigs: [remote],
                        extensions: [[$class: 'CleanBeforeCheckout']]
                    ])
                }
            }
        }



	stage('Validate Tools and Target') {
 	   steps {
       	       script {
                   withEnv([
                	"TRIVY_BIN=${params.TRIVY_BIN}",
                	"DOCKERFILE_PATH=${params.DOCKERFILE_PATH}",
                	"K8S_CONTEXT=${params.K8S_CONTEXT}",
                	"K8S_NAMESPACE=${params.K8S_NAMESPACE}",
                	"DEPLOYMENT_NAME=${params.DEPLOYMENT_NAME}"
            	    ]) {
               		 sh '''
                    	    set -eu

                    command -v git
                    command -v python3
                    command -v docker
                    command -v kubectl

                    test -x "$TRIVY_BIN"
                    test -f "$DOCKERFILE_PATH"

                    kubectl --context "$K8S_CONTEXT" \
                        get namespace "$K8S_NAMESPACE"

                    kubectl --context "$K8S_CONTEXT" \
                        -n "$K8S_NAMESPACE" \
                        get deployment "$DEPLOYMENT_NAME"
                '''
            }
        }
    }
}
        stage('Install Dependencies') {
            steps {
                sh '''
                    set -eu
                    python3 -m venv "$PYTHON_VENV"
                    "./$PYTHON_VENV/bin/pip" install --upgrade pip
                    "./$PYTHON_VENV/bin/pip" install -r requirements.txt
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


        stage('Build Docker Image') {
            steps {
                script {
                    env.LOCAL_IMAGE = "${params.IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh '''
                        set -eu
                        docker build --pull \
                            -f "$DOCKERFILE_PATH" \
                            -t "$LOCAL_IMAGE" .
                    '''
                }
            }
        }

        stage('Scan Docker Image') {
            steps {
                sh '''
                    set -eu
                    "$TRIVY_BIN" image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        "$LOCAL_IMAGE"
                '''
            }
        }

        stage('Record Build Metadata') {
            steps {
                script {
                    env.GIT_REVISION = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()
                    env.DEPLOY_IMAGE = env.LOCAL_IMAGE

                    if (params.IMAGE_DELIVERY == 'REGISTRY') {
                        env.DEPLOY_IMAGE =
                            "${params.REGISTRY_URL}/${params.IMAGE_NAME}:${env.BUILD_NUMBER}"
                    }
                }
                sh '''
                    set -eu
                    {
                        echo "BUILD_NUMBER=$BUILD_NUMBER"
                        echo "GIT_COMMIT=$GIT_REVISION"
                        echo "SOURCE_REPOSITORY=$GIT_URL"
                        echo "SOURCE_BRANCH=$GIT_BRANCH"
                        echo "LOCAL_IMAGE=$LOCAL_IMAGE"
                        echo "DEPLOY_IMAGE=$DEPLOY_IMAGE"
                        echo "IMAGE_DELIVERY=$IMAGE_DELIVERY"
                        echo "K8S_CONTEXT=$K8S_CONTEXT"
                        echo "K8S_NAMESPACE=$K8S_NAMESPACE"
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

        stage('Deliver Image') {
            steps {
                script {
                    if (params.IMAGE_DELIVERY == 'KIND') {
                        sh '''
                            set -eu
                            command -v kind
                            kind load docker-image "$LOCAL_IMAGE" \
                                --name "$K8S_CLUSTER"
                        '''
                    } else {
                        withCredentials([usernamePassword(
                            credentialsId: params.REGISTRY_CREDENTIALS_ID,
                            usernameVariable: 'REGISTRY_USER',
                            passwordVariable: 'REGISTRY_PASSWORD'
                        )]) {
                            sh '''
                                set -eu
                                set +x
                                echo "$REGISTRY_PASSWORD" |
                                    docker login "$REGISTRY_URL" \
                                        --username "$REGISTRY_USER" \
                                        --password-stdin
                                docker tag "$LOCAL_IMAGE" "$DEPLOY_IMAGE"
                                docker push "$DEPLOY_IMAGE"
                                docker logout "$REGISTRY_URL" || true
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                script {
                    try {
                        sh '''
                            set -eu
                            kubectl --context "$K8S_CONTEXT" \
                                -n "$K8S_NAMESPACE" \
                                set image "deployment/$DEPLOYMENT_NAME" \
                                "$CONTAINER_NAME=$DEPLOY_IMAGE"
                            kubectl --context "$K8S_CONTEXT" \
                                -n "$K8S_NAMESPACE" \
                                rollout status "deployment/$DEPLOYMENT_NAME" \
                                --timeout=120s
                        '''
                    } catch (err) {
                        echo 'Deployment failed. Collecting diagnostics.'
                        sh(
                            returnStatus: true,
                            script: '''
                                kubectl --context "$K8S_CONTEXT" \
                                    -n "$K8S_NAMESPACE" \
                                    describe deployment "$DEPLOYMENT_NAME" || true
                                kubectl --context "$K8S_CONTEXT" \
                                    -n "$K8S_NAMESPACE" get pods -o wide || true
                                kubectl --context "$K8S_CONTEXT" \
                                    -n "$K8S_NAMESPACE" get events \
                                    --sort-by=.lastTimestamp || true
                            '''
                        )
                        echo 'Attempting rollback to the previous Deployment revision.'
                        int rollbackStatus = sh(
                            returnStatus: true,
                            script: '''
                                kubectl --context "$K8S_CONTEXT" \
                                    -n "$K8S_NAMESPACE" \
                                    rollout undo "deployment/$DEPLOYMENT_NAME"
                                kubectl --context "$K8S_CONTEXT" \
                                    -n "$K8S_NAMESPACE" \
                                    rollout status "deployment/$DEPLOYMENT_NAME" \
                                    --timeout=120s
                            '''
                        )
                        if (rollbackStatus != 0) {
                            echo 'WARNING: Automatic rollback did not complete. Manual investigation is required.'
                        }
                        throw err
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD succeeded: tests passed, image scan passed, and rollout completed.'
        }
        failure {
            echo 'CI/CD failed. Review the failed stage and diagnostic output above.'
        }
        always {
            sh(
                returnStatus: true,
                script: '''
                    docker logout "$REGISTRY_URL" >/dev/null 2>&1 || true
                '''
            )
            echo 'Pipeline execution finished.'
        }
    }
}
