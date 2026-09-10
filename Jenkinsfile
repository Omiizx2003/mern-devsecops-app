pipeline {
    agent {
        label 'aws-build'
    }

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

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=mern-devsecops \
                              -Dsonar.projectName="MERN DevSecOps" \
                              -Dsonar.sources=backend,frontend \
                              -Dsonar.exclusions="**/node_modules/**,**/build/**,**/coverage/**" \
                              -Dsonar.sourceEncoding=UTF-8 \
                              -Dsonar.qualitygate.wait=true
                        """
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                      -t mern-backend:${BUILD_NUMBER} \
                      ./backend

                    docker build \
                      -t mern-frontend:${BUILD_NUMBER} \
                      ./frontend
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

        stage('GitOps Update') {
            steps {
                sshagent(['github-gitops-ssh']) {
                    sh '''
                        rm -rf gitops

                        git clone \
                          git@github.com:Omiizx2003/mern-devsecops-gitops.git \
                          gitops

                        cd gitops

                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@localhost"

                        sed -i \
                          "/backend:/,/service:/ s/tag:.*/tag: \\"${BUILD_NUMBER}\\"/" \
                          helm/mern-app/values.yaml

                        sed -i \
                          "/frontend:/,/service:/ s/tag:.*/tag: \\"${BUILD_NUMBER}\\"/" \
                          helm/mern-app/values.yaml

                        echo "Updated values.yaml:"
                        cat helm/mern-app/values.yaml

                        git add helm/mern-app/values.yaml

                        git commit \
                          -m "Update application images to ${BUILD_NUMBER}" || true

                        git push origin main
                    '''
                }
            }
        }
    }
}