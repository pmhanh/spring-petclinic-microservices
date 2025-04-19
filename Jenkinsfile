pipeline {
    agent any
    triggers {
    pollSCM('')
    }
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
    }
    stages {
        stage('Checkout Code') {
            steps {
                script {
                    def branchToCheckout = env.BRANCH_NAME ?: 'main'
                    echo "Checking out branch: ${branchToCheckout}"
                    git branch: branchToCheckout, url: 'https://github.com/pmhanh/spring-petclinic-microservices.git'
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
                            def imageName = "${DOCKER_USERNAME}/spring-petclinic-${serviceName}:${commitId}"
                            def jarFile = sh(script: "ls ${serviceDir}/target/*.jar | head -1", returnStdout: true).trim()

                            if (!jarFile) {
                                error "No JAR file found for ${serviceName} in ${serviceDir}/target/"
                            }

                            echo "Building Docker image for ${serviceName}..."
                            sh """
                                mkdir -p docker
                                cp ${jarFile} docker/${serviceName}.jar
                                cd docker
                                docker build --build-arg ARTIFACT_NAME=${serviceName} --build-arg EXPOSED_PORT=${exposedPort} -t ${imageName} .
                                docker push ${imageName}
                                rm ${serviceName}.jar
                                cd ..
                            """

                            if (env.BRANCH_NAME == 'main') {
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
    }
    
//    post {
//        success {
//            script {
//                def commitId = env.GIT_COMMIT
//                echo "Sending success status to GitHub for commit: ${commitId}"
//                def response = httpRequest(
//                    url: "https://api.github.com/repos/pmhanh/spring-petclinic-microservices/statuses/${commitId}",
//                    httpMode: 'POST',
//                    contentType: 'APPLICATION_JSON',
//                    requestBody: """{
//                        "state": "success",
//                        "description": "Build passed",
//                        "context": "ci/jenkins-pipeline",
//                        "target_url": "${env.BUILD_URL}"
//                    }""",
//                    authentication: 'github-token1'
//                )
//                echo "GitHub Response: ${response.status}"
//            }
//        }


//        failure {
//            script {
//                def commitId = env.GIT_COMMIT
//                echo "Sending failure status to GitHub for commit: ${commitId}"
//                def response = httpRequest(
//                    url: "https://api.github.com/repos/pmhanh/spring-petclinic-microservices/statuses/${commitId}",
//                    httpMode: 'POST',
//                    contentType: 'APPLICATION_JSON',
//                    requestBody: """{
//                        "state": "failure",
//                        "description": "Build failed",
//                        "context": "ci/jenkins-pipeline",
//                        "target_url": "${env.BUILD_URL}"
//                    }""",
//                    authentication: 'github-token1'
//                )
//                echo "GitHub Response: ${response.status}"
//            }
//        }


//     //    always {
//     //        echo "Pipeline finished."
//     //    }
//     always {
//         withCredentials([string(credentialsId: 'github-token1', variable: 'GITHUB_TOKEN1')]) {
//             echo "Token exists: ${GITHUB_TOKEN1 ? 'yes' : 'no'}"
//         }
//     }
//    }
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
        withCredentials([string(credentialsId: 'github-token1', variable: 'GITHUB_TOKEN')]) {
            echo "Token exists: ${GITHUB_TOKEN ? 'yes' : 'no'}"
        }
    }
}
}