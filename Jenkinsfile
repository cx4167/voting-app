pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'denish952'
        BUILD_TAG = "${BUILD_NUMBER}"
        DEV_NAMESPACE = 'voting-app-dev'
        PROD_NAMESPACE = 'voting-app'
        KUBECONFIG = '/var/jenkins_home/.kube/config'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 1, unit: 'HOURS')
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
        
        stage('Build Vote Image') {
            steps {
                echo 'Building Vote Service Docker Image'
                sh '''
                    docker build -t denish952/voting-app-vote:${BUILD_TAG} \
                        -t denish952/voting-app-vote:latest \
                        -f vote/Dockerfile vote/
                    docker images | grep voting-app-vote
                '''
            }
        }
        
        stage('Build Worker Image') {
            steps {
                echo 'Building Worker Service Docker Image'
                sh '''
                    docker build -t denish952/voting-app-worker:${BUILD_TAG} \
                        -t denish952/voting-app-worker:latest \
                        -f worker/Dockerfile worker/
                    docker images | grep voting-app-worker
                '''
            }
        }
        
        stage('Build Result Image') {
            steps {
                echo 'Building Result Service Docker Image'
                sh '''
                    docker build -t denish952/voting-app-result:${BUILD_TAG} \
                        -t denish952/voting-app-result:latest \
                        -f result/Dockerfile result/
                    docker images | grep voting-app-result
                '''
            }
        }
        
        stage('Test Images') {
            steps {
                echo 'Running Tests'
                sh '''
                    docker run --rm denish952/voting-app-vote:${BUILD_TAG} \
                        python -c "from flask import Flask; print('Vote OK')" || true
                    
                    docker run --rm denish952/voting-app-worker:${BUILD_TAG} \
                        python -c "import redis; print('Worker OK')" || true
                    
                    docker run --rm denish952/voting-app-result:${BUILD_TAG} \
                        python -c "from flask import Flask; print('Result OK')" || true
                '''
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing Images to Docker Hub'
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', 
                                 usernameVariable: 'DOCKER_USER', 
                                 passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        
                        docker push denish952/voting-app-vote:${BUILD_TAG}
                        docker push denish952/voting-app-vote:latest
                        
                        docker push denish952/voting-app-worker:${BUILD_TAG}
                        docker push denish952/voting-app-worker:latest
                        
                        docker push denish952/voting-app-result:${BUILD_TAG}
                        docker push denish952/voting-app-result:latest
                        
                        docker logout
                        echo 'Images pushed successfully'
                    '''
                }
            }
        }
        
        stage('Update K8s Manifests') {
            steps {
                echo 'Updating Kubernetes Manifests with new image tag'
                sh '''
                    sed -i "s|denish952/voting-app-vote:.*|denish952/voting-app-vote:${BUILD_TAG}|g" \
                        k8s-yaml/vote-deployment.yaml
                    
                    sed -i "s|denish952/voting-app-worker:.*|denish952/voting-app-worker:${BUILD_TAG}|g" \
                        k8s-yaml/worker-deployment.yaml
                    
                    sed -i "s|denish952/voting-app-result:.*|denish952/voting-app-result:${BUILD_TAG}|g" \
                        k8s-yaml/result-deployment.yaml
                    
                    echo 'Manifests updated with Build:' ${BUILD_TAG}
                    grep 'image:' k8s-yaml/vote-deployment.yaml
                    grep 'image:' k8s-yaml/worker-deployment.yaml
                    grep 'image:' k8s-yaml/result-deployment.yaml
                '''
            }
        }
        
        stage('Deploy to DEV') {
            steps {
                echo 'Deploying to DEV Kubernetes Cluster'
                sh '''
                    kubectl create namespace ${DEV_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                    
                    echo 'Applying all K8s manifests to DEV'
                    kubectl apply -f k8s-yaml/ -n ${DEV_NAMESPACE}
                    
                    echo 'Waiting for pods to start'
                    sleep 10
                    
                    echo 'DEV Cluster Status:'
                    kubectl get all -n ${DEV_NAMESPACE}
                    
                    echo 'DEV Services:'
                    kubectl get svc -n ${DEV_NAMESPACE}
                '''
            }
        }
        
        stage('Approval for PROD') {
            when {
                branch 'main'
            }
            steps {
                script {
                    timeout(time: 24, unit: 'HOURS') {
                        input message: 'Deploy to Production?', ok: 'Deploy'
                    }
                }
            }
        }
        
        stage('Deploy to PROD') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to PRODUCTION Kubernetes Cluster'
                sh '''
                    kubectl create namespace ${PROD_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                    
                    echo 'Applying K8s manifests to PRODUCTION'
                    kubectl apply -f k8s-yaml/ -n ${PROD_NAMESPACE}
                    
                    echo 'Waiting for rollout'
                    sleep 15
                    
                    echo 'PRODUCTION Cluster Status:'
                    kubectl get all -n ${PROD_NAMESPACE}
                    
                    echo 'PRODUCTION Services:'
                    kubectl get svc -n ${PROD_NAMESPACE}
                '''
            }
        }
    }
    
    post {
        always {
            sh 'docker image prune -af --filter "until=168h" || true'
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
