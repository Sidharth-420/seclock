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

                    echo "===== Environment Check ====="

                    python3 --version
                    docker --version
                    aws --version
                    git --version

                    echo "Workspace:"
                    pwd

                    echo "Project files:"
                    ls -la
                '''
            }
        }

        stage('Python Setup') {
            steps {
                sh '''
                    set -eu

                    rm -rf venv

                    python3 -m venv venv

                    . venv/bin/activate

                    python -m pip install --upgrade pip

                    python -m pip install -r requirements.txt

                    python -m pip install pytest bandit httpx2

                    echo "Python setup completed."
                '''
            }
        }

        stage('Application Import Check') {
            steps {
                sh '''
                    set -eu

                    . venv/bin/activate

                    python -c "from main import app; print('FastAPI application imported successfully')"
                '''
            }
        }

        stage('E2E Tests') {
            steps {
                sh '''
                    set -eu

                    . venv/bin/activate

                    echo "===== Running E2E Tests ====="

                    python test_e2e.py

                    echo "E2E tests completed successfully."
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    set -eu

                    . venv/bin/activate

                    echo "===== Running Bandit ====="

                    bandit \
                        -r main.py \
                        crypto_engine.py \
                        ocr_engine.py \
                        audit_ledger.py \
                        -f txt

                    echo "Security scan completed."
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -eu

                    echo "===== Building Docker Image ====="

                    docker build \
                        --pull \
                        -t "${IMAGE_NAME}" \
                        -t "${LATEST_IMAGE}" \
                        .

                    echo "Docker build completed."

                    docker images "${ECR_REGISTRY}/${ECR_REPOSITORY}"
                '''
            }
        }

        stage('Docker Smoke Test') {
            steps {
                sh '''
                    set -eu

                    echo "===== Docker Smoke Test ====="

                    TEST_CONTAINER="seclock-smoke-test"

                    docker rm -f "${TEST_CONTAINER}" 2>/dev/null || true

                    docker run -d \
                        --name "${TEST_CONTAINER}" \
                        -p 18000:8000 \
                        "${IMAGE_NAME}"

                    SUCCESS=0

                    for i in $(seq 1 30); do

                        if curl -fsS \
                            --max-time 3 \
                            "http://127.0.0.1:18000/" \
                            > /dev/null 2>&1; then

                            SUCCESS=1
                            echo "Docker application is responding."
                            break
                        fi

                        echo "Waiting for application... ${i}/30"
                        sleep 2
                    done

                    if [ "${SUCCESS}" -ne 1 ]; then

                        echo "ERROR: Docker application failed to start."

                        echo "Container status:"
                        docker ps -a \
                            --filter "name=${TEST_CONTAINER}"

                        echo "Container logs:"
                        docker logs "${TEST_CONTAINER}" || true

                        docker rm -f "${TEST_CONTAINER}" || true

                        exit 1
                    fi

                    echo "Docker smoke test passed."

                    docker logs "${TEST_CONTAINER}" || true

                    docker rm -f "${TEST_CONTAINER}" || true
                '''
            }
        }

        stage('AWS Check') {
            steps {
                sh '''
                    set -eu

                    echo "===== AWS Check ====="

                    aws sts get-caller-identity

                    aws ecr describe-repositories \
                        --repository-names "${ECR_REPOSITORY}" \
                        --region "${AWS_REGION}" \
                        > /dev/null

                    echo "ECR repository exists."
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    set -eu

                    echo "===== ECR Login ====="

                    aws ecr get-login-password \
                        --region "${AWS_REGION}" | \
                    docker login \
                        --username AWS \
                        --password-stdin "${ECR_REGISTRY}"

                    echo "ECR login successful."
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    set -eu

                    echo "===== Push Docker Image ====="

                    docker push "${IMAGE_NAME}"

                    docker push "${LATEST_IMAGE}"

                    echo "Docker images pushed successfully."
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    set -eu

                    echo "===== Deploying SEclock ====="

                    docker pull "${LATEST_IMAGE}"

                    docker stop "${CONTAINER_NAME}" 2>/dev/null || true

                    docker rm "${CONTAINER_NAME}" 2>/dev/null || true

                    docker run -d \
                        --name "${CONTAINER_NAME}" \
                        --restart unless-stopped \
                        -p "${APP_PORT}:${APP_PORT}" \
                        "${LATEST_IMAGE}"

                    echo "Container started."

                    docker ps \
                        --filter "name=${CONTAINER_NAME}" \
                        --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "===== Verifying Application ====="

                    SUCCESS=0

                    for i in $(seq 1 30); do

                        if curl -fsS \
                            --max-time 5 \
                            "http://127.0.0.1:${APP_PORT}/" \
                            > /dev/null 2>&1; then

                            SUCCESS=1

                            echo "SEclock application is running."

                            break
                        fi

                        echo "Waiting for application... ${i}/30"
                        sleep 2
                    done

                    if [ "${SUCCESS}" -ne 1 ]; then

                        echo "ERROR: Application verification failed."

                        echo "Container status:"
                        docker ps -a \
                            --filter "name=${CONTAINER_NAME}"

                        echo "Application logs:"
                        docker logs "${CONTAINER_NAME}" || true

                        exit 1
                    fi

                    echo "Application verified successfully."
                    echo "Port: ${APP_PORT}"
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
ECR Push        : PASSED
EC2 Deployment  : PASSED
Verification    : PASSED

Application Port: 8000

========================================
            '''
        }

        failure {
            sh '''
                echo "===== PIPELINE FAILED ====="

                docker ps -a \
                    --filter "name=seclock" || true

                docker logs seclock 2>/dev/null || true
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
                docker rm -f seclock-smoke-test 2>/dev/null || true
                docker image prune -f || true
            '''
        }
    }
}
