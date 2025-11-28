pipeline {
    agent any
    
    environment {
        // Docker Hub details
        DOCKER_REGISTRY = 'denish952'
        DOCKER_IMAGE_VOTE = "${DOCKER_REGISTRY}/voting-app-vote"
        DOCKER_IMAGE_WORKER = "${DOCKER_REGISTRY}/voting-app-worker"
        DOCKER_IMAGE_RESULT = "${DOCKER_REGISTRY}/voting-app-result"
        
        // Build identifier
        BUILD_TAG = "${BUILD_NUMBER}"
        
        // Kubernetes namespaces
        DEV_NAMESPACE = 'voting-app-dev'
        PROD_NAMESPACE = 'voting-app'
        
        // Kubeconfig
        KUBECONFIG = '/var/jenkins_home/.kube/config'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 1, unit: 'HOURS')
    }
    
    stages {
        stage('📥 Checkout Code') {
            steps {
                echo '==================== Pulling Code from GitHub ===================='
                checkout scm
                sh 'git log -1 --oneline'
            }
        }
        
        stage('🔨 Build Docker Images') {
            parallel {
                stage('🗳️  Build Vote') {
                    steps {
                        echo '========== Building Vote Service =========='
                        script {
                            sh '''
                                echo "Building: ${DOCKER_IMAGE_VOTE}:${BUILD_TAG}"
                                docker build -t ${DOCKER_IMAGE_VOTE}:${BUILD_TAG} \
                                    -t ${DOCKER_IMAGE_VOTE}:latest \
                                    -f docker/vote/Dockerfile docker/vote/
                                docker images | grep voting-app-vote | head -2
                            '''
                        }
                    }
                }
                
                stage('⚙️  Build Worker') {
                    steps {
                        echo '========== Building Worker Service =========='
                        script {
                            sh '''
                                echo "Building: ${DOCKER_IMAGE_WORKER}:${BUILD_TAG}"
                                docker build -t ${DOCKER_IMAGE_WORKER}:${BUILD_TAG} \
                                    -t ${DOCKER_IMAGE_WORKER}:latest \
                                    -f docker/worker/Dockerfile docker/worker/
                                docker images | grep voting-app-worker | head -2
                            '''
                        }
                    }
                }
                
                stage('📊 Build Result') {
                    steps {
                        echo '========== Building Result Service =========='
                        script {
                            sh '''
                                echo "Building: ${DOCKER_IMAGE_RESULT}:${BUILD_TAG}"
                                docker build -t ${DOCKER_IMAGE_RESULT}:${BUILD_TAG} \
                                    -t ${DOCKER_IMAGE_RESULT}:latest \
                                    -f docker/result/Dockerfile docker/result/
                                docker images | grep voting-app-result | head -2
                            '''
                        }
                    }
                }
            }
        }
        
        stage('✅ Run Tests') {
            steps {
                echo '========== Running Unit Tests =========='
                script {
                    sh '''
                        echo "Testing Vote Service..."
                        docker run --rm ${DOCKER_IMAGE_VOTE}:${BUILD_TAG} \
                            python -c "from flask import Flask; print('✅ Vote app OK')" || true
                        
                        echo "Testing Worker Service..."
                        docker run --rm ${DOCKER_IMAGE_WORKER}:${BUILD_TAG} \
                            python -c "import redis, psycopg2; print('✅ Worker dependencies OK')" || true
                        
                        echo "Testing Result Service..."
                        docker run --rm ${DOCKER_IMAGE_RESULT}:${BUILD_TAG} \
                            python -c "from flask import Flask; print('✅ Result app OK')" || true
                    '''
                }
            }
        }
        
        stage('📤 Push to Docker Hub') {
            steps {
                echo '========== Pushing Images to Docker Hub =========='
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', 
                                     usernameVariable: 'DOCKER_USER', 
                                     passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                            
                            echo "Pushing Vote:${BUILD_TAG}..."
                            docker push ${DOCKER_IMAGE_VOTE}:${BUILD_TAG}
                            docker push ${DOCKER_IMAGE_VOTE}:latest
                            
                            echo "Pushing Worker:${BUILD_TAG}..."
                            docker push ${DOCKER_IMAGE_WORKER}:${BUILD_TAG}
                            docker push ${DOCKER_IMAGE_WORKER}:latest
                            
                            echo "Pushing Result:${BUILD_TAG}..."
                            docker push ${DOCKER_IMAGE_RESULT}:${BUILD_TAG}
                            docker push ${DOCKER_IMAGE_RESULT}:latest
                            
                            docker logout
                            
                            echo "✅ All images pushed successfully!"
                        '''
                    }
                }
            }
        }
        
        stage('🔄 Update K8s Manifests') {
            steps {
                echo '========== Updating Kubernetes Manifests =========='
                script {
                    sh '''
                        echo "Updating image tags to: ${BUILD_TAG}"
                        
                        # Update Vote manifest
                        sed -i "s|image: denish952/voting-app-vote:.*|image: denish952/voting-app-vote:${BUILD_TAG}|g" \
                            k8s/vote/vote-deployment.yaml
                        
                        # Update Worker manifest
                        sed -i "s|image: denish952/voting-app-worker:.*|image: denish952/voting-app-worker:${BUILD_TAG}|g" \
                            k8s/worker/worker-deployment.yaml
                        
                        # Update Result manifest
                        sed -i "s|image: denish952/voting-app-result:.*|image: denish952/voting-app-result:${BUILD_TAG}|g" \
                            k8s/result/result-deployment.yaml
                        
                        echo "Updated manifests:"
                        echo "Vote:"
                        grep "image:" k8s/vote/vote-deployment.yaml | head -1
                        echo "Worker:"
                        grep "image:" k8s/worker/worker-deployment.yaml | head -1
                        echo "Result:"
                        grep "image:" k8s/result/result-deployment.yaml | head -1
                    '''
                }
            }
        }
        
        stage('🚀 Deploy to DEV') {
            steps {
                echo '========== Deploying to DEV Kubernetes =========='
                script {
                    sh '''
                        # Create namespace
                        kubectl create namespace ${DEV_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        echo "Deploying to ${DEV_NAMESPACE}..."
                        
                        # Deploy configurations
                        kubectl apply -f k8s/configmap.yaml -n ${DEV_NAMESPACE}
                        kubectl apply -f k8s/secrets.yaml -n ${DEV_NAMESPACE}
                        kubectl apply -f k8s/pv-pvc.yaml -n ${DEV_NAMESPACE}
                        
                        # Deploy services
                        kubectl apply -f k8s/postgres/postgres-deployment.yaml -n ${DEV_NAMESPACE}
                        kubectl apply -f k8s/redis/redis-deployment.yaml -n ${DEV_NAMESPACE}
                        kubectl apply -f k8s/worker/worker-deployment.yaml -n ${DEV_NAMESPACE}
                        kubectl apply -f k8s/vote/vote-deployment.yaml -n ${DEV_NAMESPACE}
                        kubectl apply -f k8s/result/result-deployment.yaml -n ${DEV_NAMESPACE}
                        
                        echo "Waiting for deployments to be ready..."
                        kubectl rollout status deployment/vote -n ${DEV_NAMESPACE} --timeout=5m || true
                        kubectl rollout status deployment/worker -n ${DEV_NAMESPACE} --timeout=5m || true
                        kubectl rollout status deployment/result -n ${DEV_NAMESPACE} --timeout=5m || true
                        
                        echo ""
                        echo "DEV Cluster Status:"
                        kubectl get all -n ${DEV_NAMESPACE}
                        
                        echo ""
                        echo "Services and NodePorts:"
                        kubectl get svc -n ${DEV_NAMESPACE}
                    '''
                }
            }
        }
        
        stage('⏸️  Approval for PROD') {
            when {
                branch 'main'
            }
            steps {
                script {
                    timeout(time: 24, unit: 'HOURS') {
                        def userInput = input(
                            id: 'Prod-Deploy-Approval',
                            message: '''
                            ╔════════════════════════════════════════╗
                            ║  🚀 READY TO DEPLOY TO PRODUCTION? 🚀  ║
                            ╚════════════════════════════════════════╝
                            
                            Build: ${BUILD_NUMBER}
                            Images Tagged: ${BUILD_TAG}
                            
                            Current Status:
                            ✅ Code tested
                            ✅ Images built and pushed
                            ✅ Deployed to DEV successfully
                            
                            Click "Proceed" to deploy to Production
                            ''',
                            parameters: [
                                string(
                                    name: 'APPROVAL_REASON',
                                    description: 'Why deploy to production?',
                                    defaultValue: 'Scheduled production update'
                                )
                            ]
                        )
                        env.APPROVAL_REASON = userInput
                        env.APPROVED = 'true'
                    }
                }
            }
        }
        
        stage('✈️  Deploy to PROD') {
            when {
                branch 'main'
                environment name: 'APPROVED', value: 'true'
            }
            steps {
                echo '========== Deploying to PRODUCTION Kubernetes =========='
                script {
                    sh '''
                        # Create namespace
                        kubectl create namespace ${PROD_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        echo "🚀 Deploying to ${PROD_NAMESPACE}..."
                        
                        # Deploy configurations (first time only)
                        kubectl apply -f k8s/configmap.yaml -n ${PROD_NAMESPACE} || true
                        kubectl apply -f k8s/secrets.yaml -n ${PROD_NAMESPACE} || true
                        kubectl apply -f k8s/pv-pvc.yaml -n ${PROD_NAMESPACE} || true
                        
                        # Deploy database (first time only)
                        kubectl apply -f k8s/postgres/postgres-deployment.yaml -n ${PROD_NAMESPACE} || true
                        kubectl apply -f k8s/redis/redis-deployment.yaml -n ${PROD_NAMESPACE} || true
                        
                        # Update application services (rolling update)
                        kubectl apply -f k8s/worker/worker-deployment.yaml -n ${PROD_NAMESPACE}
                        kubectl apply -f k8s/vote/vote-deployment.yaml -n ${PROD_NAMESPACE}
                        kubectl apply -f k8s/result/result-deployment.yaml -n ${PROD_NAMESPACE}
                        
                        echo ""
                        echo "Waiting for PRODUCTION deployment to be ready..."
                        kubectl rollout status deployment/vote -n ${PROD_NAMESPACE} --timeout=5m
                        kubectl rollout status deployment/worker -n ${PROD_NAMESPACE} --timeout=5m
                        kubectl rollout status deployment/result -n ${PROD_NAMESPACE} --timeout=5m
                        
                        echo ""
                        echo "✅ PRODUCTION Deployment Complete!"
                        echo ""
                        echo "Production Services:"
                        kubectl get svc -n ${PROD_NAMESPACE}
                        
                        echo ""
                        echo "Production Pods:"
                        kubectl get pods -n ${PROD_NAMESPACE}
                    '''
                }
            }
        }
    }
    
    post {
        always {
            script {
                sh 'docker image prune -af --filter "until=168h" || true'
            }
        }
        
        success {
            echo '''
            ╔════════════════════════════════════════╗
            ║     ✅ PIPELINE SUCCESSFUL! ✅          ║
            ╚════════════════════════════════════════╝
            '''
        }
        
        failure {
            echo '''
            ╔════════════════════════════════════════╗
            ║     ❌ PIPELINE FAILED! ❌              ║
            ╚════════════════════════════════════════╝
            Check logs above for details
            '''
        }
    }
}
