



pipeline {
  agent any

  environment {
    IMAGE_TAG = "${env.BRANCH_NAME}"
    COLOR = "green"
    PROJECT_ID = "sport-tournament-655af" 
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Push') {
      steps {
        sh """
        docker build -t gcr.io/$PROJECT_ID/myapp:$IMAGE_TAG .
        gcloud auth configure-docker
        docker push gcr.io/$PROJECT_ID/myapp:$IMAGE_TAG
        """
      }
    }

    stage('Deploy Green') {
      steps {
        sh """
        sed 's|<tag>|$IMAGE_TAG|g' deployment-green.yaml | kubectl apply -f -
        kubectl apply -f service.yaml
        """
      }
    }

    stage('Wait Pods') {
      steps {
        sh "kubectl rollout status deployment/myapp-green"
      }
    }

    stage('Test App') {
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
      steps {
        sh "kubectl delete deployment myapp-blue --ignore-not-found"
      }
    }
  }
}
