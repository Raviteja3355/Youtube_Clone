pipeline {
    agent any

    environment {
        NODE_ENV = 'production'
        APP_PORT = '3000'
        BUILD_DIR = 'build'
        PATH = "/usr/local/node-v18.17.0-linux-x64/bin:${PATH}"
    }

    stages {
        stage('Install Node.js') {
            steps {
                echo "========== Checking and Installing Node.js =========="
                sh '''
                    # Check if Node.js is installed
                    if command -v node &> /dev/null; then
                        echo "✅ Node.js is already installed"
                        node --version
                        npm --version
                    else
                        echo "⚠️ Node.js not found, installing..."
                        cd /tmp
                        wget https://nodejs.org/dist/v18.17.0/node-v18.17.0-linux-x64.tar.xz
                        tar -xf node-v18.17.0-linux-x64.tar.xz -C /usr/local
                        /usr/local/node-v18.17.0-linux-x64/bin/node --version
                        /usr/local/node-v18.17.0-linux-x64/bin/npm --version
                    fi
                '''
            }
        }

        stage('Checkout') {
            steps {
                echo "========== Checking out source code =========="
                checkout scm
                sh 'echo "Branch: $GIT_BRANCH"'
            }
        }

        stage('Install Dependencies') {
            steps {
                echo "========== Installing Node dependencies =========="
                sh '''
                    export PATH=/usr/local/node-v18.17.0-linux-x64/bin:$PATH
                    echo "Node version: $(node --version)"
                    echo "npm version: $(npm --version)"
                    npm cache clean --force
                    npm install
                    # Install serve locally for deployment
                    npm install serve --save-dev
                '''
            }
        }

        stage('Build') {
            steps {
                echo "========== Building React Application =========="
                sh '''
                    export PATH=/usr/local/node-v18.17.0-linux-x64/bin:$PATH
                    # Update browserslist database
                    npx update-browserslist-db@latest --update
                    # Build React app
                    CI=false npm run build
                    echo "✅ Build completed successfully!"
                    ls -lah build/ | head -20
                '''
            }
        }

        stage('Test Build') {
            steps {
                echo "========== Testing Build Output =========="
                sh '''
                    if [ -d "build" ]; then
                        echo "✅ Build directory exists"
                        if [ -f "build/index.html" ]; then
                            echo "✅ index.html found"
                        else
                            echo "❌ index.html not found"
                            exit 1
                        fi
                    else
                        echo "❌ Build directory does not exist"
                        exit 1
                    fi
                '''
            }
        }

        stage('Stop Previous Instance') {
            steps {
                echo "========== Stopping any previous running instance =========="
                sh '''
                    if lsof -Pi :${APP_PORT} -sTCP:LISTEN -t >/dev/null 2>&1; then
                        echo "Stopping application running on port ${APP_PORT}..."
                        lsof -ti:${APP_PORT} | xargs kill -9 || true
                        sleep 2
                    else
                        echo "No process running on port ${APP_PORT}"
                    fi
                '''
            }
        }

        stage('Deploy & Start Application') {
            steps {
                echo "========== Deploying and starting application =========="
                sh '''
                    export PATH=/usr/local/node-v18.17.0-linux-x64/bin:$PATH
                    cd ${WORKSPACE}/${BUILD_DIR}
                    echo "Starting application on port ${APP_PORT}..."
                    nohup npx serve -s . -l ${APP_PORT} > ${WORKSPACE}/app.log 2>&1 &
                    APP_PID=$!
                    echo $APP_PID > ${WORKSPACE}/app.pid
                    sleep 3
                    echo "Application PID: $APP_PID"
                '''
            }
        }

        stage('Health Check') {
            steps {
                echo "========== Performing health check =========="
                sh '''
                    echo "Waiting for application to be ready..."
                    sleep 5
                    
                    for i in {1..30}; do
                        if curl -s http://localhost:${APP_PORT} > /dev/null; then
                            echo "✅ Application is running successfully!"
                            echo "Application URL: http://$(hostname -I | awk '{print $1}'):${APP_PORT}"
                            exit 0
                        fi
                        echo "Attempt $i/30: Waiting for application..."
                        sleep 2
                    done
                    
                    echo "❌ Application failed to start"
                    cat ${WORKSPACE}/app.log
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo "========== BUILD SUCCESSFUL =========="
            sh '''
                echo ""
                echo "✅ YouTube Clone Application Build & Deploy Successful!"
                echo "=========================================="
                echo "Application is running on: http://$(hostname -I | awk '{print $1}'):${APP_PORT}"
                echo "Build Directory: ${WORKSPACE}/${BUILD_DIR}"
                echo "Log File: ${WORKSPACE}/app.log"
                echo "=========================================="
                echo ""
            '''
            archiveArtifacts artifacts: 'build/**/*', followSymlinks: false, allowEmptyArchive: false
        }

        failure {
            echo "========== BUILD FAILED =========="
            sh '''
                echo "❌ Build or deployment failed"
                echo "Last 50 lines of build log:"
                tail -50 ${WORKSPACE}/app.log || echo "No log file found"
            '''
        }

        always {
            echo "========== CLEANING UP =========="
            sh '''
                echo "Workspace: ${WORKSPACE}"
            '''
        }
    }
}
