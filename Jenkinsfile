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

        // Central Artifact Registry
        CENTRAL_REGISTRY  = "us-docker.pkg.dev/q-gcp-40701-gke-images-23-10/container-images-q-gcp-00098-trell-snd-bx-26-04"
        CENTRAL_FRONTEND  = "${CENTRAL_REGISTRY}/frontend"
        CENTRAL_BACKEND   = "${CENTRAL_REGISTRY}/backend"

        // GKE Details
        K8S_API_URL       = "https://34.187.145.101"
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

                    # Tag Frontend
                    docker tag ${FRONTEND_IMAGE}:latest ${CENTRAL_FRONTEND}:latest

                    # Tag Backend
                    docker tag ${BACKEND_IMAGE}:latest ${CENTRAL_BACKEND}:latest

                    # Push Frontend to Central
                    docker push ${CENTRAL_FRONTEND}:latest

                    # Push Backend to Central
                    docker push ${CENTRAL_BACKEND}:latest
                """
            }
        }

        stage('Get Image SHA') {
            steps {
                script {
                    FRONTEND_SHA = sh(
                        script: """
                            gcloud artifacts docker images describe \
                                ${CENTRAL_FRONTEND}:latest \
                                --format='get(image_summary.digest)'
                        """,
                        returnStdout: true
                    ).trim()

                    BACKEND_SHA = sh(
                        script: """
                            gcloud artifacts docker images describe \
                                ${CENTRAL_BACKEND}:latest \
                                --format='get(image_summary.digest)'
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Frontend SHA: ${FRONTEND_SHA}"
                    echo "Backend SHA: ${BACKEND_SHA}"
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                withCredentials([string(credentialsId: 'k8s-deploy-token', variable: 'K8S_TOKEN')]) {
                    script {
                        sh """
                            kubectl config set-cluster gke-cluster \
                                --server=${K8S_API_URL} \
                                --insecure-skip-tls-verify=true

                            kubectl config set-credentials jenkins-deployer \
                                --token=${K8S_TOKEN}

                            kubectl config set-context gke-context \
                                --cluster=gke-cluster \
                                --user=jenkins-deployer

                            kubectl config use-context gke-context

                            kubectl apply -f k8s/

                            kubectl set image deployment/frontend \
                                frontend=${CENTRAL_FRONTEND}@${FRONTEND_SHA}

                            kubectl set image deployment/backend \
                                backend=${CENTRAL_BACKEND}@${BACKEND_SHA}
                        """
                    }
                }
            }
        }

    }

    post {
        always {
            echo "Pipeline complete. Cleaning up local Docker images..."
            sh "docker rmi ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest || true"
            sh "docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest || true"
        }
    }
}