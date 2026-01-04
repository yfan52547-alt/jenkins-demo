pipeline {
  agent any

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('MySonarQube') {
          sh '''
            set -e

            # 用容器跑 sonar-scanner（不依赖 Jenkins 节点安装）
            docker run --rm \
              -v "$PWD:/usr/src" \
              sonarsource/sonar-scanner-cli \
              -Dsonar.projectKey=demo-pipeline \
              -Dsonar.sources=/usr/src \
              -Dsonar.sourceEncoding=UTF-8
          '''
        }
      }
    }

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
          def sha = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          def image = "crpi-qvxmqo14dnp2pn9g.cn-hangzhou.personal.cr.aliyuncs.com/ray-dev/gallery-app:commit-${sha}"

          sh """
            set -e
            docker build -t ${image} .
            echo "IMAGE=${image}" > build-info.txt
            echo "SHA=$(git rev-parse HEAD)" >> build-info.txt
            echo "TIME=$(date -Iseconds)" >> build-info.txt
          """

          env.BUILD_IMAGE = image
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
            echo "$ACR_PASS" | docker login crpi-qvxmqo14dnp2pn9g.cn-hangzhou.personal.cr.aliyuncs.com -u "$ACR_USER" --password-stdin
            docker push "$BUILD_IMAGE"
          '''
        }
        archiveArtifacts artifacts: 'build-info.txt', fingerprint: true
      }
    }
  }
}
