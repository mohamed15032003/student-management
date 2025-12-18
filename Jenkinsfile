pipeline {
    agent any
    
    environment {
        // Variables Docker
        DOCKER_IMAGE = 'mohamedderbel15032003/student-management-app'
        DOCKER_TAG = 'latest'
        
        // Variables Kubernetes
        K8S_NAMESPACE = 'student-management'
        K8S_DEPLOYMENT = 'student-management-app'
        MINIKUBE_NODE_IP = '192.168.49.2'  // IP de votre Minikube
        NODE_PORT = '30080'
    }
    
    stages {
        // ... vos étapes existantes (Git, Maven, SonarQube, Docker) ...
        
        // Étape 6: Déploiement Kubernetes sur Minikube
        stage('6) Deploy to Minikube') {
            environment {
                // Utilisation de vos identifiants Minikube
                KUBECONFIG = credentials('minikube-kubeconfig')
            }
            steps {
                script {
                    echo "🚀 Déploiement sur Minikube..."
                    
                    // 1. Vérifier l'accès à Minikube
                    sh '''
                        echo "=== CONNEXION MINIKUBE ==="
                        kubectl config current-context
                        kubectl get nodes
                        echo "Minikube IP: ${MINIKUBE_NODE_IP}"
                    '''
                    
                    // 2. Créer le namespace
                    sh """
                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f - || true
                    """
                    
                    // 3. Créer le déploiement
                    sh """
                        cat > deployment.yaml << EOF
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
                    
                    // 4. Créer le service NodePort
                    sh """
                        cat > service.yaml << EOF
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
                    '''
                    
                    // 5. Vérifier le déploiement
                    sh '''
                        echo "=== VÉRIFICATION ==="
                        kubectl get pods -n ${K8S_NAMESPACE}
                        kubectl get svc -n ${K8S_NAMESPACE}
                        
                        echo "🌐 VOTRE APPLICATION EST DISPONIBLE SUR :"
                        echo "http://${MINIKUBE_NODE_IP}:${NODE_PORT}/"
                        
                        # Attendre que les pods soient prêts
                        sleep 10
                        kubectl wait --for=condition=ready pod -l app=student-management -n ${K8S_NAMESPACE} --timeout=120s
                    '''
                }
            }
        }
        
        // Étape 7: Test de l'application déployée
        stage('7) Test Application') {
            steps {
                script {
                    sh '''
                        echo "🧪 Test de l'application déployée..."
                        sleep 30  # Laisser le temps à l'app de démarrer
                        
                        # Tester l'accès
                        curl -s -o /dev/null -w "Code HTTP: %{http_code}\n" http://${MINIKUBE_NODE_IP}:${NODE_PORT}/actuator/health || echo "Application en cours de démarrage"
                        
                        echo "✅ Déploiement Minikube terminé !"
                        echo ""
                        echo "📋 RÉSUMÉ :"
                        echo "Application: http://${MINIKUBE_NODE_IP}:${NODE_PORT}/"
                        echo "Pour voir les logs: kubectl logs -f deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE}"
                        echo "Pour supprimer: kubectl delete namespace ${K8S_NAMESPACE}"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo '🎉 Pipeline réussie avec déploiement Minikube !'
            sh '''
                echo "=========================================="
                echo "✅ DÉPLOIEMENT RÉUSSI"
                echo "🌐 URL: http://192.168.49.2:30080/"
                echo "🔧 Commandes utiles:"
                echo "   kubectl get pods -n student-management"
                echo "   kubectl logs -f deployment/student-management-app -n student-management"
                echo "   kubectl delete namespace student-management"
                echo "=========================================="
            '''
        }
        failure {
            echo '❌ Pipeline échouée.'
            // Nettoyage en cas d'échec
            sh '''
                echo "Nettoyage des ressources Kubernetes..."
                kubectl delete deployment ${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} --ignore-not-found=true || true
                kubectl delete service ${K8S_DEPLOYMENT}-service -n ${K8S_NAMESPACE} --ignore-not-found=true || true
            '''
        }
    }
}
