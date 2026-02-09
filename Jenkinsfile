pipeline {
  agent any

  environment {
    // ✅ 改成你的 ACR 信息（个人版一般是 *.personal.cr.aliyuncs.com）
    REGISTRY   = "crpi-qvxmqo14dnp2pn9g.cn-hangzhou.personal.cr.aliyuncs.com"
    NAMESPACE  = "ray-dev"
    IMAGE_NAME = "gallery-app"
  }

  stages {
    stage('Checkout SCM') {
      steps {
        echo "=== Start checkout ==="
        checkout scm
        echo "=== Checkout success ==="
      }
    }

    stage('Show files') {
      steps {
        sh '''
          set -e
          echo "=== Current dir ==="
          pwd
          echo "=== List files ==="
          ls -la
          echo "=== Git info ==="
          git status || true
          git log -1 --oneline || true
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          def sha = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.IMAGE_TAG = "commit-${sha}"
          env.IMAGE = "${REGISTRY}/${NAMESPACE}/${IMAGE_NAME}:${env.IMAGE_TAG}"

          sh '''
            set -e
            echo "Building image: ${IMAGE}"
            docker version
            docker build -t ${IMAGE} .
          '''
        }
      }
    }

    stage('Push to ACR') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'acr-push',
          usernameVariable: 'ACR_USER',
          passwordVariable: 'ACR_PASS'
        )]) {
          sh '''
            set -e
            echo "Login to ACR: ${REGISTRY}"
            echo "$ACR_PASS" | docker login ${REGISTRY} -u "$ACR_USER" --password-stdin

            echo "Push image: ${IMAGE}"
            docker push ${IMAGE}

            echo "${IMAGE}" > build-info.txt
          '''
        }

        archiveArtifacts artifacts: 'build-info.txt', fingerprint: true
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline SUCCESS: image pushed to ACR => ${env.IMAGE}"
    }
    failure {
      echo "❌ Pipeline FAILED"
    }
  }
}
