pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "kondavenkat035/dsc_bookstore"
        TAG = "${BUILD_NUMBER}"
        SONARQUBE_ENV = 'sq'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Kondavenkat035/DSC_Bookstore.git'
            }
        }

        stage('Build Maven') {
            steps {
                sh 'mvn clean package'
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
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'settings.xml', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${TAG} ."
            }
        }

        stage('Login to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                    docker tag ${DOCKER_IMAGE}:${TAG} ${DOCKER_IMAGE}:latest
                    docker push ${DOCKER_IMAGE}:${TAG}
                    docker push ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('deploy'){
            steps{
                 sh """
                 docker rm -f dsc_bookstore
                docker run -itd --name dsc_bookstore -p 8082:8080 ${DOCKER_IMAGE}:latest
                 """  
            }
        }
    }
}    


