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
                docker run -d -p 8080:80 --name jenkins-demo jenkins-demo:latest
                '''
            }
        }
    }
}
