pipeline {
    // Executes on the master node where you mounted the docker socket
    agent any

    environment {
        CHARTMUSEUM_CREDS = credentials('Jenkins-github-pat')
        CHARTMUSEUM_URL   = 'http://192.168.122.154'
        CHART_DIR         = 'charts/my-web-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Lint & Validate') {
            steps {
                echo "Executing Helm Linting directly from local host Docker cache..."
                // Runs standard docker run. Since you pulled alpine/helm:latest, it runs instantly.
                sh "docker run --rm -v \$(pwd):/apps -w /apps alpine/helm:latest lint ${CHART_DIR}"
            }
        }

        stage('Security Compliance Scan') {
            steps {
                echo "Executing Trivy Scanning directly from local host Docker cache..."
                // Runs Trivy from cache. Make sure you run 'docker pull aquasec/trivy:latest' on the host box too!
                sh "docker run --rm -v \$(pwd):/apps -w /apps aquasec/trivy:latest config ${CHART_DIR} --severity HIGH,CRITICAL --exit-code 1"
            }
        }

        stage('Package Chart') {
            when {
                branch 'main'
            }
            steps {
                sh "mkdir -p dist"
                sh "docker run --rm -v \$(pwd):/apps -w /apps alpine/helm:latest package ${CHART_DIR} --destination dist/"
            }
        }

        stage('Publish to Chartmuseum') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    CHART_FILE=$(ls dist/*.tgz)
                    echo "Uploading production release ${CHART_FILE} to internal Chartmuseum..."
                    
                    # Make sure you run 'docker pull curlimages/curl:latest' on your host box as well
                    docker run --rm -v $(pwd):/apps -w /apps curlimages/curl:latest \
                         -u "${CHARTMUSEUM_CREDS_USR}:${CHARTMUSEUM_CREDS_PSW}" \
                         --data-binary "@${CHART_FILE}" \
                         "${CHARTMUSEUM_URL}/api/charts"
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}

