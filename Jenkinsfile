pipeline {
    agent any 
    
    // Define environment variables
    environment {
        // Sonarqube related variables
        SONAR_SCANNER_HOME = '/opt/sonar-scanner'
        SONAR_PROJECT_KEY = 'jenkins'
        // Harbor registry related variables
        HARBOR_URL = 'new-harbor2.duckdns.org'
        HARBOR_REPOSITORY = 'projects'
        HARBOR_CREDENTIALS = credentials('harbor')
        // Docker related variables
        DOCKER_IMAGE = 'java:latest' // Adjust according to your Docker image name
        //DOCKER_IMAGE_TAG = ':java:latest' // Adjust according to your Docker image tag
        DOCKER_CONTAINER_NAME = 'java'
        DOCKER_CONTAINER_PORT = '8080:8080'
    }
    
    // Define tools used in the pipeline
    tools {
        maven 'maven'
    }
    
    // Define stages of the pipeline
    stages {
        stage('Build the code') {
            steps {
                sh 'mvn clean package'
            }
            post {
                success {
                    slackSend color: '#36a64f', message: "Dependencies and build Success - ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
                }
                failure {
                    slackSend color: '#ff0000', message: "Dependencies and build failed! - ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('Sonar Analysis') {
            steps {
                // Running SonarQube Scanner after Maven build
                withSonarQubeEnv(credentialsId: 'sonarqube', installationName:'sonarqube') {
                    sh """
                    ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                    -Dsonar.sources=src \
                    -Dsonar.host.url=${SONAR_SERVER_URL} \
                    -Dsonar.login=${SONAR_LOGIN}
                    """
                }
            }
            post {
                success {
                    // Send success message to Slack channel if sonar-scan success
                    slackSend color: '#36A64F', message: "Sonar-scan success - ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
                }
                failure {
                    // Send failure message to Slack channel if sonar-scan fail
                    slackSend color: '#FF0000', message: "Sonar-scan failed! - ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
                }
            }
        }
        stage('archiveArtifacts') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', followSymlinks: false
            }
        }
        
        stage('Docker build & push to harbor registry') {
            steps {
                script {
                    // Use withCredentials to securely pass Harbor credentials
                    withCredentials([usernamePassword(credentialsId: 'harbor', usernameVariable: 'HARBOR_USERNAME', passwordVariable: 'HARBOR_PASSWORD')]) {
                    // Build Docker image
                    sh "docker build -t ${DOCKER_IMAGE} ."
                    // Login to Harbor registry
                    sh "docker login ${HARBOR_URL} -u ${HARBOR_USERNAME} -p ${HARBOR_PASSWORD}"
                    // Tag Docker image
                    sh "docker tag ${DOCKER_IMAGE} ${HARBOR_URL}/${HARBOR_REPOSITORY}/${DOCKER_IMAGE}"
                    // Push Docker image to Harbor registry
                    sh "docker push ${HARBOR_URL}/${HARBOR_REPOSITORY}/${DOCKER_IMAGE}"
                    }
                }
            }      
            
        stage('Deploy the container') {
            steps {
                // Deploy the Docker container
                sh "docker run -d --name ${DOCKER_CONTAINER_NAME} -p ${DOCKER_CONTAINER_PORT} ${DOCKER_IMAGE}"
            }
            post {
                success {
                    // Send success message to Slack channel
                    slackSend color: '#36A64F', message: "Docker container created and run successfully - ${HARBOR_URL}/${HARBOR_REPOSITORY}/${DOCKER_IMAGE}"
                }
                failure {
                    // Send failure message to Slack channel
                    slackSend color: '#FF0000', message: "Docker container failed to create - ${HARBOR_URL}/${HARBOR_REPOSITORY}/${DOCKER_IMAGE}"
                }
            }
        }
    }
}
