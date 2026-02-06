pipeline {
  agent any

  environment {
    REGISTRY_URL    = "docker.io"
    IMAGE_NAMESPACE = "online-boutique"
    IMAGE_TAG       = "${BUILD_NUMBER}"
    KUBE_NAMESPACE  = "default"
    CHANGED_SERVICES = ""
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
       Stage 2: Detect Changed Services
       ========================= */
    stage('Detect Changes') {
      steps {
        script {
          def services = []

          if (env.BUILD_NUMBER == "1") {
            echo "First build detected — building all services"
            services = sh(
              script: "ls src",
              returnStdout: true
            ).trim().split("\n")
          } else {
            def changedFiles = sh(
              script: "git diff --name-only HEAD~1 HEAD || true",
              returnStdout: true
            ).trim().split("\n")

            changedFiles.each { file ->
              if (file.startsWith("src/")) {
                services << file.split("/")[1]
              }
            }
          }

          services = services.unique().findAll { it }

          if (services.isEmpty()) {
            echo "No service changes detected. Pipeline will exit successfully."
            env.CHANGED_SERVICES = ""
          } else {
            env.CHANGED_SERVICES = services.join(",")
            echo "Services to build: ${env.CHANGED_SERVICES}"
          }
        }
      }
    }

    /* =========================
       Stage 3: Build & Test
       ========================= */
    stage('Build & Test') {
      when {
        expression { env.CHANGED_SERVICES?.trim() }
      }
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
                    echo "No build tool for ${svc}"
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
      when {
        expression { env.CHANGED_SERVICES?.trim() }
      }
      steps {
        script {
          env.CHANGED_SERVICES.split(",").each { svc ->
            sh """
              docker build -t ${REGISTRY_URL}/${IMAGE_NAMESPACE}/${svc}:${IMAGE_TAG} src/${svc}
              docker push ${REGISTRY_URL}/${IMAGE_NAMESPACE}/${svc}:${IMAGE_TAG}
            """
          }
        }
      }
    }

    /* =========================
       Stage 5: Deploy
       ========================= */
    stage('Deploy') {
      when {
        expression { env.CHANGED_SERVICES?.trim() }
      }
      steps {
        script {
          env.CHANGED_SERVICES.split(",").each { svc ->
            sh """
              sed -i '/image:/c\\  image: ${REGISTRY_URL}/${IMAGE_NAMESPACE}/${svc}:${IMAGE_TAG}' \
              kubernetes-manifests/${svc}.yaml

              kubectl apply -f kubernetes-manifests/${svc}.yaml -n ${KUBE_NAMESPACE}
            """
          }
        }
      }
    }

    /* =========================
       Stage 6: Validate
       ========================= */
    stage('Validate') {
      when {
        expression { env.CHANGED_SERVICES?.trim() }
      }
      steps {
        script {
          env.CHANGED_SERVICES.split(",").each { svc ->
            sh "kubectl rollout status deployment/${svc} -n ${KUBE_NAMESPACE}"
          }
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
