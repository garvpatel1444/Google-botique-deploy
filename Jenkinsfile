pipeline {

    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'
        IMAGE_PREFIX = 'garvdeploy/online-boutique'
        K8S_NAMESPACE = 'online-boutique'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Building commit: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Test - Go') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/workspace" \
                      -v /var/lib/jenkins/go-cache:/go/pkg/mod \
                      -v /var/lib/jenkins/go-build-cache:/root/.cache/go-build \
                      -w /workspace/src/shippingservice \
                      golang:1.27 \
                      go test ./...

                    docker run --rm \
                      -v "$WORKSPACE:/workspace" \
                      -v /var/lib/jenkins/go-cache:/go/pkg/mod \
                      -v /var/lib/jenkins/go-build-cache:/root/.cache/go-build \
                      -w /workspace/src/productcatalogservice \
                      golang:1.27 \
                      go test ./...
                '''
            }
        }

        stage('Test - C#') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/workspace" \
                      -w /workspace/src/cartservice \
                      mcr.microsoft.com/dotnet/sdk:10.0 \
                      dotnet test
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -e

                    docker build -t ${IMAGE_PREFIX}/adservice:${IMAGE_TAG} \
                        src/adservice

                    docker build -t ${IMAGE_PREFIX}/cartservice:${IMAGE_TAG} \
                        src/cartservice/src

                    docker build -t ${IMAGE_PREFIX}/checkoutservice:${IMAGE_TAG} \
                        src/checkoutservice

                    docker build -t ${IMAGE_PREFIX}/currencyservice:${IMAGE_TAG} \
                        src/currencyservice

                    docker build -t ${IMAGE_PREFIX}/emailservice:${IMAGE_TAG} \
                        src/emailservice

                    docker build -t ${IMAGE_PREFIX}/frontend:${IMAGE_TAG} \
                        src/frontend

                    docker build -t ${IMAGE_PREFIX}/loadgenerator:${IMAGE_TAG} \
                        src/loadgenerator

                    docker build -t ${IMAGE_PREFIX}/paymentservice:${IMAGE_TAG} \
                        src/paymentservice

                    docker build -t ${IMAGE_PREFIX}/productcatalogservice:${IMAGE_TAG} \
                        src/productcatalogservice

                    docker build -t ${IMAGE_PREFIX}/recommendationservice:${IMAGE_TAG} \
                        src/recommendationservice

                    docker build -t ${IMAGE_PREFIX}/shippingservice:${IMAGE_TAG} \
                        src/shippingservice
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    set -e

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/frontend:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/cartservice:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/checkoutservice:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/currencyservice:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/emailservice:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/paymentservice:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/productcatalogservice:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/recommendationservice:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/shippingservice:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/adservice:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity CRITICAL,HIGH \
                        ${IMAGE_PREFIX}/loadgenerator:${IMAGE_TAG}
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin

                        docker push ${IMAGE_PREFIX}/adservice:${IMAGE_TAG}
                        docker push ${IMAGE_PREFIX}/cartservice:${IMAGE_TAG}
                        docker push ${IMAGE_PREFIX}/checkoutservice:${IMAGE_TAG}
                        docker push ${IMAGE_PREFIX}/currencyservice:${IMAGE_TAG}
                        docker push ${IMAGE_PREFIX}/emailservice:${IMAGE_TAG}
                        docker push ${IMAGE_PREFIX}/frontend:${IMAGE_TAG}
                        docker push ${IMAGE_PREFIX}/loadgenerator:${IMAGE_TAG}
                        docker push ${IMAGE_PREFIX}/paymentservice:${IMAGE_TAG}
                        docker push ${IMAGE_PREFIX}/productcatalogservice:${IMAGE_TAG}
                        docker push ${IMAGE_PREFIX}/recommendationservice:${IMAGE_TAG}
                        docker push ${IMAGE_PREFIX}/shippingservice:${IMAGE_TAG}

                        docker logout
                    '''
                }
            }
        }

        stage('Helm Validate') {
            steps {
                sh '''
                    helm lint ./helm-chart
                    helm template online-boutique ./helm-chart \
                        --set images.repository=${IMAGE_PREFIX} \
                        --set images.tag=${IMAGE_TAG} \
                        > /tmp/online-boutique-rendered.yaml
                '''
            }
        }

        stage('Deploy to MicroK8s') {
            steps {
                sh '''
                    microk8s kubectl create namespace ${K8S_NAMESPACE} \
                        --dry-run=client -o yaml | \
                        microk8s kubectl apply -f -

                    helm upgrade --install online-boutique ./helm-chart \
                        --namespace ${K8S_NAMESPACE} \
                        --set images.repository=${IMAGE_PREFIX} \
                        --set images.tag=${IMAGE_TAG} \
                        --wait \
                        --timeout 10m
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    microk8s kubectl get pods \
                        -n ${K8S_NAMESPACE}

                    microk8s kubectl get services \
                        -n ${K8S_NAMESPACE}

                    microk8s kubectl rollout status deployment/frontend \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    microk8s kubectl rollout status deployment/cartservice \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    microk8s kubectl rollout status deployment/checkoutservice \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    microk8s kubectl rollout status deployment/currencyservice \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    microk8s kubectl rollout status deployment/emailservice \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    microk8s kubectl rollout status deployment/paymentservice \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    microk8s kubectl rollout status deployment/productcatalogservice \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    microk8s kubectl rollout status deployment/recommendationservice \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    microk8s kubectl rollout status deployment/shippingservice \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    microk8s kubectl get ingress \
                        -n ${K8S_NAMESPACE}

                    microk8s kubectl get pods \
                        -n ${K8S_NAMESPACE} \
                        --field-selector=status.phase!=Running
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo ' ONLINE BOUTIQUE DEPLOYMENT SUCCESSFUL '
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo ' DEPLOYMENT FAILED '
            echo 'Check the failed Jenkins stage.'
            echo '======================================'
        }
    }
}
