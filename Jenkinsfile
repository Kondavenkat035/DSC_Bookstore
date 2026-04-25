pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    environment {
        DOCKER_IMAGE = "bookstore"
        DOCKER_TAG   = "latest"
        CONTAINER_NAME = "bookstore"
        HOST_PORT    = "8082"
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

        stage('Upload to Nexus') {
            steps {
                dir('bookstore') {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh """
                        JAR_FILE=\$(ls target/*.war | head -n 1)

                        curl -u $NEXUS_USER:$NEXUS_PASS \
                        --upload-file \$JAR_FILE \
                        ${NEXUS_URL}/repository/${NEXUS_REPO}/${GROUP_ID}/${ARTIFACT_ID}/${VERSION}/${ARTIFACT_ID}-${VERSION}.jar
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('bookstore') {
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                }
            }
        }

        stage('Run Docker Container') {
            steps {
                script {
                    sh """
                        docker ps -q --filter "name=${CONTAINER_NAME}" | grep -q . && \
                        docker stop ${CONTAINER_NAME} && \
                        docker rm ${CONTAINER_NAME} || true

                        docker run -d --name ${CONTAINER_NAME} -p ${HOST_PORT}:${CONTAINER_PORT} ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "App running at http://localhost:${HOST_PORT}"
        }
    }
}
