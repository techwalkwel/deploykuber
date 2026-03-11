/*
 * ═══════════════════════════════════════════════════════════════
 *  CI/CD Pipeline  –  React App → DockerHub → Kubernetes (kind)
 * ═══════════════════════════════════════════════════════════════
 *  Jenkins master  : 192.168.1.11:8080
 *  Remote agent    : 192.168.1.44  (Docker + kind + Helm)
 *
 *  Jenkins connects to 192.168.1.44 via SSH Agent Node.
 *  ALL docker / kubectl / helm commands run ON that remote machine.
 * ═══════════════════════════════════════════════════════════════
 */

pipeline {
    // ── Run every stage on the remote machine (192.168.1.44) ──────────────────
    agent {
        label 'docker-k8s-node'   // label assigned to the SSH node in Jenkins
    }

    environment {
        // ── DockerHub ────────────────────────────────────────────────────────
        DOCKERHUB_USER     = 'techwalkwel98'
        APP_NAME           = 'hello-world'
        IMAGE_NAME         = "${DOCKERHUB_USER}/${APP_NAME}"
        IMAGE_TAG          = 'latest'
        FULL_IMAGE         = "${IMAGE_NAME}:${IMAGE_TAG}"
        DOCKERHUB_CREDS    = 'dockerhub-credentials'   // Jenkins credential ID

        // ── Helm / Kubernetes ────────────────────────────────────────────────
        HELM_RELEASE_NAME  = 'react-app'
        HELM_CHART_PATH    = './helm/react-app'
        K8S_NAMESPACE      = 'react-app'

        // ── kind kubeconfig on the remote machine (192.168.1.44) ────────────
        //    Absolute path — avoids $HOME being resolved on Jenkins master
        KUBECONFIG         = '/home/jenkins/.kube/config'
    }

    stages {

        // ─────────────────────────────────────────────────────────────────────
        stage('Verify Remote Node') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "🖥️  Running on node: ${env.NODE_NAME}  |  ${env.NODE_LABELS}"
                sh '''
                    echo "--- Remote Machine Info ---"
                    hostname && whoami && uname -n
                    echo ""
                    echo "--- Docker ---"
                    docker version --format "Client: {{.Client.Version}}  Server: {{.Server.Version}}"
                    echo ""
                    echo "--- kind clusters ---"
                    kind get clusters
                    echo ""
                    echo "--- kubectl ---"
                    kubectl version --client
                    echo ""
                    echo "--- Helm ---"
                    helm version --short
                    echo ""
                    echo "--- Kubernetes nodes ---"
                    kubectl get nodes
                '''
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Checkout') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo '📥 Checking out source code on remote machine...'
                checkout scm
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Build Docker Image') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "🐳 Building Docker image: ${FULL_IMAGE}"
                sh """
                    docker build -t ${FULL_IMAGE} .
                    docker images ${IMAGE_NAME}
                """
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Push to DockerHub') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "📤 Pushing image to DockerHub..."
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${FULL_IMAGE}
                        docker logout
                    """
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Load Image into kind') {
        // ─────────────────────────────────────────────────────────────────────
        // kind clusters run inside Docker containers — they cannot pull images
        // directly from DockerHub by default in some setups.
        // This stage loads the locally-built image into every kind node.
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "📦 Loading image into kind cluster nodes..."
                sh """
                    KIND_CLUSTER=\$(kind get clusters | head -1)
                    echo "kind cluster: \$KIND_CLUSTER"
                    kind load docker-image ${FULL_IMAGE} --name \$KIND_CLUSTER
                """
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Create Kubernetes Namespace') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "☸️  Ensuring namespace '${K8S_NAMESPACE}' exists..."
                sh """
                    kubectl get namespace ${K8S_NAMESPACE} \
                        || kubectl create namespace ${K8S_NAMESPACE}
                """
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Deploy with Helm') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "🚀 Deploying with Helm release '${HELM_RELEASE_NAME}'..."
                sh """
                    helm upgrade --install ${HELM_RELEASE_NAME} ${HELM_CHART_PATH} \
                        --namespace ${K8S_NAMESPACE} \
                        --set image.repository=${IMAGE_NAME} \
                        --set image.tag=${IMAGE_TAG} \
                        --set image.pullPolicy=IfNotPresent \
                        --wait \
                        --timeout 5m
                """
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('Verify Deployment') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo '✅ Verifying deployment...'
                sh """
                    kubectl rollout status deployment/${HELM_RELEASE_NAME}-react-app \
                        -n ${K8S_NAMESPACE} --timeout=3m
                    echo ""
                    echo "--- Pods ---"
                    kubectl get pods -n ${K8S_NAMESPACE} -o wide
                    echo ""
                    echo "--- Services ---"
                    kubectl get svc -n ${K8S_NAMESPACE}
                    echo ""
                    echo "--- Helm releases ---"
                    helm list -n ${K8S_NAMESPACE}
                """
            }
        }
    }

    post {
        success {
            echo """
            ╔══════════════════════════════════════════════════════╗
            ║  ✅  Deployment Successful!                           ║
            ║  Image   : ${FULL_IMAGE}              ║
            ║  NS      : ${K8S_NAMESPACE}                       ║
            ║  Node    : ${env.NODE_NAME} (192.168.1.44)        ║
            ║  Access  : http://192.168.1.44:30080              ║
            ╚══════════════════════════════════════════════════════╝
            """
        }
        failure {
            echo '❌ Pipeline FAILED. Check the logs above.'
        }
        always {
            sh 'docker image prune -f || true'
        }
    }
}
