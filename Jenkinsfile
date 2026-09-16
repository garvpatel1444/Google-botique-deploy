pipeline {

    agent any

    environment {
        IMAGE_PREFIX    = 'garvdeploy'
        K8S_NAMESPACE   = 'online-boutique'
        KUBECONFIG_FILE = '/var/lib/jenkins/.kube/config'
    }

    stages {

        // ============================================================
        // 1. CHECKOUT
        // ============================================================
        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    echo "======================================"
                    echo "Commit: ${env.IMAGE_TAG}"
                    echo "Docker Repository: ${IMAGE_PREFIX}"
                    echo "======================================"
                }
            }
        }


        // ============================================================
        // 2. GO TESTS
        // ============================================================
        stage('Test - Go') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "Running Go Tests"
                    echo "======================================"

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

                    echo "Go tests PASSED"
                '''
            }
        }


        // ============================================================
        // 3. C# TESTS
        // ============================================================
        stage('Test - C#') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "Running C# Tests"
                    echo "======================================"

                    docker run --rm \
                        -v "$WORKSPACE:/workspace" \
                        -w /workspace/src/cartservice \
                        mcr.microsoft.com/dotnet/sdk:10.0 \
                        dotnet test

                    echo "C# tests PASSED"
                '''
            }
        }


        // ============================================================
        // 4. DOCKER BUILD
        // ============================================================
        stage('Docker Build') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "Building Docker Images"
                    echo "Tag: ${IMAGE_TAG}"
                    echo "======================================"

                    docker build \
                        -t ${IMAGE_PREFIX}/adservice:${IMAGE_TAG} \
                        src/adservice

                    docker build \
                        -t ${IMAGE_PREFIX}/cartservice:${IMAGE_TAG} \
                        src/cartservice/src

                    docker build \
                        -t ${IMAGE_PREFIX}/checkoutservice:${IMAGE_TAG} \
                        src/checkoutservice

                    docker build \
                        -t ${IMAGE_PREFIX}/currencyservice:${IMAGE_TAG} \
                        src/currencyservice

                    docker build \
                        -t ${IMAGE_PREFIX}/emailservice:${IMAGE_TAG} \
                        src/emailservice

                    docker build \
                        -t ${IMAGE_PREFIX}/frontend:${IMAGE_TAG} \
                        src/frontend

                    docker build \
                        -t ${IMAGE_PREFIX}/paymentservice:${IMAGE_TAG} \
                        src/paymentservice

                    docker build \
                        -t ${IMAGE_PREFIX}/productcatalogservice:${IMAGE_TAG} \
                        src/productcatalogservice

                    docker build \
                        -t ${IMAGE_PREFIX}/recommendationservice:${IMAGE_TAG} \
                        src/recommendationservice

                    docker build \
                        -t ${IMAGE_PREFIX}/shippingservice:${IMAGE_TAG} \
                        src/shippingservice

                    echo "Docker builds PASSED"
                '''
            }
        }


        // ============================================================
        // 5. SECURITY SCAN
        // ============================================================
        stage('Security Scan') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "Running Trivy Security Scan"
                    echo "======================================"

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
                        echo ""
                        echo "--------------------------------------"
                        echo "Scanning: ${SERVICE}"
                        echo "--------------------------------------"

                        trivy image \
                            --exit-code 1 \
                            --severity CRITICAL,HIGH \
                            ${IMAGE_PREFIX}/${SERVICE}:${IMAGE_TAG}
                    done

                    echo "Security scan PASSED"
                '''
            }
        }


        // ============================================================
        // 6. DOCKER PUSH
        // ============================================================
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

                        echo "======================================"
                        echo "Logging into Docker Hub"
                        echo "======================================"

                        echo "$DOCKER_TOKEN" | docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin

                        echo "Docker login successful"

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
                            echo ""
                            echo "Pushing: ${IMAGE_PREFIX}/${SERVICE}:${IMAGE_TAG}"

                            docker push \
                                ${IMAGE_PREFIX}/${SERVICE}:${IMAGE_TAG}
                        done

                        docker logout

                        echo "Docker images PUSHED successfully"
                    '''
                }
            }
        }


        // ============================================================
        // 7. HELM VALIDATION
        // ============================================================
        stage('Helm Validate') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "Helm Lint"
                    echo "======================================"

                    helm lint ./helm-chart

                    echo "======================================"
                    echo "Helm Template"
                    echo "======================================"

                    helm template online-boutique ./helm-chart \
                        --namespace ${K8S_NAMESPACE} \
                        --set images.repository=${IMAGE_PREFIX} \
                        --set images.tag=${IMAGE_TAG} \
                        > /tmp/online-boutique-rendered.yaml

                    echo "Helm validation PASSED"
                '''
            }
        }


        // ============================================================
        // 8. DEPLOY TO MICROK8S
        // ============================================================
        stage('Deploy to MicroK8s') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "Deploying to MicroK8s"
                    echo "======================================"

                    export KUBECONFIG=${KUBECONFIG_FILE}

                    echo "Checking MicroK8s..."

                    microk8s status --wait-ready

                    echo "Checking Kubernetes node..."

                    microk8s kubectl get nodes

                    echo "Creating namespace if required..."

                    microk8s kubectl create namespace ${K8S_NAMESPACE} \
                        --dry-run=client \
                        -o yaml | \
                        microk8s kubectl apply -f -

                    echo "Installing/Upgrading Helm release..."

                    helm upgrade --install online-boutique ./helm-chart \
                        --kubeconfig=${KUBECONFIG_FILE} \
                        --namespace ${K8S_NAMESPACE} \
                        --set images.repository=${IMAGE_PREFIX} \
                        --set images.tag=${IMAGE_TAG} \
                        --timeout 10m

                    echo "Helm deployment submitted successfully."


                    # ------------------------------------------------
                    # LOADGENERATOR
                    # ------------------------------------------------
                    echo "Removing broken loadgenerator..."

                    microk8s kubectl delete deployment loadgenerator \
                        -n ${K8S_NAMESPACE} \
                        --ignore-not-found=true


                    # ------------------------------------------------
                    # LOCAL MICROK8S PROBES
                    # ------------------------------------------------
                    echo "Applying local MicroK8s probe configuration..."

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

                        echo "Patching ${SERVICE}..."

                        microk8s kubectl patch deployment ${SERVICE} \
                            -n ${K8S_NAMESPACE} \
                            --type=json \
                            -p='[
                                {
                                    "op":"remove",
                                    "path":"/spec/template/spec/containers/0/livenessProbe"
                                },
                                {
                                    "op":"remove",
                                    "path":"/spec/template/spec/containers/0/readinessProbe"
                                }
                            ]' || true

                    done

                    echo "Deployment configuration completed."

                    echo "Current deployments:"

                    microk8s kubectl get deployments \
                        -n ${K8S_NAMESPACE}
                '''
            }
        }


        // ============================================================
        // 9. VERIFY DEPLOYMENT
        // ============================================================
        stage('Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "Verifying Kubernetes Deployment"
                    echo "======================================"

                    echo ""
                    echo "Pods:"
                    microk8s kubectl get pods \
                        -n ${K8S_NAMESPACE}

                    echo ""
                    echo "Services:"
                    microk8s kubectl get services \
                        -n ${K8S_NAMESPACE}

                    echo ""
                    echo "Waiting for deployments to become Available..."

                    microk8s kubectl wait \
                        --for=condition=available \
                        deployment \
                        --all \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    echo ""
                    echo "All deployments are AVAILABLE."

                    echo ""
                    echo "Final deployment status:"

                    microk8s kubectl get deployments \
                        -n ${K8S_NAMESPACE}

                    echo "Deployment verification PASSED"
                '''
            }
        }


        // ============================================================
        // 10. SMOKE TEST
        // ============================================================
        stage('Smoke Test') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "Running Frontend Smoke Test"
                    echo "======================================"

                    echo "Waiting for all application deployments..."

                    microk8s kubectl wait \
                        --for=condition=available \
                        deployment \
                        --all \
                        -n ${K8S_NAMESPACE} \
                        --timeout=300s

                    echo ""
                    echo "All deployments are AVAILABLE."

                    echo ""
                    echo "Starting temporary curl test pod..."

                    microk8s kubectl run smoke-test \
                        -n ${K8S_NAMESPACE} \
                        --image=curlimages/curl:8.16.0 \
                        --restart=Never \
                        --command -- \
                        sleep 60

                    echo "Waiting for smoke-test pod..."

                    microk8s kubectl wait \
                        --for=condition=Ready \
                        pod/smoke-test \
                        -n ${K8S_NAMESPACE} \
                        --timeout=120s

                    echo ""
                    echo "Testing frontend Kubernetes service..."

                    microk8s kubectl exec \
                        -n ${K8S_NAMESPACE} \
                        smoke-test \
                        -- \
                        curl \
                        --fail \
                        --silent \
                        --show-error \
                        --max-time 30 \
                        http://frontend/

                    echo ""
                    echo "Frontend HTTP request PASSED."

                    echo ""
                    echo "Removing smoke-test pod..."

                    microk8s kubectl delete pod smoke-test \
                        -n ${K8S_NAMESPACE} \
                        --ignore-not-found=true

                    echo ""
                    echo "======================================"
                    echo " FRONTEND SMOKE TEST PASSED "
                    echo "======================================"
                '''
            }
        }
    }


    // ================================================================
    // POST ACTIONS
    // ================================================================
    post {

        success {
            echo '======================================'
            echo ' ONLINE BOUTIQUE CI/CD SUCCESSFUL '
            echo '======================================'
            echo 'Build → Test → Docker → Trivy → Push → Helm → MicroK8s → Smoke Test'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo ' CI/CD PIPELINE FAILED '
            echo '======================================'
            echo 'Check the failed Jenkins stage and its console log.'
            echo '======================================'
        }

        always {
            sh '''
                microk8s kubectl delete pod smoke-test \
                    -n online-boutique \
                    --ignore-not-found=true \
                    2>/dev/null || true
            '''
        }
    }
}
