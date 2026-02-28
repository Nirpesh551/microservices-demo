pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'hahaha555'
        IMAGE_NAME = 'online-boutique-frontend'
        GITHUB_REPO = 'github.com/Nirpesh551/microservices-demo.git'

        DOCKER_CREDS_ID = 'docker-hub-creds'
        GIT_CREDS_ID = 'github-creds'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Automated Testing') {
            steps {
                script {
                    echo "Gate 1: Running Go Unit Tests..."
                    sh """
                        docker run --rm \
                        -v \$(pwd)/src/frontend:/app \
                        -w /app \
                        golang:1.22-alpine \
                        sh -c "go mod download && go test -v ./..."
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    echo "Building version: ${GIT_COMMIT_SHORT}"

                    sh "docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT} ./src/frontend"
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: DOCKER_CREDS_ID, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh "echo $PASS | docker login -u $USER --password-stdin"
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT}"
                    }
                }
            }
        }

        stage('Update Helm Chart') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GIT_CREDS_ID, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        sh """
                            git config user.email "jenkins-bot@example.com"
                            git config user.name "Jenkins Bot"

                            # Navigate to Helm chart
                            cd helm-chart

                            # Force update the tag in values.yaml
                            sed -i 's|tag: .*|tag: ${GIT_COMMIT_SHORT}|' values.yaml

                            # Commit and Push back to GitHub
                            git add values.yaml
                            git commit -m "Jenkins: Deployed version ${GIT_COMMIT_SHORT}"
                            git push https://${GIT_USER}:${GIT_PASS}@${GITHUB_REPO} HEAD:main
                        """
                    }
                }
            }
        }
    }
}
