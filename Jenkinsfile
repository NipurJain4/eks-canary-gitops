// Jenkins CI pipeline for the canary practice project.
//
// Flow (GitOps):
//   1. Build the app image.
//   2. Push it to ECR.
//   3. Update the image tag in the GitOps repo (apps/podinfo/deployment.yaml)
//      and commit/push. Flux then deploys it and Flagger runs the canary.
//
// Jenkins does NOT run `kubectl apply` — Flux owns the cluster (clean GitOps).
//
// Required Jenkins credentials:
//   - aws-creds        : AWS access key/secret (or use an IAM instance role)
//   - github-token     : GitHub PAT for pushing the tag commit
//
// Required parameters / env (edit ACCOUNT_ID + REGION):
pipeline {
  agent any

  environment {
    AWS_REGION   = 'ap-south-1'
    ACCOUNT_ID   = '<ACCOUNT_ID>'            // <-- fill in your 12-digit account id
    ECR_REPO     = 'podinfo'
    IMAGE_TAG    = "${env.BUILD_NUMBER}"
    REGISTRY     = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    GITOPS_REPO  = 'github.com/<your-org>/eks-canary-gitops.git'   // <-- your repo
    MANIFEST     = 'apps/podinfo/deployment.yaml'
  }

  stages {
    stage('Checkout app source') {
      steps {
        checkout scm
      }
    }

    stage('Build image') {
      steps {
        sh 'docker build -t $ECR_REPO:$IMAGE_TAG .'
      }
    }

    stage('Login + Push to ECR') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            aws ecr get-login-password --region $AWS_REGION | \
              docker login --username AWS --password-stdin $REGISTRY
            docker tag  $ECR_REPO:$IMAGE_TAG $REGISTRY/$ECR_REPO:$IMAGE_TAG
            docker push $REGISTRY/$ECR_REPO:$IMAGE_TAG
          '''
        }
      }
    }

    stage('Bump image tag in GitOps repo') {
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
          sh '''
            rm -rf gitops && git clone https://$GH_TOKEN@$GITOPS_REPO gitops
            cd gitops
            # Replace the image line with the freshly pushed tag.
            sed -i "s#image: .*/podinfo:.*#image: $REGISTRY/$ECR_REPO:$IMAGE_TAG#" $MANIFEST
            git config user.email "jenkins@ci"
            git config user.name "jenkins"
            git commit -am "ci: podinfo image -> $IMAGE_TAG (build $BUILD_NUMBER)"
            git push origin main
          '''
        }
      }
    }
  }

  post {
    success {
      echo "Pushed $REGISTRY/$ECR_REPO:$IMAGE_TAG and committed to GitOps. Flux + Flagger will roll out the canary."
    }
  }
}
