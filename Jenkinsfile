pipeline {
    agent any

    environment {
        // Artifact Registry (your project)
        GAR_REGION        = "us-west1"
        PROJECT_ID        = "q-gcp-00098-trell-snd-bx-26-04"
        GAR_REPO_NAME     = "jenkins-docker-repo"
        GAR_REPO          = "${GAR_REGION}-docker.pkg.dev/${PROJECT_ID}/${GAR_REPO_NAME}"
        FRONTEND_IMAGE    = "${GAR_REPO}/frontend"
        BACKEND_IMAGE     = "${GAR_REPO}/backend"
        IMAGE_TAG         = "${env.BUILD_ID}"

        // Central Artifact Registry
        CENTRAL_REGISTRY  = "us-docker.pkg.dev/q-gcp-40701-gke-images-23-10/container-images-q-gcp-00098-trell-snd-bx-26-04"
        CENTRAL_FRONTEND  = "${CENTRAL_REGISTRY}/frontend"
        CENTRAL_BACKEND   = "${CENTRAL_REGISTRY}/backend"

        // GKE Details
        GKE_CLUSTER_NAME  = "jenkins-k8s-cluster-mayank"
        GKE_ZONE          = "us-west1-a"
        K8S_API_URL       = "https://172.16.0.2"

        // Will be populated in the "Get Image SHA" stage
        FRONTEND_SHA      = ""
        BACKEND_SHA       = ""
    }

    stages {

        stage('Clone Code') {
            steps {
                checkout scm
                echo "Code cloned successfully."
            }
        }

        stage('Build Images') {
            steps {
                script {
                    echo "Building Frontend Docker Image..."
                    sh """
                        docker build \
                            -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                            -t ${FRONTEND_IMAGE}:latest \
                            ./frontend
                    """

                    echo "Building Backend Docker Image..."
                    sh """
                        docker build \
                            -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                            -t ${BACKEND_IMAGE}:latest \
                            ./backend
                    """
                }
            }
        }

        stage('Push to Your Artifact Registry') {
            steps {
                sh """
                    set -e

                    echo "Configuring Docker authentication for your Artifact Registry..."
                    gcloud auth configure-docker us-west1-docker.pkg.dev --quiet

                    echo "Pushing Frontend images..."
                    docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    docker push ${FRONTEND_IMAGE}:latest

                    echo "Pushing Backend images..."
                    docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                    docker push ${BACKEND_IMAGE}:latest
                """
            }
        }

        stage('Tag and Push to Central Registry') {
            steps {
                sh """
                    set -e

                    echo "Configuring Docker authentication for central registry..."
                    gcloud auth configure-docker us-docker.pkg.dev --quiet

                    echo "Tagging images for central registry..."
                    docker tag ${FRONTEND_IMAGE}:latest ${CENTRAL_FRONTEND}:latest
                    docker tag ${BACKEND_IMAGE}:latest ${CENTRAL_BACKEND}:latest

                    echo "Pushing Frontend to central registry..."
                    docker push ${CENTRAL_FRONTEND}:latest

                    echo "Pushing Backend to central registry..."
                    docker push ${CENTRAL_BACKEND}:latest
                """
            }
        }

        stage('Get Image SHA') {
            steps {
                script {
                    echo "Retrieving Frontend image digest..."
                    def frontendSha = sh(
                        script: """
                            gcloud artifacts docker images describe \
                                ${CENTRAL_FRONTEND}:latest \
                                --format='get(image_summary.digest)'
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Retrieving Backend image digest..."
                    def backendSha = sh(
                        script: """
                            gcloud artifacts docker images describe \
                                ${CENTRAL_BACKEND}:latest \
                                --format='get(image_summary.digest)'
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Frontend SHA: ${frontendSha}"
                    echo "Backend SHA: ${backendSha}"

                    // Persist values for subsequent stages
                    env.FRONTEND_SHA = frontendSha
                    env.BACKEND_SHA  = backendSha

                    // Fail fast if either digest is empty
                    if (!env.FRONTEND_SHA?.trim()) {
                        error("FRONTEND_SHA is empty.")
                    }

                    if (!env.BACKEND_SHA?.trim()) {
                        error("BACKEND_SHA is empty.")
                    }
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                sh '''
                    set -e

                    echo "========================================"
                    echo "Starting GKE Deployment"
                    echo "========================================"

                    echo "Checking gcloud version..."
                    gcloud --version

                    echo "Checking kubectl version..."
                    kubectl version --client

                    echo "Testing connectivity to private Kubernetes API..."
                    curl -k -I --connect-timeout 5 ${K8S_API_URL} || true

                    echo "Getting cluster credentials..."
                    gcloud container clusters get-credentials \
                        ${GKE_CLUSTER_NAME} \
                        --zone=${GKE_ZONE}

                    echo "Updating kubeconfig to use private endpoint..."
                    kubectl config set-cluster \
                        gke_q-gcp-00098-trell-snd-bx-26-04_us-west1-a_jenkins-k8s-cluster-mayank \
                        --server=${K8S_API_URL} \
                        --insecure-skip-tls-verify=true

                    echo "Current Kubernetes context:"
                    kubectl config current-context

                    echo "Testing cluster access..."
                    kubectl get nodes || true

                    echo "Testing RBAC permissions..."
                    kubectl auth can-i get pods || true

                    echo "Listing namespaces..."
                    kubectl get ns || true

                    echo "Applying Kubernetes manifests..."
                    kubectl apply -f k8s/ --validate=false

                    echo "Using Frontend SHA: ${FRONTEND_SHA}"
                    echo "Using Backend SHA: ${BACKEND_SHA}"

                    echo "Updating frontend deployment image..."
                    kubectl set image deployment/frontend \
                        frontend=${CENTRAL_FRONTEND}@${FRONTEND_SHA}

                    echo "Updating backend deployment image..."
                    kubectl set image deployment/backend \
                        backend=${CENTRAL_BACKEND}@${BACKEND_SHA}

                    echo "Waiting for frontend rollout..."
                    kubectl rollout status deployment/frontend --timeout=300s

                    echo "Waiting for backend rollout..."
                    kubectl rollout status deployment/backend --timeout=300s

                    echo "========================================"
                    echo "Deployment completed successfully"
                    echo "========================================"
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline complete. Cleaning up local Docker images..."

            sh "docker rmi ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest || true"
            sh "docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest || true"

            sh "docker rmi ${CENTRAL_FRONTEND}:latest || true"
            sh "docker rmi ${CENTRAL_BACKEND}:latest || true"
        }
    }
}