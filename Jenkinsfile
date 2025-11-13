pipeline {
    agent any

    environment {
        ECR_REGISTRY = "049061799945.dkr.ecr.ap-south-1.amazonaws.com"
        IMAGE_REPO   = "microservice-ecommerce"    // your ECR repo name
        IMAGE_TAG    = "${BUILD_NUMBER}"

        // THIS IS THE ONLY SERVICE THAT GIVES PUBLIC URL
        SERVICE_NAME = "frontend-external"         
        NAMESPACE    = "default"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Push All Images') {
            steps {
                // Build and push every microservice image (adjust if you have separate repos)
                sh """
                docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG} -f src/frontend/Dockerfile src/frontend || true
                docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG} -f src/emailservice/Dockerfile src/emailservice || true
                docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG} -f src/checkoutservice/Dockerfile src/checkoutservice || true
                # add more if you want...

                aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                docker push ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG} || true
                docker push ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG} || true
                docker push ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG} || true
                """
            }
        }

        stage('Update Manifest with New Tag') {
            steps {
                sh """
                sed -i 's|image: .*|image: ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG}|' kubernetes-manifest-file.yaml
                sed -i 's|image: .*|image: ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG}|' kubernetes-manifest-file.yaml
                sed -i 's|image: .*|image: ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG}|' kubernetes-manifest-file.yaml
                # add more services if needed
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f kubernetes-manifest-file.yaml'
            }
        }
    }

    post {
        success {
            script {
                echo "Waiting for your public LoadBalancer IP (max 4 minutes)..."
                timeout(time: 4, unit: 'MINUTES') {
                    waitUntil {
                        def ip = sh(script: "kubectl get svc ${SERVICE_NAME} -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].ip}' --ignore-not-found", returnStdout: true).trim()
                        return (ip != "")
                    }
                }

                def PUBLIC_IP = sh(script: "kubectl get svc ${SERVICE_NAME} -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'", returnStdout: true).trim()

                echo "=================================================================="
                echo "   YOUR ONLINE BOUTIQUE E-COMMERCE APP IS LIVE!"
                echo ""
                echo "   PUBLIC URL: http://${PUBLIC_IP}"
                echo ""
                echo "   Click or copy-paste this URL in your browser"
                echo "=================================================================="
            }
        }
    }
}
