pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    AWS_REGION     = "eu-north-1"
    CLUSTER_NAME   = "myApp"

    K8S_NAMESPACE  = "ecr-demo"
    DEPLOYMENT     = "ecr-demo"
    CONTAINER_NAME = "ecr-demo"

    DOCKERHUB_REPO = "ecr-eks-jenkins-demo"
    K8S_MANIFEST   = "k8s/deploy.yaml"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'ls -la'
      }
    }

    stage('Compute image tag') {
      steps {
        script {
          def shortSha = env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : "nogit"
          env.IMAGE_TAG = "${env.BUILD_NUMBER}-${shortSha}"
        }
        sh 'echo "IMAGE_TAG=$IMAGE_TAG"'
      }
    }

    stage('DockerHub login') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKERHUB_USERNAME',
          passwordVariable: 'DOCKERHUB_TOKEN'
        )]) {
          sh '''
            set -euo pipefail
            echo "Logging in to DockerHub as $DOCKERHUB_USERNAME"
            echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
          '''
        }
      }
    }

    stage('Build image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKERHUB_USERNAME',
          passwordVariable: 'DOCKERHUB_TOKEN'
        )]) {
          sh '''
            set -euo pipefail
            DOCKER_IMAGE="docker.io/${DOCKERHUB_USERNAME}/${DOCKERHUB_REPO}:${IMAGE_TAG}"
            echo "DOCKER_IMAGE=$DOCKER_IMAGE"
            docker build -t "$DOCKER_IMAGE" .
          '''
        }
      }
    }

    stage('Push image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKERHUB_USERNAME',
          passwordVariable: 'DOCKERHUB_TOKEN'
        )]) {
          sh '''
            set -euo pipefail
            DOCKER_IMAGE="docker.io/${DOCKERHUB_USERNAME}/${DOCKERHUB_REPO}:${IMAGE_TAG}"
            docker push "$DOCKER_IMAGE"
          '''
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        withCredentials([
          [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds'],
          usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_TOKEN')
        ]) {
          sh '''
            set -euo pipefail

            aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"

            # Ensure namespace exists
            kubectl get ns "$K8S_NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$K8S_NAMESPACE"

            # Use a real image in the manifest BEFORE apply (prevents REPLACED_BY_PIPELINE warning)
            DOCKER_IMAGE="docker.io/${DOCKERHUB_USERNAME}/${DOCKERHUB_REPO}:${IMAGE_TAG}"
            if grep -q "REPLACED_BY_PIPELINE" "$K8S_MANIFEST"; then
              sed -i "s|REPLACED_BY_PIPELINE|$DOCKER_IMAGE|g" "$K8S_MANIFEST"
            fi

            kubectl apply -f "$K8S_MANIFEST"
            kubectl rollout status -n "$K8S_NAMESPACE" deployment/"$DEPLOYMENT"
          '''
        }
      }
    }

    stage('Smoke test') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            set -euo pipefail
            aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"

            kubectl run -n "$K8S_NAMESPACE" curltest \
              --image=curlimages/curl --rm -i --restart=Never -- \
              curl -sS http://ecr-demo-svc | head -n 20
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline completed successfully"
    }

    failure {
      echo "❌ Pipeline failed – diagnostics"
      withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
        sh '''
          set +e
          aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || true
          kubectl get pods -n "$K8S_NAMESPACE" -o wide || true
          kubectl get events -n "$K8S_NAMESPACE" --sort-by=.metadata.creationTimestamp | tail -n 50 || true
        '''
      }
    }

    always {
      sh 'docker image prune -f || true'
    }
  }
}
