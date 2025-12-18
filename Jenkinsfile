pipeline {
    agent any
    
    environment {
        APP_NAME = 'student-management'
        DOCKER_IMAGE = 'student-management-app'
        DOCKER_TAG = "${BUILD_NUMBER}"
        SONARQUBE_URL = 'http://localhost:9000'
    }
    
    stages {
        // ÉTAPE 1: Récupération du code
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
        
        // ÉTAPE 2: Compilation
        stage('2) Build Application') {
            steps {
                sh '''
                    echo "=== COMPILATION ==="
                    mvn clean compile
                    echo "✅ Application compilée"
                '''
            }
        }
        
        // ÉTAPE 3: Analyse SonarQube
        stage('3) SonarQube Analysis') {
            steps {
                script {
                    echo "=== ANALYSE SONARQUBE ==="
                    
                    // Test de connexion à SonarQube
                    try {
                        sh 'timeout 10 curl -s -f http://localhost:9000 > /dev/null'
                        echo "SonarQube accessible"
                    } catch (Exception e) {
                        echo "⚠️ SonarQube non accessible - étape ignorée"
                        return
                    }
                    
                    // Exécution de l'analyse
                    withCredentials([string(credentialsId: 'SONARQUBE_TOKEN', variable: 'SONAR_TOKEN')]) {
                        sh """
                            mvn sonar:sonar \
                                -Dsonar.projectKey=${APP_NAME} \
                                -Dsonar.host.url=${SONARQUBE_URL} \
                                -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                    echo "✅ Analyse SonarQube terminée"
                }
            }
        }
        
        // ÉTAPE 4: Génération du JAR
        stage('4) Build JAR') {
            steps {
                sh '''
                    echo "=== GÉNÉRATION DU JAR ==="
                    mvn clean package -DskipTests
                    echo "=== FICHIER GÉNÉRÉ ==="
                    ls -lh target/*.jar
                    echo "Taille: $(du -h target/*.jar | cut -f1)"
                '''
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                sh 'echo "✅ JAR archivé"'
            }
        }
        
        // ÉTAPE 5: Construction de l'image Docker
        stage('5) Docker Build') {
            steps {
                script {
                    sh '''
                        echo "=== CONSTRUCTION DOCKER ==="
                        
                        # Créer Dockerfile
                        cat > Dockerfile << 'EOF'
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/student-management-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
EOF
                        
                        # Construire l'image
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                        
                        echo "✅ Images créées:"
                        docker images | grep ${DOCKER_IMAGE}
                    '''
                }
            }
        }
        
        // ÉTAPE 6: Déploiement Docker
        stage('6) Docker Deploy') {
            steps {
                script {
                    sh '''
                        echo "=== DÉPLOIEMENT ==="
                        
                        # Arrêter l'ancien conteneur
                        docker stop ${APP_NAME} 2>/dev/null || true
                        docker rm ${APP_NAME} 2>/dev/null || true
                        
                        # Démarrer le nouveau conteneur
                        docker run -d \
                            --name ${APP_NAME} \
                            -p 8080:8080 \
                            --restart unless-stopped \
                            ${DOCKER_IMAGE}:latest
                        
                        echo "✅ Application déployée"
                    '''
                }
            }
        }
        
        // ÉTAPE 7: Vérification
        stage('7) Verification') {
            steps {
                script {
                    sh '''
                        echo "=== VÉRIFICATION ==="
                        
                        # Vérifier le conteneur
                        echo "--- CONTENEUR DOCKER ---"
                        docker ps --filter "name=${APP_NAME}"
                        
                        # Tester l'application
                        echo "--- TEST D'ACCÈS ---"
                        for i in {1..6}; do
                            echo "Tentative $i/6..."
                            if curl -s -f http://localhost:8080/actuator/health > /dev/null 2>&1; then
                                echo "✅ Application accessible"
                                echo "URL: http://localhost:8080"
                                echo "Health: http://localhost:8080/actuator/health"
                                break
                            else
                                sleep 10
                            fi
                        done
                        
                        # Afficher les logs
                        echo "--- LOGS RÉCENTS ---"
                        docker logs ${APP_NAME} --tail=10
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh '''
                echo "=== NETTOYAGE ==="
                rm -f Dockerfile 2>/dev/null || true
                echo "Fichiers temporaires supprimés"
            '''
        }
        success {
            echo '🎉 DÉPLOIEMENT RÉUSSI !'
            sh '''
                echo ""
                echo "========================================"
                echo "✅ APPLICATION DÉPLOYÉE"
                echo "========================================"
                echo "Application: ${APP_NAME}"
                echo "Version: ${DOCKER_TAG}"
                echo "URL: http://localhost:8080"
                echo "Health: http://localhost:8080/actuator/health"
                echo ""
                echo "🔧 COMMANDES UTILES:"
                echo "   Voir logs: docker logs ${APP_NAME} -f"
                echo "   Arrêter:   docker stop ${APP_NAME}"
                echo "   Redémarrer: docker restart ${APP_NAME}"
                echo "========================================"
            '''
        }
        failure {
            echo '❌ DÉPLOIEMENT ÉCHOUÉ'
            sh '''
                echo "=== NETTOYAGE EN CAS D'ERREUR ==="
                docker stop ${APP_NAME} 2>/dev/null || true
                docker rm ${APP_NAME} 2>/dev/null || true
                
                echo ""
                echo "🔍 POUR DÉBOGUER:"
                echo "1. Vérifier Docker: docker ps"
                echo "2. Vérifier le port: netstat -tlnp | grep 8080"
                echo "3. Vérifier les logs: docker logs ${APP_NAME}"
                echo "4. Tester manuellement: curl http://localhost:8080/actuator/health"
            '''
        }
    }
}
