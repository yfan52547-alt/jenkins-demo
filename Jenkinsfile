pipeline {
  agent any
  options { timestamps() }

  environment {
    REGISTRY = "crpi-qvxmqo14dnp2pn9g.cn-hangzhou.personal.cr.aliyuncs.com"
    REPO = "gallery-app"

    // 禁止覆盖已有版本（强烈建议开启）
    FORBID_OVERWRITE = "1"
  }

  stages {
    stage('Checkout') {
      steps { 
        checkout scm
      }
    }

    stage('Build Image (commit-*)') {
      steps {
        script {
          def sha = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.SHA = sha
          env.COMMIT_TAG = "commit-${sha}"
          env.COMMIT_IMAGE = "${REGISTRY}/${REPO}:${env.COMMIT_TAG}"
        }
        sh """
          set -e
          docker build -t "${COMMIT_IMAGE}" .
        """
      }
    }

    stage('Push commit-* to ACR') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'acr-login', // 使用正确的凭据ID
          usernameVariable: 'ACR_USER',
          passwordVariable: 'ACR_PASS'
        )]) {
          echo "ACR User: ${env.ACR_USER}"  // 用于调试
          echo "ACR Password: ${env.ACR_PASS}"  // 用于调试
          sh """
            set -e
            echo "\$ACR_PASS" | docker login ${REGISTRY} -u "\$ACR_USER" --password-stdin
            docker push "${COMMIT_IMAGE}"
          """
        }
      }
    }

    stage('Manual Approval (Version Input for TEST/RELEASE)') {
      steps {
        script {
          def v = input(
            message: "Promote to TEST/RELEASE: input image version (SemVer)?",
            ok: "Promote",
            parameters: [
              string(
                name: 'VERSION',
                defaultValue: 'v1.0.0',
                description: 'Format: vMAJOR.MINOR.PATCH or vMAJOR.MINOR.PATCH-rc.N (e.g. v1.2.3, v1.2.3-rc.1)'
              )
            ]
          ).trim()

          // 校验版本格式
          if (!v || v.contains(' ')) {
            error "Invalid VERSION: empty or contains spaces. Got: '${v}'"
          }

          // 使用正则校验 SemVer 格式
          def semverPattern = /^v(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(-rc\.(0|[1-9]\d*))?$/
          if (!(v ==~ semverPattern)) {
            error "Invalid VERSION '${v}'. Must match: vMAJOR.MINOR.PATCH or vMAJOR.MINOR.PATCH-rc.N. Example: v1.2.3-rc.1"
          }

          // 设置环境变量
          env.VERSION = v
          env.RELEASE_IMAGE = "${REGISTRY}/${REPO}:${v}"

          echo "Will promote: ${COMMIT_IMAGE} -> ${RELEASE_IMAGE}"
        }
      }
    }

    stage('Promote: retag & push release tag') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'acr-login', // 使用正确的凭据ID
          usernameVariable: 'ACR_USER',
          passwordVariable: 'ACR_PASS'
        )]) {
          sh """
            set -e
            echo "\$ACR_PASS" | docker login ${REGISTRY} -u "\$ACR_USER" --password-stdin

            // 防止覆盖已有版本
            if [ "${FORBID_OVERWRITE}" = "1" ]; then
              if docker manifest inspect "${RELEASE_IMAGE}" >/dev/null 2>&1; then
                echo "ERROR: ${RELEASE_IMAGE} already exists. Refusing to overwrite."
                exit 1
              fi
            fi

            // 推送测试版本
            docker pull "${COMMIT_IMAGE}" || true
            docker tag  "${COMMIT_IMAGE}" "${RELEASE_IMAGE}"
            docker push "${RELEASE_IMAGE}"
          """
        }
      }
    }
  }
  post {
    always {
      echo "Pipeline finished."
    }
  }
}
