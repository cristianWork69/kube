pipeline {
    agent none

    environment {
        IMAGE_TAG = "${env.BRANCH_NAME}"
        COLOR = "green"
        PROJECT_ID = "sport-tournament-655af"
    }

    stages {
        stage('Checkout') {
            agent { kubernetes { label 'docker-agent' } }
            steps {
                checkout scm
            }
        }

        stage('Build & Push') {
            agent { 
                kubernetes {
                    label 'docker-agent'
                    defaultContainer 'docker'
                    containers {
                        containerTemplate(
                            name: 'docker',
                            image: 'docker:20.10.24-dind',
                            ttyEnabled: true,
                            command: 'cat',
                            privileged: true
                        )
                    }
                }
            }
            steps {
                container('docker') {
                    script {
                        // Docker build and push
                        sh """
                        docker build -t gcr.io/$PROJECT_ID/myapp:$IMAGE_TAG .
                        gcloud auth configure-docker
                        docker push gcr.io/$PROJECT_ID/myapp:$IMAGE_TAG
                        """
                    }
                }
            }
        }

        stage('Deploy Green') {
            agent { kubernetes { label 'docker-agent' } }
            steps {
                container('docker') {
                    script {
                        // Applicazione del deployment per la versione verde
                        sh """
                        sed 's|<tag>|$IMAGE_TAG|g' deployment-green.yaml | kubectl apply -f -
                        kubectl apply -f service.yaml
                        """
                    }
                }
            }
        }

        stage('Wait Pods') {
            agent { kubernetes { label 'docker-agent' } }
            steps {
                container('docker') {
                    sh "kubectl rollout status deployment/myapp-green"
                }
            }
        }

        stage('Test App') {
            agent { kubernetes { label 'docker-agent' } }
            steps {
                script {
                    def ip = sh(script: "kubectl get svc myapp-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'", returnStdout: true).trim()
                    def output = sh(script: "curl -s http://${ip}", returnStdout: true).trim()
                    if (!output.contains("Hello")) {
                        error("App failed health check")
                    }
                }
            }
        }

        stage('Cleanup') {
            agent { kubernetes { label 'docker-agent' } }
            steps {
                container('docker') {
                    sh "kubectl delete deployment myapp-blue --ignore-not-found"
                }
            }
        }
    }
}
