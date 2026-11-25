
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'dockernavaneeth/villa-app'
        DOCKER_CREDENTIALS = 'dockerhub-credentials'
        GITHUB_CREDENTIALS = 'github-credentials'
        GITHUB_REPO = 'https://github.com/Navaneethkrishna-coder/Villa.git'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: "${GITHUB_CREDENTIALS}",
                    url: "${GITHUB_REPO}"
            }
        }

        stage('Build and Test') {
            steps {
                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}"

                    sh '''
                        echo "Checking Dockerfile..."
                        test -f Dockerfile
                        test -f requirements.txt

                        echo "Building application image..."
                        docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .

                        echo "Running Django system checks..."
                        docker run --rm \
                          -e DJANGO_SECRET_KEY='temporary-ci-check-key' \
                          -e DJANGO_ALLOWED_HOSTS='*' \
                          ${DOCKER_IMAGE}:${IMAGE_TAG} \
                          python manage.py check
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Docker Hub Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                          -u "$DOCKER_USER" --password-stdin

                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}

                        docker logout
                    '''
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${GITHUB_CREDENTIALS}",
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e

                        sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|" \
                          k8s/deployment.yaml

                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"

                        git add k8s/deployment.yaml
                        git commit -m "Update Villa image to ${IMAGE_TAG}"

                        ASKPASS_FILE=$(mktemp)
                        trap 'rm -f "$ASKPASS_FILE"' EXIT

                        cat > "$ASKPASS_FILE" <<'EOF'
#!/bin/sh
case "$1" in
    *Username*) echo "$GIT_USER" ;;
    *) echo "$GIT_TOKEN" ;;
esac
EOF

                        chmod 700 "$ASKPASS_FILE"

                        GIT_ASKPASS="$ASKPASS_FILE" \
                        GIT_TERMINAL_PROMPT=0 \
                        git push origin HEAD:main
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Villa CI/CD pipeline completed successfully!'
            echo 'Argo CD will detect the updated Kubernetes manifest.'
        }

        failure {
            echo 'Villa CI/CD pipeline failed. Check the Jenkins console output.'
        }
    }
}

