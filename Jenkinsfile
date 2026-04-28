cat > Jenkinsfile << 'EOF'
pipeline {
    agent any

    environment {
        DOCKER_COMPOSE_FILE = 'docker-compose.yml'
        DOCKERHUB_USERNAME = 'YOUR_DOCKERHUB_USERNAME'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo '📥 Checking out source code...'
                checkout scm
            }
        }

        stage('Unit Tests') {
            steps {
                echo '🧪 Running unit tests for all services...'
                sh '''
                    cd transaction-service
                    pip install -r requirements.txt --break-system-packages -q
                    pytest tests/ -v --tb=short
                    cd ..

                    cd fraud-detection-service
                    pip install -r requirements.txt --break-system-packages -q
                    pytest tests/ -v --tb=short
                    cd ..

                    cd notification-service
                    pip install -r requirements.txt --break-system-packages -q
                    pytest tests/ -v --tb=short
                    cd ..
                '''
            }
        }

        stage('SAST Scan (Bandit)') {
            steps {
                echo '🔍 Running Bandit static security analysis...'
                sh '''
                    bandit -r transaction-service/app/ -f txt -o bandit-transaction.txt || true
                    bandit -r fraud-detection-service/app/ -f txt -o bandit-fraud.txt || true
                    bandit -r notification-service/app/ -f txt -o bandit-notification.txt || true
                    echo "=== Transaction Service ==="
                    cat bandit-transaction.txt
                    echo "=== Fraud Service ==="
                    cat bandit-fraud.txt
                    echo "=== Notification Service ==="
                    cat bandit-notification.txt
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bandit-*.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Dependency Audit (Safety)') {
            steps {
                echo '📦 Checking dependencies for vulnerabilities...'
                sh '''
                    safety check -r transaction-service/requirements.txt --output text || true
                    safety check -r fraud-detection-service/requirements.txt --output text || true
                    safety check -r notification-service/requirements.txt --output text || true
                '''
            }
        }

        stage('Docker Build') {
            steps {
                echo '🐳 Building Docker images...'
                sh '''
                    docker compose build \
                        transaction-service \
                        fraud-detection-service \
                        notification-service \
                        frontend
                '''
            }
        }

        stage('Image Scan (Trivy)') {
            steps {
                echo '🔬 Scanning Docker images for CVEs...'
                sh '''
                    trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        --format table \
                        banking-devsecops-transaction-service:latest || true

                    trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        --format table \
                        banking-devsecops-fraud-detection-service:latest || true

                    trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        --format table \
                        banking-devsecops-notification-service:latest || true
                '''
            }
        }

        stage('Docker Push') {
            steps {
                echo '📤 Pushing images to DockerHub...'
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        # Tag images with DockerHub username
                        docker tag banking-devsecops-transaction-service:latest \
                            $DOCKER_USER/banking-transaction-service:latest
                        docker tag banking-devsecops-transaction-service:latest \
                            $DOCKER_USER/banking-transaction-service:$IMAGE_TAG

                        docker tag banking-devsecops-fraud-detection-service:latest \
                            $DOCKER_USER/banking-fraud-detection-service:latest
                        docker tag banking-devsecops-fraud-detection-service:latest \
                            $DOCKER_USER/banking-fraud-detection-service:$IMAGE_TAG

                        docker tag banking-devsecops-notification-service:latest \
                            $DOCKER_USER/banking-notification-service:latest
                        docker tag banking-devsecops-notification-service:latest \
                            $DOCKER_USER/banking-notification-service:$IMAGE_TAG

                        # Push all images
                        docker push $DOCKER_USER/banking-transaction-service:latest
                        docker push $DOCKER_USER/banking-transaction-service:$IMAGE_TAG

                        docker push $DOCKER_USER/banking-fraud-detection-service:latest
                        docker push $DOCKER_USER/banking-fraud-detection-service:$IMAGE_TAG

                        docker push $DOCKER_USER/banking-notification-service:latest
                        docker push $DOCKER_USER/banking-notification-service:$IMAGE_TAG

                        echo "✅ All images pushed to DockerHub"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo '🚀 Deploying all services...'
                sh '''
                    cp /home/shrey/Documents/SPE/SPE_PROJECT\(1\)/SPE_PROJECT/banking-devsecops/.env.docker .env.docker
                    docker compose down || true
                    docker compose up -d
                    echo "✅ Deployment complete"
                    docker compose ps

                    
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed — check logs above'
        }
        always {
            echo '📊 Pipeline finished'
            sh 'docker logout || true'
        }
    }
}
EOF