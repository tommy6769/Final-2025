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

         /* -------------------------------------------------------------------
           TRIVY SCAN (NON-BLOCKING)
        -------------------------------------------------------------------*/
        stage("SECURITY-IMAGE-SCANNER") {
            steps {
                script {
                    echo "Running Trivy scan..."

                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {

                        sh """
                            docker run --rm -v \$(pwd):/workspace aquasec/trivy:latest image \
                            --exit-code 0 \
                            --format json \
                            --output /workspace/trivy-report.json \
                            --severity ${TRIVY_SEVERITY} \
                            ${IMAGE_NAME}
                        """

                        sh """
                            docker run --rm -v \$(pwd):/workspace aquasec/trivy:latest image \
                            --exit-code 0 \
                            --format template \
                            --template "@/contrib/html.tpl" \
                            --output /workspace/trivy-report.html \
                            ${IMAGE_NAME}
                        """
                    }

                    archiveArtifacts artifacts: "trivy-report.json,trivy-report.html"
                }
            }
        }

        /* -------------------------------------------------------------------
           TRIVY SUMMARY (NON-BLOCKING)
        -------------------------------------------------------------------*/
        stage("Summarize Trivy Findings") {
            steps {
                script {
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {

                        if (!fileExists("trivy-report.json")) {
                            echo "No Trivy report."
                            return
                        }

                        def highCount = sh(
                            script: "grep -o '\"Severity\": \"HIGH\"' trivy-report.json | wc -l",
                            returnStdout: true
                        ).trim()

                        def criticalCount = sh(
                            script: "grep -o '\"Severity\": \"CRITICAL\"' trivy-report.json | wc -l",
                            returnStdout: true
                        ).trim()

                        echo "HIGH: ${highCount}"
                        echo "CRITICAL: ${criticalCount}"
                    }
                }
            }
        }

        /* -------------------------------------------------------------------
           ZAP BASELINE SCAN (NON-BLOCKING)
        -------------------------------------------------------------------*/
        stage('DAST') {
            steps {
                script {
                    echo "Running OWASP ZAP..."

                    sh "mkdir -p ${REPORT_DIR}"

                    // NEVER FAIL ZAP
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        sh """
                            docker run --rm --user root --network host \
                            -v ${REPORT_DIR}:/zap/wrk \
                            -t ${ZAP_IMAGE} zap-baseline.py \
                            -t ${TARGET_URL} \
                            -r ${REPORT_HTML} -J ${REPORT_JSON} || true
                        """
                    }

                    archiveArtifacts artifacts: "zap_reports/*", allowEmptyArchive: true
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
