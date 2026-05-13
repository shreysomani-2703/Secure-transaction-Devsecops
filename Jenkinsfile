pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Unit Tests') {
            steps {
                echo 'Running unit tests for all services...'
                sh '''
                    cd transaction-service
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install -r requirements.txt
                    pip install pytest
                    pytest tests/ -v --tb=short
                    cd ..
                    cd fraud-detection-service
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install -r requirements.txt
                    pip install pytest
                    pytest tests/ -v --tb=short
                    cd ..
                    cd notification-service
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install -r requirements.txt
                    pip install pytest
                    pytest tests/ -v --tb=short
                    cd ..
                '''
            }   
        }

        stage('SAST Scan (Bandit)') {
        steps {
            echo 'Running Bandit static security analysis...'

            sh '''
                set +e

                bandit -r transaction-service/app/ -f txt -o bandit-transaction.txt -ll
                bandit -r fraud-detection-service/app/ -f txt -o bandit-fraud.txt -ll
                bandit -r notification-service/app/ -f txt -o bandit-notification.txt -ll

                echo "=== Transaction Service ==="
                cat bandit-transaction.txt

                echo "=== Fraud Service ==="
                cat bandit-fraud.txt

                echo "=== Notification Service ==="
                cat bandit-notification.txt

                # Fail only on HIGH or CRITICAL
                if grep -Eq "Severity: (High|Critical)" \
                    bandit-transaction.txt \
                    bandit-fraud.txt \
                    bandit-notification.txt; then

                    echo "HIGH or CRITICAL vulnerabilities found!"
                    exit 1
                fi

                exit 0
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
                echo 'Checking dependencies for vulnerabilities...'
                sh '''
                    safety check -r transaction-service/requirements.txt --output text
                    safety check -r fraud-detection-service/requirements.txt --output text
                    safety check -r notification-service/requirements.txt --output text
                '''
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Building Docker images...'
                sh '''
                    docker compose build transaction-service fraud-detection-service notification-service frontend
                '''
            }
        }

        stage('Image Scan (Trivy)') {
            steps {
                echo 'Scanning Docker images for CVEs...'
                sh '''
                    # Fail pipeline if CRITICAL CVEs found
                    trivy image --exit-code 1 --severity CRITICAL --format table banking-devsecops-transaction-service:latest
                    trivy image --exit-code 1 --severity CRITICAL --format table banking-devsecops-fraud-detection-service:latest
                    trivy image --exit-code 1 --severity CRITICAL --format table banking-devsecops-notification-service:latest

                    # Report HIGH CVEs but do not fail
                    trivy image --exit-code 0 --severity HIGH --format table banking-devsecops-transaction-service:latest
                    trivy image --exit-code 0 --severity HIGH --format table banking-devsecops-fraud-detection-service:latest
                    trivy image --exit-code 0 --severity HIGH --format table banking-devsecops-notification-service:latest
                '''
            }
        }

        stage('Docker Push') {
            steps {
                echo 'Pushing images to DockerHub...'
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker tag banking-devsecops-transaction-service:latest $DOCKER_USER/banking-transaction-service:latest
                        docker tag banking-devsecops-transaction-service:latest $DOCKER_USER/banking-transaction-service:$IMAGE_TAG
                        docker tag banking-devsecops-fraud-detection-service:latest $DOCKER_USER/banking-fraud-detection-service:latest
                        docker tag banking-devsecops-fraud-detection-service:latest $DOCKER_USER/banking-fraud-detection-service:$IMAGE_TAG
                        docker tag banking-devsecops-notification-service:latest $DOCKER_USER/banking-notification-service:latest
                        docker tag banking-devsecops-notification-service:latest $DOCKER_USER/banking-notification-service:$IMAGE_TAG
                    '''
                    retry(3) {
                        sh 'docker push $DOCKER_USER/banking-transaction-service:latest'
                    }
                    retry(3) {
                        sh 'docker push $DOCKER_USER/banking-transaction-service:$IMAGE_TAG'
                    }
                    retry(3) {
                        sh 'docker push $DOCKER_USER/banking-fraud-detection-service:latest'
                    }
                    retry(3) {
                        sh 'docker push $DOCKER_USER/banking-fraud-detection-service:$IMAGE_TAG'
                    }
                    retry(3) {
                        sh 'docker push $DOCKER_USER/banking-notification-service:latest'
                    }
                    retry(3) {
                        sh 'docker push $DOCKER_USER/banking-notification-service:$IMAGE_TAG'
                    }
                    sh '''
                        echo "All images pushed to DockerHub"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying all services...'
                withCredentials([file(credentialsId: 'env-docker-file', variable: 'ENV_FILE')]) {
                    sh '''
                        cp $ENV_FILE .env.docker
                        docker compose --env-file .env.docker down || true
                        docker compose --env-file .env.docker up -d
                        echo "Deployment complete"
                        docker compose ps
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully — all security checks passed!'
        }
        failure {
            echo 'Pipeline FAILED — fix the issues before deploying!'
        }
        always {
            echo 'Pipeline finished'
            sh 'docker logout || true'
        }
    }
}
