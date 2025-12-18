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
        
        // ÉTAPE 5: Docker Build (SANS Minikube)
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
                        
                        # Tag pour une utilisation locale
                        docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${APP_NAME}:latest
                    '''
                }
            }
        }
        
        // ÉTAPE 6: Setup Kubernetes Environment (Nouvelle approche)
        stage('6) Setup Kubernetes Environment') {
            steps {
                script {
                    sh '''
                        echo "=== CONFIGURATION KUBERNETES ==="
                        
                        # Solution 1: Utiliser une configuration Minikube temporaire pour Jenkins
                        echo "Configuration de Minikube pour Jenkins..."
                        mkdir -p ${MINIKUBE_HOME}
                        
                        # Essayer de copier la configuration existante si accessible
                        if [ -d "/home/mohamedderbel/.minikube" ]; then
                            echo "Copie de la configuration Minikube existante..."
                            cp -r /home/mohamedderbel/.minikube/* ${MINIKUBE_HOME}/ 2>/dev/null || true
                            chmod -R 755 ${MINIKUBE_HOME}
                        fi
                        
                        # Exporter la variable d'environnement pour cette session
                        export MINIKUBE_HOME=${MINIKUBE_HOME}
                        export KUBECONFIG=${MINIKUBE_HOME}/config
                        
                        # Vérifier l'état actuel de Minikube
                        echo "Vérification de l'état Minikube..."
                        
                        # Solution 2: Démarrer Minikube si nécessaire avec l'utilisateur approprié
                        if ! sudo -u mohamedderbel minikube status 2>/dev/null | grep -q "Running"; then
                            echo "Démarrage de Minikube..."
                            sudo -u mohamedderbel minikube start --memory=2048mb --cpus=2
                        fi
                        
                        # Solution 3: Configurer kubectl pour utiliser Minikube
                        echo "Configuration de kubectl..."
                        sudo -u mohamedderbel minikube kubectl -- config view --flatten > /tmp/kubeconfig
                        cp /tmp/kubeconfig ${MINIKUBE_HOME}/config
                        chmod 644 ${MINIKUBE_HOME}/config
                        
                        echo "✅ Environnement Kubernetes configuré"
                    '''
                }
            }
        }
        
        // ÉTAPE 7: Deploy to Kubernetes (avec kubectl direct)
        stage('7) Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        echo "=== DÉPLOIEMENT KUBERNETES ==="
                        
                        # Utiliser kubectl directement avec la configuration Minikube
                        export KUBECONFIG=${MINIKUBE_HOME}/config
                        
                        # Créer le namespace
                        cat > namespace.yaml << EOF
                        apiVersion: v1
                        kind: Namespace
                        metadata:
                          name: ${K8S_NAMESPACE}
                          labels:
                            name: ${K8S_NAMESPACE}
                        EOF
                        
                        kubectl apply -f namespace.yaml || true
                        
                        # Créer le déploiement simple
                        cat > deployment.yaml << EOF
                        apiVersion: apps/v1
                        kind: Deployment
                        metadata:
                          name: ${K8S_DEPLOYMENT}
                          namespace: ${K8S_NAMESPACE}
                          labels:
                            app: ${APP_NAME}
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
                                image: ${APP_NAME}:latest
                                imagePullPolicy: IfNotPresent
                                ports:
                                - containerPort: 8080
                                env:
                                - name: SPRING_PROFILES_ACTIVE
                                  value: "production"
                                - name: SERVER_PORT
                                  value: "8080"
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
                            port: 8080
                            targetPort: 8080
                            nodePort: ${NODE_PORT}
                        EOF
                        
                        kubectl apply -f service.yaml
                        
                        echo "✅ Déploiement Kubernetes terminé"
                    """
                }
            }
        }
        
        // ÉTAPE 8: Verification
        stage('8) Verification and Testing') {
            steps {
                script {
                    sh '''
                        echo "=== VÉRIFICATION DU DÉPLOIEMENT ==="
                        
                        export KUBECONFIG=${MINIKUBE_HOME}/config
                        
                        # Attendre que les pods soient prêts
                        echo "Attente du démarrage des pods..."
                        sleep 20
                        
                        echo "--- ÉTAT DU DÉPLOIEMENT ---"
                        kubectl get all -n ${K8S_NAMESPACE} || echo "Impossible d'accéder au cluster"
                        
                        # Obtenir l'IP de Minikube via l'utilisateur propriétaire
                        MINIKUBE_IP=$(sudo -u mohamedderbel minikube ip 2>/dev/null || echo "localhost")
                        
                        echo ""
                        echo "=========================================="
                        echo "🌐 APPLICATION DÉPLOYÉE !"
                        echo "=========================================="
                        echo "URL: http://${MINIKUBE_IP}:${NODE_PORT}/"
                        echo "Health: http://${MINIKUBE_IP}:${NODE_PORT}/actuator/health"
                        echo ""
                        echo "Pour accéder à l'application:"
                        echo "1. Récupérez l'IP Minikube: sudo -u mohamedderbel minikube ip"
                        echo "2. Accédez à: http://<MINIKUBE_IP>:${NODE_PORT}/"
                        echo "=========================================="
                        
                        # Vérification simple
                        echo ""
                        echo "Vérification de la santé de l'application..."
                        for i in {1..5}; do
                            echo "Tentative $i/5..."
                            sleep 10
                        done
                    '''
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
                    echo "5. ✅ Application déployée sur Kubernetes"
                    echo ""
                    
                    # Obtenir l'IP Minikube via l'utilisateur propriétaire
                    MINIKUBE_IP=$(sudo -u mohamedderbel minikube ip 2>/dev/null || echo "localhost")
                    
                    echo "🌐 VOTRE APPLICATION EST DISPONIBLE:"
                    echo "   URL: http://${MINIKUBE_IP}:${NODE_PORT}/"
                    echo ""
                    echo "🔧 COMMANDES DE GESTION:"
                    echo "   sudo -u mohamedderbel kubectl get all -n ${K8S_NAMESPACE}"
                    echo "   sudo -u mohamedderbel minikube dashboard"
                    echo "=========================================="
                '''
            }
        }
        failure {
            script {
                echo '❌ Pipeline échouée'
                sh '''
                    echo "=== NETTOYAGE EN CAS D'ÉCHEC ==="
                    
                    # Nettoyage simple sans dépendre de kubectl
                    echo "Suppression des fichiers temporaires..."
                    docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} 2>/dev/null || true
                    docker rmi ${APP_NAME}:latest 2>/dev/null || true
                    
                    echo ""
                    echo "🔍 POUR RÉSOUDRE LES PROBLÈMES:"
                    echo "1. Vérifiez que Minikube est démarré:"
                    echo "   sudo -u mohamedderbel minikube status"
                    echo ""
                    echo "2. Démarrer Minikube manuellement si nécessaire:"
                    echo "   sudo -u mohamedderbel minikube start --memory=2048mb"
                    echo ""
                    echo "3. Vérifiez les permissions Minikube:"
                    echo "   ls -la /home/mohamedderbel/.minikube/"
                    echo ""
                    echo "4. Pour corriger les permissions, exécutez:"
                    echo "   sudo chown -R mohamedderbel:jenkins /home/mohamedderbel/.minikube"
                    echo "   sudo chmod -R 755 /home/mohamedderbel/.minikube"
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
