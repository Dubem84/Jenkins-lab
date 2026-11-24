pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'ls -la'
            }
        }
        stage('Test') {
            steps {
                sh 'python3 -m pytest -v'
            }
        }
        stage('Build Image') {
            steps {
                sh 'docker build -t jenkins-lab:${BUILD_NUMBER} .'
            }
        }
        stage('Run Image') {
            steps {
                sh 'docker run --rm jenkins-lab:${BUILD_NUMBER}'
            }
        }
    }
}
