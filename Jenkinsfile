pipeline {
    agent any
    parameters {
        string(name: 'DOCKER_IMAGE', defaultValue: 'YOUR_DOCKERHUB_USERNAME/road-hazard-portal', description: 'Docker Repo')
        string(name: 'BUILD_TAG', defaultValue: "v${BUILD_NUMBER}", description: 'Image Tag')
    }
    environment {
        DOCKER_CREDS = 'docker-hub-credentials'
    }
    stages {
        stage('Local Build Check') {
            steps {
                sh "docker build -t ${params.DOCKER_IMAGE}:${params.BUILD_TAG} ."
                sh "docker tag ${params.DOCKER_IMAGE}:${params.BUILD_TAG} ${params.DOCKER_IMAGE}:latest"
            }
        }
        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS}", usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo '${PASS}' | docker login -u '${USER}' --password-stdin"
                    sh "docker push ${params.DOCKER_IMAGE}:${params.BUILD_TAG}"
                    sh "docker push ${params.DOCKER_IMAGE}:latest"
                }
            }
        }
        stage('Deploy & Rollout Watch') {
            steps {
                sh "sed -i 's|24mis0129/road-hazard-portal:latest|${params.DOCKER_IMAGE}:${params.BUILD_TAG}|g' deployment.yaml"
                sh "kubectl apply -f deployment.yaml"
                script {
                    try {
                        sh "kubectl rollout status deployment/road-hazard-portal --timeout=30s"
                    } catch (Exception e) {
                        echo "Deployment failed! Pulling back transaction..."
                        sh "kubectl rollout undo deployment/road-hazard-portal"
                        error("Rollback completed safely.")
                    }
                }
            }
        }
    }
}
