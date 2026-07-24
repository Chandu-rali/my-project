pipeline {
    agent any

    environment {
        IMAGE_NAME = "ralichandu/myapp"
        IMAGE_TAG = "${BUILD_NUMBER}"

        DOCKER_CONTAINER = "myapp"

        EC2_HOST = "YOUR_EC2_PUBLIC_IP"
        EC2_USER = "ec2-user"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $IMAGE_NAME:$IMAGE_TAG .
                    docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
                '''
            }
        }

        stage('Push Docker Image') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                    echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

                    docker push $IMAGE_NAME:$IMAGE_TAG
                    docker push $IMAGE_NAME:latest

                    docker logout
                    '''
                }
            }
        }

        stage('Deploy') {

            steps {

                sshagent(['ec2-ssh-key']) {

                    sh '''
                    ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST << EOF

                    docker pull $IMAGE_NAME:$IMAGE_TAG

                    docker stop $DOCKER_CONTAINER || true
                    docker rm $DOCKER_CONTAINER || true

                    docker run -d \
                      --name $DOCKER_CONTAINER \
                      -p 3000:3000 \
                      $IMAGE_NAME:$IMAGE_TAG

                    EOF
                    '''
                }

            }

        }

        stage('Health Check') {

            steps {

                script {

                    sleep 20

                    def status = sh(
                        script: '''
                        curl -f http://''' + env.EC2_HOST + ''':3000
                        ''',
                        returnStatus: true
                    )

                    if(status != 0){

                        error("Deployment Failed")

                    }

                }

            }

        }

    }

    post {

        success {

            echo "Deployment Successful"

        }

        failure {

            echo "Deployment Failed"

            echo "Starting Rollback"

            sshagent(['ec2-ssh-key']) {

                sh '''
                ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST << EOF

                docker stop $DOCKER_CONTAINER || true
                docker rm $DOCKER_CONTAINER || true

                docker run -d \
                    --name $DOCKER_CONTAINER \
                    -p 3000:3000 \
                    $IMAGE_NAME:latest

                EOF
                '''

            }

        }

        always {

            cleanWs()

        }

    }

}
