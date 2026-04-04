pipeline {
    agent any

    tools {
        nodejs 'node16.19'
    }

    environment {
        DOCKER_IMAGE = "kastrov/zomato:latest"
        TRIVY_REPORT = "trivy.txt"
        SONAR_SCANNER = tool 'sonar-scanner'
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

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh """
                    ${SONAR_SCANNER}/bin/sonar-scanner \
                    -Dsonar.projectKey=zomato \
                    -Dsonar.sources=. \
                    -Dsonar.login=$SONAR_AUTH_TOKEN
                    """
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Build App") {
            steps {
                sh 'npm run build || true'
            }
        }

        stage("Nexus Upload") {
            steps {
                script {
                    sh 'tar -czf app.tar.gz .'

                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: 'YOUR_NEXUS_IP:8081',
                        groupId: 'zomato',
                        version: '1.0',
                        repository: 'npm-repo',
                        credentialsId: 'nexus-creds',
                        artifacts: [
                            [artifactId: 'zomato-app',
                             file: 'app.tar.gz',
                             type: 'tar.gz']
                        ]
                    )
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                sh "trivy fs . > ${TRIVY_REPORT}"
            }
        }

        stage("Docker Build") {
            steps {
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage("Docker Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker push ${DOCKER_IMAGE}"
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
                subject: "Build ${currentBuild.result}",
                body: """
                <h2>Build Report</h2>
                <p><b>Project:</b> ${env.JOB_NAME}</p>
                <p><b>Build:</b> ${env.BUILD_NUMBER}</p>
                <p><b>URL:</b> ${env.BUILD_URL}</p>
                """,
                to: 'srividyapsn2014@gmail.com',
                attachmentsPattern: "${TRIVY_REPORT}"
        }
    }
}
