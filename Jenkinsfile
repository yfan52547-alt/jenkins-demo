pipeline {
  agent any

  environment {
    REGISTRY   = "crpi-qvxmqo14dnp2pn9g.cn-hangzhou.personal.cr.aliyuncs.com"  // 修改这里
    NAMESPACE  = "ray-dev"
    IMAGE_NAME = "gallery-app"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    /* ========= 新增：SonarQube 代码分析 ========= */
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonarqube') {
          sh '''
            docker run --rm \
              -v "$PWD:/usr/src" \
              sonarsource/sonar-scanner-cli \
              -Dsonar.projectKey=gallery-app \
              -Dsonar.projectName=gallery-app \
              -Dsonar.sources=. \
              -Dsonar.host.url=$SONAR_HOST_URL \
              -Dsonar.login=$SONAR_AUTH_TOKEN
          '''
        }
      }
    }

    /* ========= 新增：质量门禁 ========= */
    stage('Quality Gate') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Build Image') {
      steps {
        script {
          def sha = sh(
            script: "git rev-parse --short HEAD",
            returnStdout: true
          ).trim()

          env.IMAGE_TAG = "commit-${sha}"
          env.IMAGE = "${REGISTRY}/${NAMESPACE}/${IMAGE_NAME}:${env.IMAGE_TAG}"

          sh """
            set -e
            docker build -t ${env.IMAGE} .
          """
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
          sh """
            set -e
            echo "\$ACR_PASS" | docker login ${REGISTRY} -u "\$ACR_USER" --password-stdin
            docker push ${env.IMAGE}
            echo "${env.IMAGE}" > build-info.txt
          """
        }
        archiveArtifacts artifacts: 'build-info.txt', fingerprint: true
      }
    }
  }
}
