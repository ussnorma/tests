pipeline {
    agent { label 'node1' }
    environment {
        DOCKERHUB_CREDENTIALS=credentials('ussnorma-dockerhub')
    }
    stages {
        stage('Clone repo') {
            steps {
                git url: 'https://github.com/ussnorma/tests.git', branch: 'main'
                sh "ls -la"
            }
        }

        stage('Validate Dockerfile') {
            steps {
                sh 'hadolint Dockerfile'
            }
        }

        stage('Build and test docker image') {
            steps {
                sh "docker build -t ussnorma/golang-web-app:v3 ."
                sh "docker run -d --name golang-web-app3 -p 3000:3000 ussnorma/golang-web-app:v3"
                sh "sleep 10"

                timeout(time: 5, unit: 'SECONDS') {
                    retry(2) {
                        script {
                            try {
                                sh 'curl http://localhost:3000' 
                            } catch (Exception e) {
                                error "http/app:3000 failed"
                            }
                        }
                    }
                }

                sh "docker stop golang-web-app3"
                sh "docker rm \$(docker ps -a | grep golang-web-app3 | awk '{print \$1}')"
            }
        }

        stage('Push image to dockerhub') {
            steps {
                withCredentials([
                        usernamePassword(
                            credentialsId: 'ussnorma-dockerhub',
                            usernameVariable: 'REGISTRY_USERNAME',
                            passwordVariable: 'REGISTRY_PASSWORD'
                        )
                    ]) {
                sh "docker login -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD"
                sh "docker push ussnorma/golang-web-app:v3"
                sh "docker rmi -f ussnorma/golang-web-app:v3"
                }
            }
        }

        stage('Deploy to pre-prod') {
            steps {
                sh "kubectl apply -f pre-prod.yaml --namespace pre-prod"

                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        try {
                            sh 'kubectl rollout status deployment/myapp --namespace pre-prod'
                        } catch (Exception e) {
                            error "Deployment to Pre-Prod failed"
                        }

                        echo 'Deployment to pre-prod is successful'
                    }
                }
            }
        }

        stage('Delete from pre-prod') {
            steps {
                sh "kubectl delete -f pre-prod.yaml --namespace pre-prod"
            }
        }
        stage('Deploy to prod') {
            steps {
                sh "kubectl apply -f prod.yaml --namespace prod"

                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        try {
                            sh 'kubectl rollout status deployment/myapp --namespace prod'
                        } catch (Exception e) {
                            error "Deployment to prod failed"
                        }
                    }
                }
            }
        }
        
    }

    post {
            success {
                slackSend (channel: 'tests', color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        
            }
            failure {
                slackSend (channel: 'tests', color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
            }
    }

}
