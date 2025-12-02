pipeline {
  //initial test pipeline. will add stages for testing once more things are up and running.
    agent any

    environment {
        //credentials here later
        DOCKERHUB_CREDENTIALS = ''
        IMAGE_NAME =''
    }

    stages {

        stage('Cloning Git') {
            steps {
                checkout scm
            }
        }

        stage('Build-And-Tag') {
            agent {
                label 'appserver'
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
                label 'appserver-agent'
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
                label 'appserver'
            }
            steps {
                echo 'Starting deployment using docker-compose...'
                script {
                    dir("${WORKSPACE}") {
                        sh '''
                            docker-compose down
                            docker-compose up -d
                            docker ps
                        '''
                    }
                }
                echo 'Deployment completed successfully!'
            }
        }
    }  
}  
