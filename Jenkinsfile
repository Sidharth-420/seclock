pipeline {

    agent any

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '976193266769'
        ECR_REPOSITORY = 'seclock'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        IMAGE_TAG    = "${BUILD_NUMBER}"
        IMAGE_NAME   = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
        LATEST_IMAGE = "${ECR_REGISTRY}/${ECR_REPOSITORY}:latest"

        CONTAINER_NAME = 'seclock'
        APP_PORT       = '8000'
    }

    stages {

        stage('Environment Check') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "Environment Check"
                    echo "========================================"

                    echo "Python:"
                    python3 --version

                    echo ""
                    echo "Docker:"
                    docker --version

                    echo ""
                    echo "AWS CLI:"
                    aws --version

                    echo ""
                    echo "Git:"
                    git --version

                    echo ""
                    echo "Workspace:"
                    pwd

                    echo ""
                    echo "Repository files:"
                    ls -la

                    echo "========================================"
                '''
            }
        }

        stage('Python Setup') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "Python Setup"
                    echo "========================================"

                    rm -rf venv

                    python3 -m venv venv

                    . venv/bin/activate

                    python -m pip install --upgrade pip

                    echo "Installing application dependencies..."

                    python -m pip install -r requirements.txt

                    echo "Installing CI dependencies..."

                    python -m pip install \
                        pytest \
                        bandit \
                        httpx2

                    echo "Python setup completed."

                    echo "========================================"
                '''
            }
        }

        stage('Application Import Check') {
            steps {
                sh '''
                    set -eu

                    . venv/bin/activate

                    echo "========================================"
                    echo "FastAPI Import Check"
                    echo "========================================"

                    python -c "from main import app; print('FastAPI application imported successfully')"

                    echo "========================================"
                '''
            }
        }

        stage('E2E Tests') {
            steps {
                sh '''
                    set -eu

                    . venv/bin/activate

                    echo "========================================"
                    echo "SEclock E2E Tests"
                    echo "========================================"

                    python test_e2e.py

                    echo ""
                    echo "ALL SEclock E2E TESTS PASSED."
                    echo "========================================"
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    set -eu

                    . venv/bin/activate

                    echo "========================================"
                    echo "Bandit Security Scan"
                    echo "========================================"

                    bandit \
                        -r main.py \
                        crypto_engine.py \
                        ocr_engine.py \
                        audit_ledger.py \
                        -f txt \
                        --skip B105

                    echo ""
                    echo "Bandit security scan completed successfully."
                    echo "========================================"
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "Docker Build"
                    echo "========================================"

                    docker build \
                        --pull \
                        -t "${IMAGE_NAME}" \
                        -t "${LATEST_IMAGE}" \
                        .

                    echo ""
                    echo "Docker image built successfully."

                    echo ""
                    echo "Images:"
                    docker images "${ECR_REGISTRY}/${ECR_REPOSITORY}"

                    echo "========================================"
                '''
            }
        }

        stage('Docker Smoke Test') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "Docker Smoke Test"
                    echo "========================================"

                    TEST_CONTAINER="seclock-smoke-test"

                    docker rm -f "${TEST_CONTAINER}" 2>/dev/null || true

                    docker run -d \
                        --name "${TEST_CONTAINER}" \
                        -p 18000:8000 \
                        "${IMAGE_NAME}"

                    echo ""
                    echo "Waiting for FastAPI container..."

                    SUCCESS=0

                    for i in $(seq 1 30); do

                        if curl -fsS \
                            --max-time 3 \
                            "http://127.0.0.1:18000/" \
                            > /dev/null 2>&1; then

                            SUCCESS=1

                            echo ""
                            echo "Docker container is responding."

                            break
                        fi

                        echo "Waiting... attempt ${i}/30"

                        sleep 2
                    done

                    if [ "${SUCCESS}" -ne 1 ]; then

                        echo ""
                        echo "ERROR: Docker container did not start correctly."

                        echo ""
                        echo "Container status:"

                        docker ps -a \
                            --filter "name=${TEST_CONTAINER}"

                        echo ""
                        echo "Container logs:"

                        docker logs "${TEST_CONTAINER}" || true

                        docker rm -f "${TEST_CONTAINER}" || true

                        exit 1
                    fi

                    echo ""
                    echo "Docker smoke test PASSED."

                    docker logs "${TEST_CONTAINER}" || true

                    docker rm -f "${TEST_CONTAINER}" || true

                    echo "========================================"
                '''
            }
        }

        stage('AWS Check') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "AWS / ECR Check"
                    echo "========================================"

                    echo "AWS Identity:"

                    aws sts get-caller-identity

                    echo ""
                    echo "Checking ECR repository..."

                    aws ecr describe-repositories \
                        --repository-names "${ECR_REPOSITORY}" \
                        --region "${AWS_REGION}" \
                        > /dev/null

                    echo ""
                    echo "ECR repository exists."

                    echo "========================================"
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "Amazon ECR Login"
                    echo "========================================"

                    aws ecr get-login-password \
                        --region "${AWS_REGION}" | \
                    docker login \
                        --username AWS \
                        --password-stdin "${ECR_REGISTRY}"

                    echo ""
                    echo "ECR login successful."

                    echo "========================================"
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "Push Docker Images to ECR"
                    echo "========================================"

                    echo "Pushing build image:"
                    echo "${IMAGE_NAME}"

                    docker push "${IMAGE_NAME}"

                    echo ""
                    echo "Pushing latest image:"
                    echo "${LATEST_IMAGE}"

                    docker push "${LATEST_IMAGE}"

                    echo ""
                    echo "Docker images pushed successfully."

                    echo "========================================"
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "Deploy SEclock to EC2"
                    echo "========================================"

                    echo "Pulling latest image..."

                    docker pull "${LATEST_IMAGE}"

                    echo ""
                    echo "Stopping existing container..."

                    docker stop "${CONTAINER_NAME}" 2>/dev/null || true

                    docker rm "${CONTAINER_NAME}" 2>/dev/null || true

                    echo ""
                    echo "Starting new SEclock container..."

                    docker run -d \
                        --name "${CONTAINER_NAME}" \
                        --restart unless-stopped \
                        -p "${APP_PORT}:${APP_PORT}" \
                        "${LATEST_IMAGE}"

                    echo ""
                    echo "Container started."

                    echo ""
                    echo "Container status:"

                    docker ps \
                        --filter "name=${CONTAINER_NAME}" \
                        --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"

                    echo "========================================"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "========================================"
                    echo "Deployment Verification"
                    echo "========================================"

                    SUCCESS=0

                    for i in $(seq 1 30); do

                        if curl -fsS \
                            --max-time 5 \
                            "http://127.0.0.1:${APP_PORT}/" \
                            > /dev/null 2>&1; then

                            SUCCESS=1

                            echo ""
                            echo "SEclock application is responding."

                            break
                        fi

                        echo "Waiting for application... attempt ${i}/30"

                        sleep 2
                    done

                    if [ "${SUCCESS}" -ne 1 ]; then

                        echo ""
                        echo "ERROR: Deployment verification failed."

                        echo ""
                        echo "Container status:"

                        docker ps -a \
                            --filter "name=${CONTAINER_NAME}"

                        echo ""
                        echo "Application logs:"

                        docker logs "${CONTAINER_NAME}" || true

                        exit 1
                    fi

                    echo ""
                    echo "========================================"
                    echo "SEclock DEPLOYMENT VERIFIED"
                    echo "========================================"
                    echo ""
                    echo "Application:"
                    echo "http://EC2_PUBLIC_IP:${APP_PORT}"
                    echo ""
                    echo "Container:"
                    echo "${CONTAINER_NAME}"
                    echo ""
                    echo "Port:"
                    echo "${APP_PORT}"
                    echo ""
                    echo "========================================"
                '''
            }
        }
    }

    post {

        success {
            echo '''
========================================
       SECLOCK CI/CD SUCCESS
========================================

E2E Tests       : PASSED
Security Scan   : PASSED
Docker Build    : PASSED
Docker Test     : PASSED
ECR Login       : PASSED
ECR Push        : PASSED
EC2 Deployment  : PASSED
Verification    : PASSED

Application Port: 8000

========================================
            '''
        }

        failure {
            sh '''
                echo "========================================"
                echo "SECLOCK PIPELINE FAILED"
                echo "========================================"

                echo ""
                echo "Container status:"

                docker ps -a \
                    --filter "name=seclock" || true

                echo ""
                echo "SEclock logs:"

                docker logs seclock 2>/dev/null || true

                echo ""
                echo "========================================"
            '''

            echo '''
========================================
       SECLOCK CI/CD FAILED
========================================

Check the failed stage above.

========================================
            '''
        }

        always {
            sh '''
                echo "Cleaning temporary Docker resources..."

                docker rm -f seclock-smoke-test 2>/dev/null || true

                docker image prune -f || true
            '''
        }
    }
}
