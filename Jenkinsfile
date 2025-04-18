pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
    }
    stages {
        stage('Checkout Code') {
            steps {
                script {
                    def branchToCheckout = env.BRANCH_NAME ?: 'main'
                    git branch: branchToCheckout, url: 'https://github.com/pmhanh/spring-petclinic-microservices.git'
                    sh 'git fetch --tags'
                    def gitTag = sh(script: 'git describe --tags --exact-match 2>/dev/null || true', returnStdout: true).trim()
                    env.TAG_NAME = gitTag
                    echo "Checking out branch: ${branchToCheckout}, Tag: ${gitTag}"
                }
            }
        }
        stage('Build JAR') {
            steps {
                script {
                    echo "Building JAR for all services..."
                    sh './mvnw clean package -DskipTests'
                }
            }
        }
        stage('Build and Push Docker Images') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    def imageTag = env.TAG_NAME ? env.TAG_NAME : commitId 
                    def changedFiles = sh(script: "git diff --name-only HEAD^ HEAD || true", returnStdout: true).trim().split('\n')
                    def serviceConfigs = [
                        [name: 'admin-server', dir: 'spring-petclinic-admin-server', port: 9090],
                        [name: 'api-gateway', dir: 'spring-petclinic-api-gateway', port: 8080],
                        [name: 'config-server', dir: 'spring-petclinic-config-server', port: 8888],
                        [name: 'customers-service', dir: 'spring-petclinic-customers-service', port: 8081],
                        [name: 'discovery-server', dir: 'spring-petclinic-discovery-server', port: 8761],
                        [name: 'genai-service', dir: 'spring-petclinic-genai-service', port: 8084],
                        [name: 'vets-service', dir: 'spring-petclinic-vets-service', port: 8083],
                        [name: 'visits-service', dir: 'spring-petclinic-visits-service', port: 8082]
                    ]

                    def servicesToBuild = serviceConfigs.findAll { config ->
                        changedFiles.any { it.startsWith(config.dir + '/') }
                    }
                    if (servicesToBuild.isEmpty()) {
                        echo "No services changed. Building all services as fallback."
                        servicesToBuild = serviceConfigs
                    }

                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                        
                        for (def service in servicesToBuild) {
                            def serviceDir = service.dir
                            def serviceName = service.name
                            def exposedPort = service.port
                            def imageName = "${DOCKER_USERNAME}/spring-petclinic-${serviceName}:${imageTag}"
                            def jarFile = sh(script: "ls ${serviceDir}/target/*.jar | head -1", returnStdout: true).trim()

                            if (!jarFile) {
                                error "No JAR file found for ${serviceName} in ${serviceDir}/target/"
                            }

                            sh """
                                mkdir -p docker
                                cp ${jarFile} docker/${serviceName}.jar
                                cd docker
                                docker build --build-arg ARTIFACT_NAME=${serviceName} --build-arg EXPOSED_PORT=${exposedPort} -t ${imageName} .
                                docker push ${imageName}
                                rm ${serviceName}.jar
                                cd ..
                            """

                            if (env.BRANCH_NAME == 'main' && !env.TAG_NAME) {
                                sh """
                                    docker tag ${imageName} ${DOCKER_USERNAME}/spring-petclinic-${serviceName}:latest
                                    docker push ${DOCKER_USERNAME}/spring-petclinic-${serviceName}:latest
                                """
                            }
                        }
                    }
                }
            }
        }
        stage('Deploy to Dev') {
        steps {
            script {
                // Đăng nhập vào ArgoCD
                withCredentials([usernamePassword(credentialsId: 'argocd-credentials', usernameVariable: 'ARGOCD_USER', passwordVariable: 'ARGOCD_PASS')]) {
                    sh 'argocd login localhost:8085 --username $ARGOCD_USER --password $ARGOCD_PASS --insecure'
                }
                // Đồng bộ ứng dụng
                sh 'argocd app sync petclinic-dev'
            }
        }
        }
        stage('Deploy to Staging') {
            when {
                expression { env.TAG_NAME }
            }
            steps {
                script {
                    def tag = env.TAG_NAME
                    sh """
                        argocd app set petclinic-staging \\
                            --parameter adminServer.tag=${tag} \\
                            --parameter apiGateway.tag=${tag} \\
                            --parameter configServer.tag=${tag} \\
                            --parameter customersService.tag=${tag} \\
                            --parameter discoveryServer.tag=${tag} \\
                            --parameter genaiService.tag=${tag} \\
                            --parameter vetsService.tag=${tag} \\
                            --parameter visitsService.tag=${tag}
                        argocd app sync petclinic-staging
                    """
                }
            }
        }
    }
    post {
        success {
            script {
                def commitId = env.GIT_COMMIT
                echo "Sending success status to GitHub for commit: ${commitId}"
                withCredentials([string(credentialsId: 'github-token1', variable: 'GITHUB_TOKEN')]) {
                    def response = httpRequest(
                        url: "https://api.github.com/repos/pmhanh/spring-petclinic-microservices/statuses/${commitId}",
                        httpMode: 'POST',
                        contentType: 'APPLICATION_JSON',
                        requestBody: """{
                            "state": "success",
                            "description": "Build passed",
                            "context": "ci/jenkins-pipeline",
                            "target_url": "${env.BUILD_URL}"
                        }""",
                        customHeaders: [[name: 'Authorization', value: "token ${GITHUB_TOKEN}"]]
                    )
                    echo "GitHub Response: ${response.status}"
                }
            }
        }
        failure {
            script {
                def commitId = env.GIT_COMMIT
                echo "Sending failure status to GitHub for commit: ${commitId}"
                withCredentials([string(credentialsId: 'github-token1', variable: 'GITHUB_TOKEN')]) {
                    def response = httpRequest(
                        url: "https://api.github.com/repos/pmhanh/spring-petclinic-microservices/statuses/${commitId}",
                        httpMode: 'POST',
                        contentType: 'APPLICATION_JSON',
                        requestBody: """{
                            "state": "failure",
                            "description": "Build failed",
                            "context": "ci/jenkins-pipeline",
                            "target_url": "${env.BUILD_URL}"
                        }""",
                        customHeaders: [[name: 'Authorization', value: "token ${GITHUB_TOKEN}"]]
                    )
                    echo "GitHub Response: ${response.status}"
                }
            }
        }
        always {
            echo "Pipeline finished."
        }
    }
}