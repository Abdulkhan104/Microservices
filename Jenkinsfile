pipeline {
    agent any

    environment {
        ECR_REGISTRY = "049061799945.dkr.ecr.ap-south-1.amazonaws.com"
        IMAGE_REPO   = "microservice-ecommerce"
        IMAGE_TAG    = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') { steps { checkout scm } }

        stage('Build & Push Images') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'access-key',     usernameVariable: 'AWS_ACCESS_KEY_ID',    passwordVariable: 'DUMMY'),
                    usernamePassword(credentialsId: 'aws-secret-key', usernameVariable: 'DUMMY',                passwordVariable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                    # THIS IS THE ONLY FIX — export the variables so AWS CLI sees them
                    export AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY
                    export AWS_DEFAULT_REGION=ap-south-1

                    aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${ECR_REGISTRY}

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
                echo "  YOUR FULL E-COMMERCE SITE IS NOW LIVE!"
                echo "  OPEN THIS URL → http://${URL}"
                echo "══════════════════════════════════════════════════════════"
            }
        }
    }
}
