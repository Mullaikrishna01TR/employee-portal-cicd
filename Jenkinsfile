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
                echo 'Building Docker image now ...'

                sh '''
                    docker build \
                      -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .

                    docker tag \
                      ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                      ${DOCKER_IMAGE}:latest
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

                        docker push ${DOCKER_IMAGE}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Application Server') {
            steps {

                sshagent(['app-server-ssh']) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER} \
                            "
                            docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER} &&

                            docker rm -f employee-portal || true &&

                            docker run -d \
                              --name employee-portal \
                              -p 80:80 \
                              ${DOCKER_IMAGE}:${BUILD_NUMBER}
                            "
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {

                sh '''
                    sleep 5

                    curl -f http://51.20.70.198/
                '''
            }
        }
    }

    post {

        success {
            echo '======================================'
            echo 'Employee Portal deployment successful!'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'Employee Portal pipeline failed!'
            echo '======================================'
        }

        always {
            sh '''
                docker rm -f employee-portal-test || true
            '''
        }
    }
}
