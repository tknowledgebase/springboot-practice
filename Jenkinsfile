pipeline {
    agent any // Or agent { docker { image 'maven:3.8.5-openjdk-17' } } if you want a dedicated build agent image

    environment {
        // Replace with your Docker Hub username
        DOCKER_HUB_USERNAME = 'tknowledgebase'
        APP_NAME = 'my-spring-boot-app'
        KIND_CLUSTER_NAME = 'springboot-cluster' // Name of your KIND cluster
        DOCKER_HOST = 'unix:///var/run/docker.sock'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/tknowledgebase/springboot-practice.git'
                // Ensure your Git credentials are set up in Jenkins if your repo is private
            }
        }

        stage('Build Spring Boot Application') {
                    steps {
                        // Assuming Maven. For Gradle, use `bat 'gradlew clean build'`
                        sh  'mvn clean package -DskipTests'
                    }
                }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    // Build the Docker image
                    // The 'docker' commands here will use the DOCKER_HOST set above.
                    sh "docker build -t ${DOCKER_HUB_USERNAME}/${APP_NAME}:${env.BUILD_NUMBER} ."
                    sh "docker build -t ${DOCKER_HUB_USERNAME}/${APP_NAME}:latest ."

                    // Push to Docker Hub (ensure Docker Hub credentials are configured in Jenkins)
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