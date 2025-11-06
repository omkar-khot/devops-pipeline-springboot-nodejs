pipeline {
    agent any
    
    environment {
        // Docker Registry Configuration
        DOCKER_REGISTRY = 'nexus.yourdomain.com:8082'
        DOCKER_CREDENTIALS_ID = 'nexus-docker-credentials'
        
        // Nexus Repository Configuration
        NEXUS_VERSION = 'nexus3'
        NEXUS_PROTOCOL = 'http'
        NEXUS_URL = 'nexus.yourdomain.com:8081'
        NEXUS_REPOSITORY = 'maven-releases'
        NEXUS_CREDENTIALS_ID = 'nexus-credentials'
        
        // SonarQube Configuration
        SONARQUBE_URL = 'http://sonarqube.yourdomain.com:9000'
        SONARQUBE_CREDENTIALS_ID = 'sonarqube-token'
        
        // AWS EKS Configuration
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER_NAME = 'devops-cluster'
        AWS_CREDENTIALS_ID = 'aws-credentials'
        
        // Application Version
        APP_VERSION = "${env.BUILD_NUMBER}"
        GIT_COMMIT_SHORT = "${env.GIT_COMMIT?.take(7)}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo '========================================'
                    echo 'Stage: Checkout Source Code'
                    echo '========================================'
                }
                checkout scm
                script {
                    GIT_COMMIT_SHORT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                }
            }
        }
        
        stage('Build Backend') {
            steps {
                script {
                    echo '========================================'
                    echo 'Stage: Build Spring Boot Backend'
                    echo '========================================'
                }
                dir('backend') {
                    sh 'chmod +x mvnw'
                    sh './mvnw clean package -DskipTests'
                }
            }
        }
        
        stage('Build Frontend') {
            steps {
                script {
                    echo '========================================'
                    echo 'Stage: Build Node.js Frontend'
                    echo '========================================'
                }
                dir('frontend') {
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
        }
        
        stage('Unit Tests') {
            parallel {
                stage('Backend Tests') {
                    steps {
                        dir('backend') {
                            sh './mvnw test'
                            junit '**/target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Frontend Tests') {
                    steps {
                        dir('frontend') {
                            sh 'npm test -- --coverage --watchAll=false'
                        }
                    }
                }
            }
        }
        
        stage('Code Quality Analysis') {
            steps {
                script {
                    echo '========================================'
                    echo 'Stage: SonarQube Code Analysis'
                    echo '========================================'
                }
                withSonarQubeEnv(credentialsId: SONARQUBE_CREDENTIALS_ID) {
                    dir('backend') {
                        sh './mvnw sonar:sonar'
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    echo '========================================'
                    echo 'Stage: Wait for Quality Gate'
                    echo '========================================'
                }
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Build Backend Image') {
                    steps {
                        script {
                            dir('backend') {
                                backendImage = docker.build(
                                    "${DOCKER_REGISTRY}/springboot-backend:${APP_VERSION}",
                                    "--build-arg JAR_FILE=target/*.jar ."
                                )
                            }
                        }
                    }
                }
                stage('Build Frontend Image') {
                    steps {
                        script {
                            dir('frontend') {
                                frontendImage = docker.build(
                                    "${DOCKER_REGISTRY}/nodejs-frontend:${APP_VERSION}",
                                    "."
                                )
                            }
                        }
                    }
                }
            }
        }
        
        stage('Security Scan with Trivy') {
            parallel {
                stage('Scan Backend Image') {
                    steps {
                        script {
                            echo 'Scanning Backend Image for vulnerabilities...'
                            sh """
                                trivy image \
                                --severity HIGH,CRITICAL \
                                --format json \
                                --output backend-trivy-report.json \
                                ${DOCKER_REGISTRY}/springboot-backend:${APP_VERSION}
                            """
                            
                            // Fail build if critical vulnerabilities found
                            sh """
                                trivy image \
                                --severity CRITICAL \
                                --exit-code 1 \
                                ${DOCKER_REGISTRY}/springboot-backend:${APP_VERSION}
                            """
                        }
                    }
                }
                stage('Scan Frontend Image') {
                    steps {
                        script {
                            echo 'Scanning Frontend Image for vulnerabilities...'
                            sh """
                                trivy image \
                                --severity HIGH,CRITICAL \
                                --format json \
                                --output frontend-trivy-report.json \
                                ${DOCKER_REGISTRY}/nodejs-frontend:${APP_VERSION}
                            """
                            
                            sh """
                                trivy image \
                                --severity CRITICAL \
                                --exit-code 1 \
                                ${DOCKER_REGISTRY}/nodejs-frontend:${APP_VERSION}
                            """
                        }
                    }
                }
            }
        }
        
        stage('Push to Nexus') {
            steps {
                script {
                    echo '========================================'
                    echo 'Stage: Push Docker Images to Nexus'
                    echo '========================================'
                    
                    docker.withRegistry("http://${DOCKER_REGISTRY}", DOCKER_CREDENTIALS_ID) {
                        backendImage.push("${APP_VERSION}")
                        backendImage.push("latest")
                        
                        frontendImage.push("${APP_VERSION}")
                        frontendImage.push("latest")
                    }
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    echo '========================================'
                    echo 'Stage: Deploy to AWS EKS'
                    echo '========================================'
                    
                    withCredentials([aws(credentialsId: AWS_CREDENTIALS_ID)]) {
                        // Update kubeconfig
                        sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}"
                        
                        // Apply Kubernetes manifests
                        sh "kubectl apply -f k8s/namespace.yaml"
                        sh "kubectl apply -f k8s/configmap.yaml"
                        sh "kubectl apply -f k8s/secrets.yaml"
                        
                        // Update image tags in deployment
                        sh """
                            kubectl set image deployment/backend-deployment \
                            backend=${DOCKER_REGISTRY}/springboot-backend:${APP_VERSION} \
                            -n devops-app
                        """
                        
                        sh """
                            kubectl set image deployment/frontend-deployment \
                            frontend=${DOCKER_REGISTRY}/nodejs-frontend:${APP_VERSION} \
                            -n devops-app
                        """
                        
                        // Apply deployments and services
                        sh "kubectl apply -f k8s/backend-deployment.yaml"
                        sh "kubectl apply -f k8s/backend-service.yaml"
                        sh "kubectl apply -f k8s/frontend-deployment.yaml"
                        sh "kubectl apply -f k8s/frontend-service.yaml"
                        sh "kubectl apply -f k8s/ingress.yaml"
                        
                        // Wait for rollout to complete
                        sh "kubectl rollout status deployment/backend-deployment -n devops-app"
                        sh "kubectl rollout status deployment/frontend-deployment -n devops-app"
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo '========================================'
                    echo 'Stage: Verify Deployment'
                    echo '========================================'
                    
                    withCredentials([aws(credentialsId: AWS_CREDENTIALS_ID)]) {
                        sh "kubectl get pods -n devops-app"
                        sh "kubectl get services -n devops-app"
                        sh "kubectl get ingress -n devops-app"
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo '========================================'
            echo 'Pipeline Execution Completed'
            echo '========================================'
            
            // Archive test results
            junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
            
            // Archive Trivy reports
            archiveArtifacts artifacts: '*-trivy-report.json', allowEmptyArchive: true
            
            // Clean up Docker images
            sh 'docker image prune -f'
        }
        success {
            echo '✅ Pipeline executed successfully!'
            // Add notification logic here (Slack, Email, etc.)
        }
        failure {
            echo '❌ Pipeline failed!'
            // Add notification logic here
        }
    }
}
