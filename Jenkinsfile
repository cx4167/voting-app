pipeline {
    agent any
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    
    environment {
        DOCKER_HUB_REPO = 'denish952'
        BUILD_NUM = "${BUILD_NUMBER}"
        DOCKER_BUILDKIT = '1'
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                echo 'Pulling Code from GitHub'
                checkout scm
                sh 'git log -1 --oneline'
                sh 'ls -la'
            }
        }
        
        stage('Build All Images in Parallel') {
            parallel {
                stage('Build Vote') {
                    steps {
                        echo 'Building Vote Service'
                        sh """
                            docker build \
                              -t ${DOCKER_HUB_REPO}/voting-app-vote:${BUILD_NUM} \
                              -t ${DOCKER_HUB_REPO}/voting-app-vote:latest \
                              -f vote/Dockerfile vote/
                            docker images | grep voting-app-vote
                        """
                    }
                }
                stage('Build Worker') {
                    steps {
                        echo 'Building Worker Service'
                        sh """
                            docker build \
                              -t ${DOCKER_HUB_REPO}/voting-app-worker:${BUILD_NUM} \
                              -t ${DOCKER_HUB_REPO}/voting-app-worker:latest \
                              -f worker/Dockerfile worker/
                            docker images | grep voting-app-worker
                        """
                    }
                }
                stage('Build Result') {
                    steps {
                        echo 'Building Result Service'
                        sh """
                            docker build \
                              -t ${DOCKER_HUB_REPO}/voting-app-result:${BUILD_NUM} \
                              -t ${DOCKER_HUB_REPO}/voting-app-result:latest \
                              -f result/Dockerfile result/
                            docker images | grep voting-app-result
                        """
                    }
                }
            }
        }
        
        stage('Test Images') {
            steps {
                echo 'Running Tests'
                sh """
                    docker run --rm ${DOCKER_HUB_REPO}/voting-app-vote:${BUILD_NUM} python -c "from flask import Flask; print('Vote OK')"
                    docker run --rm ${DOCKER_HUB_REPO}/voting-app-worker:${BUILD_NUM} python -c "import redis; print('Worker OK')"
                    docker run --rm ${DOCKER_HUB_REPO}/voting-app-result:${BUILD_NUM} python -c "from flask import Flask; print('Result OK')"
                """
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing Images to Docker Hub'
                withCredentials([string(credentialsId: 'dockerhub-password', variable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u denish952 --password-stdin
                        
                        docker push ${DOCKER_HUB_REPO}/voting-app-vote:${BUILD_NUM}
                        docker push ${DOCKER_HUB_REPO}/voting-app-vote:latest
                        
                        docker push ${DOCKER_HUB_REPO}/voting-app-worker:${BUILD_NUM}
                        docker push ${DOCKER_HUB_REPO}/voting-app-worker:latest
                        
                        docker push ${DOCKER_HUB_REPO}/voting-app-result:${BUILD_NUM}
                        docker push ${DOCKER_HUB_REPO}/voting-app-result:latest
                        
                        docker logout
                        echo "Images pushed successfully"
                    '''
                }
            }
        }
        
        stage('Update K8s Manifests') {
            steps {
                echo 'Updating Kubernetes Manifests'
                sh """
                    sed -i 's|denish952/voting-app-vote:.*|denish952/voting-app-vote:${BUILD_NUM}|g' k8s-yaml/vote-deployment.yaml
                    sed -i 's|denish952/voting-app-worker:.*|denish952/voting-app-worker:${BUILD_NUM}|g' k8s-yaml/worker-deployment.yaml
                    sed -i 's|denish952/voting-app-result:.*|denish952/voting-app-result:${BUILD_NUM}|g' k8s-yaml/result-deployment.yaml
                    echo "Manifests updated with Build: ${BUILD_NUM}"
                    grep image: k8s-yaml/vote-deployment.yaml
                    grep image: k8s-yaml/worker-deployment.yaml
                    grep image: k8s-yaml/result-deployment.yaml
                """
            }
        }
        
        stage('Deploy to DEV') {
            steps {
                echo 'Deploying to DEV Kubernetes Cluster'
                sh '''
                    kubectl create namespace voting-app-dev --dry-run=client -o yaml | kubectl apply -f -
                    echo "Applying all K8s manifests to DEV"
                    kubectl apply -f k8s-yaml/ -n voting-app-dev
                    echo "Waiting for pods to start"
                    sleep 10
                    echo "DEV Cluster Status:"
                    kubectl get all -n voting-app-dev
                    echo "DEV Services:"
                    kubectl get svc -n voting-app-dev
                '''
            }
        }
        
        stage('Approval for PROD') {
            when {
                branch 'main'
            }
            steps {
                echo 'Waiting for approval to deploy to PROD'
                input message: 'Deploy to PROD?', ok: 'Deploy'
            }
        }
        
        stage('Deploy to PROD') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to PROD Kubernetes Cluster'
                sh '''
                    kubectl create namespace voting-app-prod --dry-run=client -o yaml | kubectl apply -f -
                    echo "Applying all K8s manifests to PROD"
                    kubectl apply -f k8s-yaml/ -n voting-app-prod
                    echo "Waiting for rollout"
                    kubectl rollout status deployment/vote -n voting-app-prod --timeout=120s
                    kubectl rollout status deployment/worker -n voting-app-prod --timeout=120s
                    kubectl rollout status deployment/result -n voting-app-prod --timeout=120s
                    echo "PROD Cluster Status:"
                    kubectl get all -n voting-app-prod
                '''
            }
        }
    }
    
    post {
        always {
            sh 'docker image prune -af --filter until=168h'
            echo 'Pipeline execution completed'
        }
        success {
            echo 'Pipeline succeeded'
        }
        failure {
            echo 'Pipeline failed - check logs'
        }
    }
}
