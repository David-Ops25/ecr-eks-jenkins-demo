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

    stage('Show environment') {
      steps {
        sh '''
          echo "BUILD_NUMBER=${BUILD_NUMBER}"
          echo "GIT_COMMIT=${GIT_COMMIT:-N/A}"
          echo "AWS_REGION=${AWS_REGION}"
          echo "CLUSTER_NAME=${CLUSTER_NAME}"
          echo "K8S_NAMESPACE=${K8S_NAMESPACE}"
          echo "DOCKERHUB_REPO=${DOCKERHUB_REPO}"
        '''
      }
    }

    stage('Validate files') {
      steps {
        sh '''
          test -f Dockerfile
          test -f index.html
          test -f "${K8S_MANIFEST}"

          echo "=== Dockerfile ==="
          cat Dockerfile

          echo "=== Kubernetes manifest ==="
          cat "${K8S_MANIFEST}"
        '''
      }
    }

    stage('Verify tools') {
      steps {
        sh '''
          aws --version
          kubectl version --client
          docker --version
        '''
      }
    }

    stage('Compute image tag') {
      steps {
        script {
          def shortSha = env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : "nogit"
          env.IMAGE_TAG = "${env.BUILD_NUMBER}-${shortSha}"
        }
        sh 'echo "IMAGE_TAG=${IMAGE_TAG}"'
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
            DOCKER_IMAGE="docker.io/${DOCKERHUB_USERNAME}/${DOCKERHUB_REPO}:${IMAGE_TAG}"
            docker build -t "$DOCKER_IMAGE" .
            docker image ls | head
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
            DOCKER_IMAGE="docker.io/${DOCKERHUB_USERNAME}/${DOCKERHUB_REPO}:${IMAGE_TAG}"
            docker push "$DOCKER_IMAGE"
          '''
        }
      }
    }

    stage('AWS auth sanity') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh 'aws sts get-caller-identity'
        }
      }
    }

    stage('Configure kubectl') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
            kubectl get nodes
          '''
        }
      }
    }

    stage('Pre-deploy checks') {
      steps {
        sh '''
          kubectl get ns "$K8S_NAMESPACE" || kubectl create ns "$K8S_NAMESPACE"
          kubectl get secret -n "$K8S_NAMESPACE" dockerhub-regcred
          kubectl get deploy -n "$K8S_NAMESPACE" "$DEPLOYMENT" || true
        '''
      }
    }

    stage('Apply manifests') {
      steps {
        sh 'kubectl apply -f "${K8S_MANIFEST}"'
      }
    }

    stage('Rolling deploy') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKERHUB_USERNAME',
          passwordVariable: 'DOCKERHUB_TOKEN'
        )]) {
          sh '''
            DOCKER_IMAGE="docker.io/${DOCKERHUB_USERNAME}/${DOCKERHUB_REPO}:${IMAGE_TAG}"
            kubectl set image -n "$K8S_NAMESPACE" deployment/"$DEPLOYMENT" \
              "$CONTAINER_NAME"="$DOCKER_IMAGE"
            kubectl rollout status -n "$K8S_NAMESPACE" deployment/"$DEPLOYMENT"
          '''
        }
      }
    }

    stage('Post-deploy verification') {
      steps {
        sh '''
          kubectl get pods -n "$K8S_NAMESPACE" -o wide
          kubectl get svc -n "$K8S_NAMESPACE" -o wide
          kubectl get deploy -n "$K8S_NAMESPACE" "$DEPLOYMENT" \
            -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
        '''
      }
    }

    stage('Smoke test') {
      steps {
        sh '''
          kubectl run -n "$K8S_NAMESPACE" curltest \
            --image=curlimages/curl --rm -i --restart=Never -- \
            curl -sS http://ecr-demo-svc | head -n 20
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline completed successfully"
    }

    failure {
      echo "❌ Pipeline failed – collecting diagnostics"
      sh '''
        kubectl get pods -n "$K8S_NAMESPACE" -o wide || true
        kubectl get events -n "$K8S_NAMESPACE" \
          --sort-by=.metadata.creationTimestamp | tail -n 50 || true
      '''
    }

    always {
      sh 'docker image prune -f || true'
    }
  }
}
