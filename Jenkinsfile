
pipeline {

    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '976193266769'

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
                    python -m pip install pytest bandit
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
                    bandit -r . -x ./venv
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME} .
                    docker tag ${IMAGE_NAME} ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                '''
            }
        }

        stage('Login to AWS ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS \
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

        stage('Deploy to EC2') {
            steps {
                sh '''
                    docker stop seclock || true
                    docker rm seclock || true

                    docker pull ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest

                    docker run -d \
                    --name seclock \
                    --restart unless-stopped \
                    -p 8080:8080 \
                    ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                '''
            }
        }
    }

    post {

        success {
            echo '======================================'
            echo 'SECLOCK CI/CD PIPELINE SUCCESSFUL'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'SECLOCK CI/CD PIPELINE FAILED'
            echo 'Check Jenkins console output.'
            echo '======================================'
        }

        always {
            sh '''
                docker image prune -f || true
            '''
        }
    }
}


