pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    // Lấy tên nhánh từ env.BRANCH_NAME hoặc git
                    def branchToCheckout = env.BRANCH_NAME ?: sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                    echo "Branch hiện tại: ${branchToCheckout}"

                    // Sử dụng biến cục bộ để kiểm soát pipeline
                    def shouldRunPipeline = true
                    if (branchToCheckout == 'main') {
                        echo "Đây là nhánh 'main'. Pipeline sẽ dừng lại và vẫn được đánh dấu SUCCESS."
                        shouldRunPipeline = false
                    }

                    // Lưu giá trị vào biến môi trường để sử dụng trong các stage sau
                    env.SHOULD_RUN_PIPELINE = shouldRunPipeline.toString()

                    if (!shouldRunPipeline) {
                        return
                    }

                    // Checkout nhánh động
                    checkout([$class: 'GitSCM', 
                              branches: [[name: "*/${branchToCheckout}"]], 
                              userRemoteConfigs: [[url: 'https://github.com/pmhanh/spring-petclinic-microservices.git']]])
                }
            }
        }

        stage('Build JAR') {
            when {
                expression { return env.SHOULD_RUN_PIPELINE == 'true' }
            }
            steps {
                script {
                    // Kiểm tra bổ sung để đảm bảo không chạy trên nhánh main
                    if (env.SHOULD_RUN_PIPELINE != 'true') {
                        echo "Bỏ qua stage Build JAR vì nhánh là main."
                        return
                    }
                    sh './mvnw clean package -DskipTests'
                }
            }
        }

        stage('Build and Push Docker Images') {
            when {
                expression { return env.SHOULD_RUN_PIPELINE == 'true' }
            }
            steps {
                script {
                    // Kiểm tra bổ sung để đảm bảo không chạy trên nhánh main
                    if (env.SHOULD_RUN_PIPELINE != 'true') {
                        echo "Bỏ qua stage Build and Push Docker Images vì nhánh là main."
                        return
                    }

                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    // Xử lý git diff an toàn
                    def changedFiles = sh(script: 'git diff --name-only HEAD^ HEAD 2>/dev/null || git ls-files', returnStdout: true).trim().split('\n')
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
                    } else {
                        echo "Building services with changes: ${servicesToBuild.collect { it.name }.join(', ')}"
                    }

                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                        for (def service in servicesToBuild) {
                            def imageName = "${DOCKER_USERNAME}/spring-petclinic-${service.name}:${commitId}"
                            def jarFile = sh(script: "ls ${service.dir}/target/*.jar | head -1", returnStdout: true).trim()

                            if (!jarFile) {
                                error "No JAR file found for ${service.name} in ${service.dir}/target/"
                            }

                            echo "Building and pushing Docker image for ${service.name} with tag ${commitId}..."
                            sh """
                                mkdir -p docker
                                cp ${jarFile} docker/${service.name}.jar
                                cd docker
                                docker build --build-arg ARTIFACT_NAME=${service.name} --build-arg EXPOSED_PORT=${service.port} -t ${imageName} .
                                docker push ${imageName}
                                rm ${service.name}.jar
                                cd ..
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up workspace..."
            cleanWs()
        }
        success {
            script {
                if (env.SHOULD_RUN_PIPELINE == 'false') {
                    echo "✅ Pipeline skipped successfully (main branch detected)."
                } else {
                    echo "✅ Pipeline completed successfully."
                }
            }
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}