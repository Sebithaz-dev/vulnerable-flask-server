pipeline {
    agent any

    environment {
        PROJECT_NAME = "vulnerable-server-OCY"
        SONARQUBE_URL = "http://sonarqube:9000"
        SONARQUBE_TOKEN = "sqa_135fbc1b29b75884ec54a41c3ca72b1b2a439b2c"
        TARGET_URL = "http://172.23.41.49:5000"
    }

    stages {
        stage('Install Python') {
            steps {
                sh '''
                    apt update
                    apt install -y python3 python3-venv python3-pip
                '''
            }
        }
        
        stage('Setup Environment') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }
        stage('Bandit Security Scan') {
            steps {
                sh '''
                    . venv/bin/activate
                    pip install bandit
                    mkdir -p security-reports
                    bandit -r vulnerable_flask_app.py -f html -o security-reports/bandit-report.html || true
                '''
            }
        }
        stage('Semgrep Security Scan') {
            steps {
                sh '''
                    . venv/bin/activate
                    pip install semgrep
                    mkdir -p security-reports
                    semgrep --config auto --json > security-reports/semgrep.json || true
                '''
            }
        }
        stage('Python Security Audit') {
            steps {
                sh '''
                    . venv/bin/activate
                    pip install pip-audit
                    mkdir -p dependency-check-report
                    pip-audit -r requirements.txt -f markdown -o dependency-check-report/pip-audit.md || true
                '''
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQubeScanner') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=$PROJECT_NAME \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=$SONARQUBE_URL \
                                -Dsonar.login=$SONARQUBE_TOKEN
                        """
                    }
                }
            }
        }
        stage('Dependency Check') {
            environment {
                NVD_API_KEY = credentials('nvdApiKey')
            }
            steps {
                dependencyCheck additionalArguments: "--scan ./requirements.txt --format HTML --out dependency-check-report --enableExperimental --enableRetired --nvdApiKey ${NVD_API_KEY}", odcInstallation: 'DependencyCheck'
            }
        }

        stage('Publish Reports') {
            steps {
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'dependency-check-report',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'OWASP Dependency Check Report'
                ])
            }
        }
    }

}
