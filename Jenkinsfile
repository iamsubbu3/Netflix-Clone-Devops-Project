pipeline {
    agent any

    tools {
        jdk 'jdk21'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        ansiColor('xterm')
    }

    environment {

        /* ---------- Build ---------- */
        DOCKER_BUILDKIT   = "1"

        /* ---------- Sonar ---------- */
        SCANNER_HOME      = tool 'sonar-scanner'
        SONAR_PROJECT_KEY = "Netflix-app"
        SONAR_HOST_URL    = "http://sonarqube.company.com"

        /* ---------- Docker ---------- */
        DOCKER_USER       = "iamsubbu3"
        DOCKER_IMAGE      = "netflix-app"
        REGISTRY_CRED     = "docker-credentials"

        /* Semantic Versioning */
        MAJOR_VERSION = "1"
        MINOR_VERSION = "0"
        PATCH_VERSION = "${BUILD_NUMBER}"

        DOCKER_TAG   = "v${MAJOR_VERSION}.${MINOR_VERSION}.${PATCH_VERSION}"
        FULL_IMAGE   = "${DOCKER_USER}/${DOCKER_IMAGE}:${DOCKER_TAG}"
        LATEST_IMAGE = "${DOCKER_USER}/${DOCKER_IMAGE}:latest"

        /* ---------- Kubernetes ---------- */
        AWS_REGION    = "us-east-1"
        EKS_CLUSTER   = "subbu-cluster"
        K8S_NAMESPACE = "subbu-1-ns"

        /* ---------- Notifications ---------- */
        NOTIFY_EMAIL = "subramanyam9979@gmail.com"
    }

    triggers {
        githubPush()
    }

    stages {

        /* ===================================================== */
        stage('Checkout') {
            steps {
                cleanWs()
                git branch: 'master',
                    url: 'https://github.com/iamsubbu3/Netflix-Clone-Devops-Project.git'
            }
        }

        /* ===================================================== */
        stage('SonarQube Scan') {
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
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /* ===================================================== */
        stage('Docker Build') {
            steps {
                script {
                    echo "🐳 Building ${FULL_IMAGE}"

                    docker.build(
                        FULL_IMAGE,
                        "--pull --label build_number=${BUILD_NUMBER} " +
                        "--label git_commit=${GIT_COMMIT} ."
                    )
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

        /* ===================================================== */
        stage('Docker Push Image') {
            steps {
                script {
                    withDockerRegistry(
                        [credentialsId: REGISTRY_CRED, url: 'https://index.docker.io/v1/']
                    ) {

                        echo "📦 Push immutable tag"
                        sh "docker push ${FULL_IMAGE}"

                        echo "🏷️ Push latest tag"
                        sh "docker tag ${FULL_IMAGE} ${LATEST_IMAGE}"
                        sh "docker push ${LATEST_IMAGE}"
                    }
                }
            }
        }

        /* ===================================================== */
        /*
        stage('Deploy to EKS (Auto Rollback Enabled)') {
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
                              --name ${EKS_CLUSTER}
                        """

                        script {
                            try {

                                echo "🚀 Deploying new image"

                                sh """
                                    kubectl set image deployment/netflix-deployment \
                                      netflix-app=${FULL_IMAGE} \
                                      -n ${K8S_NAMESPACE}
                                """

                                echo "⏳ Waiting for rollout..."
                                sh """
                                    kubectl rollout status deployment/netflix-deployment \
                                      -n ${K8S_NAMESPACE} \
                                      --timeout=180s
                                """

                                echo "✅ Deployment successful"

                            } catch (err) {

                                echo "❌ Deployment failed — rolling back..."

                                sh """
                                    kubectl rollout undo deployment/netflix-deployment \
                                      -n ${K8S_NAMESPACE}
                                """

                                sh """
                                    kubectl rollout status deployment/netflix-deployment \
                                      -n ${K8S_NAMESPACE}
                                """

                                error("Deployment failed. Rolled back to previous stable version.")
                            }
                        }
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
                to: "${NOTIFY_EMAIL}"
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
                to: "${NOTIFY_EMAIL}"
            )
        }

        always {
            echo "🧹 Cleanup completed (build images already removed)"
        }
    }
}
