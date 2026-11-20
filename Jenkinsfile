pipeline {
    agent any

    // Node.js from the Jenkins NodeJS plugin:
    // Manage Jenkins → Tools → NodeJS installations → Name "node22", version 22.x
    tools {
        nodejs 'node22'
    }

    environment {
        ENV_NAME    = 'production'

        REPO_URL    = 'https://github.com/ankittyagi140/euroasia_lims_fe.git'

        DOCKERFILE  = 'Dockerfile.prod'
        IMAGE_NAME  = 'lims-fe-prod'
        CONTAINER   = 'lims-fe-prod'
        PORT        = '3002'

        // The app is built here in the workspace, which persists between builds:
        // node_modules and the webpack/babel cache (node_modules/.cache) are reused.
        WEBPACK_FS_CACHE = '1'
        HUSKY            = '0'
    }

    options {
        // Per-line timestamps in the console, to see how long each step takes
        timestamps()
        // Prevent concurrent builds for production
        disableConcurrentBuilds()
        // Add timeout to prevent hanging builds
        timeout(time: 45, unit: 'MINUTES')
        // The pipeline checks out the app repo itself (release branch) below.
        skipDefaultCheckout(true)
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    echo "📥 Checking out code from repository (release branch)..."
                    git branch: 'release',
                        credentialsId: 'github-pat',
                        url: env.REPO_URL
                    echo "✅ Code checkout completed from release branch"
                }
            }
        }

        stage('Validate Environment') {
            steps {
                script {
                    echo "🔍 Validating production environment configuration..."
                    
                    // Check if .env.prod exists (now allowed in repo)
                    sh """
                        if [ ! -f .env.prod ]; then
                            echo "❌ ERROR: .env.prod file not found!"
                            echo "   Make sure .env.prod is committed to the repository"
                            exit 1
                        fi
                        
                        echo "✅ .env.prod file found"
                        
                        # Validate required environment variables in .env.prod
                        if ! grep -q "REACT_APP_API_URL" .env.prod; then
                            echo "❌ ERROR: REACT_APP_API_URL not found in .env.prod"
                            exit 1
                        fi
                        
                        if ! grep -q "REACT_APP_AZURE_AD_TENANT_ID" .env.prod; then
                            echo "❌ ERROR: REACT_APP_AZURE_AD_TENANT_ID not found in .env.prod"
                            exit 1
                        fi
                        
                        if ! grep -q "REACT_APP_AZURE_AD_CLIENT_ID" .env.prod; then
                            echo "❌ ERROR: REACT_APP_AZURE_AD_CLIENT_ID not found in .env.prod"
                            exit 1
                        fi
                        
                        echo "✅ All required environment variables found in .env.prod"
                    """
                }
            }
        }

        stage('Install') {
            steps {
                sh '''
                    set -e
                    node -v && npm -v

                    # Reinstall only when package-lock.json changed (or node_modules is missing).
                    LOCK_HASH=$(sha256sum package-lock.json | cut -d" " -f1)
                    if [ -d node_modules ] && [ "$(cat node_modules/.lock-hash 2>/dev/null)" = "$LOCK_HASH" ]; then
                      echo "✅ Dependencies unchanged: skipping npm ci"
                    else
                      echo "📦 package-lock.json changed: running npm ci"
                      npm ci --no-audit --no-fund
                      echo "$LOCK_HASH" > node_modules/.lock-hash
                    fi
                '''
            }
        }

        stage('Build') {
            steps {
                script {
                    echo "🔨 Building production app and image..."

                    sh """
                        set -e

                        echo "🔨 Building app (webpack, cached; reads .env.prod)"
                        NODE_ENV=production npm run build:prod

                        # Image = nginx + the prebuilt dist/ (a few seconds).
                        echo "🐳 Building runtime image: ${IMAGE_NAME}"
                        DOCKER_BUILDKIT=0 docker build -f "${DOCKERFILE}" -t "${IMAGE_NAME}:latest" .

                        # Tag with build number for versioning
                        docker tag "${IMAGE_NAME}:latest" "${IMAGE_NAME}:${BUILD_NUMBER}"

                        echo "✅ Docker image built successfully"
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    echo "🚀 Deploying to production..."
                    sh """
                        set -e
                        
                        echo "🛑 Stopping and removing existing container (if running)"
                        docker stop "${CONTAINER}" || true
                        docker rm "${CONTAINER}" || true
                        
                        echo "🔍 Checking if port ${PORT} is available"
                        if lsof -Pi :${PORT} -sTCP:LISTEN -t >/dev/null 2>&1 ; then
                            echo "⚠️  WARNING: Port ${PORT} is already in use!"
                            echo "Attempting to free the port..."
                            # Try to find and stop the process using the port
                            lsof -ti:${PORT} | xargs kill -9 || true
                            sleep 2
                        fi
                        
                        echo "🐳 Starting new production container"
                        # nginx:alpine has wget (busybox), not curl — match stage-style container (no false unhealthy)
                        docker run -d \
                          --name "${CONTAINER}" \
                          -p "${PORT}:80" \
                          --restart unless-stopped \
                          --health-cmd="wget -q -O /dev/null http://127.0.0.1/health || exit 1" \
                          --health-interval=30s \
                          --health-timeout=10s \
                          --health-retries=3 \
                          "${IMAGE_NAME}:latest"
                        
                        echo "⏳ Waiting for container to be healthy..."
                        sleep 5
                        
                        # Verify container is running
                        if ! docker ps | grep -q "${CONTAINER}"; then
                            echo "❌ ERROR: Container failed to start!"
                            docker logs "${CONTAINER}" || true
                            exit 1
                        fi
                        
                        echo "✅ Container is running"
                        
                        # Health check
                        echo "🏥 Performing health check..."
                        sleep 3
                        # Jenkins host usually has curl; probe published port on localhost
                        HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:${PORT}/health || echo "000")
                        
                        if [ "\$HTTP_CODE" = "200" ]; then
                            echo "✅ Health check passed (HTTP 200)"
                        else
                            echo "⚠️  WARNING: Health check returned HTTP \$HTTP_CODE"
                            echo "Container logs:"
                            docker logs "${CONTAINER}" --tail 50 || true
                        fi
                        
                        echo "🚀 Production deployment completed successfully"
                        echo "📍 Application available at: http://localhost:${PORT}"
                    """
                }
            }
        }

        stage('Cleanup') {
            steps {
                script {
                    echo "🧹 Cleaning up old Docker images..."
                    sh """
                        # Keep only the last 5 versions
                        docker images "${IMAGE_NAME}" --format "{{.Tag}}" | grep -E "^[0-9]+\$" | sort -rn | tail -n +6 | while read tag; do
                            echo "Removing old image: ${IMAGE_NAME}:\$tag"
                            docker rmi "${IMAGE_NAME}:\$tag" || true
                        done
                        
                        # Remove dangling images (and the old in-Docker builder image, if any)
                        docker rmi "${IMAGE_NAME}:builder" 2>/dev/null || true
                        docker image prune -f
                        
                        echo "✅ Cleanup completed"
                    """
                }
            }
        }
    }

    post {
        success {
            script {
                echo "✅ PRODUCTION deployment successful!"
                echo "📍 Application URL: http://localhost:${PORT}"
                echo "🏥 Health Check: http://localhost:${PORT}/health"
                echo "📦 Image: ${IMAGE_NAME}:${BUILD_NUMBER}"
            }
        }
        failure {
            script {
                echo "❌ PRODUCTION deployment failed!"
                echo "📋 Checking container logs..."
                sh """
                    docker logs "${CONTAINER}" --tail 100 || true
                """
            }
        }
        always {
            script {
                echo "📊 Deployment Summary:"
                echo "   Environment: ${ENV_NAME}"
                echo "   Build Number: ${BUILD_NUMBER}"
                echo "   Container: ${CONTAINER}"
                echo "   Port: ${PORT}"
            }
        }
    }
}
