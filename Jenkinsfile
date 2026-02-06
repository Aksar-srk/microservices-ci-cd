pipeline {
  agent any

  environment {
    DOCKERHUB_USERNAME = "aksarsr"
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
       Stage 2: Detect Services
       ========================= */
    stage('Detect Services') {
      steps {
        script {
          env.SERVICES = sh(
            script: "ls src",
            returnStdout: true
          ).trim().replaceAll("\\s+", ",")

          echo "Services detected: ${env.SERVICES}"
        }
      }
    }

    /* =========================
       Stage 3: Docker Build & Push (BUILDx ONLY)
       ========================= */
    stage('Docker Build & Push') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
          )
        ]) {

          sh '''
            set -e
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

            # Create builder if not exists
            docker buildx inspect jenkins-builder >/dev/null 2>&1 || \
              docker buildx create --name jenkins-builder --use

            docker buildx use jenkins-builder
          '''

          script {
            env.SERVICES.split(",").each { svc ->
              sh """
                echo "=============================="
                echo "Building & pushing ${svc}"
                echo "=============================="

                docker buildx build \
                  --platform linux/amd64 \
                  --push \
                  -t ${DOCKERHUB_USERNAME}/online-boutique-${svc}:${IMAGE_TAG} \
                  src/${svc}
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
        withCredentials([
          file(credentialsId: 'gke-kubeconfig', variable: 'KUBECONFIG')
        ]) {

          sh '''
            set -e
            kubectl get ns ${KUBE_NAMESPACE} || kubectl create ns ${KUBE_NAMESPACE}
          '''

          script {
            env.SERVICES.split(",").each { svc ->
              sh """
                echo "Deploying ${svc}"

                sed -i 's|image:.*|image: ${DOCKERHUB_USERNAME}/online-boutique-${svc}:${IMAGE_TAG}|' \
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
        withCredentials([
          file(credentialsId: 'gke-kubeconfig', variable: 'KUBECONFIG')
        ]) {
          sh '''
            kubectl get pods -n ${KUBE_NAMESPACE}
            kubectl get svc -n ${KUBE_NAMESPACE}
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ CI/CD pipeline completed successfully"
    }
    failure {
      echo "❌ CI/CD pipeline failed"
    }
    always {
      cleanWs()
    }
  }
}
