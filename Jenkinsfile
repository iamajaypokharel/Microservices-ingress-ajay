pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
    }

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
                script {
                    def imageTag = "${DOCKER_HUB_REPO}:${BUILD_NUMBER}"
                    def latestTag = "${DOCKER_HUB_REPO}:latest"

                    echo "Building ${imageTag}"

                    sh "docker build -t ${imageTag} ."
                    sh "docker tag ${imageTag} ${latestTag}"

                    env.IMAGE_TAG = BUILD_NUMBER

                    echo "Docker image built successfully."
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
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
                                --username "$DOCKER_USERNAME" \
                                --password-stdin
                        '''

                        sh "docker push ${DOCKER_HUB_REPO}:${IMAGE_TAG}"
                        sh "docker push ${DOCKER_HUB_REPO}:latest"
                    }
                }
            }
        }

        stage('Configure AWS and Kubectl') {
            steps {
                script {
                    withCredentials([
                        [$class: 'AmazonWebServicesCredentialsBinding',
                         credentialsId: 'aws-creds']
                    ]) {

                        sh """
                            aws eks update-kubeconfig \
                                --region ${AWS_REGION} \
                                --name ${K8S_CLUSTER_NAME}
                        """

                        sh 'kubectl config current-context'
                        sh 'kubectl get nodes'
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Deploying image ${DOCKER_HUB_REPO}:${IMAGE_TAG}"

                    sh """
                        sed -i 's|image:.*|image: ${DOCKER_HUB_REPO}:${IMAGE_TAG}|' k8s/deployment.yaml
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

        stage('Deploy Ingress') {
            steps {
                script {
                    sh 'kubectl apply -f k8s/ingress.yaml'

                    sleep(time: 10, unit: 'SECONDS')

                    sh "kubectl get ingress ${APP_NAME}-ingress"
                    sh "kubectl describe ingress ${APP_NAME}-ingress"
                }
            }
        }

        stage('Get Ingress URL') {
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {

                        waitUntil {

                            def result = sh(
                                script: """
                                    kubectl get svc \
                                        ingress-nginx-controller \
                                        -n ingress-nginx \
                                        -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
                                """,
                                returnStdout: true
                            ).trim()

                            if (result) {
                                env.INGRESS_URL = "http://${result}"

                                echo "Ingress URL: ${env.INGRESS_URL}"

                                return true
                            }

                            echo 'Waiting for AWS LoadBalancer...'
                            sleep(time: 10, unit: 'SECONDS')

                            return false
                        }
                    }

                    echo '========================================='
                    echo 'DEPLOYMENT SUCCESSFUL!'
                    echo '========================================='
                    echo "Application URL: ${env.INGRESS_URL}"
                    echo "Home:     ${env.INGRESS_URL}/"
                    echo "About:    ${env.INGRESS_URL}/about"
                    echo "Services: ${env.INGRESS_URL}/services"
                    echo "Contact:  ${env.INGRESS_URL}/contact"
                    echo '========================================='

                    sh "curl -I --max-time 15 ${env.INGRESS_URL}/ || true"
                    sh "curl -I --max-time 15 ${env.INGRESS_URL}/about || true"
                    sh "curl -I --max-time 15 ${env.INGRESS_URL}/services || true"
                    sh "curl -I --max-time 15 ${env.INGRESS_URL}/contact || true"
                }
            }
        }
    }

    post {

        always {
            echo 'Cleaning up local Docker images...'

            script {
                sh "docker rmi ${DOCKER_HUB_REPO}:${IMAGE_TAG} || true"
                sh "docker rmi ${DOCKER_HUB_REPO}:latest || true"
            }
        }

        success {
            echo 'Pipeline completed successfully!'
            echo "Application URL: ${env.INGRESS_URL}"
        }

        failure {
            echo 'Pipeline failed! Check the Jenkins console output.'
        }
    }
}
