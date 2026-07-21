pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = '567017110325' 
        AWS_DEFAULT_REGION = 'us-east-1' 
        IMAGE_REPO_NAME = 'prime-clone'
        IMAGE_TAG = "${BUILD_NUMBER}"
        GITHUB_CRED_ID = 'github-token'
    }
    stages {
        stage('Code Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Terraform Provision Infrastructure') {
            steps {
                dir('terraform') {
                    sh 'terraform init'
                    sh 'terraform apply --auto-approve'
                }
            }
        }

        stage('Establish Cluster Access') {
            steps {
                sh "aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name prime-poc-cluster"
            }
        }

        stage('NPM Install & Build') {
            steps {
                sh 'npm install'
                sh 'npm run build'
            }
        }

        stage('Docker Build Container') {
            steps {
                sh "docker build -t ${IMAGE_REPO_NAME}:${IMAGE_TAG} ."
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
                    git commit -m "Automated build update: image tag v${IMAGE_TAG} [skip ci]" || true
                    git remote set-url origin "https://\${GIT_USERNAME}:\${GIT_PASSWORD}@://github.com\${GIT_USERNAME}/POC-6.git"
                    git push origin HEAD:main
                    """
                }
            }
        }

        stage('Deploy GitOps & Helm Monitoring') {
            steps {
                sh '''
                NODE_ROLE=$(aws iam list-roles --query "Roles[?contains(RoleName, 'monitoring_node')].RoleName" --output text)
                aws iam attach-role-policy --role-name $NODE_ROLE --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly || true
        
                kubectl create namespace argocd || true
                kubectl apply --server-side -n argocd -f https://githubusercontent.com || true
                kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s || true
                kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}' || true
        
                kubectl apply -f k8s/argocd-app.yaml || true
        
                helm repo add prometheus-community https://github.io || true
                helm repo update
                kubectl create namespace monitoring || true
        
                helm upgrade --install kube-stack prometheus-community/kube-prometheus-stack \
                  --namespace monitoring \
                  --set prometheus.prometheusSpec.resources.requests.memory=400Mi \
                  --set prometheus.prometheusSpec.resources.limits.memory=1200Mi \
                  --set grafana.service.type=NodePort
                '''
            }
        }
        
        stage('Display Live Entry Details') {
            steps {
                sh '''
                echo "=========================================================="
                echo "DEVSECOPS POC LIFECYCLE CONNECTIONS"
                echo "=========================================================="
        
                # FIXED FILTER: Targets only the instance created by the EKS module, ignoring the Jenkins box
                EKS_NODE_IP=$(aws ec2 describe-instances \
                  --filters "Name=instance-state-name,Values=running" \
                            "Name=tag:kubernetes.io/cluster/prime-poc-cluster,Values=owned" \
                  --query "Reservations[*].Instances[*].PublicIpAddress" \
                  --output text | head -n1)
        
                ARGOCD_PORT=$(kubectl get svc argocd-server -n argocd -o jsonpath="{.spec.ports[0].nodePort}" 2>/dev/null || echo "N/A")
                GRAFANA_PORT=$(kubectl get svc kube-stack-grafana -n monitoring -o jsonpath="{.spec.ports[0].nodePort}" 2>/dev/null || echo "N/A")
                GRAFANA_PASS=$(kubectl get secret -n monitoring kube-stack-grafana -o jsonpath="{.data.admin-password}" 2>/dev/null | base64 --decode || echo "N/A")
        
                echo "Actual EKS Worker Node IP: ${EKS_NODE_IP}"
                echo "Application URL : http://${EKS_NODE_IP}:32080"
                echo "ArgoCD URL      : http://${EKS_NODE_IP}:${ARGOCD_PORT}"
                echo "Grafana URL     : http://${EKS_NODE_IP}:${GRAFANA_PORT}"
                echo "Grafana User    : admin"
                echo "Grafana Password: ${GRAFANA_PASS}"
                echo "=========================================================="
                '''
            }
        }
    }
}
