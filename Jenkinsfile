pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        DOCKER_IMAGE_NAME = "spring-petclinic"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: "${BRANCH_NAME}", url: 'https://github.com/spring-petclinic/spring-petclinic-microservices.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.build("${DOCKER_IMAGE_NAME}:${commitId}")
                }
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
                        def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        docker.image("${DOCKER_IMAGE_NAME}:${commitId}").push()
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                    kubectl set image deployment/${SERVICE_NAME} ${SERVICE_NAME}=docker.io/<your-dockerhub-username>/${DOCKER_IMAGE_NAME}:${commitId} --kubeconfig /path/to/kubeconfig
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
