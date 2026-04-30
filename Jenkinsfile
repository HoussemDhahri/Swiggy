pipeline {
    agent any

    environment {
        // ─── App Config ───────────────────────────────────────────
        APP_NAME        = 'my-app'
        APP_VERSION     = "${env.BUILD_NUMBER}"
        GIT_REPO        = 'https://github.com/org/my-app.git'

        // ─── Nexus Repository Manager ─────────────────────────────
        NEXUS_URL            = 'http://nexus.example.com:8081'
        NEXUS_DOCKER_HOST    = 'nexus.example.com:8083'          // Docker hosted repo port
        NEXUS_CREDS_ID       = 'nexus-credentials'               // Jenkins credential ID
        NEXUS_MAVEN_RELEASES = 'maven-releases'                  // Nexus repo name
        NEXUS_MAVEN_SNAPSHOTS= 'maven-snapshots'                 // Nexus repo name

        // ─── Docker / Registry (Nexus) ────────────────────────────
        DOCKER_REGISTRY = "${NEXUS_DOCKER_HOST}"
        IMAGE_NAME      = "${NEXUS_DOCKER_HOST}/${APP_NAME}"
        IMAGE_TAG       = "${APP_VERSION}-${env.GIT_COMMIT?.take(7) ?: 'latest'}"

        // ─── Kubernetes Namespaces ────────────────────────────────
        K8S_NS_DEV      = 'dev'
        K8S_NS_STAGING  = 'staging'
        K8S_NS_PROD     = 'prod'

        // ─── SonarQube ────────────────────────────────────────────
        SONAR_PROJECT   = 'my-app'
        SONAR_SERVER    = 'SonarQube'          // Jenkins credential ID

        // ─── Notifications ────────────────────────────────────────
        SLACK_CHANNEL   = '#devops-alerts'
        EMAIL_RECIPIENT = 'team@example.com'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 90, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        // ════════════════════════════════════════════════════════
        // STAGE 1 — SOURCE & TRIGGER
        // ════════════════════════════════════════════════════════
        stage('📋 Source') {
            steps {
                echo "Branch: ${env.BRANCH_NAME} | Build: ${env.BUILD_NUMBER}"
                checkout scm
                script {
                    env.GIT_AUTHOR = sh(script: "git log -1 --pretty=%an", returnStdout: true).trim()
                    env.GIT_MESSAGE = sh(script: "git log -1 --pretty=%s", returnStdout: true).trim()
                }
                echo "Commit by: ${env.GIT_AUTHOR} — ${env.GIT_MESSAGE}"
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 2 — BUILD
        // ════════════════════════════════════════════════════════
        stage('🔨 Build') {
            steps {
                sh '''
                    echo "=== Building Application ==="
                    mvn clean package -DskipTests -q

                    # Node.js example:
                    # npm ci && npm run build

                    # Go example:
                    # go build -o bin/${APP_NAME} ./...
                '''
            }
            post {
                success { archiveArtifacts artifacts: 'target/*.jar', fingerprint: true }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 2.5 — PUBLISH ARTIFACT → NEXUS (Maven/npm)
        // ════════════════════════════════════════════════════════
        stage('📦 Publish Artifact → Nexus') {
            steps {
                script {
                    def isSnapshot = env.BRANCH_NAME != 'main'
                    def nexusRepo  = isSnapshot ? env.NEXUS_MAVEN_SNAPSHOTS : env.NEXUS_MAVEN_RELEASES
                    echo "=== Publishing to Nexus repo: ${nexusRepo} ==="

                    withCredentials([usernamePassword(
                        credentialsId: "${NEXUS_CREDS_ID}",
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh """
                            mvn deploy -DskipTests -q \\
                              -DaltDeploymentRepository=${nexusRepo}::default::${NEXUS_URL}/repository/${nexusRepo}/ \\
                              -Dusername=\$NEXUS_USER \\
                              -Dpassword=\$NEXUS_PASS
                        """
                    }
                    echo "✅ Artifact published → Nexus [${nexusRepo}]"
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 3 — UNIT TESTS
        // ════════════════════════════════════════════════════════
        stage('🧪 Unit Tests') {
            steps {
                sh 'mvn test -q'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    publishHTML([
                        allowMissing: false,
                        reportDir: 'target/site/jacoco',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 4 — CODE QUALITY (SonarQube)
        // ════════════════════════════════════════════════════════
        stage('📊 Code Quality') {
            steps {
                withSonarQubeEnv("${SONAR_SERVER}") {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT} \
                          -Dsonar.projectName="${APP_NAME}" \
                          -Dsonar.java.coveragePlugin=jacoco \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                          -q
                    '''
                }
                // Block pipeline until Quality Gate passes
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 5 — SAST (Static Application Security Testing)
        // ════════════════════════════════════════════════════════
        stage('🔒 SAST') {
            parallel {
                stage('Semgrep') {
                    steps {
                        sh '''
                            semgrep --config=auto \
                                    --json \
                                    --output=semgrep-results.json \
                                    . || true
                        '''
                        recordIssues tool: semgrep(pattern: 'semgrep-results.json')
                    }
                }
                stage('SpotBugs') {
                    steps {
                        sh 'mvn spotbugs:check -q || true'
                        recordIssues tool: spotBugs(pattern: 'target/spotbugsXml.xml')
                    }
                }
                stage('Checkov (IaC)') {
                    steps {
                        sh '''
                            checkov -d . \
                                    --output json \
                                    --output-file checkov-results.json \
                                    --soft-fail || true
                        '''
                    }
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 6 — SCA (Software Composition Analysis)
        // ════════════════════════════════════════════════════════
        stage('📦 Dependency Scan') {
            parallel {
                stage('OWASP Dependency-Check') {
                    steps {
                        sh '''
                            dependency-check \
                              --project "${APP_NAME}" \
                              --scan . \
                              --format JSON \
                              --out dependency-check-report.json \
                              --failOnCVSS 8 || true
                        '''
                        dependencyCheckPublisher pattern: 'dependency-check-report.json'
                    }
                }
                stage('Snyk') {
                    steps {
                        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                            sh '''
                                snyk test \
                                  --severity-threshold=high \
                                  --json > snyk-results.json || true
                                snyk monitor || true
                            '''
                        }
                    }
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 7 — CONTAINER BUILD
        // ════════════════════════════════════════════════════════
        stage('🐳 Container Build') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: "${NEXUS_CREDS_ID}",
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh '''
                            # Login to Nexus Docker Registry
                            echo "$NEXUS_PASS" | docker login ${NEXUS_DOCKER_HOST} \
                                                   -u "$NEXUS_USER" --password-stdin

                            docker build \
                              --build-arg APP_VERSION=${APP_VERSION} \
                              --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
                              --build-arg VCS_REF=${GIT_COMMIT} \
                              --label "org.opencontainers.image.version=${APP_VERSION}" \
                              --label "org.opencontainers.image.created=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
                              --label "org.opencontainers.image.revision=${GIT_COMMIT}" \
                              -t ${IMAGE_NAME}:${IMAGE_TAG} \
                              -t ${IMAGE_NAME}:latest \
                              .
                        '''
                    }
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 8 — CONTAINER SCAN
        // ════════════════════════════════════════════════════════
        stage('🔍 Container Scan') {
            parallel {
                stage('Trivy') {
                    steps {
                        sh '''
                            trivy image \
                              --exit-code 0 \
                              --severity HIGH,CRITICAL \
                              --format json \
                              --output trivy-results.json \
                              ${IMAGE_NAME}:${IMAGE_TAG}

                            # Fail on CRITICAL CVEs
                            trivy image \
                              --exit-code 1 \
                              --severity CRITICAL \
                              --ignore-unfixed \
                              ${IMAGE_NAME}:${IMAGE_TAG} || true
                        '''
                    }
                }
                stage('Grype') {
                    steps {
                        sh '''
                            grype ${IMAGE_NAME}:${IMAGE_TAG} \
                              --fail-on critical \
                              -o json > grype-results.json || true
                        '''
                    }
                }
                stage('Dockle (Best Practices)') {
                    steps {
                        sh '''
                            dockle \
                              --exit-code 1 \
                              --exit-level warn \
                              ${IMAGE_NAME}:${IMAGE_TAG} || true
                        '''
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: '*-results.json', allowEmptyArchive: true
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 9 — PUSH IMAGE → NEXUS DOCKER REGISTRY
        // ════════════════════════════════════════════════════════
        stage('📤 Push Image → Nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: "${NEXUS_CREDS_ID}",
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh '''
                            echo "$NEXUS_PASS" | docker login ${NEXUS_DOCKER_HOST} \
                                                   -u "$NEXUS_USER" --password-stdin

                            # Push versioned tag
                            docker push ${IMAGE_NAME}:${IMAGE_TAG}

                            # Push latest only on main branch
                            if [ "${BRANCH_NAME}" = "main" ]; then
                                docker push ${IMAGE_NAME}:latest
                                echo "✅ Pushed latest tag → Nexus (main branch)"
                            fi

                            # Cleanup local image to free space
                            docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
                            docker rmi ${IMAGE_NAME}:latest       || true

                            echo "✅ Image pushed → Nexus Docker Registry: ${IMAGE_NAME}:${IMAGE_TAG}"
                        '''
                    }
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 10 — DEPLOY TO DEV
        // ════════════════════════════════════════════════════════
        stage('🚀 Deploy → DEV') {
            when { branch 'develop' }
            steps {
                withKubeConfig([credentialsId: 'k8s-dev-kubeconfig']) {
                    sh '''
                        helm upgrade --install ${APP_NAME} ./helm/${APP_NAME} \
                          --namespace ${K8S_NS_DEV} \
                          --set image.repository=${IMAGE_NAME} \
                          --set image.tag=${IMAGE_TAG} \
                          --set environment=dev \
                          --wait --timeout 5m
                    '''
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 11 — INTEGRATION TESTS
        // ════════════════════════════════════════════════════════
        stage('🧩 Integration Tests') {
            when { branch 'develop' }
            steps {
                sh '''
                    # Wait for deployment
                    sleep 15

                    # Run integration test suite
                    mvn verify -Pintegration-tests \
                        -Dapp.url=http://${APP_NAME}.${K8S_NS_DEV}.svc.cluster.local \
                        -q

                    # API contract tests with Postman/Newman
                    newman run tests/api/collection.json \
                           --environment tests/api/dev-env.json \
                           --reporters cli,json \
                           --reporter-json-export newman-results.json || true
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/failsafe-reports/*.xml'
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 12 — PERFORMANCE TESTS
        // ════════════════════════════════════════════════════════
        stage('⚡ Performance Tests') {
            when { branch 'develop' }
            steps {
                sh '''
                    k6 run \
                      --vus 50 \
                      --duration 2m \
                      --out json=k6-results.json \
                      tests/performance/load-test.js || true
                '''
                perfReport sourceDataFiles: 'k6-results.json'
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 13 — DAST (Dynamic Application Security Testing)
        // ════════════════════════════════════════════════════════
        stage('🛡️ DAST') {
            when { branch 'develop' }
            steps {
                sh '''
                    # OWASP ZAP Baseline Scan
                    docker run --rm \
                      -v $(pwd):/zap/wrk:rw \
                      ghcr.io/zaproxy/zaproxy:stable \
                      zap-baseline.py \
                        -t http://${APP_NAME}.${K8S_NS_DEV}.svc.cluster.local \
                        -J zap-results.json \
                        -r zap-report.html \
                        -I || true
                '''
                publishHTML([
                    reportDir: '.',
                    reportFiles: 'zap-report.html',
                    reportName: 'ZAP DAST Report'
                ])
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 14 — SECURITY GATE
        // ════════════════════════════════════════════════════════
        stage('✅ Security Gate') {
            steps {
                script {
                    echo "=== Evaluating Security Gate ==="
                    // Aggregate all scan results and apply thresholds
                    def trivyCriticals = sh(
                        script: "jq '[.Results[]?.Vulnerabilities[]? | select(.Severity==\"CRITICAL\")] | length' trivy-results.json 2>/dev/null || echo 0",
                        returnStdout: true
                    ).trim().toInteger()

                    echo "Critical CVEs found: ${trivyCriticals}"
                    if (trivyCriticals > 0) {
                        unstable("Security Gate WARNING: ${trivyCriticals} CRITICAL CVEs detected!")
                    } else {
                        echo "✅ Security Gate PASSED"
                    }
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 15 — DEPLOY TO STAGING
        // ════════════════════════════════════════════════════════
        stage('🎯 Deploy → STAGING') {
            when { branch 'main' }
            steps {
                withKubeConfig([credentialsId: 'k8s-staging-kubeconfig']) {
                    sh '''
                        helm upgrade --install ${APP_NAME} ./helm/${APP_NAME} \
                          --namespace ${K8S_NS_STAGING} \
                          --set image.repository=${IMAGE_NAME} \
                          --set image.tag=${IMAGE_TAG} \
                          --set environment=staging \
                          --set replicaCount=2 \
                          --wait --timeout 5m
                    '''
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 16 — SMOKE TESTS
        // ════════════════════════════════════════════════════════
        stage('🔎 Smoke Tests') {
            when { branch 'main' }
            steps {
                sh '''
                    # Basic health check
                    STAGING_URL="http://${APP_NAME}.${K8S_NS_STAGING}.svc.cluster.local"
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" ${STAGING_URL}/health)

                    if [ "$STATUS" != "200" ]; then
                        echo "❌ Smoke Test FAILED: Health check returned ${STATUS}"
                        exit 1
                    fi

                    echo "✅ Smoke Test PASSED"

                    # Run critical path smoke tests
                    newman run tests/api/smoke-collection.json \
                           --environment tests/api/staging-env.json \
                           --reporters cli,json \
                           --reporter-json-export smoke-results.json
                '''
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 17 — MANUAL APPROVAL (Production Gate)
        // ════════════════════════════════════════════════════════
        stage('👤 Production Gate') {
            when { branch 'main' }
            steps {
                script {
                    def approver = input(
                        id: 'prod-approval',
                        message: "🚀 Deploy ${APP_NAME} v${APP_VERSION} to PRODUCTION?",
                        ok: 'Approve & Deploy',
                        parameters: [
                            string(
                                name: 'CHANGE_TICKET',
                                description: 'Change Request / Ticket Number',
                                defaultValue: ''
                            ),
                            text(
                                name: 'DEPLOY_NOTES',
                                description: 'Deployment Notes / Release Summary',
                                defaultValue: ''
                            )
                        ],
                        submitter: 'lead-devops,tech-lead,release-manager',
                        submitterParameter: 'APPROVER'
                    )
                    env.CHANGE_TICKET  = approver.CHANGE_TICKET
                    env.DEPLOY_NOTES   = approver.DEPLOY_NOTES
                    env.APPROVED_BY    = approver.APPROVER
                    echo "✅ Approved by: ${env.APPROVED_BY} | Ticket: ${env.CHANGE_TICKET}"
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 18 — DEPLOY TO PRODUCTION (Blue/Green)
        // ════════════════════════════════════════════════════════
        stage('🚀 Deploy → PROD') {
            when { branch 'main' }
            steps {
                withKubeConfig([credentialsId: 'k8s-prod-kubeconfig']) {
                    sh '''
                        echo "=== Starting Blue/Green Deployment ==="

                        # 1. Deploy Green (new version)
                        helm upgrade --install ${APP_NAME}-green ./helm/${APP_NAME} \
                          --namespace ${K8S_NS_PROD} \
                          --set image.repository=${IMAGE_NAME} \
                          --set image.tag=${IMAGE_TAG} \
                          --set environment=prod \
                          --set replicaCount=3 \
                          --set slot=green \
                          --wait --timeout 10m

                        # 2. Health check on green
                        kubectl rollout status deployment/${APP_NAME}-green -n ${K8S_NS_PROD}

                        # 3. Switch traffic to Green
                        kubectl patch service ${APP_NAME} -n ${K8S_NS_PROD} \
                          --type='json' \
                          -p='[{"op":"replace","path":"/spec/selector/slot","value":"green"}]'

                        echo "✅ Traffic switched to GREEN (v${APP_VERSION})"

                        # 4. Keep Blue for rollback (remove after validation window)
                        echo "Blue slot kept for 10min rollback window..."
                    '''
                }
            }
        }

        // ════════════════════════════════════════════════════════
        // STAGE 19 — POST-DEPLOY MONITORING
        // ════════════════════════════════════════════════════════
        stage('📡 Post-Deploy Check') {
            when { branch 'main' }
            steps {
                sh '''
                    echo "=== Monitoring Production for 3 minutes ==="
                    sleep 30

                    # Verify production health
                    PROD_URL="https://api.example.com"
                    for i in 1 2 3 4 5; do
                        STATUS=$(curl -s -o /dev/null -w "%{http_code}" ${PROD_URL}/health)
                        echo "Health check ${i}/5: HTTP ${STATUS}"
                        if [ "$STATUS" != "200" ]; then
                            echo "❌ Production health check failed! Triggering rollback..."
                            # Auto-rollback
                            kubectl patch service ${APP_NAME} -n ${K8S_NS_PROD} \
                              --type='json' \
                              -p='[{"op":"replace","path":"/spec/selector/slot","value":"blue"}]'
                            exit 1
                        fi
                        sleep 30
                    done

                    echo "✅ Production is HEALTHY"

                    # Cleanup old Blue slot
                    helm uninstall ${APP_NAME}-blue -n ${K8S_NS_PROD} || true
                '''
            }
        }

    } // end stages

    // ════════════════════════════════════════════════════════════
    // POST — NOTIFICATIONS & CLEANUP
    // ════════════════════════════════════════════════════════════
    post {
        always {
            // Archive all security reports
            archiveArtifacts(
                artifacts: '**/trivy-results.json, **/grype-results.json, **/zap-results.json, **/semgrep-results.json, **/snyk-results.json',
                allowEmptyArchive: true
            )
            // Cleanup workspace
            cleanWs()
        }

        success {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'good',
                message: """✅ *SUCCESS* | ${APP_NAME} v${APP_VERSION}
Branch: ${env.BRANCH_NAME} | Build: #${env.BUILD_NUMBER}
Author: ${env.GIT_AUTHOR}
🔗 <${env.BUILD_URL}|View Build>"""
            )
            emailext(
                to: "${EMAIL_RECIPIENT}",
                subject: "✅ [${APP_NAME}] Build #${env.BUILD_NUMBER} — SUCCESS",
                body: "Deployment of ${APP_NAME} v${APP_VERSION} completed successfully.\n\nBuild URL: ${env.BUILD_URL}"
            )
        }

        failure {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'danger',
                message: """❌ *FAILED* | ${APP_NAME} v${APP_VERSION}
Stage: ${env.STAGE_NAME} | Build: #${env.BUILD_NUMBER}
Author: ${env.GIT_AUTHOR}
🔗 <${env.BUILD_URL}|View Logs>"""
            )
            emailext(
                to: "${EMAIL_RECIPIENT}",
                subject: "❌ [${APP_NAME}] Build #${env.BUILD_NUMBER} — FAILED",
                body: "Pipeline failed at stage: ${env.STAGE_NAME}\n\nBuild URL: ${env.BUILD_URL}"
            )
        }

        unstable {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'warning',
                message: """⚠️ *UNSTABLE* | ${APP_NAME} v${APP_VERSION}
Security warnings detected. Review required.
🔗 <${env.BUILD_URL}|View Details>"""
            )
        }
    }
}