pipeline {
    agent any
    
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'DEPLOY_ENV', choices: ['staging', 'production'], description: 'Deployment environment')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip test stages')
    }
    
    environment {
        // Application details
        APP_NAME = 'spring-petclinic'
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/your-dockerhub-username/${APP_NAME}"
        
        // Dynamic versioning: BuildNumber-CommitHash
        COMMIT_HASH = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
        BUILD_VERSION = "${BUILD_NUMBER}-${COMMIT_HASH}"
        
        // Maven options for better performance
        MAVEN_OPTS = '-Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN'
    }
    
    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                git branch: "${params.BRANCH}",
                    url: 'https://github.com/mohamedalibenchiekh/spring-petclinic.git'
            }
        }
        
        stage('Build') {
            steps {
                echo "Building version: ${BUILD_VERSION}"
                sh """
                    mvn clean compile -DskipTests
                    mvn package -DskipTests
                """
            }
            post {
                success {
                    // Archive the JAR artifact immediately after build
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
                        echo 'Running Unit Tests...'
                        sh 'mvn test -Dtest="*Test"'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        echo 'Running Integration Tests...'
                        sh 'mvn verify -Dtest="*IntegrationTest"'
                    }
                    post {
                        always {
                            junit 'target/failsafe-reports/*.xml'
                        }
                    }
                }
            }
        }
        
        stage('Docker Image Build') {
            steps {
                script {
                    // Create a simple Dockerfile if it doesn't exist
                    writeFile file: 'Dockerfile', text: '''
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
'''
                    def dockerImage = docker.build("${DOCKER_IMAGE}:${BUILD_VERSION}")
                    
                    // Optional: Push to Docker Hub (uncomment if you configured credentials)
                    /*
                    withDockerRegistry([credentialsId: 'docker-hub-credentials', url: "https://${DOCKER_REGISTRY}"]) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                    */
                    
                    // Save image to tar for archiving
                    sh "docker save ${DOCKER_IMAGE}:${BUILD_VERSION} -o target/${APP_NAME}-${BUILD_VERSION}.tar"
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
                    not { expression { return env.CHANGE_ID != null } } // Skip for PRs
                }
            }
            steps {
                script {
                    echo "Deploying to ${params.DEPLOY_ENV} environment..."
                    
                    // Create Docker network if it doesn't exist
                    sh '''
                        docker network inspect petclinic-network >/dev/null 2>&1 || \
                        docker network create petclinic-network
                    '''
                    
                    // Stop and remove existing container if exists
                    sh """
                        docker stop ${APP_NAME} || true
                        docker rm ${APP_NAME} || true
                    """
                    
                    // Run the new container
                    sh """
                        docker run -d \
                          --name ${APP_NAME} \
                          --network petclinic-network \
                          -p 8080:8080 \
                          ${DOCKER_IMAGE}:${BUILD_VERSION}
                    """
                    
                    // Simulate deployment by copying to a deployment folder
                    sh """
                        mkdir -p /var/jenkins_home/deployments/${params.DEPLOY_ENV}
                        cp target/${APP_NAME}-${BUILD_VERSION}.tar \
                           /var/jenkins_home/deployments/${params.DEPLOY_ENV}/
                    """
                    
                    echo "✅ Application deployed successfully!"
                    echo "Access it at: http://localhost:8080"
                }
            }
        }
    }
    
    post {
        always {
            // Clean workspace
            cleanWs()
        }
        
        success {
            script {
                def duration = currentBuild.durationString.replace(' and counting', '')
                emailext (
                    subject: "✅ Build Success: ${env.JOB_NAME} #${BUILD_NUMBER}",
                    body: """
                        <h2>Build Successful! 🎉</h2>
                        <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                        <p><strong>Build Number:</strong> #${BUILD_NUMBER}</p>
                        <p><strong>Version:</strong> ${BUILD_VERSION}</p>
                        <p><strong>Branch:</strong> ${params.BRANCH}</p>
                        <p><strong>Environment:</strong> ${params.DEPLOY_ENV}</p>
                        <p><strong>Duration:</strong> ${duration}</p>
                        <p><strong>Commit Hash:</strong> ${COMMIT_HASH}</p>
                        <p>View the build: <a href="${BUILD_URL}">${BUILD_URL}</a></p>
                    """,
                    to: 'your-email@example.com',
                    mimeType: 'text/html'
                )
            }
        }
        
        failure {
            script {
                emailext (
                    subject: "❌ Build Failed: ${env.JOB_NAME} #${BUILD_NUMBER}",
                    body: """
                        <h2>Build Failed! 💥</h2>
                        <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                        <p><strong>Build Number:</strong> #${BUILD_NUMBER}</p>
                        <p><strong>Version:</strong> ${BUILD_VERSION}</p>
                        <p><strong>Branch:</strong> ${params.BRANCH}</p>
                        <p><strong>Failed Stage:</strong> ${env.STAGE_NAME}</p>
                        <p>Check the logs: <a href="${BUILD_URL}console">${BUILD_URL}console</a></p>
                    """,
                    to: 'your-email@example.com',
                    mimeType: 'text/html'
                )
            }
        }
        
        unstable {
            echo 'Build is unstable - some tests may have failed'
        }
    }
}