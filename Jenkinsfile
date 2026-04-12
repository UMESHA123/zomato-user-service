pipeline {
    agent {
        label 'docker'  // Use a labeled agent with Docker installed
    }

    tools {
        jdk 'jdk-21'    // Ensure JDK 21 is configured in Jenkins Global Tool Configuration
    }

    environment {
        SERVICE_NAME    = 'user-service'
        REPO_NAME       = 'zomato-user-service'
        DOCKER_REGISTRY = 'docker.io/umesa123'
        DOCKER_CREDS    = 'docker-registry-credentials'
        IMAGE_TAG       = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
        DOCKER_BUILDKIT = '1'
        MAVEN_OPTS      = '-Dmaven.repo.local=.m2/repository'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '5'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds(abortPrevious: true)  // Abort older builds on same branch
        skipStagesAfterUnstable()
    }

    triggers {
        githubPush()
    }

    stages {

        // ==================== CHECKOUT ====================
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_AUTHOR    = sh(script: "git log -1 --pretty=format:'%an'", returnStdout: true).trim()
                    env.GIT_MSG       = sh(script: "git log -1 --pretty=format:'%s'",  returnStdout: true).trim()
                    env.GIT_SHORT     = env.GIT_COMMIT.take(7)
                    env.IS_MAIN       = (env.BRANCH_NAME == 'main') ? 'true' : 'false'
                    env.IS_PR         = (env.CHANGE_ID != null) ? 'true' : 'false'
                    env.FULL_IMAGE    = "${DOCKER_REGISTRY}/zomato-${SERVICE_NAME}"
                }
            }
        }

        // ==================== UNIT TESTS + COVERAGE ====================
        stage('Unit Tests') {
            steps {
                sh 'chmod +x mvnw'
                // Cache Maven dependencies across builds
                sh './mvnw test -B jacoco:report'
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
                    // Publish JaCoCo coverage report
                    jacoco(
                        execPattern: 'target/jacoco.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java',
                        exclusionPattern: '**/dto/**,**/config/**,**/entity/**'
                    )
                }
            }
        }

        // ==================== CODE QUALITY (SonarQube - optional) ====================
        stage('Code Quality') {
            when {
                expression {
                    // Only run if SonarQube credential exists
                    try {
                        withCredentials([string(credentialsId: 'sonarqube-url', variable: 'SONAR_URL')]) { return true }
                    } catch (Exception e) {
                        return false
                    }
                }
            }
            steps {
                withCredentials([string(credentialsId: 'sonarqube-url', variable: 'SONAR_URL')]) {
                    sh """
                        ./mvnw sonar:sonar \
                            -Dsonar.host.url=\${SONAR_URL} \
                            -Dsonar.projectKey=${REPO_NAME} \
                            -Dsonar.qualitygate.wait=true \
                            -B
                    """
                }
            }
        }

        // ==================== BUILD JAR ====================
        stage('Build') {
            steps {
                sh './mvnw package -DskipTests -B'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        // ==================== DOCKER BUILD ====================
        stage('Docker Build') {
            steps {
                sh """
                    docker build \
                        --label org.opencontainers.image.created=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
                        --label org.opencontainers.image.version=${IMAGE_TAG} \
                        --label org.opencontainers.image.revision=${env.GIT_COMMIT} \
                        --label org.opencontainers.image.source=https://github.com/UMESHA123/${REPO_NAME} \
                        --label org.opencontainers.image.title=zomato-${SERVICE_NAME} \
                        --cache-from ${FULL_IMAGE}:latest \
                        -t ${FULL_IMAGE}:${IMAGE_TAG} \
                        -t ${FULL_IMAGE}:latest \
                        .
                """
            }
        }

        // ==================== SECURITY SCAN (Trivy) ====================
        stage('Security Scan') {
            steps {
                script {
                    def trivyInstalled = sh(script: 'which trivy', returnStatus: true) == 0
                    if (trivyInstalled) {
                        // Generate JSON report for archiving
                        sh """
                            trivy image \
                                --severity HIGH,CRITICAL \
                                --format json \
                                --output trivy-report.json \
                                ${FULL_IMAGE}:${IMAGE_TAG}
                        """
                        archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true

                        // Fail build on CRITICAL vulnerabilities
                        sh """
                            trivy image \
                                --severity CRITICAL \
                                --exit-code 1 \
                                --format table \
                                ${FULL_IMAGE}:${IMAGE_TAG}
                        """
                    } else {
                        echo "WARNING: Trivy not installed. Install it for production: https://github.com/aquasecurity/trivy"
                        echo "Skipping security scan."
                    }
                }
            }
        }

        // ==================== PUSH IMAGE ====================
        stage('Push Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                retry(3) {
                    withDockerRegistry(credentialsId: DOCKER_CREDS, url: 'https://index.docker.io/v1/') {
                        sh "docker push ${FULL_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${FULL_IMAGE}:latest"
                    }
                }
            }
        }

        // ==================== DEPLOY TO STAGING ====================
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    echo "Deploying ${SERVICE_NAME}:${IMAGE_TAG} to STAGING..."

                    // ── OPTION A: SSH to staging server (recommended for Docker Compose) ──
                    // Uncomment and configure:
                    /*
                    sshagent(['staging-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no deployer@\${STAGING_SERVER} \\
                                'cd /opt/zomato && \\
                                 ./deploy.sh ${SERVICE_NAME} ${IMAGE_TAG}'
                        """
                    }
                    */

                    // ── OPTION B: Kubernetes ──
                    // sh "kubectl set image deployment/${SERVICE_NAME} ${SERVICE_NAME}=${FULL_IMAGE}:${IMAGE_TAG} -n staging --record"
                    // sh "kubectl rollout status deployment/${SERVICE_NAME} -n staging --timeout=120s"

                    // ── OPTION C: ArgoCD ──
                    // sh "argocd app set zomato-${SERVICE_NAME}-staging -p image.tag=${IMAGE_TAG}"
                    // sh "argocd app sync zomato-${SERVICE_NAME}-staging --prune"

                    echo ">>> Configure your staging deployment method above <<<"
                }
            }
        }

        // ==================== SMOKE TESTS ====================
        stage('Smoke Tests') {
            when {
                branch 'main'
            }
            steps {
                script {
                    echo "Running smoke tests against staging..."

                    // Uncomment and set your staging URL:
                    /*
                    retry(5) {
                        sleep(time: 10, unit: 'SECONDS')
                        sh """
                            HTTP_CODE=\$(curl -s -o /dev/null -w '%{http_code}' \
                                --max-time 10 \
                                http://\${STAGING_SERVER}:PORT/actuator/health)
                            if [ "\$HTTP_CODE" != "200" ]; then
                                echo "Health check returned \$HTTP_CODE"
                                exit 1
                            fi
                            echo "Health check passed (HTTP \$HTTP_CODE)"
                        """
                    }
                    */

                    echo ">>> Configure smoke tests for your staging URL <<<"
                }
            }
        }

        // ==================== DEPLOY TO PRODUCTION ====================
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                // Manual approval gate — pipeline pauses here
                timeout(time: 1, unit: 'HOURS') {
                    input message: "Deploy ${SERVICE_NAME}:${IMAGE_TAG} to PRODUCTION?",
                          ok: 'Approve & Deploy',
                          submitter: 'admin,deployer',
                          parameters: [
                              string(name: 'APPROVER_NOTE', defaultValue: '', description: 'Optional: reason for approval')
                          ]
                }

                script {
                    echo "Deploying ${SERVICE_NAME}:${IMAGE_TAG} to PRODUCTION..."
                    echo "Approved by: ${env.BUILD_USER ?: 'unknown'}"

                    // ── Same options as staging, targeting prod server ──
                    /*
                    sshagent(['production-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no deployer@\${PROD_SERVER} \\
                                'cd /opt/zomato && \\
                                 ./deploy.sh ${SERVICE_NAME} ${IMAGE_TAG}'
                        """
                    }
                    */

                    echo ">>> Configure your production deployment method above <<<"
                }
            }
        }

        // ==================== TAG RELEASE ====================
        stage('Tag Release') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Tag the commit that was deployed to production
                    sh """
                        git tag -a "v${IMAGE_TAG}" -m "Production release ${IMAGE_TAG} for ${SERVICE_NAME}"
                        git push origin "v${IMAGE_TAG}" || true
                    """
                }
            }
        }
    }

    post {
        always {
            // Clean up Docker images from build agent
            sh "docker rmi ${FULL_IMAGE}:${IMAGE_TAG} || true"
            sh "docker rmi ${FULL_IMAGE}:latest || true"
            cleanWs()
        }
        success {
            echo "SUCCESS: ${SERVICE_NAME}:${IMAGE_TAG} | ${env.GIT_MSG} | by ${env.GIT_AUTHOR}"
            // Uncomment for Slack:
            /*
            slackSend(
                color: 'good',
                channel: '#deployments',
                message: "*${SERVICE_NAME}* `${IMAGE_TAG}` — SUCCESS\n> ${env.GIT_MSG}\n> Author: ${env.GIT_AUTHOR}\n> <${env.BUILD_URL}|View Build>"
            )
            */
        }
        failure {
            echo "FAILED: ${SERVICE_NAME}:${IMAGE_TAG} | ${env.GIT_MSG} | by ${env.GIT_AUTHOR}"
            /*
            slackSend(
                color: 'danger',
                channel: '#deployments',
                message: "*${SERVICE_NAME}* — FAILED\n> ${env.GIT_MSG}\n> Author: ${env.GIT_AUTHOR}\n> <${env.BUILD_URL}|View Build>"
            )
            */
        }
        unstable {
            echo "UNSTABLE: ${SERVICE_NAME}:${IMAGE_TAG} — tests may have failed"
        }
    }
}
