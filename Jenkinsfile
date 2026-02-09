pipeline {
  agent any

  parameters {
    booleanParam(
      name: 'DO_RELEASE',
      defaultValue: false,
      description: '是否发布到测试仓库（勾选后会弹窗输入 V.x.x 或 V.x.x.x）'
    )
  }

  environment {
    REGISTRY = "crpi-qvxmqo14dnp2pn9g.cn-hangzhou.personal.cr.aliyuncs.com"

    // ✅ 研发仓库（commit-SHA）
    DEV_NAMESPACE  = "gallery-app"
    DEV_REPO       = "gallery-app"

    // ✅ 测试仓库（release版本）
    TEST_NAMESPACE = "ray-dev"
    TEST_REPO      = "gallery-app"
  }

  stages {

    stage('Checkout SCM') {
      steps {
        checkout scm
      }
    }

    stage('Build & Push Snapshot (DEV)') {
      steps {
        script {
          def sha = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.SNAPSHOT_TAG = "commit-${sha}"
          env.DEV_IMAGE = "${REGISTRY}/${DEV_NAMESPACE}/${DEV_REPO}:${env.SNAPSHOT_TAG}"

          sh """
            set -e
            echo "Build snapshot: ${DEV_IMAGE}"
            docker build -t ${DEV_IMAGE} .
          """
        }

        withCredentials([usernamePassword(
          credentialsId: 'acr-push',
          usernameVariable: 'ACR_USER',
          passwordVariable: 'ACR_PASS'
        )]) {
          sh """
            set -e
            echo "\$ACR_PASS" | docker login ${REGISTRY} -u "\$ACR_USER" --password-stdin
            echo "Push snapshot: ${DEV_IMAGE}"
            docker push ${DEV_IMAGE}
          """
        }
      }
    }

    // ✅ 只有 DO_RELEASE=true 才会执行；否则直接跳过，不会卡住占资源
    stage('Release to TEST (Manual Version)') {
      when {
        expression { return params.DO_RELEASE }
      }
      steps {
        script {
          def v = input(
            message: '请输入发布版本（格式：V.X.X 或 V.X.X.X）',
            ok: '确认发布到测试仓库',
            parameters: [
              string(name: 'RELEASE_VERSION', defaultValue: 'V.1.1', description: '例如：V.1.1 或 V.1.1.1（X为数字，可为 9 或 99）')
            ]
          ).trim()

          if (!(v ==~ /^V\.\d{1,2}\.\d{1,2}(\.\d{1,2})?$/)) {
            error("版本格式不合法：${v}。应为 V.X.X 或 V.X.X.X（X为数字，可为 9 或 99）")
          }

          env.RELEASE_VERSION = v
          env.TEST_IMAGE = "${REGISTRY}/${TEST_NAMESPACE}/${TEST_REPO}:${env.RELEASE_VERSION}"

          sh """
            set -e
            echo "Tag snapshot => release"
            echo "FROM: ${DEV_IMAGE}"
            echo "TO  : ${TEST_IMAGE}"
            docker tag ${DEV_IMAGE} ${TEST_IMAGE}
          """
        }

        withCredentials([usernamePassword(
          credentialsId: 'acr-push',
          usernameVariable: 'ACR_USER',
          passwordVariable: 'ACR_PASS'
        )]) {
          sh """
            set -e
            echo "\$ACR_PASS" | docker login ${REGISTRY} -u "\$ACR_USER" --password-stdin
            echo "Push release: ${TEST_IMAGE}"
            docker push ${TEST_IMAGE}

            echo "DEV_IMAGE=${DEV_IMAGE}" > build-info.txt
            echo "TEST_IMAGE=${TEST_IMAGE}" >> build-info.txt
          """
        }

        archiveArtifacts artifacts: 'build-info.txt', fingerprint: true
      }
    }
  }

  post {
    success {
      echo "✅ SUCCESS"
      echo "Snapshot (DEV): ${env.DEV_IMAGE}"
      if (params.DO_RELEASE) {
        echo "Release  (TEST): ${env.TEST_IMAGE}"
      } else {
        echo "Release  (TEST): skipped (DO_RELEASE=false)"
      }
    }
  }
}
