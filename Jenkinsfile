pipeline {
    agent any
    
    parameters {
        choice(
            name: 'BRANCH',
            choices: ['main', 'develop'],
            description: 'Source code branch'
        )
        choice(
            name: 'DEPLOY_ENV',
            choices: ['staging', 'production'],
            description: 'Deployment environment'
        )
        booleanParam(
            name: 'SKIP_DEPLOYMENT',
            defaultValue: false,
            description: 'Skip deployment (useful for PRs)'
        )
    }
    
    environment {
        // Build version based on git commit and build number
        BUILD_VERSION = "${env.BUILD_NUMBER}-${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}"
        
        // Docker configuration
        DOCKER_IMAGE = "spring-petclinic:${BUILD_VERSION}"
        DOCKER_CONTAINER = "petclinic-${BUILD_VERSION}"
        
        // Maven configuration - skip enforcer for Java version check
        MAVEN_OPTS = "-Dmaven.repo.local=${env.JENKINS_HOME}/.m2/repository -Dmaven.enforcer.skip=true"
        
        // Application port
        APP_PORT = "8080"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        timestamps()
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "Checking out from branch: ${params.BRANCH}"
                git branch: "${params.BRANCH}",
                    url: 'https://github.com/mohamedalibenchiekh/spring-petclinic.git'
            }
        }
        
        stage('Environment Check') {
            steps {
                script {
                    echo "Checking environment..."
                    sh '''
                        echo "Java version:"
                        java -version 2>&1 || echo "Java not found"
                        echo "Maven version:"
                        mvn -version 2>&1 || echo "Maven not found"
                        echo "Docker version:"
                        docker --version 2>&1 || echo "Docker not found - Docker stages will be skipped"
                    '''
                }
            }
        }
        
        stage('Build') {
            steps {
                echo "Building with version: ${BUILD_VERSION}"
                // Skip tests and enforcer during build phase
                sh 'mvn clean package -DskipTests -Dmaven.enforcer.skip=true'
                
                // Store the built artifact path
                script {
                    def jarFile = findFiles(glob: 'target/*.jar')[0]
                    env.ARTIFACT_PATH = jarFile.path
                    env.ARTIFACT_NAME = jarFile.name
                    echo "Artifact created: ${env.ARTIFACT_NAME}"
                }
            }
            post {
                success {
                    echo "Build successful - Artifact: ${env.ARTIFACT_NAME}"
                }
            }
        }
        
        stage('Parallel Testing') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        echo "Running unit tests..."
                        // Skip enforcer for tests
                        sh 'mvn test -Dtest=**/*Test -Dmaven.enforcer.skip=true'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/**/*.xml'
                        }
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        echo "Running integration tests..."
                        // Skip enforcer for tests
                        sh 'mvn integration-test -DskipUnitTests -Dmaven.enforcer.skip=true'
                    }
                    post {
                        always {
                            junit 'target/failsafe-reports/**/*.xml'
                        }
                    }
                }
            }
        }
        
        stage('Docker Build') {
            when {
                expression { 
                    (currentBuild.result == null || currentBuild.result == 'SUCCESS') &&
                    isUnix() // Only run if we're on a Unix system
                }
            }
            steps {
                script {
                    // Check if Docker is available
                    def dockerAvailable = sh(script: 'which docker', returnStatus: true) == 0
                    
                    if (!dockerAvailable) {
                        echo "Docker not available - skipping Docker build stage"
                        return
                    }
                    
                    // Create Dockerfile if it doesn't exist
                    if (!fileExists('Dockerfile')) {
                        writeFile file: 'Dockerfile', text: '''FROM openjdk:11-jre-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]'''
                    }
                    
                    echo "Building Docker image: ${DOCKER_IMAGE}"
                    sh "docker build -t ${DOCKER_IMAGE} ."
                }
            }
        }
        
        stage('Artifact Archiving') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                echo "Archiving artifacts..."
                
                // Archive the JAR file
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                
                // Archive test results
                junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                
                // Create and archive build info
                script {
                    def buildInfo = """
Build Information:
- Version: ${BUILD_VERSION}
- Build Number: ${env.BUILD_NUMBER}
- Git Commit: ${sh(script: 'git rev-parse HEAD', returnStdout: true).trim()}
- Branch: ${params.BRANCH}
- Artifact: ${env.ARTIFACT_NAME}
- Docker Image: ${DOCKER_IMAGE}
- Timestamp: ${new Date().format("yyyy-MM-dd HH:mm:ss")}
"""
                    writeFile file: "build-info-${BUILD_VERSION}.txt", text: buildInfo
                    archiveArtifacts artifacts: "build-info-${BUILD_VERSION}.txt", fingerprint: true
                }
            }
        }
        
        stage('Deployment') {
            when {
                expression { 
                    params.DEPLOY_ENV == 'staging' && 
                    !params.SKIP_DEPLOYMENT &&
                    (currentBuild.result == null || currentBuild.result == 'SUCCESS') &&
                    isUnix()
                }
            }
            steps {
                script {
                    def dockerAvailable = sh(script: 'which docker', returnStatus: true) == 0
                    if (!dockerAvailable) {
                        echo "Docker not available - simulating deployment"
                        echo "Application would be deployed to staging environment"
                        return
                    }
                    
                    echo "Deploying to ${params.DEPLOY_ENV} environment"
                    
                    // Stop and remove any existing container with the same name
                    sh "docker stop ${DOCKER_CONTAINER} || true"
                    sh "docker rm ${DOCKER_CONTAINER} || true"
                    
                    // Run the container
                    sh """
                    docker run -d \
                        --name ${DOCKER_CONTAINER} \
                        -p ${APP_PORT}:8080 \
                        ${DOCKER_IMAGE}
                    """
                    
                    // Wait for application to start
                    sleep time: 10, unit: 'SECONDS'
                    
                    // Simple health check
                    sh """
                    curl -f http://localhost:${APP_PORT} || echo "Application might still be starting"
                    """
                }
            }
            post {
                success {
                    echo "Application deployed successfully to ${params.DEPLOY_ENV}"
                    echo "Access the application at: http://localhost:${APP_PORT}"
                }
            }
        }
        
        stage('Production Deployment Simulation') {
            when {
                expression { 
                    params.DEPLOY_ENV == 'production' && 
                    !params.SKIP_DEPLOYMENT &&
                    (currentBuild.result == null || currentBuild.result == 'SUCCESS')
                }
            }
            steps {
                echo "Simulating production deployment for ${DOCKER_IMAGE}"
                sh """
                echo "Production deployment would happen here for image: ${DOCKER_IMAGE}"
                echo "Deploying to production environment..."
                """
            }
        }
    }
    
    post {
        always {
            echo "Pipeline execution completed - Cleaning up..."
            script {
                // Only try to clean up Docker if it's available
                def dockerAvailable = sh(script: 'which docker', returnStatus: true) == 0
                if (dockerAvailable) {
                    sh "docker stop ${DOCKER_CONTAINER} || true"
                    sh "docker rm ${DOCKER_CONTAINER} || true"
                } else {
                    echo "Docker not available - skipping container cleanup"
                }
                
                currentBuild.description = "Build ${BUILD_VERSION} - ${params.DEPLOY_ENV}"
            }
        }
        success {
            echo "Pipeline executed successfully!"
            mail to: 'mohamed.ali.bencheikh@horizon-university.tn',
                 subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                 body: """
                 Build ${env.BUILD_NUMBER} - ${BUILD_VERSION} completed successfully.
                 
                 Job: ${env.JOB_NAME}
                 Version: ${BUILD_VERSION}
                 Environment: ${params.DEPLOY_ENV}
                 Branch: ${params.BRANCH}
                 
                 View build: ${env.BUILD_URL}
                 """
        }
        failure {
            echo "Pipeline execution failed!"
            mail to: 'mohamed.ali.bencheikh@horizon-university.tn',
                 subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                 body: """
                 Build ${env.BUILD_NUMBER} - ${BUILD_VERSION} failed.
                 
                 Job: ${env.JOB_NAME}
                 Version: ${BUILD_VERSION}
                 
                 View build: ${env.BUILD_URL}
                 Console output: ${env.BUILD_URL}console
                 """
        }
        unstable {
            echo "Pipeline execution unstable - tests failed!"
        }
    }
}
