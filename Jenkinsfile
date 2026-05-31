pipeline {
    agent any

    environment {
        GIT_REPO         = 'https://github.com/Shubham-Salunke-26/onlineshop-poc.git'
        GIT_BRANCH       = 'main'

        DOCKER_IMAGE     = 'shubhyash/onlineshop-poc'
        IMAGE_TAG        = "${BUILD_NUMBER}"

        DOCKER_HUB_CREDS = 'docker-hub-cred'
        GITHUB_CREDS     = 'git-hub-cred'

        K8S_NAMESPACE    = 'onlineshop'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: "${GIT_BRANCH}",
                    credentialsId: "${GITHUB_CREDS}",
                    url: "${GIT_REPO}"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                    -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_HUB_CREDS}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''

                    sh """
                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                sh """
                    sed -i 's|IMAGE_PLACEHOLDER|${DOCKER_IMAGE}:${IMAGE_TAG}|g' k8s/deployment.yaml
                """
            }
        }

        stage('Deploy to Kind Cluster') {
            steps {
                sh """
                    set -e

                    kubectl apply -f k8s/

                    kubectl rollout status deployment/onlineshop \
                    -n ${K8S_NAMESPACE} \
                    --timeout=300s
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                    echo "========== Deployments =========="
                    kubectl get deployments -n ${K8S_NAMESPACE}

                    echo "========== Pods =========="
                    kubectl get pods -o wide -n ${K8S_NAMESPACE}

                    echo "========== Services =========="
                    kubectl get svc -n ${K8S_NAMESPACE}
                """
            }
        }
    }

    post {

        success {
            echo 'Deployment completed successfully.'

            sh """
                docker image prune -f
            """
        }

        failure {
            echo 'Deployment failed.'
        }

        always {
            cleanWs()
        }
    }
}
