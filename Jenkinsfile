// Pipeline Jenkins déclaratif pour le projet Angular + Docker.
// Prérequis sur l'agent Jenkins : Node.js (plugin NodeJS), Docker CLI,
// Chrome/Chromium (pour les tests headless) et le plugin Slack Notification.
// Credentials Jenkins attendus (Manage Jenkins > Credentials) :
//   - dockerhub-credentials : Username/Password Docker Hub
//   - sonar-token           : Secret text (token SonarQube)
pipeline {
    agent any

    tools {
        nodejs 'NodeJS-22'
    }

    environment {
        IMAGE_NAME     = 'mds81/projet-angular-docker-miage'
        SONAR_HOST_URL = 'https://sonarcloud.io'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build -- --configuration production'
            }
        }

        stage('Test') {
            steps {
                sh '''
                    export CHROME_BIN=/usr/bin/chromium
                    npx ng test --no-watch --no-progress --browsers=ChromeHeadlessNoSandbox
                '''
            }
        }

        stage('SonarQube Scan') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh "npx sonar-scanner -Dsonar.token=${SONAR_TOKEN} -Dsonar.host.url=${SONAR_HOST_URL}"
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${GIT_COMMIT} -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}:${GIT_COMMIT}
                        docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }
    }

    post {
        success {
            script {
                try {
                    slackSend(channel: '#ci-cd', color: 'good',
                        message: "✅ Build réussi : ${env.JOB_NAME} #${env.BUILD_NUMBER} (${env.GIT_COMMIT})")
                } catch (err) {
                    echo "Notification Slack ignorée (non configurée) : ${err}"
                }
            }
        }
        failure {
            script {
                try {
                    slackSend(channel: '#ci-cd', color: 'danger',
                        message: "❌ Build échoué : ${env.JOB_NAME} #${env.BUILD_NUMBER}")
                } catch (err) {
                    echo "Notification Slack ignorée (non configurée) : ${err}"
                }
            }
        }
    }
}
