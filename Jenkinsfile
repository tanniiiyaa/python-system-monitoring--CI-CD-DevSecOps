pipeline {
    agent {
        label 'docker-agent'
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'

        DOCKER_REGISTRY = 'docker.io'
        DOCKER_REPO = 'cheesepopcorn/python-system-monitoring'
        DOCKER_IMAGE = 'docker.io/cheesepopcorn/python-system-monitoring'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Checkout') {
            steps {
                git(
                    branch: 'main',
                    url: 'https://github.com/Aj7Ay/Python-System-Monitoring.git'
                )
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(
                    installationName: 'sonar-server',
                    credentialsId: 'sonar-token'
                ) {
                    sh '''
                        "$SCANNER_HOME/bin/sonar-scanner" \
                          -Dsonar.projectKey=python-system-monitoring \
                          -Dsonar.projectName=Python-System-Monitoring \
                          -Dsonar.sources=src \
                          -Dsonar.tests=tests \
                          -Dsonar.python.version=3.9
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck(
                    odcInstallation: 'DP-Check',
                    nvdCredentialsId: 'nvd-api-key',
                    additionalArguments: '--scan ./ --format XML'
                )
            }
        }

        stage('Publish OWASP Report') {
            steps {
                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report.xml',
                    skipNoReportFiles: false,
                    stopBuild: false
                )
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs \
                      --format table \
                      --output trivy-fs-report.txt \
                      --timeout 10m \
                      .
                '''
            }
        }

        stage('Archive Trivy Report') {
            steps {
                archiveArtifacts(
                    artifacts: 'trivy-fs-report.txt',
                    fingerprint: true
                )
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    make image \
                      IMAGE_REG="$DOCKER_REGISTRY" \
                      IMAGE_REPO="$DOCKER_REPO" \
                      IMAGE_TAG="$IMAGE_TAG"
                '''
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh '''
                    trivy image \
                      --format table \
                      --output trivy-image-report.txt \
                      --timeout 10m \
                      "$DOCKER_IMAGE:$IMAGE_TAG"
                '''
            }
        }

        stage('Archive Trivy Image Report') {
            steps {
                archiveArtifacts(
                    artifacts: 'trivy-image-report.txt',
                    fingerprint: true
                )
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x

                        echo "$DOCKER_PASSWORD" | docker login \
                          --username "$DOCKER_USERNAME" \
                          --password-stdin

                        make push \
                          IMAGE_REG="$DOCKER_REGISTRY" \
                          IMAGE_REPO="$DOCKER_REPO" \
                          IMAGE_TAG="$IMAGE_TAG"

                        docker logout
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
            echo "Docker image: ${DOCKER_IMAGE}:${IMAGE_TAG}"
        }

        failure {
            echo "Pipeline failed. Check the failed stage."
        }

        always {
            echo "Pipeline execution finished."
        }
    }
}
