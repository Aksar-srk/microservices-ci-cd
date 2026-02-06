pipeline {
  agent any

  environment {
    DOCKERHUB_USERNAME = "aksarsr"     // 👈 your Docker Hub username
    IMAGE_TAG          = "${BUILD_NUMBER}"
    KUBE_NAMESPACE     = "online-boutique"
  }

  options {
    timestamps()
  }

  stages {

    /* =========================
       Stage 1: Checkout
       ========================= */
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    /* =========================
       Stage 2: Select Services
       ========================= */
    stage('Select Services') {
      steps {
        script {
          env.SERVICES = sh(
            script: "ls src",
            returnStdout: true
          ).trim().split("\n").join(",")

          echo "Services: ${env.SERVICES}"
        }
      }
    }

    /* =========================
       Stage 3: Docker Build & Push
       ========================= */
    stage('Docker Build & Push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {

          sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"

          script {
            env.SERVICES.split(",").each { svc ->
              sh """
                docker build \
                  -t ${DOCKERHUB_USERNAME}/online-boutique-${svc}:${IMAGE_TAG} \
                  src/${svc}

                docker push ${DOCKERHUB_USERNAME}/online-boutique-${svc}:${IMAGE_TAG}
              """
            }
          }
        }
      }
    }

    /* =========================
       Stage 4: Deploy to GKE
       ========================= */
    stage('Deploy to GKE') {
      steps {
        withCredentials([file(credentialsId: 'gke-kubeconfig', variable: 'KUBECONFIG')]) {
          script {
            env.SERVICES.split(",").each { svc ->
              sh """
                sed -i '/image:/c\\  image: ${DOCKERHUB_USERNAME}/online-boutique-${svc}:${IMAGE_TAG}' \
                kubernetes-manifests/${svc}.yaml

                kubectl apply -f kubernetes-manifests/${svc}.yaml -n ${KUBE_NAMESPACE}
              """
            }
          }
        }
      }
    }

    /* =========================
       Stage 5: Validate
       ========================= */
    stage('Validate') {
      steps {
        withCredentials([file(credentialsId: 'gke-kubeconfig', variable: 'KUBECONFIG')]) {
          sh "kubectl get pods -n ${KUBE_NAMESPACE}"
          sh "kubectl get svc -n ${KUBE_NAMESPACE}"
        }
      }
    }
  }

  post {
    success {
      echo "CI/CD pipeline completed successfully"
    }
    failure {
      echo "CI/CD pipeline failed"
    }
    always {
      cleanWs()
    }
  }
}
