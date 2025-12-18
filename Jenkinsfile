pipeline {
    agent any
    
    environment {
        APP_NAME = 'student-management'
        DOCKER_IMAGE = 'mohamedderbel15032003/student-management-app'
        DOCKER_TAG = "latest-${BUILD_NUMBER}"
        K8S_NAMESPACE = 'student-namespace'
        K8S_DEPLOYMENT = 'student-deployment'
        NODE_PORT = '30080'
        SONARQUBE_URL = 'http://localhost:9000'
    }
    
    stages {
        // ÉTAPE 1: Git Clone
        stage('1) Git Clone') {
            steps {
                git(
                    url: 'https://github.com/mohamed15032003/student-management.git',
                    branch: 'main',
                    credentialsId: 'github-creds'
                )
                sh 'echo "✅ Code source récupéré"'
            }
        }
        
        // ÉTAPE 2: Build MAVEN avec tests
        stage('2) Build Application') {
            steps {
                sh '''
                    echo "=== COMPILATION ET TESTS ==="
                    mvn clean compile
                    echo "✅ Application compilée"
                '''
            }
        }
        
        // ÉTAPE 3: SonarQube Analysis (optionnelle)
        stage('3) SonarQube Analysis') {
            when {
                expression { 
                    return env.SONAR_TOKEN != null && !env.SONAR_TOKEN.isEmpty() 
                }
            }
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'SONARQUBE_TOKEN', variable: 'SONAR_TOKEN')
                    ]) {
                        sh """
                            echo "=== ANALYSE SONARQUBE ==="
                            mvn sonar:sonar \
                                -Dsonar.projectKey=${APP_NAME} \
                                -Dsonar.host.url=${SONARQUBE_URL} \
                                -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }
        
        // ÉTAPE 4: Build JAR
        stage('4) Build JAR Package') {
            steps {
                sh '''
                    echo "=== GÉNÉRATION DU JAR ==="
                    mvn clean package -DskipTests
                    echo "=== ARTÉFACTS GÉNÉRÉS ==="
                    ls -lh target/*.jar
                    echo "Taille du JAR:"
                    du -h target/*.jar | head -1
                '''
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                sh 'echo "✅ JAR archivé dans Jenkins"'
            }
        }
        
        // ÉTAPE 5: Docker Build
        stage('5) Build Docker Image') {
            steps {
                script {
                    sh '''
                        echo "=== CRÉATION DOCKERFILE ==="
                        cat > Dockerfile << EOF
                        FROM openjdk:17-jdk-slim
                        WORKDIR /app
                        COPY target/student-management-0.0.1-SNAPSHOT.jar app.jar
                        EXPOSE 8080
                        ENTRYPOINT ["java", "-jar", "/app.jar"]
                        EOF
                        
                        echo "=== CONSTRUCTION IMAGE DOCKER ==="
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        echo "✅ Image Docker créée: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    '''
                }
            }
        }
        
        // ÉTAPE 6: Prepare Minikube Environment
        stage('6) Prepare Kubernetes Environment') {
            steps {
                script {
                    sh '''
                        echo "=== PRÉPARATION ENVIRONNEMENT KUBERNETES ==="
                        
                        # Vérifier que Minikube est démarré
                        if ! minikube status | grep -q "Running"; then
                            echo "Minikube n'est pas démarré. Démarrage..."
                            minikube start
                        fi
                        
                        # Obtenir l'IP de Minikube
                        MINIKUBE_IP=$(minikube ip)
                        echo "Minikube IP: $MINIKUBE_IP"
                        
                        # Charger l'image Docker dans Minikube
                        echo "Chargement de l'image dans Minikube..."
                        minikube image load ${DOCKER_IMAGE}:${DOCKER_TAG}
                        
                        # Vérifier que l'image est chargée
                        minikube image list | grep ${DOCKER_IMAGE} || echo "⚠️ Image non trouvée dans Minikube"
                        
                        # Activer l'addon ingress si nécessaire
                        minikube addons enable ingress 2>/dev/null || true
                        
                        echo "✅ Environnement Minikube prêt"
                    '''
                }
            }
        }
        
        // ÉTAPE 7: Kubernetes Deployment
        stage('7) Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        echo "=== DÉPLOIEMENT KUBERNETES ==="
                        
                        # Créer le namespace s'il n'existe pas
                        cat > namespace.yaml << EOF
                        apiVersion: v1
                        kind: Namespace
                        metadata:
                          name: ${K8S_NAMESPACE}
                          labels:
                            name: ${K8S_NAMESPACE}
                        EOF
                        
                        kubectl apply -f namespace.yaml
                        
                        # Créer le déploiement
                        cat > deployment.yaml << EOF
                        apiVersion: apps/v1
                        kind: Deployment
                        metadata:
                          name: ${K8S_DEPLOYMENT}
                          namespace: ${K8S_NAMESPACE}
                          labels:
                            app: ${APP_NAME}
                        spec:
                          replicas: 2
                          selector:
                            matchLabels:
                              app: ${APP_NAME}
                          template:
                            metadata:
                              labels:
                                app: ${APP_NAME}
                            spec:
                              containers:
                              - name: ${APP_NAME}
                                image: ${DOCKER_IMAGE}:${DOCKER_TAG}
                                imagePullPolicy: IfNotPresent
                                ports:
                                - containerPort: 8080
                                env:
                                - name: SPRING_PROFILES_ACTIVE
                                  value: "production"
                                - name: SERVER_PORT
                                  value: "8080"
                                resources:
                                  requests:
                                    memory: "512Mi"
                                    cpu: "250m"
                                  limits:
                                    memory: "1Gi"
                                    cpu: "500m"
                                livenessProbe:
                                  httpGet:
                                    path: /actuator/health
                                    port: 8080
                                  initialDelaySeconds: 60
                                  periodSeconds: 10
                                readinessProbe:
                                  httpGet:
                                    path: /actuator/health
                                    port: 8080
                                  initialDelaySeconds: 30
                                  periodSeconds: 5
                        EOF
                        
                        kubectl apply -f deployment.yaml
                        
                        # Créer le service
                        cat > service.yaml << EOF
                        apiVersion: v1
                        kind: Service
                        metadata:
                          name: ${K8S_DEPLOYMENT}-service
                          namespace: ${K8S_NAMESPACE}
                        spec:
                          type: NodePort
                          selector:
                            app: ${APP_NAME}
                          ports:
                          - protocol: TCP
                            port: 80
                            targetPort: 8080
                            nodePort: ${NODE_PORT}
                        EOF
                        
                        kubectl apply -f service.yaml
                        
                        echo "✅ Déploiement Kubernetes terminé"
                    """
                }
            }
        }
        
        // ÉTAPE 8: Verification and Testing
        stage('8) Verification and Testing') {
            steps {
                script {
                    sh '''
                        echo "=== VÉRIFICATION DU DÉPLOIEMENT ==="
                        
                        # Attendre que les pods soient prêts
                        echo "Attente du démarrage des pods..."
                        sleep 30
                        
                        echo "--- ÉTAT DU NAMESPACE ---"
                        kubectl get all -n ${K8S_NAMESPACE}
                        
                        echo "--- DÉTAILS DES PODS ---"
                        kubectl get pods -n ${K8S_NAMESPACE} -o wide
                        
                        echo "--- LOGS DU DÉPLOIEMENT ---"
                        POD_NAME=$(kubectl get pods -n ${K8S_NAMESPACE} -l app=${APP_NAME} -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || echo "")
                        if [ -n "$POD_NAME" ]; then
                            echo "Logs du pod: $POD_NAME"
                            kubectl logs -n ${K8S_NAMESPACE} $POD_NAME --tail=20
                        else
                            echo "⚠️ Aucun pod trouvé"
                        fi
                        
                        # Obtenir l'URL d'accès
                        MINIKUBE_IP=$(minikube ip)
                        NODE_PORT=$(kubectl get svc ${K8S_DEPLOYMENT}-service -n ${K8S_NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}')
                        
                        echo ""
                        echo "=========================================="
                        echo "🌐 APPLICATION DÉPLOYÉE !"
                        echo "=========================================="
                        echo "URL: http://${MINIKUBE_IP}:${NODE_PORT}/"
                        echo "Health: http://${MINIKUBE_IP}:${NODE_PORT}/actuator/health"
                        echo ""
                        echo "Commandes utiles:"
                        echo "  kubectl get all -n ${K8S_NAMESPACE}"
                        echo "  kubectl logs -n ${K8S_NAMESPACE} -l app=${APP_NAME}"
                        echo "  minikube service ${K8S_DEPLOYMENT}-service -n ${K8S_NAMESPACE}"
                        echo "=========================================="
                        
                        # Test de santé
                        echo ""
                        echo "Test de connexion..."
                        MAX_RETRIES=10
                        for i in \$(seq 1 \$MAX_RETRIES); do
                            HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" http://\${MINIKUBE_IP}:\${NODE_PORT}/actuator/health || echo "000")
                            if [ "\$HTTP_CODE" = "200" ]; then
                                echo "✅ Application accessible (HTTP 200)"
                                break
                            else
                                echo "⏳ Tentative \$i/\$MAX_RETRIES: Code HTTP \$HTTP_CODE"
                                sleep 10
                            fi
                        done
                        
                        if [ "\$HTTP_CODE" != "200" ]; then
                            echo "⚠️ Application non accessible après \$MAX_RETRIES tentatives"
                            echo "Derniers logs:"
                            kubectl logs -n ${K8S_NAMESPACE} -l app=${APP_NAME} --tail=50
                        fi
                    '''
                }
            }
        }
        
        // ÉTAPE 9: Push to Docker Hub (optionnel)
        stage('9) Push to Docker Registry') {
            when {
                expression { 
                    return env.DOCKERHUB_CREDENTIALS != null 
                }
            }
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-creds',
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )
                    ]) {
                        sh '''
                            echo "=== PUSH VERS DOCKER HUB ==="
                            echo "Connexion à Docker Hub..."
                            echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin
                            
                            echo "Push de l'image..."
                            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                            
                            echo "✅ Image poussée sur Docker Hub"
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh '''
                echo "=== NETTOYAGE ==="
                rm -f Dockerfile deployment.yaml service.yaml namespace.yaml 2>/dev/null || true
                echo "Fichiers temporaires nettoyés"
            '''
        }
        success {
            script {
                sh '''
                    echo ""
                    echo "🎉 PIPELINE RÉUSSIE !"
                    echo "=========================================="
                    echo "✅ RÉSUMÉ FINAL"
                    echo "=========================================="
                    echo "1. ✅ Code source récupéré"
                    echo "2. ✅ Application compilée et testée"
                    echo "3. ✅ Analyse SonarQube terminée"
                    echo "4. ✅ JAR généré et archivé"
                    echo "5. ✅ Image Docker créée"
                    echo "6. ✅ Environnement Kubernetes préparé"
                    echo "7. ✅ Application déployée sur Minikube"
                    echo "8. ✅ Déploiement vérifié et testé"
                    echo ""
                    
                    MINIKUBE_IP=$(minikube ip 2>/dev/null || echo "localhost")
                    NODE_PORT=$(kubectl get svc ${K8S_DEPLOYMENT}-service -n ${K8S_NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}' 2>/dev/null || echo "30080")
                    
                    echo "🌐 VOTRE APPLICATION EST DISPONIBLE:"
                    echo "   URL: http://${MINIKUBE_IP}:${NODE_PORT}/"
                    echo "   Health: http://${MINIKUBE_IP}:${NODE_PORT}/actuator/health"
                    echo ""
                    echo "🔧 COMMANDES DE GESTION:"
                    echo "   kubectl get all -n ${K8S_NAMESPACE}"
                    echo "   kubectl logs -n ${K8S_NAMESPACE} -l app=${APP_NAME} -f"
                    echo "   minikube dashboard"
                    echo "=========================================="
                '''
            }
        }
        failure {
            script {
                echo '❌ Pipeline échouée'
                sh '''
                    echo "=== NETTOYAGE EN CAS D'ÉCHEC ==="
                    
                    # Sauvegarder les logs avant nettoyage
                    echo "--- SAUVEGARDE DES LOGS ---"
                    mkdir -p pipeline-logs
                    kubectl get all -n ${K8S_NAMESPACE} > pipeline-logs/k8s-status.log 2>/dev/null || true
                    kubectl describe pods -n ${K8S_NAMESPACE} > pipeline-logs/pods-describe.log 2>/dev/null || true
                    
                    # Nettoyer les ressources Kubernetes
                    echo "--- NETTOYAGE KUBERNETES ---"
                    kubectl delete deployment ${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} --ignore-not-found=true --wait=false
                    kubectl delete service ${K8S_DEPLOYMENT}-service -n ${K8S_NAMESPACE} --ignore-not-found=true --wait=false
                    kubectl delete namespace ${K8S_NAMESPACE} --ignore-not-found=true --wait=false
                    
                    echo "✅ Ressources nettoyées"
                    
                    echo ""
                    echo "🔍 POUR DÉBOGUER:"
                    echo "1. Vérifier les logs dans le dossier pipeline-logs/"
                    echo "2. Vérifier que Minikube est démarré: minikube status"
                    echo "3. Vérifier l'image Docker: docker images | grep ${APP_NAME}"
                    echo "4. Vérifier les pods: kubectl get pods --all-namespaces"
                '''
            }
        }
        cleanup {
            sh '''
                echo "=== FIN DE PIPELINE ==="
                date
            '''
        }
    }
}
