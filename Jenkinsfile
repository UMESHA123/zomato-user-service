// ============================================================
//  Java / Spring Boot service pipeline
//  Replace user-service and zomato-user-service when splitting
// ============================================================
pipeline {
    agent any

    environment {
        SERVICE_NAME    = 'user-service'
        REPO_NAME       = 'zomato-user-service'
        DOCKER_REGISTRY = 'docker.io/umesa123'
        DOCKER_CREDS    = 'docker-registry-credentials'
        IMAGE_TAG       = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
        FULL_IMAGE      = "${DOCKER_REGISTRY}/zomato-${SERVICE_NAME}"
        DOCKER_BUILDKIT = '1'
        MAVEN_OPTS      = '-Dmaven.repo.local=.m2/repository'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '5'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds(abortPrevious: true)
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
                    env.GIT_AUTHOR = sh(script: "git log -1 --pretty=format:'%an'", returnStdout: true).trim()
                    env.GIT_MSG    = sh(script: "git log -1 --pretty=format:'%s'",  returnStdout: true).trim()
                    env.GIT_SHORT  = env.GIT_COMMIT.take(7)

                    // Branch → environment mapping
                    if (env.BRANCH_NAME == 'main') {
                        env.DEPLOY_ENV = 'prod'
                    } else if (env.BRANCH_NAME == 'develop') {
                        env.DEPLOY_ENV = 'dev'
                    } else if (env.BRANCH_NAME?.startsWith('release/') || env.BRANCH_NAME == 'qa') {
                        env.DEPLOY_ENV = 'qa'
                    } else {
                        env.DEPLOY_ENV = 'none'
                    }
                    env.ENV_IMAGE_TAG = (env.DEPLOY_ENV != 'none')
                        ? "${env.DEPLOY_ENV}-${IMAGE_TAG}"
                        : IMAGE_TAG
                }
            }
        }

        // ==================== UNIT TESTS + COVERAGE ====================
        stage('Unit Tests') {
            steps {
                sh 'chmod +x mvnw 2>/dev/null || true'
                sh './mvnw test -B jacoco:report'
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
                    jacoco(
                        execPattern: 'target/jacoco.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java',
                        exclusionPattern: '**/dto/**,**/config/**,**/entity/**'
                    )
                }
            }
        }

        // ==================== CODE QUALITY (SonarQube — optional) ====================
        stage('Code Quality') {
            when {
                expression {
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
                        --label org.opencontainers.image.version=${env.ENV_IMAGE_TAG} \
                        --label org.opencontainers.image.revision=${env.GIT_COMMIT} \
                        --label org.opencontainers.image.source=https://github.com/UMESHA123/${REPO_NAME} \
                        --label org.opencontainers.image.title=zomato-${SERVICE_NAME} \
                        --cache-from ${FULL_IMAGE}:latest \
                        -t ${FULL_IMAGE}:${env.ENV_IMAGE_TAG} \
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
                        sh """
                            trivy image --severity HIGH,CRITICAL \
                                --format json --output trivy-report.json \
                                ${FULL_IMAGE}:${env.ENV_IMAGE_TAG}
                        """
                        archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
                        sh """
                            trivy image --severity CRITICAL \
                                --exit-code 1 --format table \
                                ${FULL_IMAGE}:${env.ENV_IMAGE_TAG}
                        """
                    } else {
                        echo "WARNING: Trivy not installed — skipping security scan"
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
                    branch pattern: 'release/.*', comparator: 'REGEXP'
                    branch 'qa'
                }
            }
            steps {
                retry(3) {
                    withDockerRegistry(credentialsId: DOCKER_CREDS, url: 'https://index.docker.io/v1/') {
                        sh "docker push ${FULL_IMAGE}:${env.ENV_IMAGE_TAG}"
                        sh "docker push ${FULL_IMAGE}:latest"
                    }
                }
            }
        }

        // ==================== DEPLOY TO DEV ====================
        stage('Deploy to Dev') {
            when { branch 'develop' }
            steps {
                script {
                    echo "Deploying ${SERVICE_NAME}:${env.ENV_IMAGE_TAG} to DEV..."
                    sshagent(['dev-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no deployer@\${DEV_SERVER} \
                                'cd /opt/zomato && ./deploy.sh ${SERVICE_NAME} ${env.ENV_IMAGE_TAG} dev'
                        """
                    }
                }
            }
        }

        // ==================== DEPLOY TO QA ====================
        stage('Deploy to QA') {
            when {
                anyOf {
                    branch pattern: 'release/.*', comparator: 'REGEXP'
                    branch 'qa'
                }
            }
            steps {
                script {
                    echo "Deploying ${SERVICE_NAME}:${env.ENV_IMAGE_TAG} to QA..."
                    sshagent(['qa-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no deployer@\${QA_SERVER} \
                                'cd /opt/zomato && ./deploy.sh ${SERVICE_NAME} ${env.ENV_IMAGE_TAG} qa'
                        """
                    }
                }
            }
        }

        // ==================== SMOKE TESTS (staging) ====================
        stage('Smoke Tests') {
            when { branch 'main' }
            steps {
                script {
                    echo "Deploying ${SERVICE_NAME}:${env.ENV_IMAGE_TAG} to STAGING for smoke tests..."
                    sshagent(['staging-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no deployer@\${STAGING_SERVER} \
                                'cd /opt/zomato && ./deploy.sh ${SERVICE_NAME} ${env.ENV_IMAGE_TAG} prod'
                        """
                    }
                    // Verify health after staging deploy
                    retry(5) {
                        sleep(time: 10, unit: 'SECONDS')
                        sh """
                            HTTP_CODE=\$(curl -s -o /dev/null -w '%{http_code}' \
                                --max-time 10 \
                                http://\${STAGING_SERVER}:\${STAGING_PORT}/actuator/health)
                            if [ "\$HTTP_CODE" != "200" ]; then
                                echo "Health check returned \$HTTP_CODE"
                                exit 1
                            fi
                            echo "Health check passed (HTTP \$HTTP_CODE)"
                        """
                    }
                }
            }
        }

        // ==================== PRODUCTION APPROVAL ====================
        stage('Production Approval') {
            when { branch 'main' }
            steps {
                timeout(time: 2, unit: 'HOURS') {
                    input message: "Deploy ${SERVICE_NAME}:${env.ENV_IMAGE_TAG} to PRODUCTION?",
                          ok: 'Approve & Deploy',
                          submitter: 'admin,deployer',
                          parameters: [
                              string(name: 'APPROVER_NOTE', defaultValue: '', description: 'Reason for approval')
                          ]
                }
            }
        }

        // ==================== DEPLOY TO PRODUCTION ====================
        stage('Deploy to Production') {
            when { branch 'main' }
            steps {
                script {
                    echo "Deploying ${SERVICE_NAME}:${env.ENV_IMAGE_TAG} to PRODUCTION..."
                    sshagent(['production-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no deployer@\${PROD_SERVER} \
                                'cd /opt/zomato && ./deploy.sh ${SERVICE_NAME} ${env.ENV_IMAGE_TAG} prod'
                        """
                    }
                }
            }
        }

        // ==================== TAG RELEASE ====================
        stage('Tag Release') {
            when { branch 'main' }
            steps {
                sh """
                    git tag -a "v${IMAGE_TAG}" -m "Production release ${IMAGE_TAG} for ${SERVICE_NAME}"
                    git push origin "v${IMAGE_TAG}" || true
                """
            }
        }
    }

    post {
        always {
            sh "docker rmi ${FULL_IMAGE}:${env.ENV_IMAGE_TAG} || true"
            sh "docker rmi ${FULL_IMAGE}:latest || true"
            cleanWs()
        }
        success {
            echo "SUCCESS: ${SERVICE_NAME}:${env.ENV_IMAGE_TAG} [${env.DEPLOY_ENV}] | ${env.GIT_MSG} | by ${env.GIT_AUTHOR}"
        }
        failure {
            echo "FAILED: ${SERVICE_NAME}:${env.ENV_IMAGE_TAG} [${env.DEPLOY_ENV}] | ${env.GIT_MSG} | by ${env.GIT_AUTHOR}"
        }
        unstable {
            echo "UNSTABLE: ${SERVICE_NAME}:${env.ENV_IMAGE_TAG} — tests may have failed"
        }
    }
}
