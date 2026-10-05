pipeline {
    agent any

    environment {
        // Points to the secure user/pass credential ID you created globally in Jenkins
        CHARTMUSEUM_CREDS = credentials('Jenkins-github-pat')
        
        // Update this to your exact internal Chartmuseum endpoint URL
        CHARTMUSEUM_URL   = 'http://192.168.122.154'
        
        // The path to your Helm chart source subfolder
        CHART_DIR         = 'charts/my-web' 
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
        }
    }

        stage('Lint & Validate') {
            agent {
                dockerContainer { 
                    image 'alpine/helm:latest'
                }
            }
            steps {
                sh "docker run --rm -v \$(pwd):/apps -w /apps alpine/helm:latest helm lint ${CHART_DIR}"
            }
        }

        stage('Security Compliance Scan') {
            agent {
                dockerContainer { 
                    image 'aquasec/trivy:latest' 
                }
            }
            steps {
                echo "Scanning Helm configurations for misconfigurations and secrets..."
                sh "trivy config ${CHART_DIR} --severity HIGH,CRITICAL --exit-code 1"
            }
        }

        stage('Package Chart') {
            when {
                branch 'main' 
            }
            agent {
                dockerContainer { 
                    image 'alpine/helm:latest'
                }
            }
            steps {
                sh "mkdir -p dist && helm package ${CHART_DIR} --destination dist/"
            }
        }

        stage('Publish to Chartmuseum') {
            when {
                branch 'main' 
            }
            agent {
                dockerContainer { 
                    image 'curlimages/curl:latest' 
                }
            }
            steps {
                sh '''
                    CHART_FILE=$(ls dist/*.tgz)
                    echo "Uploading production release ${CHART_FILE} to internal Chartmuseum..."
                    
                    curl -u "${CHARTMUSEUM_CREDS_USR}:${CHARTMUSEUM_CREDS_PSW}" \
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
