pipeline {
    agent any

    environment {
        IMAGE_NAME     = 'two-tier-flask-app'
        IMAGE_TAG      = "${BUILD_NUMBER}"
        SONAR_HOST_URL = 'http://172.17.0.1:9000'
    }

    stages {
        stage('1. Git Checkout') {
            steps {
                echo '=== Stage 1: Fetching Source Code ==='
                checkout scm
            }
        }

        stage('2. Trivy Filesystem & Secrets Scan') {
            steps {
                echo '=== Stage 2: Scanning Filesystem & Secrets via Trivy ==='
                sh 'trivy fs --severity HIGH,CRITICAL --exit-code 0 .'
            }
        }

        stage('3. SonarQube Code Quality Analysis') {
            steps {
                echo '=== Stage 3: Running SonarQube Analysis ==='
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        docker run --rm \
                        -e SONAR_HOST_URL="${SONAR_HOST_URL}" \
                        -e SONAR_TOKEN="${SONAR_TOKEN}" \
                        -v "${WORKSPACE}:/usr/src" \
                        sonarsource/sonar-scanner-cli
                    """
                }
            }
        }

        stage('4. SonarQube Quality Gate') {
            steps {
                echo '=== Stage 4: Waiting for Quality Gate Status ==='
                timeout(time: 2, unit: 'MINUTES') {
                    script {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('5. Build Docker Image') {
            steps {
                echo '=== Stage 5: Building Docker Image ==='
                withCredentials([usernamePassword(credentialsId: 'docker-key', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} ."
                    sh "docker tag ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_USER}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('6. Trivy Docker Image Scan') {
            steps {
                echo '=== Stage 6: Scanning Docker Image Vulnerabilities ==='
                withCredentials([usernamePassword(credentialsId: 'docker-key', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "trivy image --severity HIGH,CRITICAL --exit-code 0 ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('7. Push Image to Docker Hub') {
            steps {
                echo '=== Stage 7: Pushing Image to Docker Hub ==='
                withCredentials([usernamePassword(credentialsId: 'docker-key', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_USER}/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('8. Deploy via Docker Compose') {
            steps {
                echo '=== Stage 8: Deploying Application via Docker Compose ==='
                withCredentials([usernamePassword(credentialsId: 'docker-key', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "DOCKER_HUB_USER=${DOCKER_USER} docker compose pull"
                    sh "DOCKER_HUB_USER=${DOCKER_USER} docker compose up -d --remove-orphans"
                }
            }
        }
    }

    post {
        always {
            withCredentials([usernamePassword(credentialsId: 'docker-key', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                sh "docker rmi ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} || true"
            }
        }
        success {
            echo ' SUCCESS: Flask Application Deployed and Running Successfully!'
        }
        failure {
            echo ' FAILURE: Pipeline Executed with Errors.'
        }
    }
}
