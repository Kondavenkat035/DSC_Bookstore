pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "kondavenkat035/dsc_bookstore"
        TAG = "${BUILD_NUMBER}"
        SONARQUBE_ENV = 'sonarqube'
        AWS_DEFAULT_REGION = "us-east-1"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Kondavenkat035/DSC_Bookstore.git'
            }
        }
        stage('Inntall') {
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
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('JENKINS TO NEXUS') {
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
                    credentialsId: 'docker id',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                sh """
                docker push ${DOCKER_IMAGE}:${TAG}
                docker tag ${DOCKER_IMAGE}:${TAG} ${DOCKER_IMAGE}:latest
                docker push ${DOCKER_IMAGE}:latest
                """
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name new-cluster

                kubectl apply -f k8s/deployment.yml
                kubectl apply -f k8s/service.yml
                kubectl apply -f k8s/ingress.yml
                '''
            }
        }

        stage('Show App URL') {
            steps {
                sh '''
                echo "Waiting for Ingress LoadBalancer to provision..."
                
                RETRIES=0
                URL=""

                while [ -z "$URL" ] && [ $RETRIES -lt 18 ]; do
                    URL=$(kubectl get ingress k8s-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                    
                    if [ -z "$URL" ]; then
                        echo "Ingress hostname not ready... waiting 10s ($((RETRIES+1))/18)"
                        sleep 10
                        RETRIES=$((RETRIES+1))
                    fi
                done

                echo "======================================"
                if [ -z "$URL" ]; then
                    echo "ERROR: LoadBalancer timed out. Check AWS Console."
                else
                    echo "Application deployed successfully!"
                    echo "Rose Service: http://$URL/bookstore"
                fi
                echo "======================================"
                '''
            }
        }
    }
}
