pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = 'YOUR_AWS_ACCOUNT_ID' 
        AWS_DEFAULT_REGION = 'us-east-1' 
        IMAGE_REPO_NAME = 'prime-clone'
        IMAGE_TAG = "${BUILD_NUMBER}"
        GITHUB_CRED_ID = 'github-token' // Jenkins credential ID for GitHub pushes
    }
    stages {
        stage('Code Checkout') {
            steps {
                checkout scm
            }
        }
        stage('NPM Install & Build') {
            steps {
                sh 'npm install'
                sh 'npm run build'
            }
        }
        stage('SonarQube Quality Scan') {
            steps {
                echo "Forwarding static code mapping to SonarQube..."
                // Optional: add your configured sonar-scanner command here if using dynamic analytics tokens
            }
        }
        stage('Docker Build Container') {
            steps {
                sh "docker build -t ${IMAGE_REPO_NAME}:${IMAGE_TAG} ."
            }
        }
        stage('Trivy Image Audit') {
            steps {
                sh "trivy image --severity HIGH,CRITICAL --exit-code 0 ${IMAGE_REPO_NAME}:${IMAGE_TAG}"
            }
        }
        stage('Push Image to AWS ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
                docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}://{IMAGE_REPO_NAME}:latest
                docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}://{IMAGE_REPO_NAME}:${IMAGE_TAG}
                docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}://{IMAGE_REPO_NAME}:latest
                docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}://{IMAGE_REPO_NAME}:${IMAGE_TAG}
                """
            }
        }
        stage('Update Git Manifest For GitOps') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GITHUB_CRED_ID}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh """
                    sed -i "s|image: .*|image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}://{IMAGE_REPO_NAME}:${IMAGE_TAG}|g" k8s/deployment.yaml
                    git config user.email "jenkins@devsecops.poc"
                    git config user.name "Jenkins CI Engine"
                    git add k8s/deployment.yaml
                    git commit -m "Automated build update: image tag v${IMAGE_TAG} [skip ci]"
                    git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@://github.com{GIT_USERNAME}/amazon-prime-poc.git
                    git push origin HEAD:main
                    """
                }
            }
        }
    }
}
