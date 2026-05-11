pipeline {
    agent any

    environment {
        GAR_REGION        = "us-west1"
        PROJECT_ID        = "q-gcp-00098-trell-snd-bx-26-04"
        GAR_REPO_NAME     = "jenkins-docker-repo"

        GAR_REPO          = "${GAR_REGION}-docker.pkg.dev/${PROJECT_ID}/${GAR_REPO_NAME}"
        FRONTEND_IMAGE    = "${GAR_REPO}/frontend"
        BACKEND_IMAGE     = "${GAR_REPO}/backend"
        IMAGE_TAG         = "${env.BUILD_ID}"

        CENTRAL_REGISTRY  = "us-docker.pkg.dev/q-gcp-40701-gke-images-23-10/container-images-q-gcp-00098-trell-snd-bx-26-04"
        CENTRAL_FRONTEND  = "${CENTRAL_REGISTRY}/mayank-frontend-k8s-image"
        CENTRAL_BACKEND   = "${CENTRAL_REGISTRY}/mayank-backend-k8s-image"

        GKE_CLUSTER_NAME  = "jenkins-k8s-cluster-mayank"
        GKE_ZONE          = "us-west1-a"
        K8S_API_URL       = "https://172.16.0.2"
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
                    sh """
                        docker build \
                            -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                            -t ${FRONTEND_IMAGE}:latest \
                            ./frontend
                    """

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
                    gcloud auth configure-docker us-west1-docker.pkg.dev --quiet

                    docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    docker push ${FRONTEND_IMAGE}:latest

                    docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                    docker push ${BACKEND_IMAGE}:latest
                """
            }
        }

        stage('Tag and Push to Central Registry') {
            steps {
                sh """
                    gcloud auth configure-docker us-docker.pkg.dev --quiet

                    docker tag ${FRONTEND_IMAGE}:${IMAGE_TAG} ${CENTRAL_FRONTEND}:${IMAGE_TAG}
                    docker tag ${FRONTEND_IMAGE}:latest ${CENTRAL_FRONTEND}:latest

                    docker tag ${BACKEND_IMAGE}:${IMAGE_TAG} ${CENTRAL_BACKEND}:${IMAGE_TAG}
                    docker tag ${BACKEND_IMAGE}:latest ${CENTRAL_BACKEND}:latest

                    docker push ${CENTRAL_FRONTEND}:${IMAGE_TAG}
                    docker push ${CENTRAL_FRONTEND}:latest

                    docker push ${CENTRAL_BACKEND}:${IMAGE_TAG}
                    docker push ${CENTRAL_BACKEND}:latest
                """
            }
        }

        stage('Deploy to GKE') {
            steps {
                sh """
                    gcloud container clusters get-credentials \
                        ${GKE_CLUSTER_NAME} \
                        --zone=${GKE_ZONE}

                    kubectl config set-cluster \
                        gke_q-gcp-00098-trell-snd-bx-26-04_us-west1-a_jenkins-k8s-cluster-mayank \
                        --server=${K8S_API_URL} \
                        --insecure-skip-tls-verify=true

                    kubectl apply -f k8s/ --validate=false

                    kubectl set image deployment/frontend \
                        frontend=${CENTRAL_FRONTEND}:${IMAGE_TAG}

                    kubectl set image deployment/backend \
                        backend=${CENTRAL_BACKEND}:${IMAGE_TAG}

                    kubectl rollout status deployment/frontend --timeout=300s
                    kubectl rollout status deployment/backend --timeout=300s
                """
            }
        }
    }

    post {
        always {
            echo "Pipeline complete. Cleaning up..."

            sh "docker rmi ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest || true"
            sh "docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest || true"

            sh "docker rmi ${CENTRAL_FRONTEND}:${IMAGE_TAG} ${CENTRAL_FRONTEND}:latest || true"
            sh "docker rmi ${CENTRAL_BACKEND}:${IMAGE_TAG} ${CENTRAL_BACKEND}:latest || true"
        }
    }
}