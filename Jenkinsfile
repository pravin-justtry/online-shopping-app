pipeline {
    agent any
    
    environment {
        HARBOR_REGISTRY = '192.168.1.11:8001'
        HARBOR_PROJECT  = 'web3-apps'
        APP_NAME        = 'online-shopping'
        HARBOR_CREDS_ID = 'harbor'
        GIT_SHORT_HASH  = ''
    }
    
    stages {
        stage('Preparation') {
            steps {
                script {
                    GIT_SHORT_HASH = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
                    // Define strict explicit tag scheme
                    env.IMAGE_TAG = "${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${APP_NAME}:${BUILD_NUMBER}-${GIT_SHORT_HASH}"
                }
            }
        }
        
        stage('Docker Build & Tag') {
            steps {
                sh "docker build -t ${env.IMAGE_TAG} ."
            }
        }
        
        stage('Docker Push & Scan Injection') {
            steps {
                script {
                    // Authenticate and Push using isolated robot system account
                    withCredentials([usernamePassword(credentialsId: env.HARBOR_CREDS_ID, usernameVariable: 'ROBOT_USER', passwordVariable: 'ROBOT_PASS')]) {
                        sh "echo '${ROBOT_PASS}' | docker login ${env.HARBOR_REGISTRY} -u '${ROBOT_USER}' --password-stdin"
                        sh "docker push ${env.IMAGE_TAG}"
                    }
                }
            }
        }
        
        stage('Harbor Security Gates (Trivy Verification)') {
            steps {
                script {
                    echo "Awaiting completion of asynchronous Trivy scan on Harbor..."
                    sleep 15 // Give Trivy time to process the image push event
                    
                    withCredentials([usernamePassword(credentialsId: env.HARBOR_CREDS_ID, usernameVariable: 'ROBOT_USER', passwordVariable: 'ROBOT_PASS')]) {
                        // Query the Harbor Core API for the image report metrics
                        def apiEndpoint = "http://${env.HARBOR_REGISTRY}/api/v2.0/projects/${env.HARBOR_PROJECT}/repositories/${env.APP_NAME}/artifacts/${env.BUILD_NUMBER}-${env.GIT_SHORT_HASH}/additions/vulnerabilities"
                        
                        def response = sh(
                            returnStdout: true,
                            script: "curl -s -u '${ROBOT_USER}:${ROBOT_PASS}' -H 'Accept: application/vnd.security.vulnerability.report; version=1.1' '${apiEndpoint}'"
                        ).trim()
                        
                        echo "Raw Vulnerability Response Fragment: ${response}"
                        
                        // Parse critical severity counts
                        if (response.contains('"Critical":') || response.contains('"severity": "Critical"')) {
                            error "PIPELINE ABORTED: Trivy scanner detected Critical severity vulnerabilities within ${env.IMAGE_TAG}."
                        } else {
                            echo "Security Gate Passed: Zero critical risks discovered."
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Clean up workspace image footprints
            sh "docker rmi ${env.IMAGE_TAG} || true"
        }
    }
}
