pipeline {
    agent any

    environment {
        NEXUS_URL = "43.204.37.180:8082"      // Nexus Docker port
        IMAGE_NAME = "shopping"
        REPO_NAME = "docker-hosted"
        SONAR_URL = "http://3.108.41.2:9000"
        SONAR_PROJECT_KEY = "shopping-app"
        SONAR_PROJECT_NAME = "ShoppingApp"
    }

    stages {

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
               withSonarQubeEnv('SonarQube') {   // Name configured in Jenkins
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
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-docker',
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
