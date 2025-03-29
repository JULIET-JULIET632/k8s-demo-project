```
pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git credentialsId: 'git-cred', url: 'https://github.com/etechsconsulting/java-maven-app.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        stage('File System Scan') {
            steps {
                sh "trivy fs ."
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=java-maven-app -Dsonar.projectKey=java-maven-app \
                           -Dsonar.java.banaries=. '''     
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Build Artifact') {
            steps {
                sh "mvn package"
            }
        }
        stage('Publish To Nexus') {
            steps {
               withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                        sh "mvn deploy"
               }
            }
        }
        stage('Build & Tag Docker Image') {
            steps {
                script {
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t juliet237/java-maven-app:1.0 ."
                   }    
                }     
            } 
        }
        stage('Docker Scan Image') {
            steps {
                sh "trivy image ebonje/java-maven-app:1.0"
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                    sh "docker push ebonje/java-maven-app:1.0"
                    }
                } 
            }
        }
        stage('Deploy To Kubernetes') {
            steps {
               withKubeConfig(caCertificate: '', clusterName: ' kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapp', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.15.201:6443') {
                        sh "kubectl apply -f deployment.yaml"
               } 
            }
        }
        
        stage('Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: ' kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapp', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.15.201:6443') {
                         sh "kubectl get pods -n webapp"
                         sh "kubectl get svc -n webapp"
               }
          }
    }
}

