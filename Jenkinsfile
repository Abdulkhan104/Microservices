pipeline {
    agent any

    environment {
        ECR_REGISTRY = "049061799945.dkr.ecr.ap-south-1.amazonaws.com"
        IMAGE_REPO   = "microservice-ecommerce"
        IMAGE_TAG    = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Push Images') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'access-key',     usernameVariable: 'AWS_ACCESS_KEY_ID',    passwordVariable: 'DUMMY'),
                    usernamePassword(credentialsId: 'aws-secret-key', usernameVariable: 'DUMMY',                passwordVariable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                    # THIS LINE IS THE MAGIC FIX — RELOAD ENV SO AWS CLI SEES THE VARIABLES
                    set +o history
                    export AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID"
                    export AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY"
                    export AWS_DEFAULT_REGION=ap-south-1

                    # Now login WILL work
                    aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    # Build & Push
                    docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG}      ./src/frontend
                    docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG} ./src/emailservice
                    docker build -t ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG} ./src/checkoutservice

                    docker push ${ECR_REGISTRY}/${IMAGE_REPO}:frontend-${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${IMAGE_REPO}:emailservice-${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${IMAGE_REPO}:checkoutservice-${IMAGE_TAG}

                    echo "ALL IMAGES PUSHED SUCCESSFULLY!"
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
                echo "DEPLOYED SUCCESSFULLY!"
                '''
            }
        }
    }

    post {
        success {
            script {
                timeout(time: 6, unit: 'MINUTES') {
                    waitUntil {
                        sh(script: "kubectl get svc frontend-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' --ignore-not-found", returnStdout: true).trim() != ""
                    }
                }
                def URL = sh(script: "kubectl get svc frontend-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                echo "YOUR E-COMMERCE SITE IS LIVE BRO!"
                echo "OPEN THIS → http://${URL}"
                echo "http://${URL}"
            }
        }
    }
}
