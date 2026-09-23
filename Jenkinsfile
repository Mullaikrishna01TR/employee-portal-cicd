pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "mullaikrishna/employee-portal-cicd"
        APP_SERVER = "172.31.18.62"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'

                git branch: 'main',
                    url: 'https://github.com/Mullaikrishna01TR/employee-portal-cicd.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.GIT_SHA_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Git Commit: ${env.GIT_SHA_SHORT}"
                    echo "Jenkins Build: ${env.BUILD_NUMBER}"
                }

                sh '''
                    docker build \
                      -t ${DOCKER_IMAGE}:${GIT_SHA_SHORT} \
                      .
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                echo 'Testing Docker image...'

                sh '''
                    docker rm -f employee-portal-test || true

                    docker run -d \
                      --name employee-portal-test \
                      -p 8082:80 \
                      ${DOCKER_IMAGE}:${GIT_SHA_SHORT}

                    sleep 5

                    curl -f http://localhost:8082

                    docker rm -f employee-portal-test
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                          -u "$DOCKER_USERNAME" \
                          --password-stdin

                        docker push ${DOCKER_IMAGE}:${GIT_SHA_SHORT}

                        docker logout
                    '''
                }
            }
        }

        stage('Save Previous Version') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER} \
                            "docker inspect \
                              --format='{{.Config.Image}}' \
                              employee-portal \
                              2>/dev/null \
                              > /tmp/employee-portal-previous-image \
                              || true"
                    '''
                }
            }
        }

        stage('Deploy to Application Server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh '''
                        echo 'Pulling new image...'

                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER} \
                            "docker pull ${DOCKER_IMAGE}:${GIT_SHA_SHORT}"

                        echo 'Removing old container...'

                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER} \
                            "docker rm -f employee-portal || true"

                        echo 'Starting new container...'

                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER} \
                            "docker run -d \
                              --name employee-portal \
                              --restart unless-stopped \
                              -p 80:80 \
                              ${DOCKER_IMAGE}:${GIT_SHA_SHORT}"

                        echo 'Deployment completed.'
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh '''
                        echo 'Waiting for application to start...'
                        sleep 5

                        echo 'Running health check...'

                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER} \
                            "curl -f http://localhost"
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Employee Portal pipeline completed successfully!'
        }

        failure {
            echo 'Employee Portal pipeline failed.'
        }

        always {
            sh '''
                docker rm -f employee-portal-test || true
            '''
        }
    }
}
