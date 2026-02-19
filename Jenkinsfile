pipeline {
    agent any

    environment {
        REGISTRY = "43.204.37.180:8081"        // Nexus Docker registry URL + HTTPS port
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
                sh 'mvn clean install'
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
                    credentialsId: 'nexus-docker',
                    usernameVariable: 'USERNAME', 
                    passwordVariable: 'PASSWORD'
                )]) {
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
                aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}

                # Apply base manifest
                kubectl apply -f deployment.yaml || true
                kubectl apply -f service.yaml || true

                # Update deployment to use new image tag
                kubectl set image deployment/shopping-app \
                  shopping-app=${FULL_IMAGE}

                # Wait for rollout
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
