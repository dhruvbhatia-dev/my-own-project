pipeline {
    agent any 
    stages {
        stage('Clone repo') {
            steps {
                git branch: 'main', url: 'https://github.com/dhruvbhatia-dev/my-own-project.git'               
            }
        }
        stage('Build image') {
            steps {
                sh 'docker build -t flask-app .'
            }
        }
        stage('Deploy with docker compose') {
            steps {
                sh 'docker-compose down'

                sh 'docker-compose up -d  --build'
            }
        }
    }
}