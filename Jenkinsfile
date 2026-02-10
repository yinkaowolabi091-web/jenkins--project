pipeline {
    agent { label 'Jenkins-Agent' }
    tools {
        jdk 'java17'      // SonarQube requires Java 17
        maven 'Maven3'
    }
    environment { 
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "dreedsir12"
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }
        
    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/yinkaowolabi091-web/jenkins--project.git'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv('SonarQubeServer') {
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Build Docker Image") {
            steps {
                script {
                    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    // Run Trivy in a container to scan the built image
                    sh """
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} \
                      --no-progress \
                      --scanners vuln \
                      --exit-code 1 \
                      --severity HIGH,CRITICAL \
                      --format table
                    """
                }
            }
            post {
                always {
                    // Archive Trivy scan results for later review
                    sh """
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} \
                      --no-progress \
                      --scanners vuln \
                      --severity HIGH,CRITICAL \
                      --format json > trivy-result.json
                    """
                    archiveArtifacts artifacts: 'trivy-result.json', fingerprint: true
                }
            }
        }

        stage("Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'Dockerhub') {
                        sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                script {
                    // Remove local images to free space
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                }
            }
        }
    }

    post {
        always {
            // Final cleanup of workspace after pipeline finishes
            cleanWs()
        }
    }
}
