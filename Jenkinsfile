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
              -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
              -t ${DOCKER_IMAGE}:${GIT_SHA_SHORT} \
              -t ${DOCKER_IMAGE}:latest \
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

                docker push ${DOCKER_IMAGE}:${GIT_SHA_SHORT}

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
                    set -e

                    echo 'Pulling new image...'

                    docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER}

                    echo 'Getting currently deployed image...'

                    CURRENT_IMAGE=\\$(docker inspect \
                        --format='{{.Config.Image}}' \
                        employee-portal 2>/dev/null || true)

                    echo \\\"Current image: \\$CURRENT_IMAGE\\\"

                    echo 'Saving previous image...'

                    echo \\\"\\$CURRENT_IMAGE\\\" > /tmp/employee-portal-previous-image

                    echo 'Removing old container...'

                    docker rm -f employee-portal || true

                    echo 'Starting new container...'

                    docker run -d \
                      --name employee-portal \
                      --restart unless-stopped \
                      -p 80:80 \
                      ${DOCKER_IMAGE}:${BUILD_NUMBER}

                    echo 'Deployment completed.'
                    "
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
                    set -e

                    echo 'Pulling new image...'

                    docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER}

                    echo 'Getting currently deployed image...'

                    CURRENT_IMAGE=\\$(docker inspect \
                        --format='{{.Config.Image}}' \
                        employee-portal 2>/dev/null || true)

                    echo \\\"Current image: \\$CURRENT_IMAGE\\\"

                    echo 'Saving previous image...'

                    echo \\\"\\$CURRENT_IMAGE\\\" > /tmp/employee-portal-previous-image

                    echo 'Removing old container...'

                    docker rm -f employee-portal || true

                    echo 'Starting new container...'

                    docker run -d \
                      --name employee-portal \
                      --restart unless-stopped \
                      -p 80:80 \
                      ${DOCKER_IMAGE}:${BUILD_NUMBER}

                    echo 'Deployment completed.'
                    "
            '''
        }
    }
}

       stage('Health Check') {
    steps {
        sshagent(['app-server-ssh']) {
            sh '''
                set +e

                echo 'Waiting for application to start...'
                sleep 5

                echo 'Running health check...'

                ssh -o StrictHostKeyChecking=no \
                    ubuntu@${APP_SERVER} \
                    "curl -f http://localhost"

                HEALTH_STATUS=$?

                if [ $HEALTH_STATUS -eq 0 ]; then

                    echo '====================================='
                    echo 'Health check PASSED'
                    echo 'Deployment successful'
                    echo '====================================='

                else

                    echo '====================================='
                    echo 'Health check FAILED'
                    echo 'Starting rollback'
                    echo '====================================='

                    ssh -o StrictHostKeyChecking=no \
                        ubuntu@${APP_SERVER} \
                        "
                        PREVIOUS_IMAGE=\\$(cat /tmp/employee-portal-previous-image)

                        echo \\\"Rolling back to: \\$PREVIOUS_IMAGE\\\"

                        docker rm -f employee-portal || true

                        docker run -d \
                          --name employee-portal \
                          --restart unless-stopped \
                          -p 80:80 \
                          \\$PREVIOUS_IMAGE
                        "

                    exit 1
                fi
            '''
        }
    }
}
