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

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --no-progress \
                      --timeout 15m \
                      mern-backend:${BUILD_NUMBER}

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --no-progress \
                      --timeout 15m \
                      mern-frontend:${BUILD_NUMBER}
                '''
            }
        }

        stage('ECR Login & Push') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-jenkins-ci']
                ]) {
                    sh '''
                        aws ecr get-login-password \
                          --region ap-south-1 | \
                        docker login \
                          --username AWS \
                          --password-stdin \
                          357199109816.dkr.ecr.ap-south-1.amazonaws.com

                        docker tag \
                          mern-backend:${BUILD_NUMBER} \
                          357199109816.dkr.ecr.ap-south-1.amazonaws.com/mern-devsecops-dev-backend:${BUILD_NUMBER}

                        docker tag \
                          mern-frontend:${BUILD_NUMBER} \
                          357199109816.dkr.ecr.ap-south-1.amazonaws.com/mern-devsecops-dev-frontend:${BUILD_NUMBER}

                        docker push \
                          357199109816.dkr.ecr.ap-south-1.amazonaws.com/mern-devsecops-dev-backend:${BUILD_NUMBER}

                        docker push \
                          357199109816.dkr.ecr.ap-south-1.amazonaws.com/mern-devsecops-dev-frontend:${BUILD_NUMBER}
                    '''
                }
            }
        }
    }
}