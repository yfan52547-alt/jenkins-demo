pipeline {
  agent any

  environment {
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

    stage('Manual Input Version') {
      steps {
        script {
          def v = input(
            message: '请输入发布版本（格式：V.X.X 或 V.X.X.X）',
            ok: '确认发布',
            parameters: [
              string(name: 'RELEASE_VERSION', defaultValue: 'V.1.1', description: '例如：V.1.1 或 V.1.1.1（X为数字，可为 9 或 99）')
            ]
          ).trim()

          if (!(v ==~ /^V\.\d{1,2}\.\d{1,2}(\.\d{1,2})?$/)) {
            error("版本格式不合法：${v}。应为 V.X.X 或 V.X.X.X（X为数字，可为 9 或 99）")
          }

          env.RELEASE_VERSION = v
          echo "✅ Manual Release Version: ${env.RELEASE_VERSION}"
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          def sha = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()

          env.TAG_SHA = "commit-${sha}"
          env.TAG_VER = env.RELEASE_VERSION

          env.IMAGE_SHA = "${REGISTRY}/${NAMESPACE}/${IMAGE_NAME}:${env.TAG_SHA}"
          env.IMAGE_VER = "${REGISTRY}/${NAMESPACE}/${IMAGE_NAME}:${env.TAG_VER}"

          sh '''
            set -e
            echo "=== Build image ==="
            echo "IMAGE_SHA => ${IMAGE_SHA}"
            echo "IMAGE_VER => ${IMAGE_VER}"
            docker version

            docker build -t ${IMAGE_SHA} .
            docker tag ${IMAGE_SHA} ${IMAGE_VER}
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
            echo "=== Login to ACR ==="
            echo "$ACR_PASS" | docker login ${REGISTRY} -u "$ACR_USER" --password-stdin

            echo "=== Push images ==="
            docker push ${IMAGE_SHA}
            docker push ${IMAGE_VER}

            echo "IMAGE_SHA=${IMAGE_SHA}" > build-info.txt
            echo "IMAGE_VER=${IMAGE_VER}" >> build-info.txt
          '''
        }

        archiveArtifacts artifacts: 'build-info.txt', fingerprint: true
      }
    }
  }

  post {
    success {
      echo "✅ SUCCESS"
      echo "Published: ${env.IMAGE_VER}"
      echo "Traceable: ${env.IMAGE_SHA}"
    }
    failure {
      echo "❌ FAILED"
    }
  }
}
