pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        DOCKER_IMAGE = "bookstore"
        DOCKER_TAG   = "latest"
        CONTAINER_NAME = "bookstore"
        HOST_PORT    = "8081"
        CONTAINER_PORT = "8080"
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Avnish-web/DSC_Bookstore.git'
            }
        }
        stage('Build with Maven') {
            steps {
                dir('bookstore') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                dir('bookstore') {
                    sh "sudo docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                }
            }
        }
        stage('Run Docker Container') {
            steps {
                script {
                    sh """
                        sudo docker ps -q --filter "name=${CONTAINER_NAME}" | grep -q . && \
                        sudo docker stop ${CONTAINER_NAME} && \
                        sudo docker rm ${CONTAINER_NAME} || true
                        sudo docker run -d --name ${CONTAINER_NAME} -p ${HOST_PORT}:${CONTAINER_PORT} ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }
    }
    post {
        success {
            echo "Deployment Successful! App running at http://localhost:${HOST_PORT}"
        }
        failure {
            echo "Pipeline failed. Check logs."
        }
    }
}
