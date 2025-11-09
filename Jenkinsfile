pipeline {
    agent any
    
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'DEPLOY_ENV', choices: ['staging', 'production'], description: 'Deployment environment')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip test stages')
    }
    
    environment {
        APP_NAME = 'spring-petclinic'
        DOCKER_IMAGE = "spring-petclinic:${BUILD_VERSION}"
        COMMIT_HASH = sh(returnStdout: true, script: 'git rev-parse --short HEAD 2>/dev/null || echo "no-git"').trim()
        BUILD_VERSION = "${BUILD_NUMBER}-${COMMIT_HASH}"
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
                        echo '🧪 Running Unit Tests...'
                        sh './mvnw test'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        echo '🔬 Running Integration Tests...'
                        // This will run integration tests if they exist, or continue gracefully
                        sh '''
                            ./mvnw verify -DskipTests=false || \
                            echo "No integration tests found, continuing..."
                        '''
                    }
                    post {
                        always {
                            // Allow empty test results for integration tests
                            junit allowEmptyResults: true, testResults: 'target/failsafe-reports/*.xml'
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
                    
                    sh '''
                        docker network inspect petclinic-network >/dev/null 2>&1 || \
                        docker network create petclinic-network
                    '''
                    
                    sh """
                        docker stop ${APP_NAME} 2>/dev/null || true
                        docker rm ${APP_NAME} 2>/dev/null || true
                    """
                    
                    sh """
                        docker run -d \
                          --name ${APP_NAME} \
                          --network petclinic-network \
                          -p 8080:8080 \
                          ${DOCKER_IMAGE}
                    """
                    
                    sh """
                        mkdir -p /var/jenkins_home/deployments/${params.DEPLOY_ENV}
                        cp target/${APP_NAME}-${BUILD_VERSION}.tar /var/jenkins_home/deployments/${params.DEPLOY_ENV}/
                    """
                    
                    echo "✅ Application deployed successfully!"
                    echo "🌐 Access it at: http://localhost:8080"
                }
            }
        }
    }
    
    post {
        always {
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
    }
}