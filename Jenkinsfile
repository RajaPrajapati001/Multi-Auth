pipeline {
    agent any

    environment {
        IMAGE_NAME = 'auth-service-image'
        CONTAINER_NAME = 'auth-service'
        ENV_DIR = '/var/www/multi-auth-secrets'
        AUTH_ENV_FILE = "${ENV_DIR}/auth.env"
        PORT = '5000'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Manage Secrets') {
            steps {
                script {
                    sh "sudo mkdir -p ${ENV_DIR}"
                    
                    try {
                        withCredentials([file(credentialsId: 'auth-env', variable: 'JENKINS_ENV')]) {
                            sh "sudo cp \$JENKINS_ENV ${AUTH_ENV_FILE}"
                            echo "Successfully updated server .env from Jenkins Credentials."
                        }
                    } catch (Exception e) {
                        echo "Jenkins credential 'auth-env-secret' not found. Falling back to permanent server file."
                        
                        def fileExists = sh(script: "test -f ${AUTH_ENV_FILE} && echo 'YES' || echo 'NO'", returnStdout: true).trim()
                        if (fileExists == 'NO') {
                            error("FATAL: No env file found in Jenkins and no permanent file found on server. Cannot proceed.")
                        } else {
                            echo "Using existing permanent .env file from ${AUTH_ENV_FILE}."
                        }
                    }
                }
            }
        }

        stage('Backup Previous Version') {
            steps {
                sh '''
                    if docker image inspect ${IMAGE_NAME}:latest >/dev/null 2>&1; then
                        docker tag ${IMAGE_NAME}:latest ${IMAGE_NAME}:previous || true
                    fi
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building new Docker image..."
                sh "docker build -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Database Migration') {
            steps {
                sh '''
                    SCHEMA_CHANGED=$(git diff --name-only HEAD~1 HEAD 2>/dev/null | grep "prisma/schema.prisma" || echo "YES")
                    
                    if [ -n "$SCHEMA_CHANGED" ] || [ "$SCHEMA_CHANGED" == "YES" ]; then
                        echo "Schema changes detected. Running migration."
                        docker run --rm --env-file ${AUTH_ENV_FILE} ${IMAGE_NAME}:latest npx prisma migrate deploy
                    else
                        echo "No schema changes. Skipping migration."
                    fi
                '''
            }
        }

        stage('Deploy App') {
            steps {
                sh '''
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    
                    docker run -d --name ${CONTAINER_NAME} -p ${PORT}:${PORT} --env-file ${AUTH_ENV_FILE} --restart unless-stopped ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Health Check') {
            steps {
                echo "Waiting 10 seconds for service initialization..."
                sleep time: 10, unit: 'SECONDS'
                
                script {
                    def healthStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${PORT}/auth/verify || true", returnStdout: true).trim()
                    echo "Health Check Response Code: ${healthStatus}"
                    
                    if (healthStatus == "000" || healthStatus.startsWith("5")) {
                        error("Health check failed with HTTP ${healthStatus}. Triggering rollback.")
                    } else {
                        echo "Service is operational."
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "Pipeline failed. Rolling back to the last known-good version."
            sh '''
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
                
                if docker image inspect ${IMAGE_NAME}:previous >/dev/null 2>&1; then
                    docker run -d --name ${CONTAINER_NAME} -p ${PORT}:${PORT} --env-file ${AUTH_ENV_FILE} --restart unless-stopped ${IMAGE_NAME}:previous
                    echo "Rollback successful. System restored to previous state."
                else
                    echo "No previous backup found to rollback."
                fi
            '''
        }
        success {
            echo "Deployment completed successfully."
        }
    }
}
