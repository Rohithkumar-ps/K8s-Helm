pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        AWS_REGION       = 'ap-south-1'
        ECR_REPOSITORY   = 'cicd-demo-app'
        EKS_CLUSTER_NAME = 'REPLACE_WITH_YOUR_EKS_CLUSTER_NAME'
        K8S_NAMESPACE    = 'dev'
        HELM_RELEASE     = 'myapp-dev'
        IMAGE_TAG        = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Verify Tools and AWS') {
            steps {
                sh '''
                    set -eu
                    docker --version
                    aws --version
                    kubectl version --client
                    helm version
                    aws sts get-caller-identity
                '''
            }
        }

        stage('Prepare ECR') {
            steps {
                script {
                    env.AWS_ACCOUNT_ID = sh(
                        script: "aws sts get-caller-identity --query Account --output text",
                        returnStdout: true
                    ).trim()

                    env.ECR_REGISTRY =
                        "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                }

                sh '''
                    set -eu

                    aws ecr describe-repositories \
                      --repository-names "$ECR_REPOSITORY" \
                      --region "$AWS_REGION" >/dev/null 2>&1 || \
                    aws ecr create-repository \
                      --repository-name "$ECR_REPOSITORY" \
                      --region "$AWS_REGION"

                    aws ecr get-login-password \
                      --region "$AWS_REGION" |
                    docker login \
                      --username AWS \
                      --password-stdin "$ECR_REGISTRY"
                '''
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    set -eu
                    docker build \
                      --pull \
                      -t "$ECR_REPOSITORY:$IMAGE_TAG" .
                    docker image inspect "$ECR_REPOSITORY:$IMAGE_TAG"
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    set -eu

                    docker tag "$ECR_REPOSITORY:$IMAGE_TAG" \
                      "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

                    docker push \
                      "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"
                '''
            }
        }

        stage('Configure EKS Access') {
            steps {
                sh '''
                    set -eu
                    aws eks update-kubeconfig \
                      --region "$AWS_REGION" \
                      --name "$EKS_CLUSTER_NAME"

                    kubectl get nodes
                '''
            }
        }

        stage('Validate Helm Chart') {
            steps {
                sh '''
                    set -eu
                    helm lint ./helm/myapp
                    helm template "$HELM_RELEASE" ./helm/myapp \
                      --namespace "$K8S_NAMESPACE" \
                      --set-string image.repository="$ECR_REGISTRY/$ECR_REPOSITORY" \
                      --set-string image.tag="$IMAGE_TAG"
                '''
            }
        }

        stage('Deploy with Helm') {
            steps {
                sh '''
                    set -eu

                    helm upgrade --install "$HELM_RELEASE" ./helm/myapp \
                      --namespace "$K8S_NAMESPACE" \
                      --create-namespace \
                      --set-string image.repository="$ECR_REGISTRY/$ECR_REPOSITORY" \
                      --set-string image.tag="$IMAGE_TAG" \
                      --wait \
                      --timeout 5m

                    kubectl rollout status \
                      deployment/"$HELM_RELEASE" \
                      --namespace "$K8S_NAMESPACE" \
                      --timeout=300s

                    kubectl get deployment,service,pods \
                      --namespace "$K8S_NAMESPACE"
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Image pushed to ECR and deployed using Helm.'
        }
        failure {
            echo 'FAILED: Check the Jenkins console output for the failing stage.'
        }
        always {
            echo 'Pipeline execution finished.'
        }
    }
}
