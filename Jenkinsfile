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

        stage('Install Prometheus and Grafana') {
            steps {
                script {
                    // Add Prometheus & Grafana repositories
                    bat 'helm repo add prometheus-community https://prometheus-community.github.io/helm-charts'
                    bat 'helm repo add grafana https://grafana.github.io/helm-charts'
                
                    bat 'hrlm repo update'

                    // Install Prometheus & Grafana
                    bat 'helm install prometheus prometheus-community/prometheus'
                    bat 'helm install grafana grafana/grafana'

                    // Wait for Prometheus & Grafana pods to be ready
                    bat 'kubectl wait --for=condition=Ready pod -l app=prometheus-server --timeout=300s'
                    bat 'kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=grafana --timeout=300s'
                
                    // Expose Grafana service for external access
                    bat 'kubectl expose service grafana --type=LoadBalancer --name=grafana-ext'
                }
            }
        }
    }
}