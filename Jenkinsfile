pipeline {
    agent any
    environment {
        DOCKERHUB_CREDS = credentials('dockerhub')
        COMMIT_ID = "${env.GIT_COMMIT.take(7)}"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    // Lấy tên service từ thư mục hoặc biến
                    def serviceName = env.JOB_NAME.split('/')[1] // Giả sử job tên theo service
                    sh """
                    docker build -t $DOCKERHUB_CREDS_USR/${serviceName}:${COMMIT_ID} .
                    """
                }
            }
        }
        stage('Push to Docker Hub') {
            steps {
                sh """
                echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin
                docker push $DOCKERHUB_CREDS_USR/${serviceName}:${COMMIT_ID}
                """
            }
        }
    }
}
