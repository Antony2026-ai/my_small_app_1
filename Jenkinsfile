pipeline {
    agent any

    environment {
        DOCKER_IMAGE     = "kreajith2026/vts-main"
        DEPLOYMENT_NAME  = "vts-deployment-main"
        GITOPS_REPO      = "github.com/Antony2026-ai/argocd-test.git"
        MANIFEST_DIR     = "dev/overlays/main"
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
    }

    
    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push "${DOCKER_IMAGE}:${IMAGE_TAG}"
                    '''
                }
            }
        }

        stage('Update K8s Manifest') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'github-creds',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_TOKEN')]) {
            sh '''
                set -e

                rm -rf argocd-test
                git clone https://\$GIT_USER:\$GIT_TOKEN@github.com/Antony2026-ai/argocd-test.git
                cd argocd-test/${MANIFEST_DIR}

                kustomize edit set image "${DOCKER_IMAGE}=${DOCKER_IMAGE}:${IMAGE_TAG}"

                git config user.email "jenkins@ci.com"
                git config user.name  "Jenkins CI"

                git add "kustomization.yaml"

                git commit -m "Update ${DEPLOYMENT_NAME} image to ${DOCKER_IMAGE}:${IMAGE_TAG}"
                git push origin main
            '''
        }
    }
}

    }

    post {
        success {
            echo "${DEPLOYMENT_NAME}:${IMAGE_TAG} deployed via ArgoCD"
        }
        failure {
            echo "${DEPLOYMENT_NAME} build ${IMAGE_TAG} failed"
        }
    }
}
