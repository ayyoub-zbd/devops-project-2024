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
                        def serviceURL = kubectl.run "get service devops-project-development-service -o jsonpath={.status.loadBalancer.ingress[0].ip}"
                        print("Service URL: " + serviceURL)
                        
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

        /*stage('Install Prometheus') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    script {
                        def isPrometheusInstalled = bat(script: 'helm list -f \'\bprometheus\b\'', returnStdout: true).contains('prometheus')

                        if (!isPrometheusInstalled) {
                            // Add Prometheus repository
                            bat 'helm repo add prometheus-community https://prometheus-community.github.io/helm-charts'
                            bat 'helm repo update'

                            // Install Prometheus
                            bat 'helm install prometheus prometheus-community/prometheus'
                        } else {
                            echo "Prometheus is already installed. Skipping installation."
                        }

                        // Wait for Prometheus pod to be ready
                        bat 'kubectl wait --for=condition=available deployment/prometheus-server --timeout=300s'
                    }
                }
            }
        }

        stage('Install Grafana') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    script {
                        def isGrafanaInstalled = bat(script: 'helm list -f \'\bgrafana\b\'', returnStdout: true).contains('grafana')

                        if (!isGrafanaInstalled) {
                            // Add Grafana repository
                            bat 'helm repo add grafana https://grafana.github.io/helm-charts'
                            bat 'helm repo update'

                            // Install Grafana
                            bat 'helm install grafana grafana/grafana'
                        } else {
                            echo "Grafana is already installed. Skipping installation."
                        }

                        // Wait for Grafana pod to be ready
                        bat 'kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=grafana --timeout=300s'
                    
                        // Expose Grafana service for external access
                        bat 'kubectl expose service grafana --type=LoadBalancer --name=grafana-ext'
                    
                        // Wait for Grafana service to have an external IP
                        bat 'kubectl wait --for=condition=available --timeout=300s -n default deployment/grafana'
                        
                        def grafanaIP = bat(returnStdout: true, script: 'kubectl get svc grafana-ext -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"').trim()
                        print("Grafana IP: " + grafanaIP)
                    }
                }
            }
        }*/
    }
}