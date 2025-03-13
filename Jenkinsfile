pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool('sonar-scanner')
    }

    stages { 
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/akylgit/Mission.git'
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
        
        stage('Trivy Scan File System') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=Mission \
                        -Dsonar.projectName=Mission \
                        -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        
        stage('Deploy Artifacts To Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsFilePath: '/var/lib/jenkins/.m2/settings.xml',
                    maven: 'maven3'
                ) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t akyldocker25/mission:latest ."
                    }
                }
            }
        }
        
        stage('Trivy Scan Image') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html akyldocker25/mission:latest"
            }
        }
        
        stage('Publish Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push akyldocker25/mission:latest"
                    }    
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-token', namespace: 'webapps', serverUrl: 'https://BD2E008A180E721142C52C8D7F2AE818.gr7.ap-south-1.eks.amazonaws.com') {
                    sh "kubectl apply -f ds.yml -n webapps"
                    sh "kubectl rollout status deployment mission-deployment -n webapps"
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                withKubeConfig(credentialsId: 'k8-token', namespace: 'webapps', serverUrl: 'https://BD2E008A180E721142C52C8D7F2AE818.gr7.ap-south-1.eks.amazonaws.com') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }
    
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${env.BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'johns.ak87@gmail.com', 
                    from: 'jenkins@example.com', 
                    replyTo: 'jenkins@example.com', 
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-*.html'
                )
            }
        }
        
        success {
            archiveArtifacts artifacts: 'trivy-*.html', allowEmptyArchive: true
        }
    }
}
