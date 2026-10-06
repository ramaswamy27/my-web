pipeline {
    agent any

    environment {
        CHARTMUSEUM_CREDS = credentials('cred-2-conn-chartmuseum-2-push-charts')
        CHARTMUSEUM_URL   = 'http://192.168.122.154:8080'
        CHART_DIR         = 'charts/my-web' 
        
        // AUTOMATED FIX: Dynamically extracts whatever active directory Jenkins is currently using (e.g. includes the @6)
        REAL_WORKSPACE    = "${env.WORKSPACE}"
        GITHUB_CREDS = credentials('Jenkins-github-pat')
    }

    stages {
        stage('Checkout') {
            steps {
                deleteDir()
                checkout scm
            }
        }

        stage('Lint & Validate') {
            steps {
                echo "Executing Helm Linting inside active directory: ${REAL_WORKSPACE}"
                // FIXED: Mounts the volume and directs the working directory directly to the active path
                sh "docker run --rm -v jenkins_home:/var/jenkins_home -w ${REAL_WORKSPACE} alpine/helm:latest lint ${CHART_DIR}"
            }
        }

        stage('Security Compliance Scan') {
            steps {
                echo "Executing Trivy Scanning inside active directory: ${REAL_WORKSPACE}"
                sh "docker run --rm -v jenkins_home:/var/jenkins_home -w ${REAL_WORKSPACE} aquasec/trivy:latest config ${CHART_DIR} --severity HIGH,CRITICAL --exit-code 1 --timeout 15m --skip-check-update"
            }
        }

        stage('Package Chart') {
            when {
                branch 'main'
            }
            steps {
                sh "mkdir -p dist"
                sh "docker run --rm -v jenkins_home:/var/jenkins_home -w ${REAL_WORKSPACE} alpine/helm:latest package ${CHART_DIR} --destination dist/"
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
                    
                    docker run --rm -v jenkins_home:/var/jenkins_home -w ${REAL_WORKSPACE} curlimages/curl:latest \
                         -f -u "${CHARTMUSEUM_CREDS_USR}:${CHARTMUSEUM_CREDS_PSW}" \
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

