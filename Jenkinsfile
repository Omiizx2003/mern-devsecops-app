pipeline {
    agent any

    stages {

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs \
                      --scanners vuln \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --no-progress \
                      --timeout 15m \
                      --skip-dirs .git \
                      .
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t mern-backend:${BUILD_NUMBER} ./backend
                    docker build -t mern-frontend:${BUILD_NUMBER} ./frontend
                '''
            }
        }
    }
}