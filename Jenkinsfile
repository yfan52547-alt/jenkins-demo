pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Pulling code from GitHub'
                sh 'ls -la'
            }
        }

        stage('Hello') {
            steps {
                echo 'Hello Jenkins CI!'
                sh 'echo "Build success at $(date)"'
            }
        }
    }
}
