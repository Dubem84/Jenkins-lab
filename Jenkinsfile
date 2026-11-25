pipeline {
    agent any

    environment {
        IMAGE = 'dubemifedi/jenkins-lab'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Test') {
            steps {
                sh 'python3 -m pytest -v'
            }
        }
        stage('Build Image') {
            steps {
                sh 'docker build -t $IMAGE:${BUILD_NUMBER} -t $IMAGE:latest .'
            }
        }
        stage('Run Image') {
            steps {
                sh 'docker run --rm $IMAGE:${BUILD_NUMBER} python app.py'
            }
        }
        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
                    sh 'docker push $IMAGE:${BUILD_NUMBER}'
                    sh 'docker push $IMAGE:latest'
                }
            }
        }
    }
}
