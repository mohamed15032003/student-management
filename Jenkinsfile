pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'mohamedderbel15032003/student-management-app'
        DOCKER_TAG = 'latest'
        K8S_NAMESPACE = 'student-management'
        K8S_DEPLOYMENT = 'student-management-app'
        MINIKUBE_NODE_IP = '192.168.49.2'
        NODE_PORT = '30080'
    }
    
    stages {
        stage('1) Git Clone') {
            steps {
                git(
                    url: 'https://github.com/mohamed15032003/student-management.git',
                    branch: 'main',
                    credentialsId: 'github-creds'
                )
                sh 'echo "Code source récupéré avec succès !"'
            }
        }
        
        stage('2) Build Maven') {
            steps {
                sh 'mvn clean compile'
                sh 'echo "Compilation Maven réussie !"'
            }
        }
        
        stage('3) Build JAR') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                sh 'echo "JAR archivé dans Jenkins"'
            }
        }
        
        stage('4) SonarQube Analysis') {
            steps {
                script {
                    try {
                        withCredentials([
                            string(credentialsId: 'SONARQUBE_TOKEN', variable: 'SONAR_TOKEN')
                        ]) {
                            sh '''
                                mvn sonar:sonar \
                                -Dsonar.projectKey=student-management \
                                -Dsonar.host.url=http://54.37.78.131:9000 \
                                -Dsonar.login=${SONAR_TOKEN}
                            '''
                        }
                    } catch (Exception e) {
                        echo "⚠️ SonarQube non disponible, continuation sans analyse..."
                    }
                }
            }
        }
        
        stage('5) Docker Build & Push') {
            environment {
                DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'
            }
            steps {
                script {
                    sh """
                        echo "Construction de l'image Docker..."
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    """
                    
                    withCredentials([
                        usernamePassword(
                            credentialsId: DOCKER_CREDENTIALS_ID,
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )
                    ]) {
                        sh '''
                            echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin
                        '''
                    }
                    
                    sh """
                        echo "Push des images Docker..."
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    """
                    
                    sh 'echo "✅ Images Docker poussées avec succès !"'
                }
            }
        }
        
        stage('6) Deploy to Minikube') {
            environment {
                KUBECONFIG = credentials('minikube-kubeconfig')
            }
            steps {
                script {
                    echo "🚀 Déploiement sur Minikube..."
                    
                    sh '''
                        echo "=== CONNEXION MINIKUBE ==="
                        kubectl config current-context
                        kubectl get nodes
                    '''
                    
                    sh """
                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f - || true
                    """
                    
                    sh """
                        cat > deployment.yaml << 'EOF'
                        apiVersion: apps/v1
                        kind: Deployment
                        metadata:
                          name: ${K8S_DEPLOYMENT}
                          namespace: ${K8S_NAMESPACE}
                        spec:
                          replicas: 2
                          selector:
                            matchLabels:
                              app: student-management
                          template:
                            metadata:
                              labels:
                                app: student-management
                            spec:
                              containers:
                              - name: student-management
                                image: ${DOCKER_IMAGE}:${DOCKER_TAG}
                                ports:
                                - containerPort: 8080
                                resources:
                                  requests:
                                    memory: "512Mi"
                                    cpu: "250m"
                        EOF
                        
                        kubectl apply -f deployment.yaml -n ${K8S_NAMESPACE}
                    """
                    
                    sh """
                        cat > service.yaml << 'EOF'
                        apiVersion: v1
                        kind: Service
                        metadata:
                          name: ${K8S_DEPLOYMENT}-service
                          namespace: ${K8S_NAMESPACE}
                        spec:
                          type: NodePort
                          selector:
                            app: student-management
                          ports:
                          - port: 8080
                            targetPort: 8080
                            nodePort: ${NODE_PORT}
                        EOF
                        
                        kubectl apply -f service.yaml -n ${K8S_NAMESPACE}
                    """
                    
                    sh '''
                        echo "=== VÉRIFICATION ==="
                        kubectl get pods -n ${K8S_NAMESPACE}
                        kubectl get svc -n ${K8S_NAMESPACE}
                        echo "🌐 VOTRE APPLICATION EST DISPONIBLE SUR :"
                        echo "http://192.168.49.2:30080/"
                        sleep 10
                        kubectl wait --for=condition=ready pod -l app=student-management -n ${K8S_NAMESPACE} --timeout=120s
                    '''
                }
            }
        }
        
        stage('7) Test Application') {
            steps {
                script {
                    sh '''
                        echo "🧪 Test de l'application déployée..."
                        sleep 30
                        curl -s -o /dev/null -w "Code HTTP: %{http_code}\n" http://192.168.49.2:30080/actuator/health || echo "Application en cours de démarrage"
                        echo "✅ Déploiement Minikube terminé !"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo '🎉 Pipeline réussie avec déploiement Minikube !'
        }
        failure {
            echo '❌ Pipeline échouée.'
            sh '''
                echo "Nettoyage des ressources Kubernetes..."
                kubectl delete deployment student-management-app -n student-management --ignore-not-found=true || true
                kubectl delete service student-management-app-service -n student-management --ignore-not-found=true || true
            '''
        }
        always {
            echo '🔧 Nettoyage...'
            sh '''
                docker container prune -f || true
                docker image prune -f || true
                rm -f deployment.yaml service.yaml || true
            '''
        }
    }
}
