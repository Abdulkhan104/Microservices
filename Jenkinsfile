pipeline {
    agent any

    environment {
        ECR_REGISTRY = "049061799945.dkr.ecr.ap-south-1.amazonaws.com"
        IMAGE_REPO   = "microservice-ecommerce"
        IMAGE_TAG    = "${BUILD_NUMBER}"
        SERVICE_NAME = "frontend-external"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Push Images') {
            steps {
                // These two credential IDs exist in your Jenkins exactly as shown in the screenshot
                withCredentials([
                    usernamePassword(credentialsId: 'access-key',          usernameVariable: 'AWS_ACCESS_KEY_ID',     passwordVariable: 'DUMMY'),
                    usernamePassword(credentialsId: 'aws-ecr-credentials', usernameVariable: 'DUMMY',                 passwordVariable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                    # Login to ECR
                    aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    # Build and push
                    docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG} ./src/frontend
                    docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG} ./src/emailservice
                    docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG} ./src/checkoutservice

                    docker push ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                sed -i "s|image: .*frontend.*|image: ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG}|" kubernetes-manifest-file.yaml
                sed -i "s|image: .*emailservice.*|image: ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG}|" kubernetes-manifest-file.yaml
                sed -i "s|image: .*checkoutservice.*|image: ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG}|" kubernetes-manifest-file.yaml

                kubectl apply -f kubernetes-manifest-file.yaml
                '''
            }
        }
    }

    post {
        success {
            script {
                timeout(time: 5, unit: 'MINUTES') {
                    waitUntil {
                        sh(script: "kubectl get svc frontend-external --ignore-not-found -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim() != ""
                    }
                }
                def URL = sh(script: "kubectl get svc frontend-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                echo "══════════════════════════════════════════════════════════"
                echo "   YOUR E-COMMERCE SITE IS LIVE!"
                echo "   URL: http://${URL}"
                echo "   Open it now!"
                echo "══════════════════════════════════════════════════════════"
            }
        }
    }
}
