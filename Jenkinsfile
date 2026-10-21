
pipeline {
    agent any

    triggers {
        pollSCM('H/2 * * * *')
    }
    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'jenkins-kubernetes-cicd'
        K8S_CLUSTER = 'cicd-assessment'
        DEPLOYMENT_NAME = 'cicd-assessment-app'
        CONTAINER_NAME = 'flask-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
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
                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Load Image into kind') {
            steps {
                sh '''
                    kind load docker-image \
                      ${IMAGE_NAME}:${BUILD_NUMBER} \
                      --name ${K8S_CLUSTER}
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    kubectl set image \
                      deployment/${DEPLOYMENT_NAME} \
                      ${CONTAINER_NAME}=${IMAGE_NAME}:${BUILD_NUMBER}

                    kubectl rollout status \
                      deployment/${DEPLOYMENT_NAME} \
                      --timeout=120s
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline succeeded: application deployed.'
        }

        failure {
            echo 'Pipeline failed. Review the stage logs.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}

