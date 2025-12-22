pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Checkout code'
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t jenkins-demo:latest .'
            }
        }
        stage('Run Container') {
              steps {
                sh '''
                  docker rm -f jenkins-demo || true
                  docker run -d --name jenkins-demo \
                    -p 8081:80 \
                    jenkins-demo:latest
                '''
                  }
            }
        }
    }
}
