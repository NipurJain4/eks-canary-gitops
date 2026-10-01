// Jenkins pipeline — in-cluster CI for recharge-system.
//
// Flow (GitOps):
//   1. Build the image with KANIKO (no Docker daemon needed in K8s).
//   2. Push to ECR — auth via the NODE role (SCP-excluded, auto-refreshing;
//      no stored AWS keys, nothing to expire).
//   3. Update the image tag in the GitOps repo (apps/recharge-system/deployment.yaml)
//      and commit. Flux + Flagger then deploy + run the canary.
//
// Jenkins never runs `kubectl apply` — it only commits to Git (clean GitOps).
//
// Prerequisites configured in Jenkins:
//   - Credential 'github-token'  : GitHub PAT (repo scope) for pushing the tag commit
//
// Runs on a Kubernetes agent pod with a kaniko container + a git/aws container.

pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-agent
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:v1.23.2-debug
      command: ["/busybox/cat"]
      tty: true
    - name: tools
      image: amazon/aws-cli:2.17.0
      command: ["cat"]
      tty: true
'''
    }
  }

  environment {
    AWS_REGION  = 'ap-south-1'
    ACCOUNT_ID  = '821410798987'
    ECR_REPO    = 'podinfo'
    REGISTRY    = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    IMAGE_TAG   = "recharge-b${env.BUILD_NUMBER}"
    APP_REPO    = 'https://github.com/NipurJain4/recharge-system.git'
    GITOPS_REPO = 'github.com/NipurJain4/eks-canary-gitops.git'
    MANIFEST    = 'apps/recharge-system/deployment.yaml'
  }

  stages {
    stage('Checkout app source') {
      steps {
        git branch: 'main', url: "${APP_REPO}"
      }
    }

    stage('Build & push image (Kaniko)') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --context `pwd` \
              --dockerfile Dockerfile \
              --destination ${REGISTRY}/${ECR_REPO}:${IMAGE_TAG} \
              --verbosity info
          '''
        }
      }
    }

    stage('Bump image tag in GitOps repo') {
      steps {
        container('tools') {
          withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
            sh '''
              yum install -y git >/dev/null 2>&1 || true
              rm -rf gitops && git clone https://$GH_TOKEN@$GITOPS_REPO gitops
              cd gitops
              sed -i "s#${ECR_REPO}:recharge-[a-zA-Z0-9._-]*#${ECR_REPO}:${IMAGE_TAG}#" ${MANIFEST}
              git config user.email "jenkins@ci"
              git config user.name "jenkins-ci"
              git commit -am "ci: recharge-system image -> ${IMAGE_TAG} (build ${BUILD_NUMBER})"
              git push origin main
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo "Pushed ${REGISTRY}/${ECR_REPO}:${IMAGE_TAG} and committed to GitOps. Flux + Flagger will run the canary."
    }
  }
}
