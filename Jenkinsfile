pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        MAVEN_OPTS = '-Xmx1024m'
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Git Checkout") {
            steps {
                git url: 'https://github.com/Arunasri-0096/devops-zomato.git', branch: 'master'
            }
        }

       stage("Install Dependencies") {
    steps {
        sh 'npm install'
    }
}

stage("Build App") {
    steps {
        sh 'npm run build || true'
    }
}
        stage('Test') {
    when {
        expression { return false }
    }
    steps {
        echo "Skipping test stage"
    }
}

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato
                    """
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage("Install NPM Dependencies") {
            steps {
                sh 'npm install || true'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', 
                odcInstallation: 'DP-Check'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("Trivy File Scan") {
            steps {
                sh 'trivy fs . > trivy.txt'
            }
        }

        stage("Docker Build") {
            steps {
                sh 'docker build -t zomato .'
            }
        }

        stage("Tag & Push Docker Image") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh 'docker tag zomato kastrov/zomato:latest'
                        sh 'docker push kastrov/zomato:latest'
                    }
                }
            }
        }

        stage('Docker Scout Scan') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker-scout quickview kastrov/zomato:latest'
                        sh 'docker-scout cves kastrov/zomato:latest'
                        sh 'docker-scout recommendations kastrov/zomato:latest'
                    }
                }
            }
        }

        stage("Deploy Container") {
            steps {
                sh '''
                docker rm -f zomato || true
                docker run -d --name zomato -p 3000:3000 kastrov/zomato:latest
                '''
            }
        }
    }

    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: """
                <html>
                <body>
                    <h2>Build Report</h2>
                    <p><b>Project:</b> ${env.JOB_NAME}</p>
                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                    <p><b>URL:</b> ${env.BUILD_URL}</p>
                </body>
                </html>
                """,
                to: 'srividyapsn2014@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy.txt'
        }

        success {
            echo "✅ Pipeline Success"
        }

        failure {
            echo "❌ Pipeline Failed"
        }
    }
}
