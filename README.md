pipeline {
    agent any // Use any available Jenkins agent/node

    // Define the environment variables
    environment {
        DOCKER_REGISTRY = 'dockerhub_account/my-web-app' // Docker image registry
        STAGING_ENV = 'staging-server-url'
        PRODUCTION_ENV = 'production-server-url'
        // Secure credentials
        DOCKER_CREDENTIALS = credentials('dockerhub-credentials-id')
        SSH_CREDENTIALS = credentials('ssh-credentials-id')
    }

    stages {
        // Stage 1: Build the application
        stage('Build') {
            steps {
                echo 'Compiling the application...'
                sh 'mvn clean install' // Example for a Java app using Maven; replace with your build command
            }
        }

        // Stage 2: Run Unit Tests
        stage('Test') {
            steps {
                echo 'Running unit tests...'
                sh 'mvn test' // Replace with the appropriate command for your framework
            }
            post {
                // If tests fail, mark the stage as failed
                failure {
                    echo 'Unit tests failed!'
                }
            }
        }

        // Stage 3: Package the application
        stage('Package') {
            steps {
                echo 'Packaging the application into a Docker image...'
                sh '''
                docker login -u $DOCKER_CREDENTIALS_USR -p $DOCKER_CREDENTIALS_PSW
                docker build -t $DOCKER_REGISTRY:latest .
                docker push $DOCKER_REGISTRY:latest
                '''
            }
        }

        // Stage 4: Deploy to Staging
        stage('Deploy to Staging') {
            steps {
                echo 'Deploying to the staging environment...'
                sh '''
                ssh -i $SSH_CREDENTIALS /path/to/deployment/script.sh staging $DOCKER_REGISTRY:latest
                '''
            }
        }

        // Stage 5: Approval for Production Deployment
        stage('Approval') {
            steps {
                script {
                    echo 'Waiting for manual approval to deploy to production...'
                    timeout(time: 1, unit: 'HOURS') {
                        input message: 'Approve deployment to production?'
                    }
                }
            }
        }

        // Stage 6: Deploy to Production
        stage('Deploy to Production') {
            steps {
                echo 'Deploying to the production environment...'
                sh '''
                ssh -i $SSH_CREDENTIALS /path/to/deployment/script.sh production $DOCKER_REGISTRY:latest
                '''
            }
        }
    }

    // Post-build actions
    post {
        always {
            echo 'Pipeline completed.'
        }
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed. Sending notifications...'
            // Add email/slack notifications here
        }
    }
}
