pipeline {

    agent {
        label 'aws-build'
    }

    stages {

        stage('Get Git Commit SHA') {
            steps {
                script {
                    env.GIT_SHA = sh(
                        script: 'git rev-parse --short=7 HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${BUILD_NUMBER}-${GIT_SHA}"

                    echo "========================================"
                    echo "Jenkins Build Number : ${BUILD_NUMBER}"
                    echo "Git Commit SHA       : ${GIT_SHA}"
                    echo "Docker Image Tag     : ${IMAGE_TAG}"
                    echo "========================================"
                }
            }
        }

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

        stage('OWASP Dependency Check') {
            steps {
                sh '''
                    mkdir -p dependency-check-report

                    dependency-check.sh \
                      --project "MERN DevSecOps" \
                      --scan . \
                      --format HTML \
                      --format JSON \
                      --out dependency-check-report \
                      --failOnCVSS 7
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
                      -t mern-backend:${IMAGE_TAG} \
                      ./backend

                    docker build \
                      -t mern-frontend:${IMAGE_TAG} \
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
                      mern-backend:${IMAGE_TAG}

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --no-progress \
                      --timeout 15m \
                      mern-frontend:${IMAGE_TAG}
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
                          mern-backend:${IMAGE_TAG} \
                          357199109816.dkr.ecr.ap-south-1.amazonaws.com/mern-devsecops-dev-backend:${IMAGE_TAG}

                        docker tag \
                          mern-frontend:${IMAGE_TAG} \
                          357199109816.dkr.ecr.ap-south-1.amazonaws.com/mern-devsecops-dev-frontend:${IMAGE_TAG}

                        docker push \
                          357199109816.dkr.ecr.ap-south-1.amazonaws.com/mern-devsecops-dev-backend:${IMAGE_TAG}

                        docker push \
                          357199109816.dkr.ecr.ap-south-1.amazonaws.com/mern-devsecops-dev-frontend:${IMAGE_TAG}
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
                          "/backend:/,/service:/ s/tag:.*/tag: \\"${IMAGE_TAG}\\"/" \
                          helm/mern-app/values.yaml

                        sed -i \
                          "/frontend:/,/service:/ s/tag:.*/tag: \\"${IMAGE_TAG}\\"/" \
                          helm/mern-app/values.yaml

                        echo "========================================"
                        echo "Updated values.yaml:"
                        echo "========================================"

                        cat helm/mern-app/values.yaml

                        git add helm/mern-app/values.yaml

                        git commit \
                          -m "Update application images to ${IMAGE_TAG}" || true

                        git push origin main
                    '''
                }
            }
        }
    }
}