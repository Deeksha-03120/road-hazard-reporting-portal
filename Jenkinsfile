pipeline {
    agent any
    parameters {
        string(name: 'DOCKER_IMAGE', defaultValue: 'deeksha03120/road-hazard-portal', description: 'Docker Hub Repository Path')
        string(name: 'BUILD_TAG', defaultValue: "v${BUILD_NUMBER}", description: 'Image Tag')
    }
    environment {
        DOCKER_CREDS = 'docker-hub-credentials'
        KUBE_CONFIG  = 'kubeconfig-credentials'
    }
    stages {
        stage('Local Build Check') {
            steps {
                // Using 'bat' for Windows command execution
                bat "docker build -t %DOCKER_IMAGE%:%BUILD_TAG% ."
                bat "docker tag %DOCKER_IMAGE%:%BUILD_TAG% %DOCKER_IMAGE%:latest"
            }
        }
        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS}", usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    // Windows Docker Login authentication routing
                    bat "echo %PASS% | docker login -u %USER% --password-stdin"
                    bat "docker push %DOCKER_IMAGE%:%BUILD_TAG%"
                    bat "docker push %DOCKER_IMAGE%:latest"
                }
            }
        }
        stage('Deploy & Rollout Watch') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CONFIG}", variable: 'KUBECONFIG_FILE')]) {
                    script {
                        // Update deployment manifest dynamically on Windows using PowerShell syntax via batch
                        bat "powershell -Command \"(Get-Content deployment.yaml) -replace 'YOUR_DOCKERHUB_USERNAME/road-hazard-portal:latest', '%DOCKER_IMAGE%:%BUILD_TAG%' | Set-Content deployment.yaml\""
                        
                        // Deploy using the target Kubeconfig profile
                        bat "kubectl apply -f deployment.yaml --kubeconfig=\"%KUBECONFIG_FILE%\""
                        
                        try {
                            bat "kubectl rollout status deployment/road-hazard-portal --timeout=45s --kubeconfig=\"%KUBECONFIG_FILE%\""
                            echo "Application Rollout marked: FUNCTIONAL & SUCCESSFUL."
                        } catch (Exception e) {
                            echo "!! ROLLOUT DETECTED AS CRITICAL OR FAILING !! Initiating Automated Fallback..."
                            bat "kubectl rollout undo deployment/road-hazard-portal --kubeconfig=\"%KUBECONFIG_FILE%\""
                            bat "kubectl rollout status deployment/road-hazard-portal --kubeconfig=\"%KUBECONFIG_FILE%\""
                            error("Pipeline aborted: Rollout failed, cluster reverted back to previous deployment instance.")
                        }
                    }
                }
            }
        }
    }
}
