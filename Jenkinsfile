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
        SONARQUBE_ENV = 'sonarqube'

        NEXUS_URL = "http://13.201.94.125:8081"
        NEXUS_REPO = "maven-releases"
        GROUP_ID = "com/bookstore"
        ARTIFACT_ID = "bookstore"
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
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ✅ ADD THIS STAGE (VERY IMPORTANT)
        stage('Get Version') {
            steps {
                dir('bookstore') {
                    script {
                        env.VERSION = sh(
                            script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }

        // ✅ FIXED NEXUS STAGE
        stage('Upload to Nexus') {
            steps {
                dir('bookstore') {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh '''
                        FILE=$(ls target/*.war | head -n 1)

                        curl -u $NEXUS_USER:$NEXUS_PASS \
                        --upload-file $FILE \
                        $NEXUS_URL/repository/maven-releases/com/bookstore/bookstore/${VERSION}/bookstore-${VERSION}.war
                        '''
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
