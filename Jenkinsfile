pipeline {
    agent any
    
    environment {
        APP_NAME = 'student-management'
        DOCKER_IMAGE = 'student-management-app'
        DOCKER_TAG = "latest-${BUILD_NUMBER}"
        K8S_NAMESPACE = 'student-namespace'
        K8S_DEPLOYMENT = 'student-deployment'
        NODE_PORT = '30080'
        MINIKUBE_HOME = '/tmp/minikube-jenkins'
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
        
        // ÉTAPE 2: Build Application
        stage('2) Build Application') {
            steps {
                sh '''
                    echo "=== COMPILATION ==="
                    mvn clean compile
                    echo "✅ Application compilée"
                '''
            }
        }
        
        // ÉTAPE 3: Build JAR
        stage('3) Build JAR Package') {
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
        
        // ÉTAPE 4: Build Docker Image
        stage('4) Build Docker Image') {
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
        
        // ÉTAPE 5: Setup Minikube Environment (SANS SUDO)
        stage('5) Setup Minikube Environment') {
            steps {
                script {
                    sh '''
                        echo "=== CONFIGURATION MINIKUBE ==="
                        
                        # Créer un environnement Minikube temporaire pour Jenkins
                        mkdir -p ${MINIKUBE_HOME}
                        
                        # Vérifier si Minikube est accessible
                        if [ -x "$(command -v minikube)" ]; then
                            echo "Minikube est installé"
                            
                            # Essayer de démarrer Minikube sans sudo
                            echo "Tentative de démarrage de Minikube..."
                            
                            # Option A: Utiliser le driver Docker (recommandé)
                            MINIKUBE_START_CMD="minikube start --driver=docker --memory=2048mb --cpus=2"
                            
                            # Essayer différentes approches
                            echo "Approche 1: Démarrer Minikube normalement..."
                            if ${MINIKUBE_START_CMD} 2>/dev/null; then
                                echo "✅ Minikube démarré avec succès"
                            else
                                echo "⚠️ Impossible de démarrer Minikube normalement"
                                
                                # Option B: Vérifier si Minikube tourne déjà
                                if minikube status 2>/dev/null | grep -q "Running"; then
                                    echo "✅ Minikube est déjà en cours d'exécution"
                                else
                                    echo "⚠️ Minikube ne fonctionne pas avec l'utilisateur Jenkins"
                                    echo "Solution: Démarrer Minikube manuellement avant d'exécuter le pipeline"
                                fi
                            fi
                        else
                            echo "⚠️ Minikube n'est pas installé ou non accessible"
                        fi
                        
                        # Configurer kubectl pour utiliser Minikube
                        echo "Configuration de kubectl..."
                        if [ -x "$(command -v kubectl)" ]; then
                            # Essayer de récupérer la configuration
                            if minikube kubectl -- config view 2>/dev/null; then
                                minikube kubectl -- config view --flatten > ${MINIKUBE_HOME}/config
                                echo "✅ Configuration kubectl générée"
                            else
                                # Créer une configuration de secours
                                cat > ${MINIKUBE_HOME}/config << EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://localhost:8443
    insecure-skip-tls-verify: true
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
users:
- name: minikube
EOF
                                echo "⚠️ Configuration kubectl par défaut créée"
                            fi
                        fi
                        
                        echo "Environnement préparé"
                    '''
                }
            }
        }
        
        // ÉTAPE 6: Deploy with Docker Compose (Alternative à Kubernetes)
        stage('6) Deploy with Docker Compose') {
            steps {
                script {
                    sh '''
                        echo "=== DÉPLOIEMENT AVEC DOCKER COMPOSE ==="
                        
                        # Créer un docker-compose.yml
                        cat > docker-compose.yml << EOF
version: '3.8'
services:
  ${APP_NAME}:
    image: ${DOCKER_IMAGE}:${DOCKER_TAG}
    container_name: ${APP_NAME}
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=production
      - SERVER_PORT=8080
    restart: unless-stopped
    networks:
      - student-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  student-network:
    driver: bridge
EOF
                        
                        # Arrêter et supprimer les anciens conteneurs
                        echo "Nettoyage des anciens conteneurs..."
                        docker-compose down 2>/dev/null || true
                        
                        # Démarrer l'application
                        echo "Démarrage de l'application..."
                        docker-compose up -d
                        
                        echo "✅ Application déployée avec Docker Compose"
                    '''
                }
            }
        }
        
        // ÉTAPE 7: Verification
        stage('7) Verification and Testing') {
            steps {
                script {
                    sh '''
                        echo "=== VÉRIFICATION ==="
                        
                        # Attendre le démarrage
                        echo "Attente du démarrage de l'application..."
                        sleep 30
                        
                        # Vérifier les conteneurs
                        echo "--- ÉTAT DES CONTENEURS ---"
                        docker ps --filter "name=${APP_NAME}"
                        
                        # Vérifier la santé
                        echo "--- VÉRIFICATION DE SANTÉ ---"
                        for i in {1..10}; do
                            echo "Tentative $i/10..."
                            HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/actuator/health || echo "000")
                            
                            if [ "$HTTP_CODE" = "200" ]; then
                                echo "✅ Application accessible (HTTP 200)"
                                
                                # Afficher les logs de démarrage
                                echo "--- DERNIERS LOGS ---"
                                docker logs ${APP_NAME} --tail=20
                                break
                            else
                                echo "⏳ Application en cours de démarrage (Code: $HTTP_CODE)"
                                sleep 10
                            fi
                        done
                        
                        if [ "$HTTP_CODE" != "200" ]; then
                            echo "⚠️ Application non accessible après 10 tentatives"
                            echo "Logs complets:"
                            docker logs ${APP_NAME}
                        fi
                        
                        echo ""
                        echo "=========================================="
                        echo "🌐 APPLICATION DÉPLOYÉE !"
                        echo "=========================================="
                        echo "URL: http://localhost:8080/"
                        echo "Health: http://localhost:8080/actuator/health"
                        echo ""
                        echo "🔧 COMMANDES:"
                        echo "   docker logs ${APP_NAME} -f"
                        echo "   docker-compose down"
                        echo "=========================================="
                    '''
                }
            }
        }
        
        // ÉTAPE 8: Optional - Kubernetes if Minikube works
        stage('8) Optional - Kubernetes Deployment') {
            when {
                expression {
                    // Cette étape ne s'exécute que si Minikube fonctionne
                    return fileExists("${env.MINIKUBE_HOME}/config")
                }
            }
            steps {
                script {
                    sh '''
                        echo "=== DÉPLOIEMENT KUBERNETES (OPTIONNEL) ==="
                        
                        export KUBECONFIG=${MINIKUBE_HOME}/config
                        
                        # Créer namespace
                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Créer déploiement
                        cat > k8s-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${K8S_DEPLOYMENT}
  namespace: ${K8S_NAMESPACE}
spec:
  replicas: 1
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
        ports:
        - containerPort: 8080
EOF
                        
                        kubectl apply -f k8s-deployment.yaml
                        
                        # Créer service
                        cat > k8s-service.yaml << EOF
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
                        
                        kubectl apply -f k8s-service.yaml
                        
                        echo "✅ Déploiement Kubernetes terminé"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh '''
                echo "=== NETTOYAGE ==="
                rm -f Dockerfile docker-compose.yml k8s-deployment.yaml k8s-service.yaml 2>/dev/null || true
                echo "Fichiers temporaires nettoyés"
            '''
        }
        success {
            echo '🎉 PIPELINE RÉUSSIE !'
            sh '''
                echo ""
                echo "=========================================="
                echo "✅ RÉSUMÉ FINAL"
                echo "=========================================="
                echo "1. ✅ Code source récupéré"
                echo "2. ✅ Application compilée"
                echo "3. ✅ JAR généré et archivé"
                echo "4. ✅ Image Docker créée"
                echo "5. ✅ Application déployée avec Docker Compose"
                echo ""
                echo "🌐 VOTRE APPLICATION EST DISPONIBLE:"
                echo "   URL: http://localhost:8080/"
                echo "   Health: http://localhost:8080/actuator/health"
                echo ""
                echo "🔧 COMMANDES DE GESTION:"
                echo "   docker logs student-management -f"
                echo "   docker-compose down"
                echo "=========================================="
            '''
        }
        failure {
            echo '❌ Pipeline échouée'
            sh '''
                echo "=== NETTOYAGE EN CAS D'ÉCHEC ==="
                docker-compose down 2>/dev/null || true
                docker rm -f ${APP_NAME} 2>/dev/null || true
                
                echo ""
                echo "🔍 POUR RÉSOUDRE LES PROBLÈMES:"
                echo "1. Vérifiez que Docker fonctionne:"
                echo "   docker ps"
                echo ""
                echo "2. Pour utiliser Kubernetes, démarrer Minikube manuellement:"
                echo "   minikube start --driver=docker --memory=2048mb"
                echo ""
                echo "3. Pour configurer sudo sans mot de passe pour Jenkins:"
                echo "   sudo visudo"
                echo "   # Ajouter: jenkins ALL=(ALL) NOPASSWD: ALL"
            '''
        }
    }
}
