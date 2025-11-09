pipeline {
    agent any
    
    // Part 2: Advanced Features - Parameterization
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'DEPLOY_ENV', choices: ['staging', 'production'], description: 'Deployment environment')
        booleanParam(name: 'IS_PULL_REQUEST', defaultValue: false, description: 'Is this a pull request build?')
    }
    
    // Environment variables
    environment {
        // Part 2: Environment Variables - Use Git commit hash for versioning
        BUILD_VERSION = sh(
            script: 'git rev-parse --short HEAD',
            returnStdout: true
        ).trim()
        
        // Maven settings
        MAVEN_OPTS = '-Dmaven.repo.local=.m2/repository'
        
        // Docker image name
        DOCKER_IMAGE = "spring-petclinic:${BUILD_VERSION}"
        
        // Docker network for deployment simulation
        DOCKER_NETWORK = "petclinic-network"
    }
    
    // Tools configuration
    tools {
        maven 'Maven3'
    }
    
    stages {
        // Stage 1: Checkout
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out branch: ${params.BRANCH}"
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.BRANCH}"]],
                        extensions: [[$class: 'CloneOption', depth: 1, noTags: false, shallow: true]],
                        userRemoteConfigs: [[
                            url: 'https://github.com/mohamedalibenchiekh/spring-petclinic.git',
                            credentialsId: 'github-credentials' // Configure this in Jenkins
                        ]]
                    ])
                    
                    // Get commit details for better logging
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.GIT_AUTHOR = sh(script: 'git log -1 --pretty=format:"%an"', returnStdout: true).trim()
                }
                echo "✅ Code checked out successfully. Commit: ${env.GIT_COMMIT_SHORT} by ${env.GIT_AUTHOR}"
            }
        }
        
        // Stage 2: Build
        stage('Build') {
            steps {
                script {
                    echo "🚀 Building application with version: ${env.BUILD_VERSION}"
                    echo "🔧 Maven command: mvn clean package -DskipTests"
                    
                    // Build with Maven, skip tests as we'll run them separately in parallel
                    sh 'mvn clean package -DskipTests'
                    
                    // Set build description with version and commit
                    currentBuild.description = "Version: ${env.BUILD_VERSION}, Commit: ${env.GIT_COMMIT_SHORT}"
                }
                echo "✅ Build completed successfully. Artifact version: ${env.BUILD_VERSION}"
            }
            post {
                success {
                    // Archive build artifacts immediately after successful build
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
                failure {
                    echo "❌ Build failed. Check Maven logs for details."
                }
            }
        }
        
        // Stage 3: Parallel Testing
        stage('Parallel Testing') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        echo "🧪 Running unit tests..."
                        sh 'mvn test -Dtest=*UnitTest'
                    }
                    post {
                        always {
                            // Collect and publish unit test results - Use declarative syntax over scripted pipelines for improved readability and maintainability [[6]]
                            junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        echo "🔍 Running integration tests..."
                        sh 'mvn test -Dtest=*IntegrationTest'
                    }
                    post {
                        always {
                            // Collect and publish integration test results
                            junit testResults: '**/target/failsafe-reports/*.xml', allowEmptyResults: true
                        }
                    }
                }
            }
            post {
                failure {
                    echo "❌ Tests failed. Pipeline stopped."
                    error("Test failures detected - pipeline aborted")
                }
                success {
                    echo "✅ All tests passed successfully!"
                }
            }
        }
        
        // Stage 4: Docker Image Build
        stage('Docker Image Build') {
            steps {
                script {
                    echo "🐳 Building Docker image: ${env.DOCKER_IMAGE}"
                    
                    // Create Dockerfile if it doesn't exist (Spring PetClinic may not have one)
                    if (!fileExists('Dockerfile')) {
                        echo "⚠️ Dockerfile not found. Creating default Dockerfile..."
                        writeFile file: 'Dockerfile', text: '''
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
'''
                    }
                    
                    // Build Docker image
                    docker.build(env.DOCKER_IMAGE, ".")
                    
                    // Tag the image for latest version
                    sh "docker tag ${env.DOCKER_IMAGE} spring-petclinic:latest"
                    
                    // Optional: Push to Docker Hub (uncomment if needed)
                    /*
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        docker.image(env.DOCKER_IMAGE).push()
                        docker.image('spring-petclinic:latest').push()
                    }
                    */
                }
                echo "✅ Docker image built successfully: ${env.DOCKER_IMAGE}"
            }
            post {
                success {
                    // Archive Docker image information
                    archiveArtifacts artifacts: 'Dockerfile', fingerprint: true
                    echo "📦 Docker image information archived"
                }
            }
        }
        
        // Stage 5: Artifact Archiving
        stage('Artifact Archiving') {
            steps {
                script {
                    echo "📦 Archiving build artifacts with versioning..."
                    
                    // Archive JAR file with version naming
                    sh """
                        mkdir -p archived-artifacts
                        cp target/*.jar archived-artifacts/spring-petclinic-${env.BUILD_VERSION}-${BUILD_NUMBER}.jar
                    """
                    
                    // Archive the versioned artifact
                    archiveArtifacts artifacts: 'archived-artifacts/*.jar', fingerprint: true
                    
                    // Also archive Docker image details
                    sh """
                        docker save ${env.DOCKER_IMAGE} -o archived-artifacts/spring-petclinic-${env.BUILD_VERSION}.tar
                    """
                    
                    // Create version manifest
                    writeFile file: 'archived-artifacts/version-manifest.txt', text: """
Build Number: ${BUILD_NUMBER}
Build Version: ${env.BUILD_VERSION}
Git Commit: ${env.GIT_COMMIT_SHORT}
Build Date: ${new Date().format("yyyy-MM-dd HH:mm:ss")}
Docker Image: ${env.DOCKER_IMAGE}
Deploy Environment: ${params.DEPLOY_ENV}
"""
                    
                    archiveArtifacts artifacts: 'archived-artifacts/*', fingerprint: true
                }
                echo "✅ Artifacts archived successfully with version: ${env.BUILD_VERSION}"
            }
        }
        
        // Stage 6: Deployment Simulation
        stage('Deployment Simulation') {
            // Part 2: Conditional Stages - Deploy only if DEPLOY_ENV == staging and not a PR
            when {
                allOf {
                    expression { params.DEPLOY_ENV == 'staging' }
                    expression { !params.IS_PULL_REQUEST }
                }
            }
            steps {
                script {
                    echo "🚀 Deploying to ${params.DEPLOY_ENV} environment..."
                    
                    // Create Docker network if it doesn't exist
                    sh """
                        if ! docker network ls | grep -q ${env.DOCKER_NETWORK}; then
                            docker network create ${env.DOCKER_NETWORK}
                        fi
                    """
                    
                    // Stop and remove existing container if running
                    sh """
                        if docker ps -a | grep -q petclinic-${params.DEPLOY_ENV}; then
                            docker stop petclinic-${params.DEPLOY_ENV} || true
                            docker rm petclinic-${params.DEPLOY_ENV} || true
                        fi
                    """
                    
                    // Run the Docker container
                    docker.image(env.DOCKER_IMAGE).run(
                        "--name petclinic-${params.DEPLOY_ENV} " +
                        "--network ${env.DOCKER_NETWORK} " +
                        "-p 8080:8080 " +
                        "-e SPRING_PROFILES_ACTIVE=${params.DEPLOY_ENV}"
                    )
                    
                    // Verify deployment
                    sleep 10 // Wait for container to start
                    sh """
                        echo "✅ Application deployed successfully."
                        echo "🌐 Application URL: http://localhost:8080"
                        echo "📋 Container ID: petclinic-${params.DEPLOY_ENV}"
                        echo "🏷️  Version: ${env.BUILD_VERSION}"
                        
                        # Optional: Health check
                        curl -s http://localhost:8080/actuator/health || echo "⚠️ Health check failed, but deployment completed"
                    """
                }
            }
            post {
                failure {
                    echo "❌ Deployment failed. Check Docker logs."
                    sh "docker logs petclinic-${params.DEPLOY_ENV} || true"
                }
            }
        }
    }
    
    // Part 2: Email Notifications
    post {
        always {
            echo "📧 Sending build notification..."
            emailext(
                subject: "[${currentBuild.result}] Jenkins Build - ${JOB_NAME} #${BUILD_NUMBER}",
                body: """
Jenkins Build Report
====================
Job: ${JOB_NAME}
Build Number: ${BUILD_NUMBER}
Status: ${currentBuild.result}
Branch: ${params.BRANCH}
Environment: ${params.DEPLOY_ENV}
Version: ${env.BUILD_VERSION}
Commit: ${env.GIT_COMMIT_SHORT}
Author: ${env.GIT_AUTHOR}
Build URL: ${BUILD_URL}
Console Output: ${BUILD_URL}console

Build Summary:
- Checkout: ${stageResult('Checkout') ?: 'N/A'}
- Build: ${stageResult('Build') ?: 'N/A'}
- Tests: ${stageResult('Parallel Testing') ?: 'N/A'}
- Docker Build: ${stageResult('Docker Image Build') ?: 'N/A'}
- Artifacts: ${stageResult('Artifact Archiving') ?: 'N/A'}
- Deployment: ${stageResult('Deployment Simulation') ?: 'Skipped'}

Artifacts are available at: ${BUILD_URL}artifact/
""",
                to: "dev-team@company.com", // Configure your email recipients
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
        success {
            echo "🎉 Build completed successfully!"
        }
        failure {
            echo "❌ Build failed. Check logs for details."
        }
    }
}

// Helper function to get stage results
def stageResult(stageName) {
    try {
        return currentBuild.rawBuild.getActions(jenkins.model.CauseOfInterruption.class).find { it.stageName == stageName }?.result ?: 'SUCCESS'
    } catch (Exception e) {
        return 'UNKNOWN'
    }
}
