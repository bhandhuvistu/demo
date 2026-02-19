pipeline {
    agent any

    environment {
        // Nexus Docker registry (HTTP port 8082)
        REGISTRY = "13.233.225.225:8082"
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
                // Skip tests here, they run in the next stage
                sh 'mvn clean install -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
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
                    credentialsId: 'nexus-docker',  // Jenkins credentials for Nexus
                    usernameVariable: 'USERNAME', 
                    passwordVariable: 'PASSWORD'
                )]) {
                    // Secure Docker login
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
                # Configure kubectl for EKS
                aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}

                # Apply base manifests (ignore errors if already exists)
                kubectl apply -f deployment.yaml || true
                kubectl apply -f service.yaml || true

                # Update deployment with new image
                #kubectl set image deployment/shopping-app \
                  #shopping-app=${FULL_IMAGE}

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
