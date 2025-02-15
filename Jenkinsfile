pipeline {
    agent any
    
    environment {
        GIT_REPO = 'https://github.com/amulya0000/trivy-upgrad.git'
        GIT_BRANCH = 'main'
        DOCKER_REGISTRY = 'localhost:5010'  // Updated to use port 5010
        IMAGE_NAME = 'myimage'
        IMAGE_TAG = "${BUILD_NUMBER}"       // Using build number for versioning
        DOCKERFILE_PATH = 'Dockerfile'
    }
    
    stages {
        stage('Checkout') {
            steps {
                cleanWs()  // Clean workspace before build
                git branch: "${GIT_BRANCH}", 
                    url: "${GIT_REPO}"
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image
                    sh """
                        docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                        -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest \
                        -f ${DOCKERFILE_PATH} .
                    """
                }
            }
        }
        
        stage('Push to Registry') {
            steps {
                script {
                    // Push both tagged and latest versions
                    sh """
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }
        
        stage('Trivy Scan') {
            steps {
                script {
                    try {
                        // Run Trivy scan and output results to a file
                        sh """
                            trivy image \
                            --exit-code 1 \
                            --severity HIGH,CRITICAL \
                            --no-progress \
                            --format table \
                            --output trivy_report.txt \
                            ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        """
                    } catch (err) {
                        currentBuild.result = 'UNSTABLE'
                        error "Trivy scan found HIGH/CRITICAL vulnerabilities. Check trivy_report.txt for details."
                    }
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                expression { 
                    return currentBuild.result == null || currentBuild.result == 'SUCCESS'
                }
            }
            steps {
                script {
                    try {
                        // Stop and remove existing container if it exists
                        sh 'docker rm -f myapp || true'
                        
                        // Deploy new container
                        sh """
                            docker run -d \
                            --name myapp \
                            --restart unless-stopped \
                            -p 8000:8000 \
                            ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        """
                        
                        // Verify container is running
                        sh 'docker ps | grep myapp'
                    } catch (err) {
                        error "Deployment failed: ${err.message}"
                    }
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                script {
                    // Remove old images to free up space
                    sh """
                        docker image prune -a --force --filter "until=24h"
                        docker container prune --force --filter "until=24h"
                    """
                }
            }
        }
    }
    
    post {
        always {
            // Archive the Trivy scan report
            archiveArtifacts artifacts: 'trivy_report.txt', 
                           fingerprint: true, 
                           allowEmptyArchive: true
        }
        success {
            echo "Pipeline completed successfully!"
        }
        unstable {
            echo "Pipeline is unstable, likely due to security vulnerabilities."
        }
        failure {
            echo "Pipeline failed! Check the logs for details."
        }
        cleanup {
            // Clean up workspace
            cleanWs(cleanWhenNotBuilt: false,
                   deleteDirs: true,
                   disableDeferredWipeout: true,
                   notFailBuild: true,
                   patterns: [[pattern: 'trivy_report.txt', type: 'INCLUDE']])
        }
    }
}
