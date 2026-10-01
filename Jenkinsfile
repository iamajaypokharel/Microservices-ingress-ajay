pipeline {
    agent any

    environment {
        DOCKER_HUB_REPO = 'ajaypokharel444/microservice-app-ajayman'
        K8S_CLUSTER_NAME = 'kastro-cluster'
        AWS_REGION = 'us-east-1'
        NAMESPACE = 'default'
        APP_NAME = 'techsolutions'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'

                git branch: 'main',
                    url: 'https://github.com/iamajaypokharel/Microservices-ingress-ajay.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'

                script {
                    def buildNumber = env.BUILD_NUMBER
                    def imageTag = "${DOCKER_HUB_REPO}:${buildNumber}"
                    def latestTag = "${DOCKER_HUB_REPO}:latest"

                    sh "docker build -t ${imageTag} ."
                    sh "docker tag ${imageTag} ${latestTag}"

                    env.IMAGE_TAG = buildNumber

                    echo "Docker image: ${imageTag}"
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo 'Pushing Docker image to DockerHub...'

                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-credentials',
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )
                    ]) {

                        sh '''
                            echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin
                        '''

                        sh "docker push ${DOCKER_HUB_REPO}:${env.IMAGE_TAG}"
                        sh "docker push ${DOCKER_HUB_REPO}:latest"
                    }
                }
            }
        }

        stage('Configure AWS and Kubectl') {
            steps {
                echo 'Configuring AWS CLI and kubectl...'

                script {
                    withCredentials([
                        [$class: 'AmazonWebServicesCredentialsBinding',
                         credentialsId: 'aws-creds']
                    ]) {

                        sh "aws configure set region ${AWS_REGION}"

                        sh """
                            aws eks update-kubeconfig \
                            --region ${AWS_REGION} \
                            --name ${K8S_CLUSTER_NAME}
                        """

                        sh "kubectl config current-context"
                        sh "kubectl get nodes"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying application to Kubernetes...'

                script {
                    withCredentials([
                        [$class: 'AmazonWebServicesCredentialsBinding',
                         credentialsId: 'aws-creds']
                    ]) {

                        /*
                         * Replace the image in deployment.yaml
                         * with the image built by this Jenkins build.
                         */

                        sh """
                            sed -i 's|image:.*|image: ${DOCKER_HUB_REPO}:${env.IMAGE_TAG}|' k8s/deployment.yaml
                        """

                        sh "kubectl apply -f k8s/deployment.yaml"

                        sh """
                            kubectl rollout status \
                            deployment/${APP_NAME}-deployment \
                            --timeout=300s
                        """

                        sh "kubectl get pods -l app=${APP_NAME}"
                        sh "kubectl get svc ${APP_NAME}-service"
                    }
                }
            }
        }

        stage('Deploy Ingress') {
            steps {
                echo 'Deploying Ingress resource...'

                script {
                    withCredentials([
                        [$class: 'AmazonWebServicesCredentialsBinding',
                         credentialsId: 'aws-creds']
                    ]) {

                        sh "kubectl apply -f k8s/ingress.yaml"

                        sleep(10)

                        sh "kubectl get ingress ${APP_NAME}-ingress"

                        sh "kubectl describe ingress ${APP_NAME}-ingress"
                    }
                }
            }
        }

        stage('Get Ingress URL') {
            steps {
                echo 'Getting Ingress URL...'

                script {
                    withCredentials([
                        [$class: 'AmazonWebServicesCredentialsBinding',
                         credentialsId: 'aws-creds']
                    ]) {

                        timeout(time: 10, unit: 'MINUTES') {

                            waitUntil {

                                script {

                                    def result = sh(
                                        script: """
                                            kubectl get svc \
                                            ingress-nginx-controller \
                                            -n ingress-nginx \
                                            -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
                                        """,
                                        returnStdout: true
                                    ).trim()

                                    if (result && result != '') {

                                        env.INGRESS_URL = "http://${result}"

                                        echo "Ingress URL: ${env.INGRESS_URL}"

                                        return true
                                    }

                                    echo "Waiting for LoadBalancer..."

                                    return false
                                }
                            }
                        }

                        echo "========================================="
                        echo "DEPLOYMENT SUCCESSFUL!"
                        echo "========================================="

                        echo "Application URL: ${env.INGRESS_URL}"

                        echo ""
                        echo "Available Paths:"
                        echo "- Home Page: ${env.INGRESS_URL}/"
                        echo "- About Page: ${env.INGRESS_URL}/about"
                        echo "- Services Page: ${env.INGRESS_URL}/services"
                        echo "- Contact Page: ${env.INGRESS_URL}/contact"

                        echo "========================================="

                        sh "curl -I ${env.INGRESS_URL}/ || true"
                        sh "curl -I ${env.INGRESS_URL}/about || true"
                        sh "curl -I ${env.INGRESS_URL}/services || true"
                        sh "curl -I ${env.INGRESS_URL}/contact || true"
                    }
                }
            }
        }
    }

    post {

        always {
            echo 'Cleaning up Docker images...'

            script {
                if (env.IMAGE_TAG) {
                    sh "docker rmi ${DOCKER_HUB_REPO}:${env.IMAGE_TAG} || true"
                }

                sh "docker rmi ${DOCKER_HUB_REPO}:latest || true"
            }
        }

        success {
            echo 'Pipeline completed successfully!'
            echo "Access your application at: ${env.INGRESS_URL}"
        }

        failure {
            echo 'Pipeline failed! Please check the logs.'
        }
    }
}

