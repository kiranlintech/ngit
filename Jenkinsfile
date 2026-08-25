pipeline {
    agent any

    tools {
        // Change this if your Jenkins Maven installation has a different name
        maven 'Maven3'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        APP_NAME       = 'ngit'
        IMAGE_NAME     = 'kiranlintech/ngit'
        IMAGE_TAG      = "${BUILD_NUMBER}"

        // Jenkins credentials IDs
        DOCKER_CREDS   = 'dockerhub-credentials'
        VPS_SSH        = 'vps-ssh-key'
        MYLAB_SSH      = 'mylab-ssh'
        HOMELAB_HOST = '192.168.5.9'
        VPS_HOST     = '213.210.37.106'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'git@github.com:kiranlintech/ngit.git',
                    credentialsId: 'github-ssh'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -e

                    echo "Building Docker image..."

                    docker build \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} \
                        -t ${IMAGE_NAME}:latest \
                        .
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDS}",
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
                sh '''
                    set -e

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Deploy to VPS') {
            steps {
                sshagent(credentials: ["${VPS_SSH}"]) {

                    sh '''
                        set -e

                        echo "Deploying ${APP_NAME} to VPS..."

                        ssh -o StrictHostKeyChecking=no ubuntu@VPS_HOST << EOF

                            set -e

                            echo "Logging into Docker Hub..."

                            docker pull ${IMAGE_NAME}:${IMAGE_TAG}

                            docker stop ${APP_NAME} 2>/dev/null || true
                            docker rm ${APP_NAME} 2>/dev/null || true

                            docker run -d \
                                --name ${APP_NAME} \
                                --restart unless-stopped \
                                -p 8085:8080 \
                                ${IMAGE_NAME}:${IMAGE_TAG}

                            docker image prune -f

                            echo "VPS deployment completed."

                        EOF
                    '''
                }
            }
        }

        stage('Deploy to mylab') {
            steps {
                sshagent(credentials: ["${MYLAB_SSH}"]) {

                    sh '''
                        set -e

                        echo "Deploying ${APP_NAME} to mylab..."

                        ssh -o StrictHostKeyChecking=no ubuntu@HOMELAB_HOST << EOF

                            set -e

                            echo "Pulling latest image..."

                            docker pull ${IMAGE_NAME}:${IMAGE_TAG}

                            docker stop ${APP_NAME} 2>/dev/null || true
                            docker rm ${APP_NAME} 2>/dev/null || true

                            docker run -d \
                                --name ${APP_NAME} \
                                --restart unless-stopped \
                                -p 8085:8080 \
                                ${IMAGE_NAME}:${IMAGE_TAG}

                            docker image prune -f

                            echo "mylab deployment completed."

                        EOF
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Deployment completed successfully."
                    echo "Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                '''
            }
        }
    }

    post {

        success {
            echo "========================================"
            echo "NGIT DEPLOYMENT SUCCESSFUL"
            echo "Build: ${BUILD_NUMBER}"
            echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "========================================"
        }

        failure {
            echo "========================================"
            echo "NGIT DEPLOYMENT FAILED"
            echo "Check Jenkins console output."
            echo "========================================"
        }

        always {
            sh '''
                docker logout || true
            '''
        }
    }
}
