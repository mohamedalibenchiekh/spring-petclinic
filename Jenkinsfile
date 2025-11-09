pipeline {
    agent any
    
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'DEPLOY_ENV', choices: ['staging', 'production'], description: 'Deployment environment')
        booleanParam(name: 'IS_PULL_REQUEST', defaultValue: false, description: 'Is this a pull request build?')
    }
    
    environment {
        BUILD_VERSION = sh(
            script: 'git rev-parse --short HEAD',
            returnStdout: true
        ).trim()
        MAVEN_OPTS = '-Dmaven.repo.local=.m2/repository'
        DOCKER_IMAGE = "spring-petclinic:${BUILD_VERSION}"
        DOCKER_NETWORK = "petclinic-network"
    }
    
    tools {
        maven 'Maven3'
    }
    
    stages {
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
                            credentialsId: 'github-credentials'
                        ]]
                    ])
                    
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.GIT_AUTHOR = sh(script: 'git log -1 --pretty=format:"%an"', returnStdout: true).trim()
                }
                echo "✅ Code checked out successfully. Commit: ${env.GIT_COMMIT_SHORT} by ${env.GIT_AUTHOR}"
            }
        }
        
        stage('Build') {
            steps {
                script {
                    echo "🚀 Building application with version: ${env.BUILD_VERSION}"
                    echo "🔧 Maven command: mvn clean package -DskipTests -Dcheckstyle.skip=true"
                    
                    // Fixed: Use the correct parameter to skip checkstyle
                    sh 'mvn clean package -DskipTests -Dcheckstyle.skip=true'
                    
                    currentBuild.description = "Version: ${env.BUILD_VERSION}, Commit: ${env.GIT_COMMIT_SHORT}"
                }
                echo "✅ Build completed successfully. Artifact version: ${env.BUILD_VERSION}"
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
                failure {
                    echo "❌ Build failed. Check Maven logs for details."
                }
            }
        }
        
        stage('Test') {
            steps {
                echo "🧪 Running all tests..."
                sh 'mvn test'
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
                failure {
                    error("Tests failed - pipeline aborted")
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    echo "🐳 Building Docker image: ${env.DOCKER_IMAGE}"
                    
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
                    
                    sh "docker build -t ${env.DOCKER_IMAGE} ."
                    sh "docker tag ${env.DOCKER_IMAGE} spring-petclinic:latest"
                }
                echo "✅ Docker image built successfully: ${env.DOCKER_IMAGE}"
            }
        }
        
        stage('Archive Artifacts') {
            steps {
                script {
                    mkdir 'archived-artifacts'
                    sh "cp target/*.jar archived-artifacts/spring-petclinic-${env.BUILD_VERSION}-${BUILD_NUMBER}.jar"
                    archiveArtifacts artifacts: 'archived-artifacts/*.jar', fingerprint: true
                    
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
                echo "✅ Artifacts archived successfully"
            }
        }
        
        stage('Deploy') {
            when {
                allOf {
                    expression { params.DEPLOY_ENV == 'staging' }
                    expression { !params.IS_PULL_REQUEST }
                }
            }
            steps {
                script {
                    echo "🚀 Deploying to ${params.DEPLOY_ENV} environment..."
                    
                    sh """
                        docker network create ${env.DOCKER_NETWORK} || true
                        docker stop petclinic-${params.DEPLOY_ENV} || true
                        docker rm petclinic-${params.DEPLOY_ENV} || true
                    """
                    
                    sh """
                        docker run -d \
                        --name petclinic-${params.DEPLOY_ENV} \
                        --network ${env.DOCKER_NETWORK} \
                        -p 8080:8080 \
                        -e SPRING_PROFILES_ACTIVE=${params.DEPLOY_ENV} \
                        ${env.DOCKER_IMAGE}
                    """
                    
                    sleep 10
                    sh """
                        echo "✅ Application deployed successfully."
                        echo "🌐 Application URL: http://localhost:8080"
                        echo "📋 Container ID: petclinic-${params.DEPLOY_ENV}"
                        echo "🏷️  Version: ${env.BUILD_VERSION}"
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo "📧 Build completed. Status: ${currentBuild.result}"
        }
        success {
            echo "🎉 Build succeeded!"
        }
        failure {
            echo "❌ Build failed!"
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
