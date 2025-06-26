pipeline {
    agent any
    tools {
            // This assumes the Maven tool installer plugin is configured in Jenkins
            // Go to Manage Jenkins -> Tools -> Maven Installations
            // Add a Maven installation and give it a name like 'M3'
            maven 'M3' // Replace 'M3' with the name you configure in Jenkins Global Tool Configuration
        }
    environment {
        // Replace with your Docker Hub username and image name
        DOCKER_IMAGE_NAME = "tknowledgebase/springboot-app"
        // Replace with your Spring Boot application's JAR file name if different
        JAR_FILE_NAME = "demo.jar"
        DOCKER_CRED_ID = "dockerhub-credentials" // Jenkins credential ID for Docker Hub
        GIT_CRED_ID = "git-credentials" // Jenkins credential ID for Git
        KIND_CLUSTER_NAME = "springboot-cluster" // Name of your KIND cluster
    }

    stages {
            stage('Checkout Source Code') {
                steps {
                    sh 'git checkout .' // Example, if git is needed again explicitly
                    sh "git pull origin ${env.BRANCH_NAME ?: 'main'}" // Example for pulling
                    // Often, 'git' step like below is sufficient and handled by Jenkins SCM
                    // git branch: 'main', credentialsId: "${GIT_CRED_ID}", url: 'https://github.com/your-username/your-springboot-repo.git'
                }
            }

            stage('Build Spring Boot Application') {
                steps {
                    echo 'Building Spring Boot application...'
                    // For Maven:
                    sh 'mvn clean package -DskipTests' // Changed from bat to sh
                    // For Gradle:
                    // sh 'gradlew clean build'
                }
            }

            stage('Build Docker Image') {
                steps {
                    script {
                        echo 'Building Docker image...'
                        withDockerRegistry(credentialsId: "${DOCKER_CRED_ID}", url: 'https://index.docker.io/v1/') {
                            sh "docker build -t ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} --build-arg JAR_FILE=${JAR_FILE_NAME} ." // Changed from bat to sh
                            sh "docker tag ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ${DOCKER_IMAGE_NAME}:latest" // Changed from bat to sh
                        }
                    }
                }
            }

            stage('Push Docker Image') {
                steps {
                    script {
                        echo 'Pushing Docker image to Docker Hub...'
                        withDockerRegistry(credentialsId: "${DOCKER_CRED_ID}", url: 'https://index.docker.io/v1/') {
                            sh "docker push ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}" // Changed from bat to sh
                            sh "docker push ${DOCKER_IMAGE_NAME}:latest" // Changed from bat to sh
                        }
                    }
                }
            }

            stage('Deploy to KIND') {
                steps {
                    script {
                        echo 'Deploying to KIND cluster...'
                        sh "kubectl config use-context kind-${KIND_CLUSTER_NAME}" // Changed from bat to sh
                        sh "kubectl apply -f kubernetes/deployment.yaml" // Changed from bat to sh
                        sh "kubectl apply -f kubernetes/service.yaml" // Changed from bat to sh

                        echo "Deployment to KIND cluster: ${KIND_CLUSTER_NAME} initiated."
                        echo "You can verify the deployment status using: 'kubectl get pods' and 'kubectl get service springboot-app-service'."
                    }
                }
            }

            stage('Cleanup') {
                steps {
                    echo 'Cleaning workspace...'
                    cleanWs()
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