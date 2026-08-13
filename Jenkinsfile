pipeline {
    agent any

    options {
        skipDefaultCheckout(false)
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        DOCKER_IMAGE      = "mds81/projet-angular-docker-miage"
        DOCKERHUB_CREDS   = credentials('dockerhub-credentials')
        SONAR_TOKEN       = credentials('sonar-token')
        SLACK_WEBHOOK_URL = credentials('slack-webhook-url')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install dependencies') {
            agent { docker { image 'node:22-alpine'; reuseNode true } }
            steps {
                sh 'npm ci'
            }
        }

        stage('Unit tests') {
            agent { docker { image 'markhobson/node-chrome:22'; reuseNode true } }
            steps {
                sh 'npm test -- --no-watch --no-progress --browsers=ChromeHeadless'
            }
        }

        stage('Build') {
            agent { docker { image 'node:22-alpine'; reuseNode true } }
            steps {
                sh 'npm run build -- --configuration production'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'npx sonar-scanner -Dsonar.token=$SONAR_TOKEN'
                }
            }
        }

        stage('Docker build') {
            when { branch 'main' }
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:latest -t ${DOCKER_IMAGE}:${GIT_COMMIT} ."
            }
        }

        stage('Docker push') {
            when { branch 'main' }
            steps {
                sh 'echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin'
                sh "docker push ${DOCKER_IMAGE}:latest"
                sh "docker push ${DOCKER_IMAGE}:${GIT_COMMIT}"
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        success {
            sh """
                curl -X POST -H 'Content-type: application/json' \
                --data '{"text":":white_check_mark: Build Jenkins reussi pour ${env.JOB_NAME} #${env.BUILD_NUMBER} (commit ${env.GIT_COMMIT})"}' \
                ${SLACK_WEBHOOK_URL}
            """
        }
        failure {
            sh """
                curl -X POST -H 'Content-type: application/json' \
                --data '{"text":":x: Build Jenkins echoue pour ${env.JOB_NAME} #${env.BUILD_NUMBER} - voir ${env.BUILD_URL}"}' \
                ${SLACK_WEBHOOK_URL}
            """
        }
    }
}
