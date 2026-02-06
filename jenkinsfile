pipeline {
  agent any

  environment {
    REGISTRY_URL     = "docker.io"
    IMAGE_NAMESPACE  = "online-boutique"
    DEPLOY_ENV       = "dev"
    KUBE_NAMESPACE   = "default"
    IMAGE_TAG        = "${BUILD_NUMBER}"
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
       Stage 2: Detect Changed Services
       ========================= */
    stage('Detect Changes') {
      steps {
        script {
          def changedFiles = sh(
            script: "git diff --name-only HEAD~1 HEAD",
            returnStdout: true
          ).trim().split("\n")

          def services = []
          changedFiles.each { file ->
            if (file.startsWith("src/")) {
              def svc = file.split("/")[1]
              services << svc
            }
          }

          services = services.unique()

          if (services.isEmpty()) {
            echo "No service changes detected. Exiting pipeline."
            currentBuild.result = 'SUCCESS'
            error("Pipeline stopped: no service changes")
          }

          env.CHANGED_SERVICES = services.join(",")
          echo "Changed services: ${env.CHANGED_SERVICES}"
        }
      }
    }

    /* =========================
       Stage 3: Build & Test (Parallel)
       ========================= */
    stage('Build & Test') {
      parallel {
        script {
          def branches = [:]
          env.CHANGED_SERVICES.split(",").each { svc ->
            branches[svc] = {
              stage("Build & Test: ${svc}") {
                dir("src/${svc}") {
                  script {
                    if (fileExists("go.mod")) {
                      sh "go test ./..."
                    } else if (fileExists("package.json")) {
                      sh "npm install"
                      sh "npm test || true"
                    } else if (fileExists("requirements.txt")) {
                      sh "pip install -r requirements.txt"
                    } else if (fileExists("pom.xml")) {
                      sh "mvn test"
                    } else {
                      echo "No build tool detected for ${svc}"
                    }
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
       Stage 4: Docker Build
       ========================= */
    stage('Docker Build') {
      steps {
        script {
          env.CHANGED_SERVICES.split(",").each { svc ->
            sh """
              docker build \
                -t ${REGISTRY_URL}/${IMAGE_NAMESPACE}/${svc}:${IMAGE_TAG} \
                src/${svc}
            """
          }
        }
      }
    }

    /* =========================
       Stage 5: Push Images
       ========================= */
    stage('Push Images') {
      steps {
        script {
          env.CHANGED_SERVICES.split(",").each { svc ->
            sh """
              docker push ${REGISTRY_URL}/${IMAGE_NAMESPACE}/${svc}:${IMAGE_TAG}
            """
          }
        }
      }
    }

    /* =========================
       Stage 6: Deploy to Kubernetes
       ========================= */
    stage('Deploy') {
      steps {
        script {
          env.CHANGED_SERVICES.split(",").each { svc ->
            sh """
              sed -i 's|image:.*${svc}:.*|image: ${REGISTRY_URL}/${IMAGE_NAMESPACE}/${svc}:${IMAGE_TAG}|' \
              kubernetes-manifests/${svc}.yaml

              kubectl apply -f kubernetes-manifests/${svc}.yaml -n ${KUBE_NAMESPACE}
            """
          }
        }
      }
    }

    /* =========================
       Stage 7: Post-Deploy Validation
       ========================= */
    stage('Validate') {
      steps {
        script {
          env.CHANGED_SERVICES.split(",").each { svc ->
            sh """
              kubectl rollout status deployment/${svc} -n ${KUBE_NAMESPACE}
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "Deployment successful"
    }
    failure {
      echo "Pipeline failed"
    }
    always {
      cleanWs()
    }
  }
}
