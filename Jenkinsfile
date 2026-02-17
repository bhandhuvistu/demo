pipeline {
    agent any

    environment {
        NEXUS_URL = "43.204.37.180:8082"    // Nexus Docker port
        IMAGE_NAME = "shopping"
        REPO_NAME = "docker-hosted"
    }

    stages {

        stage('Git Checkout') {
            steps {
                script {
                    git branch: 'feature/changing-port-in-dockerfile', 
                        url: 'https://github.com/bhandhuvistu/demo.git'
                }
            }
        }

        stage('Maven Build') {
            steps {
                script {
                    sh 'mvn clean install'
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Docker Image Build') {
            steps {
                sh "docker image build -t $IMAGE_NAME:v.$BUILD_NUMBER ."
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
                // Use Jenkins credentials instead of hardcoding
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-docker',   // This ID must match your Jenkins credential
                    usernameVariable: 'USERNAME', 
                    passwordVariable: 'PASSWORD'
                )]) {
                    sh """
                    echo $PASSWORD | docker login $NEXUS_URL -u $USERNAME --password-stdin
                    """
                }
            }
        }

        stage('Push Image to Nexus') {
            steps {
                sh "docker push $NEXUS_URL/$REPO_NAME/$IMAGE_NAME:v.$BUILD_NUMBER"
            }
        }
    }
}
