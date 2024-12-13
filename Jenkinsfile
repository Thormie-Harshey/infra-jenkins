pipeline {
    agent {
        label 'node-ec'
    }

    /*environment {
        S3_BUCKET = "your-s3-bucket-name"
        AWS_REGION = "your-aws-region"
        EKS_CLUSTER = "your-eks-cluster-name"
        ECR_REPO = "your-ecr-repository-name"
    }*/

    parameters {
        choice(
            name: 'ACTION',
            choices: ['apply', 'destroy'],
            description: 'Choose the action to perform: apply (default) or destroy.'
        )
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('Setup Terraform') {
            steps {
                sh 'terraform -version'
            }
        }

        stage('Terraform Init') {
            steps {
                dir('terraform') {
                    sh 'terraform init'
                }
            }
        }

        stage('Apply or Destroy') {
            when {
                expression { params.ACTION == 'apply' || params.ACTION == 'destroy' }
            }
            steps {
                script {
                    if (params.ACTION == 'apply') {
                        dir('terraform') {
                            sh 'terraform validate'
                            sh 'terraform plan -no-color -input=false -out planfile'
                            sh 'terraform apply -auto-approve -input=false -parallelism=1 planfile'
                        }
                        sh "aws eks update-kubeconfig --name ${env.EKS_CLUSTER} --region ${env.AWS_REGION}"
                        sh 'kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0-beta.0/deploy/static/provider/aws/deploy.yaml'
                    
                    } else if (params.ACTION == 'destroy') {
                        sh 'kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0-beta.0/deploy/static/provider/aws/deploy.yaml || true'
                        script {
                            def images = sh(
                                script: "aws ecr list-images --repository-name ${env.ECR_REPO} --query 'imageIds' --output json",
                                returnStdout: true
                            ).trim()

                            if (images != '[]') {
                                sh "aws ecr batch-delete-image --repository-name ${env.ECR_REPO} --image-ids '${images}'"
                            } else {
                                echo "No images found in ${env.ECR_REPO}."
                            }
                        }
                        dir('terraform') {
                            sh 'terraform destroy -auto-approve -input=false'
                        }
                    }
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
