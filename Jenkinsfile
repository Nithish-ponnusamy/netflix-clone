pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/kishgi/netflix-clone.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps { sh "npm install" }
        }
        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') {
            steps { sh "trivy fs . > trivyfs.txt" }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build --build-arg TMDB_V3_API_KEY=bc373fb16fe8de9c49dd747e899dbaee -t netflix ."
                        sh "docker tag netflix kishgi/netflix:latest"
                        sh "docker push kishgi/netflix:latest"
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps { sh "trivy image kishgi/netflix:latest > trivyimage.txt" }
        }
        stage('Deploy to Container') {
            steps {
                sh "docker ps -q --filter 'publish=8081' | xargs -r docker stop"
                sh "docker ps -aq --filter 'publish=8081' | xargs -r docker rm"
                sh "docker run -d -p 8081:80 --name netflix-app kishgi/netflix:latest"
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'kishgi1234@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}