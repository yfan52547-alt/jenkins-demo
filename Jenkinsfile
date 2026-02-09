pipeline {
  agent any

  environment {
    // ✅ 改成你的 ACR 信息（只写域名，不要带 /namespace/repo）
    REGISTRY   = "crpi-xxxxxxxxxxxxxxxx.cn-hangzhou.personal.cr.aliyuncs.com"
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

    // ✅ 从 Git 仓库读取 VERSION，并强制校验格式（每次必填）
    stage('Read & Validate Version') {
      steps {
        script {
          if (!fileExists('VERSION')) {
            error("❌ 缺少 VERSION 文件：请在仓库根目录创建 VERSION，内容如 V.1.1 或 V.1.1.1")
          }

          def v = readFile('VERSION').trim()

          // 支持：V.1.1 或 V.1.1.1（每段 1-2 位数字：9 或 99 都可以）
          if (!(v ==~ /^V\.\d{1,2}\.\d{1,2}(\.\d{1,2})?$/)) {
            error("❌ VERSION 格式不合法：${v}。应为 V.X.X 或 V.X.X.X（X为数字，可为 9 或 99）")
          }

          env.RELEASE_VERSION = v
          echo "✅ Release Version (from VERSION): ${env.RELEASE_VERSION}"
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

            # 构建：先用 commit-SHA 标签（唯一可追溯）
            docker build -t ${IMAGE_SHA} .

            # 再打一个“手动版本号”标签（便于发布/沟通）
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
