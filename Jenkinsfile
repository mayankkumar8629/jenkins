pipeline {
    agent any

    environment {
        GAR_REGION     = "us-west1"
        PROJECT_ID     = "q-gcp-00098-trell-snd-bx-26-04"
        GAR_REPO_NAME  = "jenkins-docker-repo"

        GAR_REPO       = "${GAR_REGION}-docker.pkg.dev/${PROJECT_ID}/${GAR_REPO_NAME}"
        FRONTEND_IMAGE = "${GAR_REPO}/frontend"
        BACKEND_IMAGE  = "${GAR_REPO}/backend"
        IMAGE_TAG      = "${env.BUILD_ID}"
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

        stage('Push to Artifact Registry') {
            steps {
                sh '''
                    echo "Configuring Docker authentication for Google Artifact Registry..."
                    gcloud auth configure-docker us-west1-docker.pkg.dev --quiet

                    echo "Pushing Frontend image..."
                    docker push us-west1-docker.pkg.dev/q-gcp-00098-trell-snd-bx-26-04/jenkins-docker-repo/frontend:latest

                    echo "Pushing Backend image..."
                    docker push us-west1-docker.pkg.dev/q-gcp-00098-trell-snd-bx-26-04/jenkins-docker-repo/backend:latest
                '''
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