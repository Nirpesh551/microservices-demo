pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
    }

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

       stage('Infrastructure Security Scan (IaC)') {
            steps {
                script {
                    echo "Downloading Trivy natively to bypass Docker-in-Docker volume limits..."
                    sh '''
                        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b .

                        echo "Scanning Helm Chart..."
                        ./trivy config --exit-code 0 --severity HIGH,CRITICAL ./helm-chart

                        echo "Scanning Dockerfile..."
                        ./trivy config --exit-code 0 --severity HIGH,CRITICAL ./src/frontend/Dockerfile
                    '''
                }
            }
        }

       stage('Automated Testing (Go)') {
            steps {
                script {
                    echo "Running Go Unit Tests..."
                    sh """
                        tar -cf - -C src/frontend . | docker run --rm -i golang:1.25-alpine sh -c "mkdir /app && cd /app && tar -xf - && go mod download && go test -v ./..."
                    """
                }
            }
        }

        stage('SonarQube Static Analysis') {
            steps {
                script {
                    echo "Sending code to SonarQube for static analysis (Bugs, Vulnerabilities, Code Smells)..."

                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=online-boutique-frontend \
                            -Dsonar.projectName="Online Boutique Frontend" \
                            -Dsonar.sources=./src/frontend \
                            -Dsonar.exclusions=**/*_test.go
                        """
                    }
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

        stage('Container Security Scan') {
            steps {
                script {
                    echo "Scanning Docker image for CRITICAL vulnerabilities using Trivy..."
                    sh """
                        docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image \
                        --severity CRITICAL \
                        --exit-code 1 \
                        --no-progress \
                        ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                    """
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

        stage('Production Approval Gate') {
            steps {
                script {
                    slackSend(color: 'warning', message: "⏳ *WAITING FOR APPROVAL*: Build #${env.BUILD_NUMBER} passed DevSecOps scans.\nVerify and approve deployment here: ${env.BUILD_URL}")

                    timeout(time: 1, unit: 'HOURS') {
                        input message: "Deploy version ${GIT_COMMIT_SHORT} to Production via GitOps?", ok: 'Approve & Deploy'
                    }
                    echo "Approval granted! Proceeding with GitOps deployment..."
                }
            }
        }

        stage('Update Helm Chart (GitOps)') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GIT_CREDS_ID, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        sh """
                            git config user.email "jenkins-bot@example.com"
                            git config user.name "Jenkins Bot"

                            cd helm-chart
                            sed -i 's|tag: .*|tag: ${GIT_COMMIT_SHORT}|' values.yaml

                            git add values.yaml
                            git commit -m "Jenkins: Deployed version ${GIT_COMMIT_SHORT}"
                            git push https://${GIT_USER}:${GIT_PASS}@${GITHUB_REPO} HEAD:main
                        """
                    }
                }
            }
        }

        stage('DAST (OWASP ZAP)') {
            steps {
                script {
                    def targetUrl = "http://shop.92.4.77.2.nip.io"

                    echo "Waiting 60 seconds for ArgoCD..."
                    sleep time: 60, unit: 'SECONDS'

                    echo "Initiating OWASP ZAP Scan on ${targetUrl}..."
                    sh """
                        docker run --rm --network host -u root ghcr.io/zaproxy/zaproxy:stable \
                        sh -c "mkdir -p /zap/wrk && /zap/zap-baseline.py -t ${targetUrl} -I"
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Build completed: ${currentBuild.currentResult}"
            cleanWs(deleteDirs: true, disableDeferredWipeout: true)
        }
        success {
            slackSend(color: 'good', message: "✅ *SUCCESS*: The 'online-boutique-frontend' deployment passed all quality gates and is live! \nLogs: ${env.BUILD_URL}")
        }
        failure {
            slackSend(color: 'danger', message: "❌ *FAILED*: The deployment pipeline broke or was rejected. \nInvestigate here: ${env.BUILD_URL}")
        }
    }
}
