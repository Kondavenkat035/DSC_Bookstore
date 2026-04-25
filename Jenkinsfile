pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    environment {
        // Docker
        DOCKER_IMAGE = "bookstore"
        DOCKER_TAG   = "latest"
        CONTAINER_NAME = "bookstore"
        HOST_PORT    = "8082"
        CONTAINER_PORT = "8080"

        // SonarQube
        SONARQUBE_ENV = 'sonarqube'

        // Nexus
        NEXUS_URL  = "http://13.201.94.125:8081"
        NEXUS_REPO = "maven-releases"
        GROUP_ID   = "com/bookstore"
        ARTIFACT_ID = "bookstore"
    }

    stages {

        // 🔹 Checkout
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Kondavenkat035/DSC_Bookstore.git'
            }
        }

        // 🔹 BUILD STAGE (ONLY BUILD)
        stage('Build') {
            steps {
                dir('bookstore') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        // 🔹 SONARQUBE ANALYSIS (SEPARATE)
        stage('SonarQube Analysis') {
            steps {
                dir('bookstore') {
                    withSonarQubeEnv("${SONARQUBE_ENV}") {
                        sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=bookstore \
                        -Dsonar.projectName=bookstore
                        '''
                    }
                }
            }
        }

        // 🔹 QUALITY GATE
        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // 🔹 GET VERSION
        stage('Get Version') {
            steps {
                dir('bookstore') {
                    script {
                        env.VERSION = sh(
                            script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                            returnStdout: true
                        ).trim()
                    }
                    echo "Version: ${env.VERSION}"
                }
            }
        }

        // 🔹 UPLOAD TO NEXUS
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

                        echo "Uploading $FILE..."

                        curl -u $NEXUS_USER:$NEXUS_PASS \
                        --upload-file $FILE \
                        $NEXUS_URL/repository/$NEXUS_REPO/com/bookstore/bookstore/${VERSION}/bookstore-${VERSION}.war
                        '''
                    }
                }
            }
        }

        // 🔹 BUILD DOCKER IMAGE
        stage('Build Docker Image') {
            steps {
                dir('bookstore') {
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                }
            }
        }

        // 🔹 RUN CONTAINER
        stage('Run Docker Container') {
            steps {
                sh '''
                docker rm -f ${CONTAINER_NAME} || true

                docker run -d \
                -p ${HOST_PORT}:${CONTAINER_PORT} \
                --name ${CONTAINER_NAME} \
                ${DOCKER_IMAGE}:${DOCKER_TAG}
                '''
            }
        }
    }

    post {
        success {
            echo " Application running at: http://<your-server-ip>:${HOST_PORT}"
        }
        failure {
            echo " Pipeline failed. Check logs."
        }
    }
}
