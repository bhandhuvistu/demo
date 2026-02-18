pipeline {
    agent any

    environment {
        NEXUS_URL = "43.204.37.180:8082"      // Nexus Docker registry
        IMAGE_NAME = "shopping"
        REPO_NAME = "docker-hosted"
        SONAR_PROJECT_KEY = "shopping-app"
        SONAR_PROJECT_NAME = "ShoppingApp"
        EKS_CLUSTER_NAME = "eks-cluster"      // Your EKS cluster
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
                            -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                            -Dsonar.projectName=$SONAR_PROJECT_NAME \
                            -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('Docker Image Build') {
            steps {
                sh "docker build -t $IMAGE_NAME:v.$BUILD_NUMBER ."
            }
        }

        stage('Tag Image for Nexus') {
            steps {
                sh """
                docker tag $IMAGE_NAME:v.$BUILD_NUMBER \
                $NEXUS_URL/$REPO_NAME/$IMAGE_NAME:v.$BUILD_NUMBER
                """
            }
        }

        stage('Login to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-docker',
                    usernameVariable: 'USERNAME', 
                    passwordVariable: 'PASSWORD'
                )]) {
                    sh "echo $PASSWORD | docker login $NEXUS_URL -u $USERNAME --password-stdin"
                }
            }
        }

        stage('Push Image to Nexus') {
            steps {
                sh "docker push $NEXUS_URL/$REPO_NAME/$IMAGE_NAME:v.$BUILD_NUMBER"
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
                # Configure kubeconfig for EKS cluster (IAM role used automatically)
                aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME

                # First-time deployment only
                kubectl apply -f k8s/deployment.yaml || true
                kubectl apply -f k8s/service.yaml || true

                # Update deployment image dynamically for this build
                kubectl set image deployment/shopping-app \
                shopping-app=$NEXUS_URL/$REPO_NAME/$IMAGE_NAME:v.$BUILD_NUMBER

                # Wait until deployment is rolled out
                kubectl rollout status deployment/shopping-app
                """
            }
        }
    }

    post {
        success {
            echo "Build, Docker push, and EKS deployment completed successfully!"
            cleanWs()  // Clean workspace after success
        }
        failure {
            echo "Pipeline failed!"
            cleanWs()  // Clean workspace after failure
        }
    }
}
