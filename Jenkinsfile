pipeline {
    agent any

    environment {
        DOCKER_IMAGE     = "kreajith2026/argocd-1"
        DEPLOYMENT_NAME  = "my-java-app"                                // ⭐ change per service
        GITOPS_REPO      = "github.com/Antony2026-ai/argocd-test.git"
        MANIFEST_DIR     = "dev"                                        // ⭐ directory, not file, now
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
    }

    options {
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {

        stage('Preflight Check') {
            steps {
                sh '''
                    echo "=== Preflight checks ==="
                    command -v docker    >/dev/null 2>&1 && docker --version    || { echo "❌ docker missing"; exit 1; }
                    command -v git       >/dev/null 2>&1 && git --version       || { echo "❌ git missing"; exit 1; }
                    command -v kustomize >/dev/null 2>&1 && kustomize version   || {
                        echo "❌ kustomize missing. Install with:"
                        echo "   docker cp /usr/local/bin/kustomize jenkins:/usr/local/bin/kustomize"
                        echo "   docker exec -u root jenkins chmod +x /usr/local/bin/kustomize"
                        exit 1
                    }
                    echo "✅ All tools present"
                '''
            }
        }

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

                        rm -rf gitops-tmp
                        git clone https://${GIT_USER}:${GIT_TOKEN}@${GITOPS_REPO} gitops-tmp
                        cd gitops-tmp/${MANIFEST_DIR}

                        # Sanity check: this deployment must exist in the manifest
                        grep -q "name: ${DEPLOYMENT_NAME}" deployment.yaml || {
                            echo "❌ Deployment '${DEPLOYMENT_NAME}' not found in ${MANIFEST_DIR}/deployment.yaml"
                            exit 1
                        }

                        # ⭐ Only rewrites the matching image entry in kustomization.yaml
                        kustomize edit set image "${DOCKER_IMAGE}=${DOCKER_IMAGE}:${IMAGE_TAG}"

                        cd ../..
                        git config user.email "jenkins@ci.com"
                        git config user.name  "Jenkins CI"
                        git -C gitops-tmp add "${MANIFEST_DIR}/kustomization.yaml"

                        if git -C gitops-tmp diff --cached --quiet; then
                            echo "No changes to commit"
                        else
                            git -C gitops-tmp commit -m "chore(${DEPLOYMENT_NAME}): image ${IMAGE_TAG}"
                            git -C gitops-tmp push origin main
                            echo "✅ Updated ${DEPLOYMENT_NAME} → ${IMAGE_TAG}"
                        fi
                    '''
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'rm -rf gitops-tmp || true'
            }
        }
    }

    post {
        always {
            sh 'docker logout || true; rm -rf gitops-tmp || true'
        }
        success {
            echo "✅ ${DEPLOYMENT_NAME}:${IMAGE_TAG} deployed via ArgoCD"
        }
        failure {
            echo "❌ ${DEPLOYMENT_NAME} build ${IMAGE_TAG} failed"
        }
    }
}
