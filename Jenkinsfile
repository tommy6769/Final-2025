pipeline {
  //initial test pipeline. will add stages for testing once more things are up and running.
    agent any

    environment {
        //credentials here later
        DOCKERHUB_CREDENTIALS = 'docker-id'
        IMAGE_NAME ='tommy6769/final-project:latest'
    }

    stages {

        stage('Cloning Git') {
            steps {
                checkout scm
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

        stage('Snyk-Container-Scan') {
            agent {
                label 'Final-Agent'
            }
            steps {
                withCredentials([string(credentialsId: 'Snyk-API-Token-Credential-CC', variable: 'SNYK_TOKEN')]) {
                    sh """
                        echo "Authenticating Snyk..."
                        snyk auth ${SNYK_TOKEN}

                        echo "Running Snyk container scan for ${IMAGE_NAME}..."
                        snyk container test ${IMAGE_NAME} --file=Dockerfile --severity-threshold=medium
                    """
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
    }  
}
