pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME   = "mydocker3692/boardgame"
        GITOPS_PATH  = "k8s-manifests"
    }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/yesk993-ops/Boardgame.git'
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

        stage('Trivy File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    $SCANNER_HOME/bin/sonar-scanner \
                      -Dsonar.projectName=BoardGame \
                      -Dsonar.projectKey=BoardGame \
                      -Dsonar.java.binaries=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }

        stage('Build') {
            steps {
                sh "mvn package"
            }
        }

        stage('Publish Artifacts to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                    mvn deploy -U \
                      -Dnexus.username=$NEXUS_USER \
                      -Dnexus.password=$NEXUS_PASS
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-credentials', toolName: 'docker') {
                        sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html ${IMAGE_NAME}:${BUILD_NUMBER}"
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-credentials', toolName: 'docker') {
                        sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Update GitOps Manifest (Trigger Argo CD)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'git-cred',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh """
                    git clone https://github.com/yesk993-ops/Boardgame.git gitops
                    cd gitops/${GITOPS_PATH}

                    sed -i 's|image: .*|image: ${IMAGE_NAME}:${BUILD_NUMBER}|' deployment.yaml

                    git config user.email "jenkins@local"
                    git config user.name "jenkins"

                    git commit -am "Update image to ${BUILD_NUMBER}"
                    git push https://${GIT_USER}:${GIT_PASS}@github.com/yesk993-ops/Boardgame.git main
                    """
                }
            }
        }
    }
}
