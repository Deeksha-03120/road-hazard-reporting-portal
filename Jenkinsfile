pipeline {
    agent any
    parameters {
        string(name: 'DOCKER_IMAGE', defaultValue: '24mis0129/road-hazard-portal', description: 'Docker Hub Repository Path')
        string(name: 'BUILD_TAG', defaultValue: "v${BUILD_NUMBER}", description: 'Image Tag')
    }
    environment {
        DOCKER_CREDS = 'docker-hub-credentials'
        KUBE_CONFIG  = 'kubeconfig-credentials'
        DOCKER_HOST  = 'tcp://127.0.0.1:2375'
    }
    stages {
        stage('Local Build Check') {
            steps {
                bat "docker build -t %DOCKER_IMAGE%:%BUILD_TAG% ."
                bat "docker tag %DOCKER_IMAGE%:%BUILD_TAG% %DOCKER_IMAGE%:latest"
            }
        }
        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS}", usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    // Injecting absolute index registry URL mappings dynamically into the workspace run stream
                    bat """
                    powershell -Command "New-Item -Path . -Name 'docker-config' -ItemType 'Directory' -Force | Out-Null"
                    powershell -Command "\$Bytes = [System.Text.Encoding]::UTF8.GetBytes('%USER%:%PASS%'); \$Base64 = [Convert]::ToBase64String(\$Bytes); \$Json = @{ auths = @{ 'https://docker.io' = @{ auth = \$Base64 } } } | ConvertTo-Json -Depth 4; Set-Content -Path 'docker-config\\config.json' -Value \$Json -Force"
                    """
                    bat "docker --config docker-config push %DOCKER_IMAGE%:%BUILD_TAG%"
                    bat "docker --config docker-config push %DOCKER_IMAGE%:latest"
                }
            }
        }
        stage('Deploy & Rollout Watch') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CONFIG}", variable: 'KUBECONFIG_FILE')]) {
                    script {
                        bat "powershell -Command \"(Get-Content deployment.yaml) -replace 'YOUR_DOCKERHUB_USERNAME/road-hazard-portal:latest', '%DOCKER_IMAGE%:%BUILD_TAG%' | Set-Content deployment.yaml\""
                        bat "kubectl apply -f deployment.yaml --kubeconfig=\"%KUBECONFIG_FILE%\""
                        try {
                            echo "Monitoring rollout status stability indicators..."
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
    post {
        always {
            bat "powershell -Command \"if (Test-Path 'docker-config') { Remove-Item -Path 'docker-config' -Recurse -Force }\""
        }
    }
}
