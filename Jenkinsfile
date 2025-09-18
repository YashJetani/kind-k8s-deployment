pipeline {
    agent any
    
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        DOCKER_IMAGE_NAME = 'yourdockerhubusername/nginx-devops'
        DOCKER_TAG = "${BUILD_NUMBER}"
        GITHUB_CREDENTIALS = credentials('github-credentials')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Checked out code from ${env.GIT_BRANCH}"
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE_NAME}:${DOCKER_TAG}"
                    docker.build("${DOCKER_IMAGE_NAME}:${DOCKER_TAG}")
                    docker.build("${DOCKER_IMAGE_NAME}:latest")
                }
            }
        }
        
        stage('Test Docker Image') {
            steps {
                script {
                    echo "Testing Docker image..."
                    sh """
                        docker run --rm -d --name nginx-test -p 8081:80 ${DOCKER_IMAGE_NAME}:${DOCKER_TAG}
                        sleep 10
                        curl -f http://localhost:8081/health || exit 1
                        docker stop nginx-test
                    """
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo "Pushing image to Docker Hub..."
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                        docker.image("${DOCKER_IMAGE_NAME}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_IMAGE_NAME}:latest").push()
                    }
                }
            }
        }
        
        stage('Update Kubernetes Manifest') {
            steps {
                script {
                    echo "Updating Kubernetes deployment manifest..."
                    sh """
                        sed -i 's|image: ${DOCKER_IMAGE_NAME}:.*|image: ${DOCKER_IMAGE_NAME}:${DOCKER_TAG}|' k8s/deployment.yaml
                        git config user.name "Jenkins"
                        git config user.email "jenkins@yourdomain.com"
                        git add k8s/deployment.yaml
                        git commit -m "Update image tag to ${DOCKER_TAG}" || exit 0
                        git push origin HEAD:main
                    """
                }
            }
        }
        
        stage('Trigger ArgoCD Sync') {
            steps {
                script {
                    echo "Triggering ArgoCD synchronization..."
                    // ArgoCD will automatically detect changes and deploy
                    sh """
                        echo "ArgoCD will automatically sync the changes within 3 minutes"
                        echo "Or you can manually sync via ArgoCD UI"
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo "Cleaning up..."
            sh 'docker system prune -f'
        }
        success {
            echo "Pipeline completed successfully!"
            echo "New image: ${DOCKER_IMAGE_NAME}:${DOCKER_TAG}"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}