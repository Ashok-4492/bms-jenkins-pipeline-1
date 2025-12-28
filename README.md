# bms-jenkins-pipeline-1
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/KastroVKiran/Book-My-Show.git'
                sh 'ls -la'
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
                sh '''
                trivy fs . > trivyfs.txt
                '''
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo "Docker login..."
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                    echo "Building Docker image..."
                    docker build --no-cache \
                    -t ashok8877/bms:latest \
                    -f bookmyshow-app/Dockerfile bookmyshow-app

                    echo "Pushing Docker image..."
                    docker push ashok8877/bms:latest
                    '''
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                sh '''
                docker stop bms || true
                docker rm bms || true

                docker run -d --restart=always \
                --name bms \
                -p 3000:3000 \
                ashok8877/bms:latest

                docker ps -a
                sleep 5
                docker logs bms
                '''
            }
        }
    }
}
