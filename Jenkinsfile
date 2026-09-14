pipeline {
    agent any

    environment {
        // --- AWS Configuration ---
        AWS_ACCOUNT_ID          = '316412036553'              // Replace with your AWS Account ID
        AWS_REGION              = 'us-east-1'                            // Replace with your AWS Region
        ECR_REGISTRY            = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        AWS_CREDENTIALS_ID      = 'AnilAWS-Cred'       // Jenkins AWS Credential ID

        // --- GitOps / CD Configuration ---
        // GITOPS_REPO_URL         = 'github.com/your-org/streamingapp-gitops.git' // Replace with GitOps Repo URL
        // GITOPS_CREDENTIALS_ID   = 'github-bot-pat'                       // Jenkins Secret Text ID for GitHub PAT
        // GITOPS_VALUES_PATH      = 'helm/streamingapp/values-prod.yaml'

        // --- Versioning Configuration ---
        IMAGE_TAG               = "${env.GIT_COMMIT.take(7)}"
        APP_PREFIX              = 'streamingapp'
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
        ansiColor('xterm')
    }

    stages {
        // stage('Helm Chart Validation') {
        //     steps {
        //         echo '=== Step 1: Linting Local Helm Chart Templates ==='
        //         sh 'helm lint helm/streamingapp'
        //     }
        // }

        stage('AWS ECR Authentication') {
            steps {
                script {
                    echo '=== Step 1 Logging into AWS ECR ==='
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "${AWS_CREDENTIALS_ID}"
                    ]]) {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                    }
                }
            }
        }

        stage('Build, Scan & Push Backend Services') {
            matrix {
                axes {
                    axis {
                        name 'SERVICE'
                        values 'auth', 'streaming', 'admin', 'chat'
                    }
                }
                stages {
                    stage('Build Image') {
                        steps {
                            script {
                                echo "=== Building ${SERVICE} Service ==="
                                sh "docker build -t ${ECR_REGISTRY}/${APP_PREFIX}-${SERVICE}:${IMAGE_TAG} -f ./backend/${SERVICE}Service/Dockerfile"
                            }
                        }
                    }
                    stage('Push to ECR') {
                        when {
                            branch 'main'
                        }
                        steps {
                            script {
                                echo "=== Pushing ${SERVICE} to ECR ==="
                                sh "docker push ${ECR_REGISTRY}/${APP_PREFIX}-${SERVICE}:${IMAGE_TAG}"
                                sh "docker tag ${ECR_REGISTRY}/${APP_PREFIX}-${SERVICE}:${IMAGE_TAG} ${ECR_REGISTRY}/${APP_PREFIX}-${SERVICE}:latest"
                                sh "docker push ${ECR_REGISTRY}/${APP_PREFIX}-${SERVICE}:latest"
                            }
                        }
                    }
                }
            }
        }

        stage('Build, Scan & Push Frontend Service') {
            stages {
                stage('Build Frontend with Build-Args') {
                    steps {
                        script {
                            echo "=== Building Frontend React Service with Build Args ==="
                            sh """
                                docker build \
                                  --build-arg REACT_APP_API_GATEWAY_URL="/api" \
                                  --build-arg REACT_APP_AUTH_URL="/api/auth" \
                                  --build-arg REACT_APP_STREAMING_URL="/api/streaming" \
                                  --build-arg REACT_APP_ADMIN_URL="/api/admin" \
                                  --build-arg REACT_APP_CHAT_SOCKET_URL="/api/chat" \
                                  -t ${ECR_REGISTRY}/${APP_PREFIX}-frontend:${IMAGE_TAG} \
                                  -f ./frontend-service/Dockerfile ./frontend-service
                            """
                        }
                    }
                }

                stage('Push Frontend to ECR') {
                    steps {
                        script {
                            echo "=== Pushing Frontend to ECR ==="
                            sh "docker push ${ECR_REGISTRY}/${APP_PREFIX}-frontend:${IMAGE_TAG}"
                            sh "docker tag ${ECR_REGISTRY}/${APP_PREFIX}-frontend:${IMAGE_TAG} ${ECR_REGISTRY}/${APP_PREFIX}-frontend:latest"
                            sh "docker push ${ECR_REGISTRY}/${APP_PREFIX}-frontend:latest"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo '=== Cleaning dangling docker images on Jenkins agent ==='
            sh "docker image prune -f --filter 'label=stage=builder'"
        }
        success {
            echo "CI Execution Succeeded! Images deployed with commit tag: ${IMAGE_TAG}"
        }
        failure {
            echo "CI Execution Failed! Check the step logs above for details."
        }
    }
}