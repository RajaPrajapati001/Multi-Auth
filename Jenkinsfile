pipeline {
    agent any

    environment {
        // Auth Service Variables
        AUTH_IMAGE = 'auth-service-img'
        AUTH_CONTAINER = 'auth-service'
        AUTH_PORT = '5000'
        
        // HRM Service Variables
        HRM_IMAGE = 'hrm-service-img'
        HRM_CONTAINER = 'hrm-service'
        HRM_PORT = '5001'
        
        // Secrets Directory
        ENV_DIR = '/var/www/multi-auth-secrets'
    }

    stages {
        stage('1. Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('2. Manage Secrets (Both Apps)') {
            steps {
                script {
                    sh "sudo mkdir -p ${ENV_DIR}"
                    
                    // Auth Env Fetch
                    try {
                        withCredentials([file(credentialsId: 'auth-env-secret', variable: 'AUTH_ENV')]) {
                            sh "sudo cp \$AUTH_ENV ${ENV_DIR}/auth.env"
                        }
                    } catch (Exception e) {
                        echo "auth-env-secret not found in Jenkins. Falling back to permanent server file."
                    }

                    // HRM Env Fetch
                    try {
                        withCredentials([file(credentialsId: 'hrm-env-secret', variable: 'HRM_ENV')]) {
                            sh "sudo cp \$HRM_ENV ${ENV_DIR}/hrm.env"
                        }
                    } catch (Exception e) {
                        echo "hrm-env-secret not found in Jenkins. Falling back to permanent server file."
                    }
                }
            }
        }

        stage('3. Build Docker Images') {
            steps {
                // Build Auth Image (From root folder)
                echo "Building Auth Service Image..."
                sh "docker build -t ${AUTH_IMAGE}:latest ."
                
                // Build HRM Image (From HRM_AuthApp folder)
                echo "Building HRM Service Image..."
                sh "docker build -t ${HRM_IMAGE}:latest ./HRM_AuthApp"
            }
        }

        stage('4. Smart Database Migrations') {
            steps {
                // Auth DB Migration Logic
                sh '''
                    AUTH_SCHEMA_CHANGED=$(git diff --name-only HEAD~1 HEAD 2>/dev/null | grep "^prisma/schema.prisma" || echo "YES")
                    if [ -n "$AUTH_SCHEMA_CHANGED" ] || [ "$AUTH_SCHEMA_CHANGED" == "YES" ]; then
                        echo "Running Auth Service Migration..."
                        docker run --rm --env-file ${ENV_DIR}/auth.env ${AUTH_IMAGE}:latest npx prisma migrate deploy || true
                    else
                        echo "No Auth schema changes. Skipping."
                    fi
                '''
                
                // HRM DB Migration Logic
                sh '''
                    HRM_SCHEMA_CHANGED=$(git diff --name-only HEAD~1 HEAD 2>/dev/null | grep "^HRM_AuthApp/prisma/schema.prisma" || echo "YES")
                    if [ -n "$HRM_SCHEMA_CHANGED" ] || [ "$HRM_SCHEMA_CHANGED" == "YES" ]; then
                        echo "Running HRM Service Migration..."
                        docker run --rm --env-file ${ENV_DIR}/hrm.env ${HRM_IMAGE}:latest npx prisma migrate deploy || true
                    else
                        echo "No HRM schema changes. Skipping."
                    fi
                '''
            }
        }

        stage('5. Deploy Containers') {
            steps {
                sh '''
                    // Clean up old containers
                    docker stop ${AUTH_CONTAINER} ${HRM_CONTAINER} || true
                    docker rm ${AUTH_CONTAINER} ${HRM_CONTAINER} || true
                    
                    // Start Auth Service on port 5000
                    docker run -d --name ${AUTH_CONTAINER} -p ${AUTH_PORT}:${AUTH_PORT} --env-file ${ENV_DIR}/auth.env --restart unless-stopped ${AUTH_IMAGE}:latest
                    
                    // Start HRM Service on port 5001
                    docker run -d --name ${HRM_CONTAINER} -p ${HRM_PORT}:${HRM_PORT} --env-file ${ENV_DIR}/hrm.env --restart unless-stopped ${HRM_IMAGE}:latest
                '''
            }
        }

        stage('6. Health Checks') {
            steps {
                sleep time: 10, unit: 'SECONDS'
                script {
                    def authStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${AUTH_PORT}/auth/verify || true", returnStdout: true).trim()
                    def hrmStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${HRM_PORT}/auth/me || true", returnStdout: true).trim()
                    
                    if (authStatus == "000" || authStatus.startsWith("5")) {
                        error("Auth Service Health Check Failed (HTTP ${authStatus})")
                    }
                    if (hrmStatus == "000" || hrmStatus.startsWith("5")) {
                        error("HRM Service Health Check Failed (HTTP ${hrmStatus})")
                    }
                    
                    echo "Success: Both Auth (Port 5000) and HRM (Port 5001) are UP!"
                }
            }
        }
    }
}
