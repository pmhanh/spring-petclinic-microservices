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
                    // Checkout với fetch tags
                    checkout([$class: 'GitSCM',
                              branches: [[name: branchToCheckout]],
                              userRemoteConfigs: [[url: 'https://github.com/pmhanh/spring-petclinic-microservices.git']],
                              extensions: [[$class:lasting 'CloneOption', noTags: false]]])
                    // Lấy tag nếu có
                    def gitTag = sh(script: 'git describe --tags --exact-match 2>/dev/null || true', returnStdout: true).trim()
                    env.TAG_NAME = gitTag
                    echo "Checking out branch: ${branchToCheckout}, Tag: ${gitTag ?: 'none'}"
                    if (!gitTag) {
                        echo "No git tag found, will use 'latest' as image tag."
                    }
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
                    // Dùng tag nếu có, không thì 'latest'
                    def imageTag = env.TAG_NAME ?: 'latest'
                    echo "Using image tag: ${imageTag}"
                    def changedFiles = sh(script: "git diff --name-only HEAD^ HEAD || true", returnStdout: true).trim().split('\n')
                    // Skip build nếu chỉ thay đổi file không liên quan
                    if (changedFiles.every { it ==~ /(README\.md|\.gitignore)/ }) {
                        echo "Only non-code files changed. Skipping build."
                        return
                    }
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
                    } ?: serviceConfigs // Build tất cả nếu không detect thay đổi
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        try {
                            sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                        } catch (Exception e) {
                            error "Docker login failed: ${e.message}"
                        }
                        for (def service in servicesToBuild) {
                            def serviceDir = service.dir
                            def serviceName = service.name
                            def exposedPort = service.port
                            def imageName = "${DOCKER_USERNAME}/spring-petclinic-${serviceName}:${imageTag}"
                            def jarFile = sh(script: "ls ${serviceDir}/target/*.jar | head -1", returnStdout: true).trim()
                            if (!jarFile) {
                                error "No JAR file found for ${serviceName} in ${serviceDir}/target/"
                            }
                            echo "Building Docker image for ${serviceName} with tag ${imageTag}..."
                            dir('docker') {
                                sh """
                                    cp ${jarFile} ${serviceName}.jar
                                    docker build --build-arg ARTIFACT_NAME=${serviceName} --build-arg EXPOSED_PORT=${exposedPort} -t ${imageName} .
                                    docker push ${imageName}
                                """
                                deleteDir() // Cleanup
                            }
                        }
                    }
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
                    if (response.status != 201) {
                        error "Failed to update GitHub status: ${response.status}"
                    }
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
                    if (response.status != 201) {
                        error "Failed to update GitHub status: ${response.status}"
                    }
                }
            }
        }
        always {
            echo "Pipeline finished."
            withCredentials([string(credentialsId: 'github-token1', variable: 'GITHUB_TOKEN')]) {
                echo "Token exists: ${GITHUB_TOKEN ? 'yes' : 'no'}"
            }
        }
    }
}