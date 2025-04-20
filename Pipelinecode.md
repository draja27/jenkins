
pipeline {
    agent any
   
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner' // Fixed typo
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/yourgitid/Boardgame.git'
            }
        }
       
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
       
        stage('SonarQube Analysis') { // Fixed typo in "Analysis"
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=.
                    '''
                }
            }
        }
       
        stage('TRIVY Scan') {
            steps {
                sh "/usr/local/bin/trivy fs --security-checks vuln /var/lib/jenkins/workspace/cicd"
            }
        }

        stage('CODE BUILD') {
            steps {
                sh "mvn clean install"
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: '727fa460-e370-4d59', toolName: 'docker-latest') {
                        // Build the Docker image first
                        sh "docker build -t cicd ."
                       
                        // Tag the Docker image with the build ID
                        sh "docker tag cicd draja27/test:$BUILD_ID"
                       
                        // Push the tagged Docker image to Docker Hub
                        sh "docker push draja27/test:$BUILD_ID"
                    }
                }
            }
        }

        stage('Kubernetes Deploy') { // Fixed stage name, 'CODE BUILD' appeared twice
            steps {
                script {
                    kubeconfig(credentialsId: '2632adda-7241', serverUrl: 'https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy') {
                        sh "kubectl apply -f deployment-service.yaml"
                    }
                }
            }
        }
    }
}