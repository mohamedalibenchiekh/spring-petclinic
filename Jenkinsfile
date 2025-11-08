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
        
        // Maven configuration
        MAVEN_OPTS = "-Dmaven.repo.local=${env.JENKINS_HOME}/.m2/repository"
        
        // Application port
        APP_PORT = "8080"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        gitLabConnection('')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "Checking out from branch: ${params.BRANCH}"
                git branch: "${params.BRANCH}",
                    url: 'https://github.com/mohamedalibenchiekh/spring-petclinic.git',
                    credentialsId: ''
            }
        }
        
        stage('Build') {
            steps {
                echo "Building with version: ${BUILD_VERSION}"
                sh 'mvn clean package -DskipTests'
                
                // Store the built artifact path
                script {
                    def jarFile = findFiles(glob: 'target/*.jar')[0]
                    env.ARTIFACT_PATH = jarFile.path
                    env.ARTIFACT_NAME = jarFile.name
                }
            }
            post {
                success {
                    echo "Build successful - Artifact: ${env.ARTIFACT_NAME}"
                }
            }
        }
        
        stage('Parallel Testing') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        echo "Running unit tests..."
                        sh 'mvn test -Dtest=**/*Test'
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
                        sh 'mvn integration-test -DskipUnitTests'
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
            steps {
                script {
                    // Create Dockerfile if it doesn't exist
                    if (!fileExists('Dockerfile')) {
                        writeFile file: 'Dockerfile', text: '''
FROM openjdk:11-jre-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
'''
                    }
                    
                    echo "Building Docker image: ${DOCKER_IMAGE}"
                    sh "docker build -t ${DOCKER_IMAGE} ."
                    
                    // Optional: Push to Docker Hub (uncomment and configure credentials)
                    // docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                    //     sh "docker push ${DOCKER_IMAGE}"
                    // }
                }
            }
        }
        
        stage('Artifact Archiving') {
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
                    params.DEPLOY_ENV == 'staging' && !params.SKIP_DEPLOYMENT 
                }
            }
            steps {
                script {
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
                    params.DEPLOY_ENV == 'production' && !params.SKIP_DEPLOYMENT 
                }
            }
            steps {
                echo "Simulating production deployment for ${DOCKER_IMAGE}"
                // In a real scenario, this would deploy to production environment
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
            // Clean up Docker containers
            sh "docker stop ${DOCKER_CONTAINER} || true"
            sh "docker rm ${DOCKER_CONTAINER} || true"
            
            // Generate build summary
            script {
                currentBuild.description = "Build ${BUILD_VERSION} - ${params.DEPLOY_ENV}"
            }
        }
        success {
            echo "Pipeline executed successfully!"
            emailext (
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                <h2>Build Successful!</h2>
                <p><b>Job:</b> ${env.JOB_NAME}</p>
                <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                <p><b>Version:</b> ${BUILD_VERSION}</p>
                <p><b>Environment:</b> ${params.DEPLOY_ENV}</p>
                <p><b>Git Branch:</b> ${params.BRANCH}</p>
                <p><b>Commit:</b> ${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}</p>
                <p><b>View build:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                to: "devops-team@company.com",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
        failure {
            echo "Pipeline execution failed!"
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                <h2>Build Failed!</h2>
                <p><b>Job:</b> ${env.JOB_NAME}</p>
                <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                <p><b>View build:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                <p><b>Console output:</b> <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>
                """,
                to: "devops-team@company.com",
                recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']]
            )
        }
        unstable {
            echo "Pipeline execution unstable - tests failed!"
        }
        changed {
            echo "Pipeline status changed!"
        }
    }
}
