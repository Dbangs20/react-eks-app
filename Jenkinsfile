pipeline {
    agent any
    tools {
        nodejs 'NodeJS'
    }
    environment {
        DOCKER_CREDS = credentials('dockerhub-secret-id')
        IMAGE_NAME   = 'diah2012/react-eks-app'
        IMAGE_TAG    = "${BUILD_NUMBER}"
    }
    stages {
        stage('Checkout Source Code') {
            steps {
                cleanWs()
                checkout scm
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }
        stage('Unit Testing') {
            steps {
                sh 'npm test -- --watchAll=false --ci'
            }
        }
        stage('SonarQube Code Scan') {
            steps {
                scannerHome = tool 'SonarQubeServer'
                withSonarQubeEnv('SonarQubeServer') {
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=react-eks-app -Dsonar.sources=src"
                }
            }
        }
        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
        stage('Push to Docker Hub') {
            steps {
                sh "echo ${DOCKER_CREDS_PSW} | docker login -u ${DOCKER_CREDS_USR} --password-stdin"
                sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }
        stage('Deploy to Amazon EKS') {
            steps {
                sh "aws eks update-kubeconfig --region eu-north-1 --name react-app-eks-cluster"
                sh "sed -i 's|image:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g' k8s/deployment.yaml"
                sh "kubectl apply -f k8s/"
            }
        }
        stage('Verify Rollout') {
            steps {
                sh "kubectl rollout status deployment/react-app-deployment --timeout=180s"
                sh "kubectl get svc react-app-service"
            }
        }
    }
}
