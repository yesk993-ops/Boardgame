pipeline {
    agent any
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/yesk993-ops/Boardgame.git'
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
        stage('Trivy File system scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                      -Dsonar.projectName=BoardGame \
                      -Dsonar.projectKey=BoardGame \
                      -Dsonar.java.binaries=.
                    '''
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
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        stage('Publish Artifacts to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-cred',
                                                 usernameVariable: 'NEXUS_USER',
                                                 passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                        mvn deploy -U \
                          -Dnexus.username=$NEXUS_USER \
                          -Dnexus.password=$NEXUS_PASS
                    """
                }
            }
        }
        stage('Build and Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-credentials', toolName: 'docker') {
                        sh "docker build -t mydocker3692/boardgame:latest ."
                    }
                }
            }
        }
        stage('Docker Image Scan') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-credentials', toolName: 'docker') {
                        sh "trivy image --format table -o trivy-image-report.html mydocker3692/boardgame:latest"
                    }
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-credentials', toolName: 'docker') {
                        sh "docker push mydocker3692/boardgame:latest"
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f deployment-service.yaml -n webapps"
                sh "kubectl get pods -n webapps"
            }
        }
    }
}
