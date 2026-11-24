pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'ls -la'
            }
        }
        stage('Info') {
            steps {
                sh 'git log -1 --oneline'
                sh 'cat Dockerfile'
            }
        }
        stage('Test') {
            steps {
                sh 'python3 --version || echo "Python is not installed in this Jenkins container"'
            }
        }
    }
}
