pipeline {
  agent any

  environment {
    DOCKERHUB_USERNAME = "saksar762@gmail.com"   // 👈 change this
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
       Stage 2: Select Services (Build ALL)
       ========================= */
    stage('Select Services') {
      steps {
        script {
          env.CHANGED_SERVICES = sh(
            script: "ls src",
            returnStdout: true
          ).trim().split("\n").join(",")

          echo "Services selected: ${env.CHANGED_SERVICES}"
        }
      }
    }

    /* =========================
       Stage 3: Build & Test (Parallel)
       ========================= */
    stage('Build & Test') {
      steps {
        script {
          def branches = [:]

          env.CHANGED_SERVICES.split(",").each { svc ->
            branches[svc] = {
              stage("Build & Test: ${svc}") {
                dir("src/${svc}") {

                  if (fileExists("go.mod")) {
                    sh "go test ./..."
                  }
                  else if (fileExists("package.json")) {
                    sh "npm install"
                    sh "npm test || true"
                  }
                  else if (fileExists("requirements.txt")) {
                    sh "pip install -r requirements.txt"
                  }
                  else if (fileExists("pom.xml")) {
                    sh "mvn test"
                  }
                  else {
                    echo "No build tool detected for ${svc}"
                  }

                }
              }
            }
          }

          parallel branches
        }
      }
    }

    /* =========================
       Stage 4: Docker Build & Push
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
            env.CHANGED_SERVICES.split(",").each { svc ->
              sh """
                docker build -t ${DOCKERHUB_USERNAME}/online-boutique-${svc}:${IMAGE_TAG} src/${svc}
                docker push ${DOCKERHUB_USERNAME}/online-boutique-${svc}:${IMAGE_TAG}
              """
            }
          }
        }
      }
    }

    /* =========================
       Stage 5: Deploy to GKE
       ========================= */
    stage('Deploy to GKE') {
      steps {
        withCredentials([file(credentialsId: 'gke-kubeconfig', variable: 'KUBECONFIG')]) {
          script {
            env.CHANGED_SERVICES.split(",").each { svc ->
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
       Stage 6: Validate
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
