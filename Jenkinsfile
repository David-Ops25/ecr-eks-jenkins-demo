pipeline {
  agent any

  environment {
    AWS_REGION    = "eu-north-1"
    CLUSTER_NAME  = "myApp"
    ECR_REPO_NAME = "myapp-ecr-demo"
    K8S_NAMESPACE = "ecr-demo"
    DEPLOYMENT    = "ecr-demo"
    CONTAINER     = "ecr-demo"
  }

  stages {
    stage('Checkout') {
      steps {
        sh 'ls -la'
      }
    }

    stage('Build → Push → Deploy') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            set -euo pipefail

            AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
            ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
            ECR_URI="${ECR_REGISTRY}/${ECR_REPO_NAME}"
            IMAGE_TAG="${BUILD_NUMBER}"

            aws ecr get-login-password --region "$AWS_REGION" | \
              docker login --username AWS --password-stdin "$ECR_REGISTRY"

            aws ecr describe-repositories --repository-names "$ECR_REPO_NAME" --region "$AWS_REGION" >/dev/null 2>&1 \
              || aws ecr create-repository --repository-name "$ECR_REPO_NAME" --region "$AWS_REGION"

            docker build -t "$ECR_REPO_NAME:$IMAGE_TAG" .
            docker tag "$ECR_REPO_NAME:$IMAGE_TAG" "$ECR_URI:$IMAGE_TAG"
            docker push "$ECR_URI:$IMAGE_TAG"

            aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"

            kubectl get ns "$K8S_NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$K8S_NAMESPACE"
            kubectl apply -n "$K8S_NAMESPACE" -f k8s/deploy.yaml

            kubectl set image -n "$K8S_NAMESPACE" deployment/"$DEPLOYMENT" \
              "$CONTAINER"="$ECR_URI:$IMAGE_TAG"
            kubectl rollout status -n "$K8S_NAMESPACE" deployment/"$DEPLOYMENT"
          '''
        }
      }
    }
  }
}
