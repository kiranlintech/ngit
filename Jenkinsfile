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
    parameters {
        choice(
            name: 'DEPLOY_TARGET',
            choices: ['HOMELAB', 'VPS'],
            description: 'Select deployment target'
        )
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

        stage('Push to Docker Hub') {
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
                            docker push ${IMAGE_NAME}:${IMAGE_TAG}
                            docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to host server') {
        steps {
            script {

                def target = params.DEPLOY_TARGET == "HOMELAB" ?
                             "ubuntu@${HOMELAB_HOST}" :
                             "ubuntu@${VPS_HOST}"

                sh """
                ssh -o StrictHostKeyChecking=no ${target} '

                    docker pull ${IMAGE_NAME}:latest

                    docker stop rudram || true
                    docker rm rudram || true

                    docker image prune -f

                    docker run -d \
                      --name rudram \
                      --restart unless-stopped \
                      -p 8082:8080 \
                      ${IMAGE_NAME}:latest
                '
                """
            }
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
