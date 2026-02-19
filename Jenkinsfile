pipeline {
    agent any

    environment {
        // Use the Docker registry port you configured in Nexus (HTTP/HTTPS). 
        // Make sure the port matches your Docker hosted repo.
        REGISTRY = "43.204.37.180:8082"  // Changed from 8443 to 8082 if you use HTTP
        IMAGE_NAME = "shopping"
        FULL_IMAGE = "${REGISTRY}/${IMAGE_NAME}:v.${BUILD_NUMBER}"
        SONAR_PROJECT_KEY = "shopping-app"
        SONAR_PROJECT_NAME = "ShoppingApp"
        EKS_CLUSTER_NAME = "eks-cluster"
        AWS_REGION = "ap-south-1"
    }

    stages {

        stage('Clean Workspace Before Build') {
            steps {
                cleanWs()
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: 'feature/changing-port-in-dockerfile', 
                    url: 'https://github.com/bhandhuvistu/demo.git'
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean install -DskipTests'  // Skip tests here; they run in the Test stage
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // Make sure 'SonarQube' server is configured in Jenkins → Manage Jenkins → Configure System
                withSonarQubeEnv('SonarQube') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                            -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Docker Image Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:v.${BUILD_NUMBER} ."
            }
        }

        stage('Tag Image for Nexus') {
            steps {
                sh "docker tag ${IMAGE_NAME}:v.${BUILD_NUMBER} ${FULL_IMAGE}"
            }
        }

        stage('Login to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-docker',  // Jenkins credential with Nexus username/password
                    usernameVariable: 'USERNAME', 
                    passwordVariable: 'PASSWORD'
                )]) {
                    // Login using standard Docker CLI; --password-stdin is secure
                    sh "echo \$PASSWORD | docker login ${REGISTRY} -u \$USERNAME --password-stdin"
                }
            }
        }

        stage('Push Image to Nexus') {
            steps {
                sh "docker push ${FULL_IMAGE}"
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
                # Update kubeconfig for the cluster
                aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}

                # Apply manifests (ignore errors if already exists)
                kubectl apply -f deployment.yaml || true
                kubectl apply -f service.yaml || true

                # Update deployment with new image
                kubectl set image deployment/shopping-app \
                  shopping-app=${FULL_IMAGE}

                # Wait for rollout to finish
                kubectl rollout status deployment/shopping-app
                """
            }
        }
    }

    post {
        success {
            echo "Build, push, and deployment succeeded!"
            cleanWs()
        }
        failure {
            echo "Pipeline failed!"
            cleanWs()
        }
    }
}
