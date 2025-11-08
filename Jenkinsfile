pipeline {
    agent any

    parameters {
        choice(
            name: 'BRANCH',
            choices: ['main', 'develop', 'feature/*'],
            description: 'Select the branch to build'
        )
        choice(
            name: 'DEPLOY_ENV',
            choices: ['staging', 'production'],
            description: 'Select the deployment environment'
        )
    }

    environment {
        // Dynamic environment variables
        BUILD_VERSION = "${env.BUILD_NUMBER}-${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}"
        DOCKER_IMAGE = "spring-petclinic:${BUILD_VERSION}"
        DOCKER_NETWORK = "petclinic-net"
    }

    stages {
        // Stage 1: Checkout
        stage('Checkout') {
            steps {
                echo "Checking out branch: ${params.BRANCH}"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${params.BRANCH}"]],
                    extensions: [],
                    userRemoteConfigs: [[url: 'https://github.com/mohamedalibenchiekh/spring-petclinic.git']]
                ])
            }
        }

        // Stage 2: Build
        agent any
        tools {
            jdk 'jdk25'
        }
        stage('Build') {
            steps {
                echo "Building with Maven..."
                sh 'mvn clean package'
            }
        }

        // Stage 3: Parallel Testing
        stage('Parallel Testing') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test'
                        junit '**/target/surefire-reports/*.xml'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'mvn verify -DskipUnitTests'
                        junit '**/target/failsafe-reports/*.xml'
                    }
                }
            }
            // Fail the build if any test fails
            post {
                failure {
                    error "Tests failed. Check the logs."
                }
            }
        }

        // Stage 4: Docker Image Build
        stage('Docker Image Build') {
            steps {
                echo "Building Docker image: ${DOCKER_IMAGE}"
                sh "docker build -t ${DOCKER_IMAGE} ."
                // Optional: Push to Docker Hub
                // sh "docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}"
                // sh "docker push ${DOCKER_IMAGE}"
            }
        }

        // Stage 5: Artifact Archiving
        stage('Artifact Archiving') {
            steps {
                echo "Archiving artifacts..."
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                // Archive Docker image reference (optional)
                writeFile file: 'docker-image.txt', text: "${DOCKER_IMAGE}"
                archiveArtifacts artifacts: 'docker-image.txt'
            }
        }

        // Stage 6: Deployment Simulation
        stage('Deployment Simulation') {
            when {
                // Conditional: Only deploy if DEPLOY_ENV == staging
                expression { params.DEPLOY_ENV == 'staging' }
            }
            steps {
                echo "Deploying to ${params.DEPLOY_ENV}..."
                sh "docker network create ${DOCKER_NETWORK} || true"
                sh "docker run -d --network ${DOCKER_NETWORK} --name petclinic-${BUILD_VERSION} ${DOCKER_IMAGE}"
                echo "Application deployed successfully."
            }
        }
    }

    // Part 2: Email Notifications
    post {
        always {
            echo "Sending email notification..."
            emailext (
                subject: "Build ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <p>Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}</p>
                    <p>Status: ${currentBuild.currentResult}</p>
                    <p>Branch: ${params.BRANCH}</p>
                    <p>Deploy Environment: ${params.DEPLOY_ENV}</p>
                    <p>Docker Image: ${DOCKER_IMAGE}</p>
                    <p>Check console output at: ${env.BUILD_URL}</p>
                """,
                recipientProviders: [[$class: 'RequesterRecipientProvider']]
            )
        }
    }
}
