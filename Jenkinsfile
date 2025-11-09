pipeline {
    agent any
    
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'DEPLOY_ENV', choices: ['staging', 'production'], description: 'Deployment environment')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip test stages')
    }
    
    environment {
        APP_NAME = 'spring-petclinic'
        GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse --short HEAD 2>/dev/null || echo "no-git"').trim()
        BUILD_VERSION = "${BUILD_NUMBER}-${GIT_COMMIT}"
        DOCKER_IMAGE = "spring-petclinic:${BUILD_VERSION}"
        // 🔑 UNIQUE project name per build to prevent conflicts
        COMPOSE_PROJECT_NAME = "petclinic-${BUILD_NUMBER}-${GIT_COMMIT}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scmGit(
                    branches: [[name: "*/${params.BRANCH}"]],
                    userRemoteConfigs: [[url: 'https://github.com/mohamedalibenchiekh/spring-petclinic.git']]
                )
            }
        }
        
        stage('Pre-Test Cleanup') {
            when {
                not { expression { return params.SKIP_TESTS } }
            }
            steps {
                script {
                    echo "🧹 Cleaning up Docker resources from previous builds..."
                    // Aggressive cleanup of ANY remnants
                    sh '''
                        # Remove containers
                        docker ps -a --filter "name=petclinic-*" -q | xargs -r docker rm -f
                        
                        # Remove networks
                        docker network ls --filter "name=petclinic-*" -q | xargs -r docker network rm
                        
                        # Remove volumes (optional)
                        docker volume ls --filter "name=petclinic-*" -q | xargs -r docker volume rm
                    '''
                }
            }
        }
        
        stage('Build') {
            steps {
                echo "🔨 Building version: ${BUILD_VERSION}"
                sh '''
                    ./mvnw clean compile -DskipTests
                    ./mvnw package -DskipTests
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }
        
        stage('Parallel Testing') {
            when {
                not { expression { return params.SKIP_TESTS } }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        echo '🧪 Running Unit Tests (H2 only)...'
                        // Skip Docker Compose for unit tests
                        sh './mvnw test -Dspring.docker.compose.skip.in-tests=true'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        echo '🔬 Running Integration Tests (with Docker Compose)...'
                        script {
                            try {
                                // Use unique project name for isolation
                                sh """
                                    export COMPOSE_PROJECT_NAME=${COMPOSE_PROJECT_NAME}
                                    ./mvnw verify -Dspring.docker.compose.project.name=${COMPOSE_PROJECT_NAME}
                                """
                            } catch (Exception e) {
                                echo "Integration tests failed: ${e.getMessage()}"
                                currentBuild.result = 'UNSTABLE'
                            }
                        }
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'target/failsafe-reports/*.xml'
                            
                            // Cleanup integration test containers
                            sh '''
                                export COMPOSE_PROJECT_NAME=${COMPOSE_PROJECT_NAME}
                                docker-compose -f docker-compose.yml -p ${COMPOSE_PROJECT_NAME} down --volumes --remove-orphans || true
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Docker Image Build') {
            steps {
                script {
                    writeFile file: 'Dockerfile', text: """
                    FROM openjdk:17-jdk-slim
                    WORKDIR /app
                    COPY target/spring-petclinic-*.jar app.jar
                    EXPOSE 8080
                    ENTRYPOINT ["java", "-jar", "app.jar"]
                    """
                    sh "docker build -t ${DOCKER_IMAGE} ."
                    sh "docker save ${DOCKER_IMAGE} -o target/${APP_NAME}-${BUILD_VERSION}.tar"
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: "target/${APP_NAME}-${BUILD_VERSION}.tar", fingerprint: true
                }
            }
        }
        
        stage('Deployment Simulation') {
            when {
                allOf {
                    expression { return params.DEPLOY_ENV == 'staging' }
                    not { expression { return env.CHANGE_ID != null } }
                }
            }
            steps {
                script {
                    echo "🚀 Deploying to ${params.DEPLOY_ENV} environment..."
                    
                    // Unique container name
                    def containerName = "${APP_NAME}-${BUILD_VERSION}"
                    
                    sh """
                        docker network inspect petclinic-network >/dev/null 2>&1 || \
                        docker network create petclinic-network
                    """
                    
                    sh """
                        docker stop ${containerName} 2>/dev/null || true
                        docker rm ${containerName} 2>/dev/null || true
                    """
                    
                    sh """
                        docker run -d \
                          --name ${containerName} \
                          --network petclinic-network \
                          -p 8080:8080 \
                          ${DOCKER_IMAGE}
                    """
                    
                    sh """
                        mkdir -p /var/jenkins_home/deployments/${params.DEPLOY_ENV}
                        cp target/${APP_NAME}-${BUILD_VERSION}.tar /var/jenkins_home/deployments/${params.DEPLOY_ENV}/
                    """
                    
                    echo "✅ Application deployed successfully!"
                    echo "🌐 Access it at: http://localhost:8080 (Container: ${containerName})"
                    
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "🧹 Final cleanup..."
                // Ensure no resources are left behind
                sh """
                    docker ps -a --filter "name=petclinic-*" -q | xargs -r docker rm -f
                    docker network ls --filter "name=petclinic-*" -q | xargs -r docker network rm
                    docker volume ls --filter "name=petclinic-*" -q | xargs -r docker volume rm
                """
            }
            cleanWs()
        }
        
        success {
            emailext (
                subject: "✅ Build Success: ${env.JOB_NAME} #${BUILD_NUMBER}",
                body: """
                    <h2>Build Successful! 🎉</h2>
                    <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build:</strong> #${BUILD_NUMBER}</p>
                    <p><strong>Version:</strong> ${BUILD_VERSION}</p>
                    <p><strong>Branch:</strong> ${params.BRANCH}</p>
                    <p><strong>Duration:</strong> ${currentBuild.durationString}</p>
                    <p><a href="${BUILD_URL}">View Build</a></p>
                """,
                to: 'mohamed.ali.bencheikh@craftschoolship.com',
                mimeType: 'text/html'
            )
        }
        
        failure {
            emailext (
                subject: "❌ Build Failed: ${env.JOB_NAME} #${BUILD_NUMBER}",
                body: """
                    <h2>Build Failed! 💥</h2>
                    <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build:</strong> #${BUILD_NUMBER}</p>
                    <p><strong>Version:</strong> ${BUILD_VERSION}</p>
                    <p><strong>Failed Stage:</strong> ${env.STAGE_NAME}</p>
                    <p><a href="${BUILD_URL}console">View Console</a></p>
                """,
                to: 'mohamed.ali.bencheikh@craftschoolship.com',
                mimeType: 'text/html'
            )
        }
        
        unstable {
            emailext (
                subject: "⚠️ Build Unstable: ${env.JOB_NAME} #${BUILD_NUMBER}",
                body: """
                    <h2>Build Unstable - Integration Tests Failed 🟡</h2>
                    <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build:</strong> #${BUILD_NUMBER}</p>
                    <p><strong>Version:</strong> ${BUILD_VERSION}</p>
                    <p><a href="${BUILD_URL}">View Build</a></p>
                """,
                to: 'mohamed.ali.bencheikh@craftschoolship.com',
                mimeType: 'text/html'
            )
        }
    }
}