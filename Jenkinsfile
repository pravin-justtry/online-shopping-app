pipeline {
    agent any
    
    environment {
        HARBOR_REGISTRY = '127.0.0.1:8001'
        HARBOR_PROJECT  = 'web3-apps'
        APP_NAME        = 'online-shopping'
        HARBOR_CREDS_ID = 'harbor'
    }
    
    stages {
        stage('Preparation') {
            steps {
                script {
                    // Extract git short hash directly into target execution parameters
                    def gitHash = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
                    env.IMAGE_TAG = "${BUILD_NUMBER}-${gitHash}"
                    env.FULL_IMAGE_NAME = "${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${APP_NAME}:${env.IMAGE_TAG}"
                }
            }
        }
        
        stage('Docker Build & Tag') {
            steps {
                sh "docker build -t ${env.FULL_IMAGE_NAME} ."
            }
        }
        
        stage('Docker Push & Scan Injection') {
            steps {
                script {
                    // Authenticate and Push using isolated robot credentials
                    withCredentials([usernamePassword(credentialsId: env.HARBOR_CREDS_ID, usernameVariable: 'ROBOT_USER', passwordVariable: 'ROBOT_PASS')]) {
                        sh "echo '${ROBOT_PASS}' | docker login ${env.HARBOR_REGISTRY} -u '${ROBOT_USER}' --password-stdin"
                        sh "docker push ${env.FULL_IMAGE_NAME}"
                    }
                }
            }
        }
        
        stage('Harbor Security Gates (Trivy Verification)') {
            steps {
                script {
                    echo "Awaiting completion of asynchronous Trivy scan on Harbor..."
                    sleep 15 // Give Trivy time to process the image push event
                    
                    withCredentials([usernamePassword(credentialsId: env.HARBOR_CREDS_ID, usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                        // Correctly maps clean API endpoint coordinates via V2.0 specification
                        def apiEndpoint = "http://${env.HARBOR_REGISTRY}/api/v2.0/projects/${env.HARBOR_PROJECT}/repositories/${env.APP_NAME}/artifacts/${env.IMAGE_TAG}/additions/vulnerabilities"
                        
                        // FIXED: Replaced incorrect ROBOT variables with matching REG_USER/REG_PASS credentials 
                        def response = sh(
                            returnStdout: true,
                            script: "curl -s -u '${REG_USER}:${REG_PASS}' -H 'Accept: application/vnd.security.vulnerability.report; version=1.1' '${apiEndpoint}'"
                        ).trim()
                        
                        echo "Raw Vulnerability Response: ${response}"
                        
                        // Parse critical severity counts or direct matches returned from Trivy database map
                        if (response.contains('"Critical":') || response.contains('"severity": "Critical"')) {
                            error "PIPELINE ABORTED: Trivy scanner detected Critical severity vulnerabilities within ${env.FULL_IMAGE_NAME}."
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
            // Clean up workspace local image footprint to optimize node memory space
            sh "docker rmi ${env.FULL_IMAGE_NAME} || true"
        }
    }
}
