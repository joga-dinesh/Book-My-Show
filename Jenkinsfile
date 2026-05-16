pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {

                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'git-cred', url: 'https://github.com/joga-dinesh/Book-My-Show.git']])

                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {

                withSonarQubeEnv('sonar-server') {

                    sh """
                    \$SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BMS \
                    -Dsonar.projectKey=BMS
                    """
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
            steps {

                sh '''
                cd bookmyshow-app

                ls -la

                if [ -f package.json ]; then
                    rm -rf node_modules package-lock.json
                    npm install
                else
                    echo "package.json not found!"
                    exit 1
                fi
                '''
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {

                script {

                    withDockerRegistry(
                        credentialsId: 'docker-cred',
                        toolName: 'docker'
                    ) {

                        sh '''
                        echo "Building Docker image..."

                        docker build --no-cache \
                        -t jogad/bms:latest \
                        -f bookmyshow-app/Dockerfile \
                        bookmyshow-app

                        echo "Pushing Docker image..."

                        docker push jogad/bms:latest
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image jogad/bms:latest > trivyimage.txt'
            }
        }

        stage('Deploy Container') {
            steps {

                sh '''
                echo "Stopping old container..."

                docker stop bms || true
                docker rm bms || true

                echo "Running new container..."

                docker run -d \
                --restart=always \
                --name bms \
                -p 3000:3000 \
                subhashrokkala/bms:latest

                echo "Checking running containers..."

                docker ps -a

                echo "Fetching container logs..."

                sleep 10

                docker logs bms
                '''
            }
        }
    }

    post {

        always {

            emailext(
                attachLog: true,

                subject: "${currentBuild.result}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",

                body: """
                <h2>Jenkins Build Report</h2>

                <p><b>Project:</b> ${env.JOB_NAME}</p>

                <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>

                <p><b>Status:</b> ${currentBuild.result}</p>

                <p><b>Build URL:</b>
                <a href="${env.BUILD_URL}">
                ${env.BUILD_URL}
                </a></p>
                """,

                to: 'jogadinesh8@gmail.com',

                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
