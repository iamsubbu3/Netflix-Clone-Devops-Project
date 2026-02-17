pipeline {
    agent any

    tools {
        jdk 'jdk21'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {

        /* ---------- Build ---------- */
        DOCKER_BUILDKIT      = "1"

        /* ---------- SonarQube ---------- */
        SCANNER_HOME         = tool 'sonar-scanner'
        SONAR_HOST_URL       = 'http://sonarqube.company.com'
        SONAR_PROJECT_KEY    = 'Netflix-app'

        /* ---------- Docker ---------- */
        DOCKER_USER_NAME     = 'iamsubbu3'
        DOCKER_IMAGE         = 'netflix-app'
        REGISTRY_CRED        = 'docker-credentials'

        /* Semantic Versioning */
        MAJOR_VERSION        = "1"
        MINOR_VERSION        = "0"
        PATCH_VERSION        = "${BUILD_NUMBER}"

        DOCKER_TAG           = "v${MAJOR_VERSION}.${MINOR_VERSION}.${PATCH_VERSION}"
        FULL_IMAGE           = "${DOCKER_USER_NAME}/${DOCKER_IMAGE}:${DOCKER_TAG}"
        LATEST_IMAGE         = "${DOCKER_USER_NAME}/${DOCKER_IMAGE}:latest"

        /* ---------- AWS / EKS ---------- */
        EKS_CLUSTER_NAME     = 'subbu-cluster'
        AWS_REGION           = 'us-east-1'
        K8S_NAMESPACE        = 'subbu-1-ns'

        /* ---------- Notifications ---------- */
        NOTIFY_EMAILS        = 'subramanyam9979@gmail.com'
    }

    triggers {
        githubPush()
    }

    stages {

        /* ===================================================== */
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        /* ===================================================== */
        stage('Checkout Source Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/iamsubbu3/Netflix-Clone-Devops-Project.git'
            }
        }

        /* ===================================================== */
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=.
                    """
                }
            }
        }

        /* ===================================================== */
        stage('Sonar Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /* ===================================================== */
        stage('Docker Build & Push') {
            steps {
                script {

                    echo "🐳 Building Docker image: ${FULL_IMAGE}"

                    // Build image once
                    def myImage = docker.build("${FULL_IMAGE}", "--pull .")

                    // Push to Docker Hub (production style)
                    docker.withRegistry('https://index.docker.io/v1/', REGISTRY_CRED) {

                        echo "📦 Pushing immutable tag: ${DOCKER_TAG}"
                        myImage.push()

                        echo "🏷️ Pushing latest tag"
                        myImage.push('latest')
                    }

                    // Cleanup ONLY images created in this build
                    echo "🧹 Removing pipeline-built images..."
                    sh """
                        docker rmi ${FULL_IMAGE} || true
                        docker rmi ${LATEST_IMAGE} || true
                    """
                }
            }
        }

        /* ===================================================== */
        stage('Trivy Security Scan') {
            steps {
                script {
                    echo "🔍 Running Trivy scan..."

                    sh "trivy image --severity LOW,MEDIUM,HIGH ${FULL_IMAGE}"
                    sh "trivy image --exit-code 1 --severity CRITICAL ${FULL_IMAGE}"
                    sh "trivy image --format json -o trivy-report.json ${FULL_IMAGE}"

                    archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
                }
            }
        }

        // =====================================================
        // OPTIONAL DEPLOY STAGE (Enable when ready)
        /*
        stage('Deploy to EKS') {
            steps {
                dir('k8s-manifests') {

                    withCredentials([
                        aws(
                            credentialsId: 'aws-keys',
                            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                        )
                    ]) {

                        sh """
                            aws eks update-kubeconfig \
                              --region ${AWS_REGION} \
                              --name ${EKS_CLUSTER_NAME}
                        """

                        sh "kubectl apply -f . -n ${K8S_NAMESPACE}"

                        sh """
                            kubectl set image deployment/netflix-deployment \
                              netflix-app=${FULL_IMAGE} \
                              -n ${K8S_NAMESPACE}
                        """

                        sh """
                            kubectl rollout status deployment/netflix-deployment \
                              -n ${K8S_NAMESPACE}
                        """
                    }
                }
            }
        }
        */
    }

    /* ===================================================== */
    post {

        success {
            emailext(
                subject: "✅ SUCCESS | ${JOB_NAME} #${BUILD_NUMBER}",
                body: """
Hi Team,

✅ Pipeline executed successfully.

🔹 Job Name : ${JOB_NAME}
🔹 Build No : ${BUILD_NUMBER}

🐳 Immutable Image:
${FULL_IMAGE}

🏷️ Latest Tag:
${LATEST_IMAGE}

🔗 Jenkins Build:
${BUILD_URL}

🔗 SonarQube:
${SONAR_HOST_URL}/dashboard?id=${SONAR_PROJECT_KEY}

Regards,
DevOps Automation
""",
                to: "${NOTIFY_EMAILS}"
            )
        }

        failure {
            emailext(
                subject: "❌ FAILURE | ${JOB_NAME} #${BUILD_NUMBER}",
                body: """
Hi Team,

❌ Pipeline FAILED.

🔹 Job Name : ${JOB_NAME}
🔹 Build No : ${BUILD_NUMBER}

🔗 Logs:
${BUILD_URL}console

Regards,
DevOps Automation
""",
                to: "${NOTIFY_EMAILS}"
            )
        }

        always {
            echo "🧹 Cleanup completed (build images already removed)"
        }
    }
}
