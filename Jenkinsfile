pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION       = 'ap-south-1'
        ECR_REPOSITORY   = 'cicd-demo-app'
        EKS_CLUSTER_NAME = 'jenkins-eks-cluster'
        K8S_NAMESPACE    = 'dev'
        HELM_RELEASE     = 'myapp-dev'
        IMAGE_TAG        = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh '''
                    set -eu
                    git log -1 --oneline
                    test -f app/Dockerfile
                    test -f app/index.html
                    test -f app/helm/myapp/Chart.yaml
                    test -f app/helm/myapp/templates/deployment.yaml
                    test -f app/helm/myapp/templates/service.yaml
                '''
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
                        script: 'aws sts get-caller-identity --query Account --output text',
                        returnStdout: true
                    ).trim()

                    env.ECR_REGISTRY =
                        "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                }

                sh '''
                    set -eu

                    # Verify the existing repository; do not create another.
                    aws ecr describe-repositories \
                      --repository-names "$ECR_REPOSITORY" \
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

                    # Dockerfile and build context are inside app/.
                    docker build --pull \
                      -t "$ECR_REPOSITORY:$IMAGE_TAG" app/

                    docker image inspect "$ECR_REPOSITORY:$IMAGE_TAG"

                    # Validate the NGINX configuration inside the image.
                    docker run --rm \
                      "$ECR_REPOSITORY:$IMAGE_TAG" nginx -t
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    set -eu

                    docker tag \
                      "$ECR_REPOSITORY:$IMAGE_TAG" \
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

                    helm lint ./app/helm/myapp

                    helm template "$HELM_RELEASE" ./app/helm/myapp \
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

                    helm upgrade --install "$HELM_RELEASE" ./app/helm/myapp \
                      --namespace "$K8S_NAMESPACE" \
                      --create-namespace \
                      --set-string image.repository="$ECR_REGISTRY/$ECR_REPOSITORY" \
                      --set-string image.tag="$IMAGE_TAG" \
                      --wait \
                      --timeout 10m

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
            echo 'SUCCESS: Image pushed to ECR and deployed to EKS using Helm.'
        }
        failure {
            echo 'FAILED: Check the first failing stage in the Jenkins console output.'
        }
        always {
            echo 'Pipeline execution finished.'
        }
    }
}
