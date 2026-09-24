pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "mullaikrishna/employee-portal-cicd"

        APP_SERVER_1 = "172.31.18.62"
        APP_SERVER_2 = "172.31.40.2"
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
                      -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Test Docker Image') {
            steps {

                sh '''
                    docker rm -f employee-portal-test || true

                    docker run -d \
                      --name employee-portal-test \
                      -p 8082:80 \
                      ${DOCKER_IMAGE}:${BUILD_NUMBER}

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

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Application Server 1') {
            steps {

                sshagent(['app-server-ssh']) {

                    sh '''
                        echo "Deploying build ${BUILD_NUMBER} to App Server 1..."

                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER_1} << EOF

                        set -e

                        echo "Pulling new image..."

                        docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER}

                        echo "Removing old container..."

                        docker rm -f employee-portal || true

                        echo "Starting new container..."

                        docker run -d \
                          --name employee-portal \
                          --restart unless-stopped \
                          -p 80:80 \
                          ${DOCKER_IMAGE}:${BUILD_NUMBER}

                        echo "Waiting for application..."

                        sleep 5

                        echo "Running health check..."

                        curl -f http://localhost

                        echo "App Server 1 deployment successful."

                        EOF
                    '''
                }
            }
        }

        stage('Deploy to Application Server 2') {
            steps {

                sshagent(['app-server-ssh']) {

                    sh '''
                        echo "Deploying build ${BUILD_NUMBER} to App Server 2..."

                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER_2} << EOF

                        set -e

                        echo "Pulling new image..."

                        docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER}

                        echo "Removing old container..."

                        docker rm -f employee-portal || true

                        echo "Starting new container..."

                        docker run -d \
                          --name employee-portal \
                          --restart unless-stopped \
                          -p 80:80 \
                          ${DOCKER_IMAGE}:${BUILD_NUMBER}

                        echo "Waiting for application..."

                        sleep 5

                        echo "Running health check..."

                        curl -f http://localhost

                        echo "App Server 2 deployment successful."

                        EOF
                    '''
                }
            }
        }
    }

    post {

        success {
            echo 'Employee Portal rolling deployment completed successfully!'
        }

        failure {
            echo 'Employee Portal deployment failed.'
        }

        always {
            sh '''
                docker rm -f employee-portal-test || true
            '''
        }
    }
}
