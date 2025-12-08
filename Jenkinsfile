pipeline {
  //initial test pipeline. will add stages for testing once more things are up and running.
    agent any

    environment {
        //credentials here later
        DOCKERHUB_CREDENTIALS = 'docker-id'
        IMAGE_NAME ='tommy6769/final-project:latest'
        SNYK_TOKEN = 'Snyk-API-Token-Credential-CC'
      
        TRIVY_SEVERITY = "HIGH,CRITICAL"

        TARGET_URL = "http://172.238.185.189/" 
        REPORT_HTML = "zap_report.html"
        REPORT_JSON = "zap_report.json"
        ZAP_IMAGE = "ghcr.io/zaproxy/zaproxy:stable"
        REPORT_DIR = "${env.WORKSPACE}/zap_reports"
    }

    stages {

        stage('Cloning Git') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            agent {
                label 'Final-Agent'
            }
            steps {
                script {
                    def scannerHome = tool 'SonarQube-Scanner'
                    withSonarQubeEnv('SonarQube-installations') {
                        sh "${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=gameapp \
                            -Dsonar.sources=."
                    }
                }
            }
        }
 
        stage('Build-And-Tag') {
            agent {
                label 'Final-Agent'
            }
            steps {
                script {
                    // Build Docker image using Jenkins Docker Pipeline API
                    echo "Building Docker image ${IMAGE_NAME}..."
                    app = docker.build("${IMAGE_NAME}")
                    app.tag("latest")
                }
            }
        }


// Scan the things
        stage('SAST-TEST')
        {
            agent any
            steps
            {
                script 
                {
                    snykSecurity(
                        snykInstallation: 'Snyk-installations',
                        snykTokenId: 'Snyk-API-Token-Credential-CC',
                        severity: 'critical'
                    )
                }
            }
        }
        stage('Post-To-Dockerhub') {    
            agent {
                label 'Final-Agent'
            }
            steps {
                script {
                    echo "Pushing image ${IMAGE_NAME}:latest to Docker Hub..."
                    docker.withRegistry('https://registry.hub.docker.com', "${DOCKERHUB_CREDENTIALS}") {
                        app.push("latest")
                    }
                }
            }
        }

        stage('DEPLOYMENT') {    
            agent {
                label 'Final-Agent'
            }
            steps {
                echo 'Starting deployment using docker-compose...'
                script {
                    dir("${WORKSPACE}") {
                        sh '''
                            docker-compose down || true
                            docker-compose up -d || true
                            docker ps || true
                        '''
                    }
                }
                echo 'Deployment completed successfully!'
            }
        }
       post {
        always {
            publishHTML(target: [
                reportName: 'Trivy Image Security Report',
                reportDir: '.',
                reportFiles: 'trivy-report.html',
                alwaysLinkToLastBuild: true
            ])

            publishHTML(target: [
                reportName: 'OWASP ZAP DAST Report',
                reportDir: 'zap_reports',
                reportFiles: 'zap_report.html',
                alwaysLinkToLastBuild: true
            ])
        }
    }
    }  
}
