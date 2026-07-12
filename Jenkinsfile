pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO = '000540852129.dkr.ecr.us-east-1.amazonaws.com/techchallenge2'
        CLUSTER_NAME = 'techchallenge2-cluster'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${ECR_REPO}:${BUILD_NUMBER} ."
                sh "docker tag ${ECR_REPO}:${BUILD_NUMBER} ${ECR_REPO}:latest"
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                        docker push ${ECR_REPO}:${BUILD_NUMBER}
                        docker push ${ECR_REPO}:latest
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh """
                        aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
                        kubectl set image deployment/techchallenge2 techchallenge2=${ECR_REPO}:${BUILD_NUMBER}
                        kubectl rollout status deployment/techchallenge2
                    """
                }
            }
        }
    }
}
