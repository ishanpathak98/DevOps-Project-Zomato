pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        nodejs 'node24'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE_NAME = 'zomato'
        DOCKER_REGISTRY_USER = 'ishanpathak98'
        CONTAINER_NAME = 'zomato-app'
        APP_PORT = '3000'
        // Security: Don't expose sensitive data in environment
        TRIVY_TIMEOUT = '10m'
        TRIVY_FORMAT = 'table'
    }
    
    options {
        // Build timeout to prevent hanging builds
        timeout(time: 45, unit: 'MINUTES')
        // Keep only last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Disable concurrent builds on same branch
        disableConcurrentBuilds()
        // Add timestamps to console output
        timestamps()
    }
    
    stages {
        stage('Clean Workspace') {
            steps {
                script {
                    echo "🧹 Cleaning workspace..."
                    cleanWs()
                }
            }
        }
        
        stage('Git Checkout') {
            steps {
                script {
                    echo "📥 Checking out source code..."
                    try {
                        git branch: 'master', 
                            url: 'https://github.com/ishanpathak98/DevOps-Project-Zomato.git'
                        
                        // Get commit information for build metadata
                        env.GIT_COMMIT = sh(
                            script: 'git rev-parse HEAD',
                            returnStdout: true
                        ).trim()
                        
                        env.GIT_BRANCH = sh(
                            script: 'git rev-parse --abbrev-ref HEAD',
                            returnStdout: true
                        ).trim()
                        
                        echo "✅ Git checkout completed"
                        echo "📋 Commit: ${env.GIT_COMMIT}"
                        echo "🌿 Branch: ${env.GIT_BRANCH}"
                        
                    } catch (Exception e) {
                        error "❌ Git checkout failed: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    echo "🔍 Running SonarQube Analysis..."
                    try {
                        withSonarQubeEnv('sonar-server') {
                            sh """
                                ${SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectName=zomato-web-app \
                                -Dsonar.projectKey=zomato-web-app \
                                -Dsonar.sources=. \
                                -Dsonar.exclusions=node_modules/**,coverage/**,dist/**
                            """
                        }
                    } catch (Exception e) {
                        error "❌ SonarQube analysis failed: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Code Quality Gate') {
            steps {
                script {
                    echo "🎯 Waiting for Quality Gate results..."
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            def qg = waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                            if (qg.status != 'OK') {
                                echo "⚠️  Quality Gate status: ${qg.status}"
                                // Continue build but mark as unstable
                                currentBuild.result = 'UNSTABLE'
                            } else {
                                echo "✅ Quality Gate passed!"
                            }
                        }
                    } catch (Exception e) {
                        echo "⚠️  Quality Gate timeout or error: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Install NPM Dependencies') {
            steps {
                script {
                    echo "📦 Installing NPM dependencies..."
                    try {
                        // Check if package.json exists
                        if (fileExists('package.json')) {
                            sh """
                                echo "Node version: \$(node --version)"
                                echo "NPM version: \$(npm --version)"
                                npm ci --only=production --no-audit --no-fund
                            """
                        } else {
                            echo "⚠️  No package.json found, skipping NPM install"
                        }
                    } catch (Exception e) {
                        error "❌ NPM installation failed: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('OWASP Dependency Check') {
            steps {
                script {
                    echo "🛡️  Running OWASP Dependency Check..."
                    try {
                        dependencyCheck additionalArguments: '''
                            --scan ./
                            --disableYarnAudit
                            --disableNodeAudit
                            --format XML
                            --format HTML
                            --prettyPrint
                        ''', odcInstallation: 'DP-Check'
                        
                        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                        
                        // Archive HTML report for easy viewing
                        if (fileExists('dependency-check-report.html')) {
                            archiveArtifacts artifacts: 'dependency-check-report.html', fingerprint: true
                        }
                    } catch (Exception e) {
                        echo "⚠️  OWASP Dependency Check failed: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Trivy File System Scan') {
            steps {
                script {
                    echo "🔒 Running Trivy File System scan..."
                    try {
                        sh """
                            # Check if trivy is installed
                            if ! command -v trivy &> /dev/null; then
                                echo "Installing Trivy..."
                                sudo apt-get update
                                sudo apt-get install wget apt-transport-https gnupg lsb-release -y
                                wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                                echo "deb https://aquasecurity.github.io/trivy-repo/deb \$(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
                                sudo apt-get update
                                sudo apt-get install trivy -y
                            fi
                            
                            # Run Trivy scan
                            trivy fs . \
                                --timeout ${TRIVY_TIMEOUT} \
                                --format ${TRIVY_FORMAT} \
                                --output trivy-fs-report.txt \
                                --severity HIGH,CRITICAL
                                
                            echo "Trivy scan completed. Check trivy-fs-report.txt for details."
                        """
                        
                        // Archive the report
                        archiveArtifacts artifacts: 'trivy-fs-report.txt', fingerprint: true
                        
                    } catch (Exception e) {
                        echo "⚠️  Trivy scan failed: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "🐳 Building Docker image..."
                    try {
                        // Check if Dockerfile exists
                        if (!fileExists('Dockerfile')) {
                            error "❌ Dockerfile not found in the repository"
                        }
                        
                            sh """
                                # Build the image with build args and labels
                                docker build \
                                    --build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
                                    --build-arg VCS_REF=${env.GIT_COMMIT} \
                                    --build-arg BUILD_NUMBER=${env.BUILD_NUMBER} \
                                    --label "build.number=${env.BUILD_NUMBER}" \
                                    --label "build.url=${env.BUILD_URL}" \
                                    --label "git.commit=${env.GIT_COMMIT}" \
                                    --label "git.branch=${env.GIT_BRANCH}" \
                                    -t ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} \
                                    -t ${DOCKER_IMAGE_NAME}:latest .
                            """
                        
                        echo "✅ Docker image built successfully"
                        
                    } catch (Exception e) {
                        error "❌ Docker build failed: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    echo "🚀 Tagging and pushing to DockerHub..."
                    try {
                        withDockerRegistry(credentialsId: 'docker') {
                            sh """
                                # Tag images
                                docker tag ${DOCKER_IMAGE_NAME}:latest ${DOCKER_REGISTRY_USER}/${DOCKER_IMAGE_NAME}:latest
                                docker tag ${DOCKER_IMAGE_NAME}:latest ${DOCKER_REGISTRY_USER}/${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}
                                
                                # Push images
                                docker push ${DOCKER_REGISTRY_USER}/${DOCKER_IMAGE_NAME}:latest
                                docker push ${DOCKER_REGISTRY_USER}/${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}
                                
                                echo "✅ Images pushed successfully"
                            """
                        }
                    } catch (Exception e) {
                        error "❌ Docker push failed: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Docker Security Scan') {
            steps {
                script {
                    echo "🔍 Running Docker security scan..."
                    try {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                            sh """
                                # Check Docker version and Scout availability
                                docker version
                                
                                # Try integrated Docker Scout first
                                if docker scout version >/dev/null 2>&1; then
                                    echo "Using integrated Docker Scout..."
                                    docker scout cves ${DOCKER_REGISTRY_USER}/${DOCKER_IMAGE_NAME}:latest || echo "Scout CVE scan completed with warnings"
                                    docker scout recommendations ${DOCKER_REGISTRY_USER}/${DOCKER_IMAGE_NAME}:latest || echo "Scout recommendations completed"
                                else
                                    # Fallback to Trivy for container scanning
                                    echo "Docker Scout not available, using Trivy for container scan..."
                                    trivy image \
                                        --timeout ${TRIVY_TIMEOUT} \
                                        --format table \
                                        --output trivy-docker-report.txt \
                                        --severity HIGH,CRITICAL \
                                        ${DOCKER_REGISTRY_USER}/${DOCKER_IMAGE_NAME}:latest
                                fi
                            """
                        }
                        
                        // Archive scan results if they exist
                        script {
                            if (fileExists('trivy-docker-report.txt')) {
                                archiveArtifacts artifacts: 'trivy-docker-report.txt', fingerprint: true
                            }
                        }
                        
                    } catch (Exception e) {
                        echo "⚠️  Docker security scan failed: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Deploy to Container') {
            steps {
                script {
                    echo "🚀 Deploying to container..."
                    try {
                        sh """
                            # Stop and remove existing container if it exists
                            if docker ps -a --format 'table {{.Names}}' | grep -q '^${CONTAINER_NAME}\$'; then
                                echo "Stopping existing container..."
                                docker stop ${CONTAINER_NAME} || true
                                docker rm ${CONTAINER_NAME} || true
                            fi
                            
                            # Run new container with health check and resource limits
                            docker run -d \
                                --name ${CONTAINER_NAME} \
                                --restart unless-stopped \
                                -p ${APP_PORT}:${APP_PORT} \
                                --memory="512m" \
                                --cpus="1.0" \
                                --health-cmd="curl -f http://localhost:${APP_PORT}/health || exit 1" \
                                --health-interval=30s \
                                --health-timeout=10s \
                                --health-retries=3 \
                                ${DOCKER_REGISTRY_USER}/${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}
                            
                            # Wait for container to be healthy
                            echo "Waiting for container to be healthy..."
                            timeout 60 sh -c 'until docker inspect --format="{{.State.Health.Status}}" ${CONTAINER_NAME} | grep -q "healthy"; do sleep 5; done' || {
                                echo "⚠️  Container health check timeout"
                                docker logs ${CONTAINER_NAME}
                            }
                            
                            echo "✅ Container deployed successfully"
                            docker ps | grep ${CONTAINER_NAME}
                        """
                        
                    } catch (Exception e) {
                        error "❌ Deployment failed: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Post-Deploy Verification') {
            steps {
                script {
                    echo "✅ Running post-deployment verification..."
                    try {
                        sh """
                            # Wait a bit for the application to fully start
                            sleep 10
                            
                            # Check if application is responding
                            curl -f http://localhost:${APP_PORT} || {
                                echo "❌ Application health check failed"
                                docker logs ${CONTAINER_NAME}
                                exit 1
                            }
                            
                            echo "✅ Application is responding correctly"
                            echo "🌟 Deployment completed successfully!"
                            echo "📱 Application URL: http://localhost:${APP_PORT}"
                        """
                    } catch (Exception e) {
                        error "❌ Post-deployment verification failed: ${e.getMessage()}"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "🏁 Pipeline execution completed"
                
                // Clean up Docker images to save space
                sh """
                    docker image prune -f || true
                    docker builder prune -f || true
                """
                
                // Collect all artifacts for email
                def artifactList = []
                if (fileExists('trivy-fs-report.txt')) artifactList.add('trivy-fs-report.txt')
                if (fileExists('trivy-docker-report.txt')) artifactList.add('trivy-docker-report.txt')
                if (fileExists('dependency-check-report.html')) artifactList.add('dependency-check-report.html')
                
                def attachments = artifactList.join(',')
                
                emailext (
                    attachLog: true,
                    subject: "🚀 ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ${currentBuild.result ?: 'SUCCESS'}",
                    body: """
                        <html>
                        <head>
                            <style>
                                body { font-family: Arial, sans-serif; margin: 20px; }
                                .header { background-color: #2E86AB; color: white; padding: 15px; border-radius: 5px; }
                                .success { background-color: #28a745; color: white; padding: 10px; border-radius: 5px; margin: 10px 0; }
                                .warning { background-color: #ffc107; color: black; padding: 10px; border-radius: 5px; margin: 10px 0; }
                                .failure { background-color: #dc3545; color: white; padding: 10px; border-radius: 5px; margin: 10px 0; }
                                .info { background-color: #17a2b8; color: white; padding: 10px; border-radius: 5px; margin: 10px 0; }
                                .section { margin: 15px 0; padding: 10px; border-left: 4px solid #007bff; }
                                table { width: 100%; border-collapse: collapse; margin: 10px 0; }
                                th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
                                th { background-color: #f2f2f2; }
                            </style>
                        </head>
                        <body>
                            <div class="header">
                                <h2>🚀 DevOps Pipeline Report</h2>
                                <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                            </div>
                            
                            <div class="${currentBuild.result == 'SUCCESS' ? 'success' : currentBuild.result == 'UNSTABLE' ? 'warning' : 'failure'}">
                                <h3>Build Status: ${currentBuild.result ?: 'SUCCESS'}</h3>
                            </div>
                            
                            <div class="section">
                                <h3>📊 Build Information</h3>
                                <table>
                                    <tr><th>Build Number</th><td>${env.BUILD_NUMBER}</td></tr>
                                    <tr><th>Build URL</th><td><a href="${env.BUILD_URL}">${env.BUILD_URL}</a></td></tr>
                                    <tr><th>Git Branch</th><td>${env.GIT_BRANCH ?: 'master'}</td></tr>
                                    <tr><th>Git Commit</th><td>${env.GIT_COMMIT ?: 'N/A'}</td></tr>
                                    <tr><th>Build Duration</th><td>${currentBuild.durationString}</td></tr>
                                    <tr><th>Docker Image</th><td>${DOCKER_REGISTRY_USER}/${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}</td></tr>
                                </table>
                            </div>
                            
                            <div class="section">
                                <h3>🔧 Pipeline Stages</h3>
                                <p>All configured stages have been executed. Check the Jenkins console output for detailed logs.</p>
                            </div>
                            
                            <div class="info">
                                <p><strong>📱 Application URL:</strong> http://localhost:${APP_PORT}</p>
                                <p><strong>📋 Attachments:</strong> Security scan reports and build logs are attached.</p>
                            </div>
                            
                            <div class="section">
                                <p><em>This is an automated message from Jenkins CI/CD Pipeline</em></p>
                                <p><strong>Timestamp:</strong> ${new Date()}</p>
                            </div>
                        </body>
                        </html>
                    """,
                    to: 'ishaanpathak94@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: attachments ?: ''
                )
            }
        }
        
        success {
            echo "🎉 Pipeline completed successfully!"
        }
        
        failure {
            echo "❌ Pipeline failed. Check the logs for details."
            // Additional failure handling can be added here
        }
        
        unstable {
            echo "⚠️  Pipeline completed with warnings. Some non-critical steps failed."
        }
        
        cleanup {
            echo "🧹 Performing cleanup..."
            // Additional cleanup steps can be added here
        }
    }
}
