// Simple Node.js CI/CD demo pipeline
// Flow: Checkout -> Test -> Docker Build -> Docker Push (DockerHub) -> Deploy (Kubernetes/Minikube) -> Verify
//
// One-time setup before running:
//   1. Jenkins > Manage Jenkins > Credentials > add "Username with password"
//      with ID "DOCKERHUB_CREDENTIALS" (your DockerHub username + access token)
//   2. Update IMAGE_NAME below to <your-dockerhub-username>/ibm-cicd-demo
//   3. Run `kubectl apply -f k8s/` once manually so the Deployment/Service exist
//      before this pipeline tries to `kubectl set image` on it
//   4. Jenkins agent (native install) needs docker, kubectl, and node on PATH,
//      and its kubectl context must already point at docker-desktop

pipeline {
    agent any

    //     parameters {
    //     booleanParam(name: 'PUSH_TO_DOCKERHUB', defaultValue: false, description: 'Push image to Docker Hub')
    // }

    environment {
        IMAGE_NAME = "vishal/ibm-cicd-demo"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/vishal-singh24/ibm-cicd-demo-4-jun-2026.git'
            }
        }

        stage('Install & Test') {
            steps {
                dir('app') {
                    bat 'npm install'
                    bat 'npm test'
                }
            }
        }

        stage('Docker Build') {
            steps {
                dir('app') {
                    bat "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
                }
            }
        }

        stage('Docker Push') {
            // when {
            //     expression { return params.PUSH_TO_DOCKERHUB == true }
            // }
            steps {
                withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_CREDENTIALS', usernameVariable: 'DOCKERHUB_CREDENTIALS_USR', passwordVariable: 'DOCKERHUB_CREDENTIALS_PSW')]) {
                    // bat "echo %DOCKERHUB_CREDENTIALS_PSW% | docker login -u %DOCKERHUB_CREDENTIALS_USR% --password-stdin"
                    bat '''
                    @echo off
                    powershell -Command "$env:DOCKERHUB_CREDENTIALS_PSW | docker login -u $env:DOCKERHUB_CREDENTIALS_USR --password-stdin"
                    '''
                    bat "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    bat "docker push ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Mirrors PUSH_TO_DOCKERHUB: if we didn't push, the image only
                    // exists locally, so tell Kubernetes to never attempt a registry
                    // pull -- just use whatever's already on the node.
                    def pullPolicy = params.PUSH_TO_DOCKERHUB ? 'IfNotPresent' : 'Never'
                    writeFile file: 'pull-policy-patch.yaml', text: """spec:
  template:
    spec:
      containers:
      - name: ibm-cicd-demo
        imagePullPolicy: ${pullPolicy}
"""
                }
                bat "kubectl patch deployment ibm-cicd-demo --patch-file=pull-policy-patch.yaml"
                bat "kubectl set image deployment/ibm-cicd-demo ibm-cicd-demo=${IMAGE_NAME}:${IMAGE_TAG} --record"
                bat "kubectl rollout status deployment/ibm-cicd-demo --timeout=90s"
            }
        }

        stage('Verify') {
            steps {
                bat "kubectl get pods -l app=ibm-cicd-demo"
                // Docker Desktop's Kubernetes exposes NodePort services on localhost directly
                bat "curl -s http://localhost:30080/health || true"
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded -- ${IMAGE_NAME}:${IMAGE_TAG} is live on Kubernetes"
        }
        failure {
            echo "Pipeline failed -- check the stage logs above"
        }
    }
}
