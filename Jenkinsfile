pipeline {
    agent any

    environment {
        AWS_REGION         = 'ap-south-1'
        AWS_ACCOUNT_ID     = '547268988661'
        ECR_BACKEND        = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/greencart-backend"
        ECR_FRONTEND       = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/greencart-frontend"
        EKS_CLUSTER        = 'greencart-cluster'
        SONAR_PROJECT_KEY  = 'greencart'
        IMAGE_TAG          = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/akashak0717/DevOps-Project-05.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    script {
                        def scannerHome = tool 'sonar-scanner'
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.projectName=GreenCart \
                              -Dsonar.sources=. \
                              -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/.git/**
                        """
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --format table \
                      -o trivy-fs-report.txt \
                      .
                    cat trivy-fs-report.txt
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    # Build backend image
                    docker build \
                      -t ${ECR_BACKEND}:${IMAGE_TAG} \
                      -t ${ECR_BACKEND}:latest \
                      ./server

                    # Build frontend image (pass backend URL as build arg)
                    docker build \
                      --build-arg VITE_BACKEND_URL=http://${BACKEND_SERVICE_URL} \
                      -t ${ECR_FRONTEND}:${IMAGE_TAG} \
                      -t ${ECR_FRONTEND}:latest \
                      ./client
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    # Scan backend image
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --format table \
                      -o trivy-backend-image-report.txt \
                      ${ECR_BACKEND}:${IMAGE_TAG}

                    # Scan frontend image
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --format table \
                      -o trivy-frontend-image-report.txt \
                      ${ECR_FRONTEND}:${IMAGE_TAG}

                    cat trivy-backend-image-report.txt
                    cat trivy-frontend-image-report.txt
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        # Authenticate Docker with ECR
                        aws ecr get-login-password \
                          --region ${AWS_REGION} | \
                        docker login \
                          --username AWS \
                          --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        # Push backend
                        docker push ${ECR_BACKEND}:${IMAGE_TAG}
                        docker push ${ECR_BACKEND}:latest

                        # Push frontend
                        docker push ${ECR_FRONTEND}:${IMAGE_TAG}
                        docker push ${ECR_FRONTEND}:latest
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        # Update kubeconfig for EKS cluster
                        aws eks update-kubeconfig \
                          --region ${AWS_REGION} \
                          --name ${EKS_CLUSTER}

                        # Apply Kubernetes manifests
                        kubectl apply -f k8s/

                        # Update images to new build
                        kubectl set image deployment/greencart-backend \
                          backend=${ECR_BACKEND}:${IMAGE_TAG} \
                          -n greencart

                        kubectl set image deployment/greencart-frontend \
                          frontend=${ECR_FRONTEND}:${IMAGE_TAG} \
                          -n greencart

                        # Wait for rollout
                        kubectl rollout status deployment/greencart-backend -n greencart
                        kubectl rollout status deployment/greencart-frontend -n greencart
                    '''
                }
            }
        }

    }

    post {
        always {
            // Archive Trivy scan reports
            archiveArtifacts artifacts: 'trivy-*.txt', allowEmptyArchive: true
        }
        success {
            echo 'Pipeline completed successfully! GreenCart deployed to EKS.'
        }
        failure {
            echo 'Pipeline failed. Check the logs above for details.'
        }
    }
}