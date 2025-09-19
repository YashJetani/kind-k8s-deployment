pipeline {
    agent any
    
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        DOCKER_IMAGE_NAME = 'yashjetani1408/nginx-devops'
        DOCKER_TAG = "${BUILD_NUMBER}"
        KIND_CLUSTER = 'devops-project'
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
                    sh """
                        docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_TAG} .
                        docker build -t ${DOCKER_IMAGE_NAME}:latest .
                    """
                }
            }
        }
        
        stage('Load Image to Kind') {
            steps {
                script {
                    echo "Loading image to Kind cluster..."
                    sh """
                        kind load docker-image ${DOCKER_IMAGE_NAME}:${DOCKER_TAG} --name ${KIND_CLUSTER}
                        kind load docker-image ${DOCKER_IMAGE_NAME}:latest --name ${KIND_CLUSTER}
                    """
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo "Pushing image to Docker Hub..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh """
                            echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin
                            docker push ${DOCKER_IMAGE_NAME}:${DOCKER_TAG}
                            docker push ${DOCKER_IMAGE_NAME}:latest
                        """
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
                        
                        # Ensure ImagePullPolicy is Never for Kind
                        if ! grep -q "imagePullPolicy: Never" k8s/deployment.yaml; then
                            sed -i '/image: ${DOCKER_IMAGE_NAME}:${DOCKER_TAG}/a\\        imagePullPolicy: Never' k8s/deployment.yaml
                        fi
                        
                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"
                        git add k8s/deployment.yaml
                        git commit -m "Update image tag to ${DOCKER_TAG}" || exit 0
                        git push origin HEAD:main
                    """
                }
            }
        }
        
        stage('Deploy to Kind') {
            steps {
                script {
                    echo "Deploying to Kind cluster..."
                    sh """
                        kubectl apply -f k8s/namespace.yaml
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        
                        # Wait for deployment
                        kubectl rollout status deployment/nginx-deployment -n nginx-app --timeout=300s
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo "Cleaning up..."
            sh 'docker system prune -f || true'
        }
        success {
            script {
                sh '''
                    PUBLIC_IP=$(curl -s http://checkip.amazonaws.com)
                    echo "=========================================="
                    echo "Deployment Successful!"
                    echo "Application URL: http://$PUBLIC_IP:30080"
                    echo "ArgoCD URL: https://$PUBLIC_IP:30443"
                    echo "New image: ''' + DOCKER_IMAGE_NAME + ''':''' + DOCKER_TAG + '''"
                    echo "=========================================="
                '''
            }
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
