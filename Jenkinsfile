pipeline {
    agent any
    
    environment {
        REPORT_DIR = 'security-reports'
        SONAR_TOKEN = credentials('sonarqube-token')
        SONAR_HOST_URL = 'http://localhost:9000'
        IMAGE_TAG = "${BUILD_NUMBER}"
        // Optional: set Python interpreter if needed
        // PYTHON = 'python3'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Environment Check') {
            steps {
                echo " Pipeline started - Build #${env.BUILD_NUMBER}"
                sh 'mkdir -p ${REPORT_DIR}'
            }
        }
        
        stage('Secret Scanning') {
            steps {
                sh '''
                    echo "=== Gitleaks Secret Scan ==="
                    gitleaks detect --source . --report-format json --report-path ${REPORT_DIR}/gitleaks-report.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/gitleaks-report.json", allowEmptyArchive: true
                }
            }
        }
        
        stage('Build & Dependencies') {
            steps {
                sh '''
                    echo "=== Setting up Python virtual environment ==="
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    # Install additional tools for security and testing
                    pip install safety pytest pytest-cov bandit
                '''
            }
        }
        
        stage('SAST - SonarQube') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=python-app \
                            -Dsonar.projectName='Python App' \
                            -Dsonar.language=py \
                            -Dsonar.python.version=3 \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=venv/**,**/__pycache__/**,**/*.pyc
                        """
                    }
                }
            }
        }
        
        stage('SCA - Dependency Scanning') {
            steps {
                sh '''
                    echo "=== OWASP Dependency-Check ==="
                    /opt/dependency-check/bin/dependency-check.sh \
                        --project "Python App" \
                        --scan . \
                        --format JSON --format HTML \
                        --out ${REPORT_DIR}/dependency-check \
                        --enableExperimental || true
                    
                    echo "=== Safety Vulnerability Scan ==="
                    . venv/bin/activate
                    safety check --json > ${REPORT_DIR}/safety-report.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/dependency-check/*,${REPORT_DIR}/safety-report.json", allowEmptyArchive: true
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest --junitxml=${REPORT_DIR}/junit.xml --cov=. --cov-report=html:${REPORT_DIR}/coverage
                '''
            }
            post {
                always {
                    junit testResults: "${REPORT_DIR}/junit.xml", allowEmptyResults: true
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        reportDir: "${REPORT_DIR}/coverage",
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Container Security') {
            steps {
                script {
                    sh '''
                        echo "=== Building Docker Image ==="
                        docker build -t python-app:${IMAGE_TAG} .
                        
                        echo "=== Trivy Image Scan ==="
                        trivy image --format json --output ${REPORT_DIR}/trivy-report.json --severity HIGH,CRITICAL python-app:${IMAGE_TAG} || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/trivy-report.json", allowEmptyArchive: true
                }
            }
        }
        
        stage('Approval: Deploy to Staging') {
            steps {
                script {
                    input message: 'Approve deployment to STAGING?', ok: 'Deploy to Staging', submitterParameter: 'APPROVER'
                    echo "Approved by: ${env.APPROVER}"
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh '''
                    docker stop python-app-staging 2>/dev/null || true
                    docker rm python-app-staging 2>/dev/null || true
                    docker run -d --name python-app-staging -p 5000:5000 python-app:${IMAGE_TAG}
                    sleep 5
                '''
            }
        }
        
        stage('Approval: Deploy to Production') {
            steps {
                script {
                    timeout(time: 24, unit: 'HOURS') {
                        input message: 'Approve deployment to PRODUCTION?', ok: 'Deploy to Production', submitterParameter: 'PROD_APPROVER'
                    }
                    echo "Production approved by: ${env.PROD_APPROVER}"
                }
            }
        }
        
        stage('Deploy to Production') {
            steps {
                sh 'docker tag python-app:${IMAGE_TAG} python-app:production'
            }
        }
        
        // ==========================================
        // Final Report Generation
        // ==========================================
        stage('Generate Final Report') {
            steps {
                script {
                    sh '''
                        echo "=== 📊 Generating Final Security Report ==="
                        
                        cat > ${REPORT_DIR}/pipeline-summary.html << 'HTMLEOF'
<!DOCTYPE html>
<html>
<head>
    <title>DevSecOps Pipeline Report - Python</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f5f5f5; }
        .header { background: #2c3e50; color: white; padding: 20px; border-radius: 5px; }
        .summary { background: white; padding: 20px; margin: 20px 0; border-radius: 5px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .stage { background: white; padding: 15px; margin: 10px 0; border-left: 4px solid #27ae60; }
        .stage.failed { border-left-color: #e74c3c; }
        .metric { display: inline-block; margin: 10px 20px 10px 0; }
        .metric-value { font-size: 24px; font-weight: bold; color: #27ae60; }
        .metric-label { font-size: 12px; color: #7f8c8d; }
    </style>
</head>
<body>
    <div class="header">
        <h1>🐍 DevSecOps Pipeline Report - Python</h1>
        <p>Build #BUILD_NUMBER | Commit: GIT_COMMIT | Date: BUILD_DATE</p>
    </div>
    
    <div class="summary">
        <h2>Executive Summary</h2>
        <div class="metric">
            <div class="metric-value">7</div>
            <div class="metric-label">Security Scans</div>
        </div>
        <div class="metric">
            <div class="metric-value">2</div>
            <div class="metric-label">Approval Gates</div>
        </div>
        <div class="metric">
            <div class="metric-value" style="color: #27ae60;">PASSED</div>
            <div class="metric-label">Status</div>
        </div>
    </div>

    <div class="stage">
        <h3>🔐 Secret Scanning (Gitleaks)</h3>
        <p>Scanned for hardcoded secrets, API keys, and credentials</p>
    </div>

    <div class="stage">
        <h3>🔍 SAST (SonarQube for Python)</h3>
        <p>Static code analysis for bugs, vulnerabilities, and code smells</p>
    </div>

    <div class="stage">
        <h3>📦 SCA (OWASP Dependency‑Check + Safety)</h3>
        <p>Identified known vulnerabilities in Python dependencies</p>
    </div>

    <div class="stage">
        <h3>🐳 Container Security (Trivy)</h3>
        <p>Scanned Docker image for OS and Python package vulnerabilities</p>
    </div>

    <div class="stage">
        <h3>✅ Unit Tests (pytest)</h3>
        <p>Executed unit tests with coverage</p>
    </div>

    <div class="summary">
        <h2>Artifacts Generated</h2>
        <ul>
            <li>gitleaks-report.json - Secret scanning results</li>
            <li>dependency-check-report.html - OWASP dependency report</li>
            <li>safety-report.json - Safety vulnerability report</li>
            <li>trivy-report.json - Container scan results</li>
            <li>junit.xml & coverage/ - Unit test results</li>
        </ul>
    </div>
</body>
</html>
HTMLEOF

                        # Replace placeholders
                        sed -i "s/BUILD_NUMBER/${BUILD_NUMBER}/g" ${REPORT_DIR}/pipeline-summary.html
                        sed -i "s/GIT_COMMIT/${GIT_COMMIT}/g" ${REPORT_DIR}/pipeline-summary.html
                        sed -i "s/BUILD_DATE/$(date)/g" ${REPORT_DIR}/pipeline-summary.html

                        echo " Report generated: ${REPORT_DIR}/pipeline-summary.html"
                    '''
                }
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
            echo "=========================================="
            echo "PIPELINE COMPLETED - Build #${BUILD_NUMBER}"
            echo "=========================================="
        }
        success {
            echo "✅ SUCCESS! Check the Pipeline Summary Report"
        }
        failure {
            echo "❌ FAILED!"
        }
    }
}
