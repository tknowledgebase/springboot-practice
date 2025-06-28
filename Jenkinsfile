pipeline {
    agent any

    environment {
        DOCKER_HUB_USERNAME = 'tknowledgebase'
        APP_NAME = 'my-spring-boot-app'
        KIND_CLUSTER_NAME = 'springboot-cluster'
        DOCKER_HOST = 'unix:///var/run/docker.sock'
        DOCKER_TLS_VERIFY = '0'
        DOCKER_CERT_PATH = ''
    }

    stages {
        stage('Checkout') { /* ... */ }

        stage('Build Spring Boot Application') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Debug Docker Environment') { // <<< NEW STAGE FOR DEBUGGING
            steps {
                sh 'env | grep -i docker || true' // List all env vars, filter for docker-related, || true to prevent failure if grep fails
                sh 'echo "DOCKER_HOST: ${DOCKER_HOST}"'
                sh 'echo "DOCKER_TLS_VERIFY: ${DOCKER_TLS_VERIFY}"'
                sh 'echo "DOCKER_CERT_PATH: ${DOCKER_CERT_PATH}"'
                sh 'ls -l /var/run/docker.sock || true' // Check if socket exists and permissions
                sh 'docker info || true' // Attempt docker info to see connection status
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    // This is where the error occurs
                    sh "docker build -t ${DOCKER_HUB_USERNAME}/${APP_NAME}:${env.BUILD_NUMBER} ."
                    sh "docker build -t ${DOCKER_HUB_USERNAME}/${APP_NAME}:latest ."

                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                        sh "docker push ${DOCKER_HUB_USERNAME}/${APP_NAME}:${env.BUILD_NUMBER}"
                        sh "docker push ${DOCKER_HUB_USERNAME}/${APP_NAME}:latest"
                        sh "docker logout"
                    }
                }
            }
        }

        stage('Deploy to KIND') {
            steps {
                script {
                    // Create KIND cluster if it doesn't exist
                    try {
                        sh "kind get clusters | findstr /i /c:\"${KIND_CLUSTER_NAME}\""
                    } catch (Exception e) {
                        echo "KIND cluster ${KIND_CLUSTER_NAME} does not exist, creating it..."
                        sh "kind create cluster --name ${KIND_CLUSTER_NAME}"
                    }

                    // Export Kubeconfig to make it accessible to kubectl inside the Jenkins container
                    sh "kind get kubeconfig --name ${KIND_CLUSTER_NAME} > kubeconfig-${KIND_CLUSTER_NAME}.yaml"
                    env.KUBECONFIG = "kubeconfig-${KIND_CLUSTER_NAME}.yaml" // Set KUBECONFIG for current stage

                    // Apply Kubernetes deployment
                    sh "kubectl apply -f springboot-deployment.yaml"
                    sh "kubectl apply -f springboot-service.yaml"

                    // Wait for deployment to be ready (optional, but good practice)
                    sh "kubectl rollout status spring-boot-app-deployment --timeout=300s"

                    // (Optional) Get service URL for NodePort
                    echo "Access your application using:"
                    sh "kubectl get service spring-boot-app-service -o jsonpath='{.spec.ports[0].nodePort}'"
                    sh "kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type==\"InternalIP\")].address}'"
                    echo "Combine the Node IP and NodePort to access your application."
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished."
            // Clean up workspace if needed
            // deleteDir()
        }
        failure {
            echo "Pipeline failed! Check logs for details."
        }
    }
}