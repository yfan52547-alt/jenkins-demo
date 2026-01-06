pipeline {
  agent any

  environment {
    REGISTRY = "crpi-qvxmqo14dnp2pn9g.cn-hangzhou.personal.cr.aliyuncs.com"

    // ① commit-* 推到这里（你的“commit 仓库”）
    COMMIT_NAMESPACE = "gallery-app"     // ← 按你实际的命名空间改
    COMMIT_REPO      = "gallery-app"     // ← 按你实际的仓库名改

    // ② v1/v2/v3 推到这里（你的“release 仓库”）
    RELEASE_NAMESPACE = "ray-dev"        // ← 你新建的命名空间
    RELEASE_REPO      = "ray-dev"        // ← 你新建的仓库名（截图里是 ray-dev）
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Image (commit-*)') {
      steps {
        script {
          def sha = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()

          env.IMAGE_TAG   = "commit-${sha}"
          env.COMMIT_IMAGE = "${REGISTRY}/${COMMIT_NAMESPACE}/${COMMIT_REPO}:${env.IMAGE_TAG}"

          sh """
            set -e
            docker build -t "${env.COMMIT_IMAGE}" .
          """
        }
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
            docker push "${env.COMMIT_IMAGE}"
            echo "${env.COMMIT_IMAGE}" > build-info.txt
          """
        }
        archiveArtifacts artifacts: 'build-info.txt', fingerprint: true
      }
    }

    stage('Manual Approval (Promote to V*)') {
      steps {
        script {
          // 人工选择 v1/v2/v3
          def v = input(
            message: "Promote this commit image to TEST?",
            ok: "Promote",
            parameters: [
              choice(name: 'VERSION', choices: "v1\nv2\nv3", description: 'Select version tag for TEST')
            ]
          )
          env.VERSION = v
          env.RELEASE_IMAGE = "${REGISTRY}/${RELEASE_NAMESPACE}/${RELEASE_REPO}:${v}"
        }
      }
    }

    stage('Promote: retag & push V*') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'acr-push',
          usernameVariable: 'ACR_USER',
          passwordVariable: 'ACR_PASS'
        )]) {
          sh """
            set -e
            echo "\$ACR_PASS" | docker login ${REGISTRY} -u "\$ACR_USER" --password-stdin

            # 从本地已有镜像 retag；为稳妥起见也 pull 一下
            docker pull "${env.COMMIT_IMAGE}" || true

            docker tag  "${env.COMMIT_IMAGE}" "${env.RELEASE_IMAGE}"
            docker push "${env.RELEASE_IMAGE}"

            echo "commit=${env.COMMIT_IMAGE}" > promote-info.txt
            echo "release=${env.RELEASE_IMAGE}" >> promote-info.txt
          """
        }
        archiveArtifacts artifacts: 'promote-info.txt', fingerprint: true
      }
    }
  }
}
