pipeline {
    agent any

    environment {
        ECR_REGISTRY = "049061799945.dkr.ecr.ap-south-1.amazonaws.com"
        IMAGE_REPO   = "microservice-ecommerce"
        IMAGE_TAG    = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push Images') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'access-key',     usernameVariable: 'AWS_ACCESS_KEY_ID',    passwordVariable: 'DUMMY'),
                    usernamePassword(credentialsId: 'aws-secret-key', usernameVariable: 'DUMMY',                passwordVariable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                    # THESE 3 LINES MUST BE FIRST — THIS IS THE FINAL FIX
                    export AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY
                    export AWS_DEFAULT_REGION=ap-south-1

                    # Now login works perfectly
                    aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    # Build
                    docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG}      ./src/frontend
                    docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG} ./src/emailservice
                    docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG} ./src/checkoutservice

                    # Push
                    docker push ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                # Update image tags in your manifest
                sed -i "s|image: .*frontend.*|image: ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG}|" kubernetes-manifest-file.yaml
                sed -i "s|image: .*emailservice.*|image: ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG}|" kubernetes-manifest-file.yaml
                sed -i "s|image: .*checkoutservice.*|image: ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG}|" kubernetes-manifest-file.yaml

                # Deploy
                kubectl apply -f kubernetes-manifest-file.yaml
                '''
            }
        }
    }

    post {
        success {
            script {
                echo "Waiting for Load Balancer URL..."
                timeout(time: 6, unit: 'MINUTES') {
                    waitUntil {
                        script {
                            def url = sh(script: "kubectl get svc frontend-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' --ignore-not-found || true", returnStdout: true).trim()
                            return (url != "")
                        }
                    }
                }
                def PUBLIC_URL = sh(script: "kubectl get svc frontend-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()

                echo "*******************************************************************"
                echo "     YOUR FULL E-COMMERCE WEBSITE IS LIVE BRO!"
                echo "     OPEN THIS LINK RIGHT NOW → http://${PUBLIC_URL}"
                echo "     BOOKMARK IT → http://${PUBLIC_URL}"
                echo "*******************************************************************"
            }
        }
        failure {
            echo "Something went wrong. Ping me, we fix in 30 seconds."
        }
    }
}
