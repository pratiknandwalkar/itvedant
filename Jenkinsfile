pipeline {
    agent any

    environment {
        AWS_REGION   = 'ap-south-1'                 // change to your region
        AWS_ACCOUNT  = '123456789012'               // change to your 12-digit account ID
        ECR_REPO     = 'my-app'                     // ECR repository name
        EKS_CLUSTER  = 'my-eks-cluster'             // EKS cluster name
        K8S_NAMESPACE = 'default'
        ECR_REGISTRY = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_TAG    = "${env.BUILD_NUMBER}"
        IMAGE_URI    = "${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Run Tests') {
            steps {
                // Runs tests inside a throwaway Node container (swap for your stack)
                sh '''
                    docker run --rm -v "$PWD":/app -w /app node:20-alpine \
                      sh -c "npm ci && npm test"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_URI} -t ${ECR_REGISTRY}/${ECR_REPO}:latest .'
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} \
                      | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker push ${IMAGE_URI}
                    docker push ${ECR_REGISTRY}/${ECR_REPO}:latest
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                    sed "s|IMAGE_PLACEHOLDER|${IMAGE_URI}|g" k8s/deployment.yaml > k8s/deployment.rendered.yaml
                    kubectl apply -n ${K8S_NAMESPACE} -f k8s/deployment.rendered.yaml
                    kubectl apply -n ${K8S_NAMESPACE} -f k8s/service.yaml
                    kubectl rollout status deployment/my-app -n ${K8S_NAMESPACE} --timeout=180s
                '''
            }
        }
    }

    post {
        success { echo "Deployed ${IMAGE_URI}" }
        failure { echo 'Pipeline failed - check the stage logs above.' }
        always  { sh 'docker image prune -f || true' }
    }
}
