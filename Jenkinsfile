pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    choice(name: 'DEPLOY_ENV', choices: ['dev', 'prod'], description: 'Deploy target environment/namespace')
  }

  environment {
    AWS_REGION     = "eu-north-1"
    CLUSTER_NAME   = "myApp"

    APP_NAME       = "ecr-demo"
    SERVICE_NAME   = "ecr-demo-svc"
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
        sh 'echo "DEPLOY_ENV=$DEPLOY_ENV"'
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

    stage('Prod approval') {
      when { expression { return params.DEPLOY_ENV == 'prod' } }
      steps {
        input message: "Deploy ${IMAGE_TAG} to PROD?", ok: "Deploy to PROD"
      }
    }

    stage('Deploy to EKS (selected env)') {
      steps {
        withCredentials([
          [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds'],
          usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_TOKEN')
        ]) {
          sh '''
            set -euo pipefail

            aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"

            TARGET_NS="$DEPLOY_ENV"
            echo "Target namespace: $TARGET_NS"

            kubectl get ns "$TARGET_NS" >/dev/null 2>&1 || kubectl create ns "$TARGET_NS"

            DOCKER_IMAGE="docker.io/${DOCKERHUB_USERNAME}/${DOCKERHUB_REPO}:${IMAGE_TAG}"

            # Apply the manifest into the target namespace (don’t rely on the namespace inside YAML)
            sed "s/namespace: ecr-demo/namespace: $TARGET_NS/g" "$K8S_MANIFEST" | kubectl apply -f -

            # Ensure correct image (avoid REPLACED_BY_PIPELINE problems)
            kubectl set image -n "$TARGET_NS" deployment/"$APP_NAME" "$CONTAINER_NAME"="$DOCKER_IMAGE"
            kubectl rollout status -n "$TARGET_NS" deployment/"$APP_NAME"
          '''
        }
      }
    }

    stage('Smoke test (selected env)') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            set -euo pipefail
            aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"

            TARGET_NS="$DEPLOY_ENV"
            echo "Smoke testing namespace: $TARGET_NS"

            kubectl run -n "$TARGET_NS" curltest \
              --image=curlimages/curl --rm -i --restart=Never -- \
              curl -sS http://ecr-demo-svc | head -n 20
          '''
        }
      }
    }
  }

  post {
    success { echo "✅ Pipeline completed successfully" }

    failure {
      echo "❌ Pipeline failed – diagnostics"
      withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
        sh '''
          set +e
          aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || true
          kubectl get pods -n "$DEPLOY_ENV" -o wide || true
          kubectl get events -n "$DEPLOY_ENV" --sort-by=.metadata.creationTimestamp | tail -n 50 || true
        '''
      }
    }

    always { sh 'docker image prune -f || true' }
  }
}
