pipeline {
  agent any

  stages {
    stage('Checkout SCM') {
      steps {
        echo "=== Start checkout ==="
        checkout scm
        echo "=== Checkout success ==="
      }
    }

    stage('Show files') {
      steps {
        sh '''
          echo "=== Current dir ==="
          pwd
          echo "=== List files ==="
          ls -la
          echo "=== Git info ==="
          git status || true
          git log -1 --oneline || true
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline SUCCESS: Git checkout works!"
    }
    failure {
      echo "❌ Pipeline FAILED: Git checkout failed!"
    }
  }
}
