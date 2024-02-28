def buildId = env.BUILD_ID

pipeline {
    agent any
    
    tools {
        maven 'Default'
    }
    
    stages {
        stage('Cloning Git') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/ayyoub-zbd/devops-project-2024']])
            }
        }
        
        stage('Update version in pom.xml') {
            steps {
                bat "mvn versions:set -DnewVersion=${buildId}"
            }
        }
        
        stage('Building Docker image') {
            steps {
                script {
                    projectImage = docker.build("ayyoubzbd/devops-project:${buildId}", "--build-arg VARIABLE=${buildId} .")
                }
            }
        }
        
        stage('Publish Docker image on DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-credentials') {
                        projectImage.push()
                    }
                }
            }
        }
        
        stage('Deploy to development') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    bat 'kubectl apply -f kubernetes/development.yaml'
                }
            }
        }
        
        stage('Validate development') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    script {                        
                        def response = httpRequest 'http://127.0.0.1:8001/api/v1/namespaces/default/services/devops-project-development-service' // Require "HTTP Request" plugin

                        if (response.status == 200) {
                            echo("Test passed!")
                        } else {
                            error "Test failed..."
                        }
                    }
                }
            }
        }
        
        stage('Deploy to production') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    bat 'kubectl apply -f kubernetes/production.yaml'
                }
            }
        }
    }
}