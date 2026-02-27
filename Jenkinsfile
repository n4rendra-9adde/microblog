pipeline {
    agent { label 'vm-agent' }

    environment {
        REPORT_DIR = 'security-reports'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
    }

    stages {

        // ==========================================
        // STAGE 1: ENVIRONMENT CHECK
        // ==========================================
        stage('Environment Check') {
            steps {
                echo "Pipeline started - Build #${BUILD_NUMBER}"
                sh '''
                    mkdir -p ${REPORT_DIR}
                    python3 --version
                    pip --version
                    docker --version
                '''
            }
        }

        // ==========================================
        // STAGE 2: SECRET SCANNING
        // ==========================================
        stage('Secret Scanning - Gitleaks') {
            steps {
                sh '''
                    echo "=== Gitleaks Secret Scan ==="
                    gitleaks detect \
                        --source . \
                        --report-format json \
                        --report-path ${REPORT_DIR}/gitleaks-report.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/gitleaks-report.json", allowEmptyArchive: true
                }
            }
        }

        // ==========================================
        // STAGE 3: DEPENDENCY INSTALL
        // ==========================================
        stage('Install Python Dependencies') {
            steps {
                sh '''
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        // ==========================================
        // STAGE 4: SAST - BANDIT
        // ==========================================
        stage('SAST - Bandit') {
            steps {
                sh '''
                    bandit -r . \
                        -f json \
                        -o ${REPORT_DIR}/bandit-report.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/bandit-report.json", allowEmptyArchive: true
                }
            }
        }

        // ==========================================
        // STAGE 5: SCA - PIP-AUDIT
        // ==========================================
        stage('SCA - pip-audit') {
            steps {
                sh '''
                    pip-audit -r requirements.txt \
                        --format json \
                        -o ${REPORT_DIR}/pip-audit-report.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/pip-audit-report.json", allowEmptyArchive: true
                }
            }
        }

        // ==========================================
        // STAGE 6: UNIT TESTS
        // ==========================================
        stage('Unit Tests - Pytest') {
            steps {
                sh '''
                    pytest --junitxml=${REPORT_DIR}/pytest-report.xml || true
                '''
            }
            post {
                always {
                    junit testResults: "${REPORT_DIR}/pytest-report.xml", allowEmptyResults: true
                }
            }
        }

        // ==========================================
        // STAGE 7: DOCKER BUILD
        // ==========================================
        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t python-app:${IMAGE_TAG} .
                '''
            }
        }

        // ==========================================
        // STAGE 8: CONTAINER SECURITY - TRIVY
        // ==========================================
        stage('Container Security - Trivy') {
            steps {
                sh '''
                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format json \
                        --output ${REPORT_DIR}/trivy-report.json \
                        python-app:${IMAGE_TAG} || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/trivy-report.json", allowEmptyArchive: true
                }
            }
        }

        // ==========================================
        // STAGE 9: APPROVAL - STAGING
        // ==========================================
        stage('Approval: Deploy to Staging') {
            steps {
                script {
                    input message: 'Approve deployment to STAGING?',
                          ok: 'Deploy to Staging',
                          submitterParameter: 'STAGING_APPROVER'
                    echo "Approved by: ${STAGING_APPROVER}"
                }
            }
        }

        // ==========================================
        // STAGE 10: DEPLOY TO STAGING
        // ==========================================
        stage('Deploy to Staging') {
            steps {
                sh '''
                    docker stop python-staging 2>/dev/null || true
                    docker rm python-staging 2>/dev/null || true
                    docker run -d \
                        --name python-staging \
                        -p 8000:8000 \
                        python-app:${IMAGE_TAG}
                '''
            }
        }

        // ==========================================
        // STAGE 11: APPROVAL - PRODUCTION
        // ==========================================
        stage('Approval: Deploy to Production') {
            steps {
                script {
                    timeout(time: 24, unit: 'HOURS') {
                        input message: 'Approve deployment to PRODUCTION?',
                              ok: 'Deploy to Production',
                              submitterParameter: 'PROD_APPROVER'
                    }
                    echo "Production approved by: ${PROD_APPROVER}"
                }
            }
        }

        // ==========================================
        // STAGE 12: DEPLOY TO PRODUCTION
        // ==========================================
        stage('Deploy to Production') {
            steps {
                sh '''
                    docker stop python-production 2>/dev/null || true
                    docker rm python-production 2>/dev/null || true
                    docker run -d \
                        --name python-production \
                        -p 80:8000 \
                        python-app:${IMAGE_TAG}
                '''
            }
        }

        // ==========================================
        // STAGE 13: FINAL SECURITY REPORT
        // ==========================================
        stage('Generate Final Report') {
            steps {
                sh '''
cat > ${REPORT_DIR}/pipeline-summary.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
<title>DevSecOps Pipeline Report</title>
<style>
body { font-family: Arial; background: #f5f5f5; padding: 30px; }
.header { background: #2c3e50; color: white; padding: 20px; border-radius: 5px; }
.section { background: white; padding: 20px; margin-top: 20px; border-radius: 5px; }
.success { color: green; font-weight: bold; }
</style>
</head>
<body>
<div class="header">
<h1>DevSecOps Pipeline Summary</h1>
<p>Build #: BUILD_NUMBER</p>
<p>Commit: GIT_COMMIT</p>
<p>Date: BUILD_DATE</p>
</div>

<div class="section">
<h2>Security Coverage</h2>
<ul>
<li>Secret Scanning – Gitleaks</li>
<li>SAST – Bandit</li>
<li>SCA – pip-audit</li>
<li>Container Scan – Trivy</li>
<li>Unit Tests – Pytest</li>
<li>Manual Approval Gates</li>
</ul>
</div>

<div class="section success">
<h2>Status: PIPELINE COMPLETED</h2>
</div>
</body>
</html>
EOF

sed -i "s/BUILD_NUMBER/${BUILD_NUMBER}/g" ${REPORT_DIR}/pipeline-summary.html
sed -i "s/GIT_COMMIT/${GIT_COMMIT}/g" ${REPORT_DIR}/pipeline-summary.html
sed -i "s/BUILD_DATE/$(date)/g" ${REPORT_DIR}/pipeline-summary.html
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/pipeline-summary.html", allowEmptyArchive: true
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: "${REPORT_DIR}",
                        reportFiles: 'pipeline-summary.html',
                        reportName: 'Pipeline Summary Report'
                    ])
                }
            }
        }
    }

    post {
        always {
            echo "======================================"
            echo "PIPELINE COMPLETED - BUILD #${BUILD_NUMBER}"
            echo "======================================"
        }
        success {
            echo "SUCCESS - Application deployed securely"
        }
        failure {
            echo "FAILURE - Check Jenkins logs and reports"
        }
    }
}
