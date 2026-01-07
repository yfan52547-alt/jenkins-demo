pipeline {
  agent any
  options { timestamps() }

  environment {
    REGISTRY = "crpi-qvxmqo14dnp2pn9g.cn-hangzhou.personal.cr.aliyuncs.com"

    // commit-* 推到这里（按你实际改）
    COMMIT_NAMESPACE = "gallery-app"
    COMMIT_REPO      = "gallery-app"

    // release(v*) 推到这里（按你实际改）
    RELEASE_NAMESPACE = "ray-dev"
    RELEASE_REPO      = "ray-dev"

    // 1=禁止覆盖已有版本（强烈建议开启）
    FORBID_OVERWRITE = "1"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Image (commit-*)') {
      steps {
        script {
          def sha = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.SHA = sha
          env.COMMIT_TAG = "commit-${sha}"
          env.COMMIT_IMAGE = "${REGISTRY}/${COMMIT_NAMESPACE}/${COMMIT_REPO}:${env.COMMIT_TAG}"
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
          credentialsId: 'acr-push',
          usernameVariable: 'ACR_USER',
          passwordVariable: 'ACR_PASS'
        )]) {
          sh """
            set -e
            echo "\$ACR_PASS" | docker login ${REGISTRY} -u "\$ACR_USER" --password-stdin
            docker push "${COMMIT_IMAGE}"
          """
        }
      }
    }

    stage('Manual Approval (SemVer)') {
      steps {
        script {
          def v = input(
            message: "Promote to TEST/RELEASE: input image version (SemVer)?",
            ok: "Promote",
            parameters: [
              string(
                name: 'VERSION',
                defaultValue: 'v1.0.0-rc.1',
                description: 'Format: vMAJOR.MINOR.PATCH or vMAJOR.MINOR.PATCH-rc.N (e.g. v1.2.3, v1.2.3-rc.1)'
              )
            ]
          ).trim()

          // 基础防呆：不允许空值/空格
          if (!v || v.contains(' ')) {
            error "Invalid VERSION: empty or contains spaces. Got: '${v}'"
          }

          // 企业常用 SemVer + rc（严格一点）
          // 允许：
          //  - v1.2.3
          //  - v1.2.3-rc.1
          //  - v0.1.0-rc.12
          def semverRcPattern = /^v(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(-rc\.(0|[1-9]\d*))?$/
          if (!(v ==~ semverRcPattern)) {
            error "Invalid VERSION '${v}'. Must match: vMAJOR.MINOR.PATCH or vMAJOR.MINOR.PATCH-rc.N. Example: v1.2.3-rc.1"
          }

          env.VERSION = v
          env.RELEASE_IMAGE = "${REGISTRY}/${RELEASE_NAMESPACE}/${RELEASE_REPO}:${v}"

          echo "Will promote: ${COMMIT_IMAGE} -> ${RELEASE_IMAGE}"
        }
      }
    }

    stage('Promote: retag & push release tag') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'acr-push',
          usernameVariable: 'ACR_USER',
          passwordVariable: 'ACR_PASS'
        )]) {
          sh """
            set -e
            echo "\$ACR_PASS" | docker login ${REGISTRY} -u "\$ACR_USER" --password-stdin

            # 禁止覆盖（企业强烈建议：制品不可变）
            if [ "${FORBID_OVERWRITE}" = "1" ]; then
              if docker manifest inspect "${RELEASE_IMAGE}" >/dev/null 2>&1; then
                echo "ERROR: ${RELEASE_IMAGE} already exists. Refusing to overwrite."
                exit 1
              fi
            fi

            # promote（不重建）：pull -> tag -> push
            docker pull "${COMMIT_IMAGE}" || true
            docker tag  "${COMMIT_IMAGE}" "${RELEASE_IMAGE}"
            docker push "${RELEASE_IMAGE}"
          """
        }
      }
    }
  }
}
