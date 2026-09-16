pipeline {

    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '859925121963'

        ECR_REPOSITORY = 'seclock'
        IMAGE_TAG = "${BUILD_NUMBER}"

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Python Setup') {
    steps {
        sh '''
            python3 --version
            python3 -m venv venv
            . venv/bin/activate

            python -m pip install --upgrade pip
            python -m pip install -r requirements.txt
        '''
    }
}

        stage('Unit Tests') {
    steps {
        sh '''
            . venv/bin/activate
            python -m pytest test_e2e.py -v
        '''
    }
}

        stage('Security Scan - Bandit') {
            steps {
                sh '''
                    . venv/bin/activate

                    bandit -r . \
                        --exclude ./venv \
                        -f json \
                        -o bandit-report.json || true

                    echo "Bandit scan completed"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                        -t ${IMAGE_NAME} \
                        -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest \
                        .
                '''
            }
        }

        stage('Docker Image Scan - Trivy') {
            steps {
                sh '''
                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        ${IMAGE_NAME}
                '''
            }
        }

        stage('Login to AWS ECR') {
            steps {
                sh '''
                    aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                    docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    docker push ${IMAGE_NAME}
                    docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                '''
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                sh '''
                    echo "Image successfully pushed:"
                    echo "${IMAGE_NAME}"

                    # This stage will later update your GitOps repository
                    # for Argo CD deployment.

                    echo "Kubernetes image: ${IMAGE_NAME}"
                '''
            }
        }
    }

    post {

        success {
            echo "======================================"
            echo "SECLOCK CI/CD PIPELINE SUCCESSFUL"
            echo "Image: ${IMAGE_NAME}"
            echo "======================================"
        }

        failure {
            echo "======================================"
            echo "SECLOCK CI/CD PIPELINE FAILED"
            echo "Check the failed stage above."
            echo "======================================"
        }

        always {
            sh '''
                docker image prune -f || true
            '''
        }
    }
}
