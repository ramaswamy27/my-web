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
                // Safely pulls down the code from GitHub
                checkout scm
            }
        }

        stage('Lint & Validate') {
            agent {
              docker {
                image 'alpine/helm:latest'
                reuseNode same
              }
            }
            steps {
                // This stage runs on BOTH Pull Requests and Main branch commits to catch structural errors
                sh "helm lint ${CHART_DIR}"
            }
        }

  	stage('Security Complaince Scan') {
	  agent {
	    docker {
	      image 'aquasec/trivy:latest'
	      reuseNode true
	    }
	  }
	  steps {
	    echo "Scanning Helm configurations for misconfigurations and secrets"
	    sh "trivy config ${CHART_DIR} --severity HIGH,CRITICAL --exit-code 1"
	  }

        stage('Package Chart') {
            when {
                // DECOUPLING RULE: Only package the chart binary when changes hit the production branch
                branch 'main'
            }
            steps {
                sh '''
                    mkdir -p dist
                    helm package ${CHART_DIR} --destination dist/
                '''
            }
        }

        stage('Publish to Chartmuseum') {
            when {
                // DECOUPLING RULE: Only publish to your in-house registry after a PR is approved and merged into main
                branch 'main'
            }
            steps {
                sh '''
                    CHART_FILE=$(ls dist/*.tgz)
                    echo "Uploading production release ${CHART_FILE} to internal Chartmuseum..."
                    
                    # POSTs the packaged binary directly to Chartmuseum's standard REST API
                    curl -u "${CHARTMUSEUM_CREDS_USR}:${CHARTMUSEUM_CREDS_PSW}" \
                         --data-binary "@${CHART_FILE}" \
                         "${CHARTMUSEUM_URL}/api/charts"
                '''
            }
        }
    }

    post {
        always {
            // Workspace Hygiene: Wipe local binary artifacts so they don't consume Jenkins worker storage
            cleanWs()
        }
    }
}

