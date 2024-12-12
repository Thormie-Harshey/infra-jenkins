pipeline {
    agent {
        label 'node-ec'
    }

    environment {
        AWS_REGION = "your-aws-region"
        ECR_REPOSITORY = "your-ecr-repository-name"
        EKS_CLUSTER = "your-eks-cluster-name"
        REGISTRY = "your-registry-url"
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push to ECR') {
            agent {
                docker {
                    image 'docker:24.0.5-cli'
                    args '--privileged -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    sh "docker build -t ${REGISTRY}/${ECR_REPOSITORY}:latest ."
                    sh "docker tag ${REGISTRY}/${ECR_REPOSITORY}:latest ${REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${REGISTRY}"
                    sh "docker push ${REGISTRY}/${ECR_REPOSITORY}:latest"
                    sh "docker push ${REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    sh "aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}"
                    sh "kubectl get secret regcred || kubectl create secret docker-registry regcred --docker-server=${REGISTRY} --docker-username=AWS --docker-password=$(aws ecr get-login-password --region ${AWS_REGION})"
                    sh "helm upgrade --install webapp-stack helm/appcharts --namespace default --set appimage=${REGISTRY}/${ECR_REPOSITORY},apptag=${BUILD_NUMBER}"
                    sh "kubectl get pods -A"
                    sh "kubectl get ingress"
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
        }
    }
}