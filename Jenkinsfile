pipeline {

    agent any

    environment {
        IMAGE_PREFIX = 'garvdeploy'
        K8S_NAMESPACE = 'online-boutique'
        KUBECONFIG_FILE = '/var/lib/jenkins/.kube/config'
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
                    echo "Docker repository: ${IMAGE_PREFIX}"
                }
            }
        }

        stage('Test - Go') {
            steps {
                sh '''
                    set -e

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
                    set -e

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

                    for SERVICE in \
                        adservice \
                        cartservice \
                        checkoutservice \
                        currencyservice \
                        emailservice \
                        frontend \
                        paymentservice \
                        productcatalogservice \
                        recommendationservice \
                        shippingservice
                    do
                        echo "======================================"
                        echo "Scanning ${SERVICE}"
                        echo "======================================"

                        trivy image \
                            --exit-code 1 \
                            --severity CRITICAL,HIGH \
                            ${IMAGE_PREFIX}/${SERVICE}:${IMAGE_TAG}
                    done
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
                        set -e

                        echo "$DOCKER_TOKEN" | docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin

                        for SERVICE in \
                            adservice \
                            cartservice \
                            checkoutservice \
                            currencyservice \
                            emailservice \
                            frontend \
                            paymentservice \
                            productcatalogservice \
                            recommendationservice \
                            shippingservice
                        do
                            docker push ${IMAGE_PREFIX}/${SERVICE}:${IMAGE_TAG}
                        done

                        docker logout
                    '''
                }
            }
        }

        stage('Helm Validate') {
            steps {
                sh '''
                    set -e

                    helm lint ./helm-chart

                    helm template online-boutique ./helm-chart \
                        --set images.repository=${IMAGE_PREFIX} \
                        --set images.tag=${IMAGE_TAG} \
                        > /tmp/online-boutique-rendered.yaml

                    echo "======================================"
                    echo " Helm Validation Passed"
                    echo "======================================"
                '''
            }
        }

        stage('Deploy to MicroK8s') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo " Deploying to MicroK8s"
                    echo "======================================"

                    export KUBECONFIG=${KUBECONFIG_FILE}

                    microk8s kubectl create namespace ${K8S_NAMESPACE} \
                        --dry-run=client -o yaml | \
                        microk8s kubectl apply -f -

                    helm upgrade --install online-boutique ./helm-chart \
                        --kubeconfig=${KUBECONFIG_FILE} \
                        --namespace ${K8S_NAMESPACE} \
                        --set images.repository=${IMAGE_PREFIX} \
                        --set images.tag=${IMAGE_TAG} \
                        --timeout 10m

                    echo "Helm deployment submitted."

                    # Loadgenerator is currently excluded because
                    # its image is not runtime-correct.
                    microk8s kubectl delete deployment loadgenerator \
                        -n ${K8S_NAMESPACE} \
                        --ignore-not-found=true

                    # Remove the overly aggressive 1-second probes
                    # for this local MicroK8s environment.
                    for SERVICE in \
                        adservice \
                        cartservice \
                        checkoutservice \
                        currencyservice \
                        emailservice \
                        frontend \
                        paymentservice \
                        productcatalogservice \
                        recommendationservice \
                        shippingservice
                    do
                        microk8s kubectl patch deployment ${SERVICE} \
                            -n ${K8S_NAMESPACE} \
                            --type=json \
                            -p='[
                              {"op":"remove","path":"/spec/template/spec/containers/0/livenessProbe"},
                              {"op":"remove","path":"/spec/template/spec/containers/0/readinessProbe"}
                            ]' || true
                    done

                    echo "Deployment patches completed."
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo " Verifying Deployment"
                    echo "======================================"

                    microk8s kubectl get pods \
                        -n ${K8S_NAMESPACE}

                    microk8s kubectl get services \
                        -n ${K8S_NAMESPACE}

                    microk8s kubectl wait \
                        --for=condition=available \
                        deployment \
                        --all \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    echo "All deployments are Available."

                    microk8s kubectl get deployments \
                        -n ${K8S_NAMESPACE}
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo " Running Smoke Test"
                    echo "======================================"

                    microk8s kubectl get pods \
                        -n ${K8S_NAMESPACE}

                    microk8s kubectl get pods \
                        -n ${K8S_NAMESPACE} \
                        --field-selector=status.phase!=Running

                    # Forward frontend temporarily.
                    microk8s kubectl port-forward \
                        -n ${K8S_NAMESPACE} \
                        svc/frontend \
                        18080:80 \
                        > /tmp/frontend-port-forward.log 2>&1 &

                    PF_PID=$!

                    trap 'kill $PF_PID 2>/dev/null || true' EXIT

                    sleep 5

                    curl --fail --silent --show-error \
                        http://127.0.0.1:18080/ \
                        > /dev/null

                    echo "======================================"
                    echo " Frontend Smoke Test PASSED"
                    echo "======================================"
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo ' ONLINE BOUTIQUE CI/CD SUCCESSFUL '
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo ' CI/CD PIPELINE FAILED '
            echo 'Check the failed Jenkins stage.'
            echo '======================================'
        }
    }
}
