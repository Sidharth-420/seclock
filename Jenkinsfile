pipeline {

    agent any

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '976193266769'

        ECR_REPOSITORY = 'seclock'
        IMAGE_TAG      = "${BUILD_NUMBER}"

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME   = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

        CONTAINER_NAME = 'seclock'
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
                    set -e

                    python3 --version

                    rm -rf venv
                    python3 -m venv venv

                    . venv/bin/activate

                    python -m pip install --upgrade pip
                    python -m pip install -r requirements.txt

                    python -m pip install pytest bandit httpx2
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate

                    python -m pytest test_e2e.py -v
                    TEST_EXIT=$?

                    if [ $TEST_EXIT -eq 5 ]; then
                        echo "WARNING: No pytest tests were collected."
                        echo "Continuing pipeline..."
                        exit 0
                    fi

                    exit $TEST_EXIT
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
                    set -e

                    docker build \
                        -t ${IMAGE_NAME} \
                        -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest \
                        .
                '''
            }
        }

        stage('Login to AWS ECR') {
            steps {
                sh '''
                    set -e

                    echo "Checking AWS identity..."
                    aws sts get-caller-identity

                    echo "Logging into Amazon ECR..."

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
                    set -e

                    docker push ${IMAGE_NAME}

                    docker push \
                        ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    set -e

                    echo "Stopping old container..."

                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true

                    echo "Pulling latest image..."

                    docker pull \
                        ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest

                    echo "Starting new container..."

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p 8080:8080 \
                        ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest

                    sleep 5

                    echo "Checking container..."

                    docker ps | grep ${CONTAINER_NAME}
                '''
            }
        }

        stage('Verify Application') {
            steps {
                sh '''
                    set -e

                    echo "Testing application..."

                    curl -f http://localhost:8080/

                    echo ""
                    echo "SEclock application is running successfully!"
                '''
            }
        }
    }

    post {

        success {
            echo '''
========================================
 SEclock CI/CD PIPELINE SUCCESSFUL
========================================

Docker Image:
${IMAGE_NAME}

ECR:
${ECR_REGISTRY}

Application:
http://EC2_PUBLIC_IP:8080

========================================
            '''
        }

        failure {
            echo '''
========================================
 SEclock CI/CD PIPELINE FAILED
========================================

Check the failed stage above.

========================================
            '''
        }

        always {
            sh '''
                docker image prune -f || true
            '''
        }
    }
}

