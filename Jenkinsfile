pipeline {

    agent any

    environment {
        AWS_ACCOUNT_ID  = "REPLACE_WITH_AWS_ACCOUNT_ID"
        AWS_REGION      = "ap-south-1"
        ECR_REPO_NAME   = "smart-hostel-website"
        PROD_EC2_IP     = "REPLACE_WITH_PROD_ELASTIC_IP"
        PROD_EC2_USER   = "ubuntu"
        CONTAINER_NAME  = "smart-hostel-app"
        APP_PORT        = "3000"
        ECR_REGISTRY    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME      = "${ECR_REGISTRY}/${ECR_REPO_NAME}"
        IMAGE_TAG_BUILD = "${BUILD_NUMBER}"
        IMAGE_TAG_LATEST= "latest"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                cleanWs()
                git(
                    url: 'https://github.com/28092005/SMART_HOSTEL_WEBSITE.git',
                    branch: 'main',
                    credentialsId: 'github-credentials'
                )
                echo "✅ Checkout done"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci --frozen-lockfile'
                echo "✅ Dependencies installed"
            }
        }

        stage('Docker Build and Tag') {
            steps {
                script {
                    sh """
                        docker build \
                            -t ${IMAGE_NAME}:${IMAGE_TAG_BUILD} \
                            -t ${IMAGE_NAME}:${IMAGE_TAG_LATEST} \
                            .
                    """
                }
                echo "✅ Docker image built — build #${BUILD_NUMBER}"
            }
        }

        stage('Push to AWS ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            docker push ${IMAGE_NAME}:${IMAGE_TAG_BUILD}
                            docker push ${IMAGE_NAME}:${IMAGE_TAG_LATEST}
                        """
                    }
                }
                echo "✅ Images pushed to ECR"
            }
        }

        stage('Deploy to Production EC2') {
            steps {
                withCredentials([
                    string(credentialsId: 'mongodb-uri', variable: 'MONGODB_URI'),
                    string(credentialsId: 'jwt-secret',  variable: 'JWT_SECRET'),
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials',
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {
                    sshagent(credentials: ['prod-ec2-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${PROD_EC2_USER}@${PROD_EC2_IP} '
                                export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
                                export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}"
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                docker pull ${IMAGE_NAME}:${IMAGE_TAG_LATEST}
                                docker stop ${CONTAINER_NAME} 2>/dev/null || true
                                docker rm   ${CONTAINER_NAME} 2>/dev/null || true
                                docker run -d \
                                    --name ${CONTAINER_NAME} \
                                    --restart always \
                                    -p ${APP_PORT}:3000 \
                                    -e NODE_ENV=production \
                                    -e PORT=3000 \
                                    -e MONGODB_URI="${MONGODB_URI}" \
                                    -e JWT_SECRET="${JWT_SECRET}" \
                                    ${IMAGE_NAME}:${IMAGE_TAG_LATEST}
                                docker image prune -f
                            '
                        """
                    }
                }
                echo "✅ Deployed to ${PROD_EC2_IP}:${APP_PORT}"
            }
        }

        stage('Smoke Test') {
            steps {
                script {
                    sh """
                        sleep 10
                        curl -f http://${PROD_EC2_IP}:${APP_PORT}/health && echo "✅ App is live!" || echo "⚠️ Check app logs"
                    """
                }
            }
        }

    }

    post {
        success {
            echo "✅ BUILD #${BUILD_NUMBER} SUCCEEDED — http://${PROD_EC2_IP}:${APP_PORT}"
        }
        failure {
            echo "❌ BUILD #${BUILD_NUMBER} FAILED — check console output"
        }
        always {
            sh """
                docker rmi ${IMAGE_NAME}:${IMAGE_TAG_BUILD}  2>/dev/null || true
                docker rmi ${IMAGE_NAME}:${IMAGE_TAG_LATEST} 2>/dev/null || true
            """
        }
    }
}
