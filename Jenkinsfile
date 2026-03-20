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
                    # Update Browserslist database
                    npx update-browserslist-db@latest --update
                    # Build React app (ignore CI warnings temporarily)
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
                    if [ -d "${BUILD_DIR}" ]; then
                        echo "✅ Build directory exists"
                        if [ -f "${BUILD_DIR}/index.html" ]; then
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
                    if [ -f "${WORKSPACE}/app.pid" ]; then
                        PREV_PID=$(cat ${WORKSPACE}/app.pid)
                        if ps -p $PREV_PID > /dev/null; then
                            echo "Stopping previous application process $PREV_PID..."
                            kill -9 $PREV_PID
                            sleep 2
                        fi
                        rm -f ${WORKSPACE}/app.pid
                    else
                        echo "No previous PID file found"
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
                    # Run serve in non-interactive background mode
                    nohup npx serve -s . -l ${APP_PORT} --no-clipboard > ${WORKSPACE}/app.log 2>&1 &
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
                echo "Application running on: http://$(hostname -I | awk '{print $1}'):${APP_PORT}"
                echo "Build Directory: ${WORKSPACE}/${BUILD_DIR}"
                echo "Log File: ${WORKSPACE}/app.log"
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
            sh 'echo "Workspace: ${WORKSPACE}"'
        }
    }
}
