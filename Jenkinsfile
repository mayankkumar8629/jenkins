pipeline {
    agent any

    environment {
        GAR_REGION       = "us-west1"
        GAR_REPO         = "us-west1-docker.pkg.dev/q-gcp-00098-trell-snd-bx-26-04/jenkins-docker-repo"
        FRONTEND_IMAGE   = "${GAR_REPO}/frontend"
        BACKEND_IMAGE    = "${GAR_REPO}/backend"
        IMAGE_TAG        = "${env.BUILD_ID}"
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
                    sh "docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} -t ${FRONTEND_IMAGE}:latest ./frontend"

                    echo "Building Backend Docker Image..."
                    sh "docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} -t ${BACKEND_IMAGE}:latest ./backend"
                }
            }
        }

        stage('Push to Artifact Registry') {
            steps {
                withCredentials([file(credentialsId: 'gcp-artifact-registry-key', variable: 'GCP_SA_KEY')]) {
                    script {
                        echo "Authenticating with Google Artifact Registry..."
                        sh "docker login -u _json_key --password-stdin ${GAR_REGION}-docker.pkg.dev < ${GCP_SA_KEY}"

                        echo "Pushing Frontend Image..."
                        sh "docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${FRONTEND_IMAGE}:latest"

                        echo "Pushing Backend Image..."
                        sh "docker push ${BACKEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${BACKEND_IMAGE}:latest"
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