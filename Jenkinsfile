pipeline {
    agent any

    environment {
        APP_NAME = "nodetodo-app"
        DOCKERHUB_USERNAME = "19simran"
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/${APP_NAME}"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    options {
        timestamps()
        ansiColor('xterm')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(
            numToKeepStr: '10',
            artifactNumToKeepStr: '5'
        ))
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Verify Environment') {
            steps {
                sh '''
                    echo "Checking installed tools..."
                    node --version
                    npm --version
                    docker --version
                    trivy --version
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    .
            '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def dockerImage = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                        -u "$DOCKER_USERNAME" \
                        --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                    docker push ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy Locally') {
            steps {
                sh '''
                    docker stop ${APP_NAME} || true
                    docker rm ${APP_NAME} || true

                    docker run -d \
                        --name ${APP_NAME} \
                        -p 3000:3000 \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }
    }

    post {

    always {
        sh 'docker image prune -f || true'
        cleanWs()
    }

    success {
        echo "Pipeline completed successfully."

        slackSend(
            channel: '#devops-alerts',
            color: "good",
            message: """
✅ *Build Successful*

*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Image:* ${IMAGE_NAME}:${IMAGE_TAG}
*URL:* ${env.BUILD_URL}
"""
        )
    }

    failure {
        echo "Pipeline failed."

        slackSend(
            channel: "#devops-alerts",
            color: "danger",
            message: """
❌ *Build Failed*

*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Check Logs:* ${env.BUILD_URL}
"""
        )
    }

    unstable {
        slackSend(
            channel: "#devops-alerts",
            color: "warning",
            message: """
⚠️ *Build Unstable*

*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*URL:* ${env.BUILD_URL}
"""
        )
    }
}

}