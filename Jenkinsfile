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

        stage('Deploy to Kubernetes (EKS)') {
            steps {
                withCredentials([file(credentialsId: 'k8s-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG
                
                        echo "Deploying to EKS..."

                        kubectl apply -f k8s/deployment.yml
                        kubectl apply -f k8s/service.yml

                        echo "Deployment status:"
                        kubectl get pods
                        kubectl get svc
                    '''
                }
            }
        }

        stage('Show Application URL') {
            steps {
                withCredentials([file(credentialsId: 'k8s-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG
                        echo "Fetching LoadBalancer URL..."

                        for i in {1..2}; do
                            URL=$(kubectl get svc bookstore-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)

                            if [ "$URL" != "" ]; then
                                echo "========================================"
                                echo "Application Deployed Successfully 🚀"
                                echo "URL: http://$URL"
                                echo "========================================"
                                exit 0
                            fi

                            echo "Waiting for LoadBalancer... attempt $i/20"
                            sleep 5
                        done

                        echo "ERROR: LoadBalancer not ready. Check AWS console."
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "PIPELINE SUCCESS 🚀"
        }
        failure {
            echo "PIPELINE FAILED ❌"
        }
    }
}


