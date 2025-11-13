pipeline {
    agent any

    environment {
        ECR_REGISTRY = "049061799945.dkr.ecr.ap-south-1.amazonaws.com"
        IMAGE_REPO   = "microservice-ecommerce"
        IMAGE_TAG    = "${BUILD_NUMBER}"
        SERVICE_NAME = "frontend-external"
        NAMESPACE    = "default"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Push Images') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-ecr-credentials',
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh '''
                    # Login to ECR
                    aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    # Build and push the main services
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

        stage('Update Manifest & Deploy') {
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
                echo "Waiting for your site to be live..."
                timeout(time: 5, unit: 'MINUTES') {
                    waitUntil {
                        def url = sh(script: "kubectl get svc frontend-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' --ignore-not-found", returnStdout: true).trim()
                        return (url != "")
                    }
                }
                def URL = sh(script: "kubectl get svc frontend-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                echo "══════════════════════════════════════════════════════════"
                echo "  YOUR FULL E-COMMERCE SITE IS NOW LIVE!"
                echo "  → http://${URL}"
                echo "  Click or copy the link above!"
                echo "══════════════════════════════════════════════════════════"
            }
        }
    }
}
