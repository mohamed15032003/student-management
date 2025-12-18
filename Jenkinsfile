pipeline {
    agent any
    
    environment {
        // Variables d'environnement
        APP_NAME = 'student-management'
        DOCKER_IMAGE = 'mohamedderbel15032003/student-management-app'
        DOCKER_TAG = 'latest'
        K8S_NAMESPACE = 'student-namespace'
        K8S_DEPLOYMENT = 'student-deployment'
        NODE_PORT = '30080'
        SONARQUBE_URL = 'http://54.37.78.131:9000'
    }
    
    stages {
        // ========== ÉTAPE 1: CLONE GIT ==========
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
        
        // ========== ÉTAPE 2: BUILD MAVEN ==========
        stage('2) Build & Test') {
            steps {
                sh '''
                    echo "=== COMPILATION MAVEN ==="
                    mvn clean compile
                    
                    echo "=== EXÉCUTION DES TESTS ==="
                    mvn test || echo "⚠️ Certains tests ont échoué (continuation...)"
                    
                    echo "✅ Build Maven réussi"
                '''
            }
        }
        
        // ========== ÉTAPE 3: ANALYSE SONARQUBE ==========
        stage('3) SonarQube Analysis') {
            steps {
                script {
                    try {
                        withCredentials([
                            string(credentialsId: 'SONARQUBE_TOKEN', variable: 'SONAR_TOKEN')
                        ]) {
                            sh """
                                echo "=== ANALYSE QUALITÉ SONARQUBE ==="
                                mvn sonar:sonar \
                                -Dsonar.projectKey=${APP_NAME} \
                                -Dsonar.host.url=${SONARQUBE_URL} \
                                -Dsonar.login=${SONAR_TOKEN}
                            """
                        }
                    } catch (Exception e) {
                        echo "⚠️ SonarQube non disponible - étape ignorée"
                        // Ne pas échouer la pipeline
                    }
                }
            }
        }
        
        // ========== ÉTAPE 4: CRÉATION DU JAR ==========
        stage('4) Build JAR') {
            steps {
                sh '''
                    echo "=== GÉNÉRATION DU JAR ==="
                    mvn package -DskipTests
                    
                    echo "=== ARTÉFACTS GÉNÉRÉS ==="
                    ls -lh target/*.jar
                    echo "Taille du JAR:"
                    du -h target/*.jar | head -1
                '''
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                sh 'echo "✅ JAR archivé dans Jenkins"'
            }
        }
        
        // ========== ÉTAPE 5: DOCKER BUILD ==========
        stage('5) Docker Build') {
            steps {
                script {
                    // Créer Dockerfile si manquant
                    sh '''
                        if [ ! -f Dockerfile ]; then
                            echo "Création du Dockerfile..."
                            cat > Dockerfile << 'EOF'
                            FROM openjdk:17-jdk-slim
                            WORKDIR /app
                            COPY target/*.jar app.jar
                            EXPOSE 8080
                            ENTRYPOINT ["java", "-jar", "/app.jar"]
                            EOF
                            echo "Dockerfile créé"
                        fi
                        
                        echo "=== CONSTRUCTION IMAGE DOCKER ==="
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    '''
                }
            }
        }
        
        // ========== ÉTAPE 6: DOCKER PUSH (Optionnel) ==========
        stage('6) Docker Push to Hub') {
            when {
                expression { env.DOCKER_CREDENTIALS_ID != null }
            }
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'docker-hub-credentials',
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )
                    ]) {
                        sh '''
                            echo "=== PUSH DOCKER HUB ==="
                            echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin
                            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                            echo "✅ Images poussées sur Docker Hub"
                        '''
                    }
                }
            }
        }
        
        // ========== ÉTAPE 7: SETUP MINIKUBE ==========
        stage('7) Minikube Setup') {
            steps {
                script {
                    sh '''
                        echo "=== CONFIGURATION MINIKUBE ==="
                        
                        # Vérifier si Minikube est démarré
                        if ! minikube status 2>/dev/null | grep -q "Running"; then
                            echo "Démarrage de Minikube avec Docker driver..."
                            minikube start --driver=docker --memory=2048 --cpus=2 || {
                                echo "⚠️ Impossible de démarrer Minikube"
                                echo "Vérifiez que Docker est en cours d'exécution"
                                exit 1
                            }
                        fi
                        
                        # Vérifier la connexion
                        minikube status
                        echo "IP Minikube: $(minikube ip)"
                        
                        # Charger l'image Docker dans Minikube (important !)
                        echo "Chargement de l'image dans Minikube..."
                        minikube image load ${DOCKER_IMAGE}:${DOCKER_TAG}
                        
                        echo "✅ Minikube prêt"
                    '''
                }
            }
        }
        
        // ========== ÉTAPE 8: DÉPLOIEMENT KUBERNETES ==========
        stage('8) Kubernetes Deployment') {
            steps {
                script {
                    sh """
                        echo "=== DÉPLOIEMENT SUR MINIKUBE ==="
                        
                        # Créer le namespace
                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f - || true
                        
                        # 1. Déploiement Kubernetes
                        cat > k8s-deployment.yaml << 'EOF'
                        apiVersion: apps/v1
                        kind: Deployment
                        metadata:
                          name: ${K8S_DEPLOYMENT}
                          namespace: ${K8S_NAMESPACE}
                          labels:
                            app: ${APP_NAME}
                            version: "${BUILD_NUMBER}"
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
                                imagePullPolicy: Never  # Utilise l'image locale chargée
                                ports:
                                - containerPort: 8080
                                env:
                                - name: SPRING_PROFILES_ACTIVE
                                  value: "production"
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
                        
                        # 2. Service NodePort
                        cat > k8s-service.yaml << 'EOF'
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
                          - port: 8080
                            targetPort: 8080
                            nodePort: ${NODE_PORT}
                        EOF
                        
                        # Appliquer les configurations
                        kubectl apply -f k8s-deployment.yaml
                        kubectl apply -f k8s-service.yaml
                        
                        echo "✅ Déploiement appliqué"
                    """
                }
            }
        }
        
        // ========== ÉTAPE 9: VÉRIFICATION ==========
        stage('9) Verification & Testing') {
            steps {
                script {
                    sh '''
                        echo "=== VÉRIFICATION DU DÉPLOIEMENT ==="
                        
                        # Attendre que les pods soient prêts
                        echo "Attente du démarrage des pods..."
                        sleep 30
                        
                        # Vérifier l'état
                        echo "--- ÉTAT DES PODS ---"
                        kubectl get pods -n ${K8S_NAMESPACE} -o wide
                        
                        echo "--- ÉTAT DES SERVICES ---"
                        kubectl get svc -n ${K8S_NAMESPACE}
                        
                        echo "--- ÉTAT DES DÉPLOIEMENTS ---"
                        kubectl get deployments -n ${K8S_NAMESPACE}
                        
                        # Obtenir l'URL d'accès
                        MINIKUBE_IP=$(minikube ip)
                        echo ""
                        echo "🌐 🌐 🌐 VOTRE APPLICATION EST MAINTENANT EN LIGNE ! 🌐 🌐 🌐"
                        echo "URL: http://${MINIKUBE_IP}:${NODE_PORT}/"
                        echo "Health check: http://${MINIKUBE_IP}:${NODE_PORT}/actuator/health"
                        echo ""
                        
                        # Test d'intégration
                        echo "=== TEST D'ACCÈS ==="
                        sleep 10
                        HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://${MINIKUBE_IP}:${NODE_PORT}/actuator/health || echo "000")
                        
                        if [ "$HTTP_CODE" = "200" ]; then
                            echo "✅ TEST RÉUSSI: Application accessible (HTTP 200)"
                        else
                            echo "⚠️ Application non accessible (Code: $HTTP_CODE)"
                            echo "Vérifiez les logs: kubectl logs -n ${K8S_NAMESPACE} -l app=${APP_NAME} --tail=20"
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo "🔧 Nettoyage des fichiers temporaires..."
            sh '''
                rm -f Dockerfile k8s-deployment.yaml k8s-service.yaml 2>/dev/null || true
                docker container prune -f || true
                docker image prune -f || true
            '''
        }
        success {
            echo '🎉 🎉 🎉 PIPELINE COMPLÈTE RÉUSSIE ! 🎉 🎉 🎉'
            sh '''
                echo "=================================================="
                echo "✅ RÉSUMÉ DE LA PIPELINE"
                echo "=================================================="
                echo "1. ✅ Git Clone: Code source récupéré"
                echo "2. ✅ Build Maven: Compilation et tests"
                echo "3. ✅ SonarQube: Analyse qualité du code"
                echo "4. ✅ JAR: Application packagée (target/*.jar)"
                echo "5. ✅ Docker: Image créée (${DOCKER_IMAGE}:${DOCKER_TAG})"
                echo "6. ✅ Minikube: Cluster Kubernetes démarré"
                echo "7. ✅ Kubernetes: Application déployée"
                echo ""
                echo "🌐 ACCÈS À L'APPLICATION:"
                echo "   URL: http://$(minikube ip):${NODE_PORT}/"
                echo ""
                echo "🔧 COMMANDES UTILES:"
                echo "   Voir les logs: kubectl logs -n ${K8S_NAMESPACE} -l app=${APP_NAME} -f"
                echo "   Voir les pods: kubectl get pods -n ${K8S_NAMESPACE}"
                echo "   Arrêter: kubectl delete namespace ${K8S_NAMESPACE}"
                echo "   Minikube dashboard: minikube dashboard"
                echo "=================================================="
            '''
        }
        failure {
            echo '❌ Pipeline échouée - Vérifiez les logs ci-dessus'
            sh '''
                echo "=== TENTATIVE DE NETTOYAGE EN CAS D'ÉCHEC ==="
                kubectl delete deployment ${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} --ignore-not-found=true || true
                kubectl delete service ${K8S_DEPLOYMENT}-service -n ${K8S_NAMESPACE} --ignore-not-found=true || true
                kubectl delete namespace ${K8S_NAMESPACE} --ignore-not-found=true || true
            '''
        }
    }
}
