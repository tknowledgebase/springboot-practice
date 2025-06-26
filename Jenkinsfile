pipeline {
    agent any
    environment {
        // Replace with your Docker Hub username and image name
        DOCKER_IMAGE_NAME = "your-dockerhub-username/springboot-app"
        // Replace with your Spring Boot application's JAR file name if different
        JAR_FILE_NAME = "your-spring-boot-app.jar"
        DOCKER_CRED_ID = "dockerhub-credentials" // Jenkins credential ID for Docker Hub
        GIT_CRED_ID = "github-credentials" // Jenkins credential ID for Git
        KIND_CLUSTER_NAME = "springboot-cluster" // Name of your KIND cluster
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                git branch: 'main', credentialsId: "${GIT_CRED_ID}", url: 'https://github.com/your-username/your-springboot-repo.git'
            }
        }

        stage('Build Spring Boot Application') {
            steps {
                // Assuming Maven. For Gradle, use `bat 'gradlew clean build'`
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Use Docker Desktop's Docker daemon
                    withDockerRegistry(credentialsId: "${DOCKER_CRED_ID}", url: 'https://index.docker.io/v1/') {
                        bat "docker build -t ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ."
                        bat "docker tag ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ${DOCKER_IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: "${DOCKER_CRED_ID}", url: 'https://index.docker.io/v1/') {
                        bat "docker push ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                        bat "docker push ${DOCKER_IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to KIND') {
            steps {
                script {
                    // Ensure kubectl context is set to KIND cluster
                    bat "kubectl config use-context kind-${KIND_CLUSTER_NAME}"

                    // Apply Kubernetes deployment and service manifests
                    // Adjust paths if your YAMLs are in a subdirectory (e.g., kubernetes/deployment.yaml)
                    bat "kubectl apply -f deployment.yaml"
                    bat "kubectl apply -f service.yaml"

                    echo "Deployed to KIND cluster: ${KIND_CLUSTER_NAME}"
                    echo "You can check the deployment with: kubectl get pods -n default"
                    echo "And access the service by running: kubectl get service springboot-app-service -n default"
                    echo "For NodePort, you can find the port by running 'kubectl get service springboot-app-service -n default' and typically access it via localhost:<NodePort>"
                }
            }
        }

        stage('Cleanup') {
            steps {
                cleanWs() // Clean the workspace
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}