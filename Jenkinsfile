pipeline {
    agent any
    
    environment {
        SONAR_TOKEN = credentials('jenkins_sonar')
        APP_PORT = '9090'
        DOCKER_REPO = 'mohamedderbel/student-management'
    }

    stages {
        // 1. CLONE
        stage('1) 📥 Clone du Code') {
            steps {
                echo "Étape 1/8 : Récupération du code source"
                checkout scm
                sh 'echo "📁 Repository cloné avec succès"'
                sh 'ls -la'
            }
        }

        // 2. BUILD
        stage('2) 🔨 Build Maven') {
            steps {
                echo "Étape 2/8 : Compilation du projet"
                sh 'mvn clean compile'
                echo "✅ Build Maven réussi"
            }
        }

        // 3. TESTS
        stage('3) 🧪 Tests Unitaires') {
            steps {
                echo "Étape 3/8 : Exécution des tests"
                sh 'mvn test'
                junit 'target/surefire-reports/*.xml'
                echo "✅ Tests exécutés avec succès"
            }
        }

        // 4. PACKAGE JAR
        stage('4) 📦 Package JAR') {
            steps {
                echo "Étape 4/8 : Génération du JAR"
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                echo "✅ JAR généré et archivé"
                sh 'ls -la target/*.jar'
            }
        }

        // 5. DOCKER BUILD
        stage('5) 🐳 Docker Build') {
            steps {
                echo "Étape 5/8 : Construction image Docker"
                script {
                    sh "docker build -t ${DOCKER_REPO}:${BUILD_NUMBER} ."
                    sh "docker build -t ${DOCKER_REPO}:latest ."
                    echo "✅ Images Docker construites"
                    sh "docker images | grep ${DOCKER_REPO} || true"
                }
            }
        }

        // 6. SONARQUBE ANALYSIS
        stage('6) 📊 Analyse SonarQube') {
            steps {
                echo "Étape 6/8 : Analyse qualité du code"
                withSonarQubeEnv('sonarqube') {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=student-management \
                          -Dsonar.host.url=http://192.168.136.129:9000 \
                          -Dsonar.login=${SONAR_TOKEN}
                    """
                }
                echo "✅ Analyse SonarQube terminée"
            }
        }

        // 7. QUALITY GATE
        stage('7) ✅ Quality Gate') {
            steps {
                echo "Étape 7/8 : Vérification qualité"
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
                echo "✅ Quality Gate passé"
            }
        }

        // 8. DOCKER PUSH & DEPLOY
        stage('8) 🚀 Docker Push & Deploy') {
            steps {
                script {
                    echo "Étape 8/8 : Déploiement Docker"
                    echo "👤 Username: Mohamed Derbel"
                    echo "🔑 Password: user123@Med"
                    echo "📦 Repository: ${DOCKER_REPO}"
                    
                    // Login Docker Hub avec mot de passe CORRECT
                    sh '''
                        echo "user123@Med" | docker login -u "Mohamed Derbel" --password-stdin && \
                        echo "✅ Login Docker Hub réussi" || \
                        echo "❌ Login Docker Hub échoué"
                    '''
                    
                    // Push vers Docker Hub
                    sh """
                        echo "📤 Push de l'image version ${BUILD_NUMBER}..."
                        docker push ${DOCKER_REPO}:${BUILD_NUMBER} && \
                        echo "✅ Push version ${BUILD_NUMBER} réussi" || \
                        echo "⚠️  Push version ${BUILD_NUMBER} échoué"
                        
                        echo "📤 Push de l'image latest..."
                        docker push ${DOCKER_REPO}:latest && \
                        echo "✅ Push latest réussi" || \
                        echo "⚠️  Push latest échoué"
                    """
                    
                    // Arrêt conteneur existant
                    sh 'docker rm -f student-app 2>/dev/null || echo "ℹ️  Aucun conteneur existant à arrêter"'
                    
                    // Déploiement local
                    sh """
                        echo "🚀 Déploiement de l'application..."
                        docker run -d \
                            --name student-app \
                            -p ${APP_PORT}:8080 \
                            ${DOCKER_REPO}:latest && \
                        echo "✅ Conteneur démarré" || \
                        echo "❌ Échec du démarrage du conteneur"
                    """
                    
                    // Vérification
                    sh """
                        echo "🔍 Vérification du déploiement..."
                        sleep 15
                        
                        # Vérification 1: Le conteneur tourne
                        docker ps | grep student-app && \
                        echo "✅ Conteneur en cours d'exécution" || \
                        echo "❌ Conteneur non démarré"
                        
                        # Vérification 2: L'application répond
                        if curl -s -f http://localhost:${APP_PORT}/actuator/health > /dev/null 2>&1; then
                            echo "✅ Application fonctionnelle sur http://localhost:${APP_PORT}"
                            echo "✅ Application accessible sur http://192.168.136.129:${APP_PORT}"
                        else
                            echo "⚠️  L'application ne répond pas sur le port ${APP_PORT}"
                            echo "📋 Logs du conteneur:"
                            docker logs student-app --tail 20 2>/dev/null || echo "Impossible de récupérer les logs"
                        fi
                    """
                }
            }
        }
    }

    post {
        always {
            echo "=========================================="
            echo "               RÉSUMEX FINAL               "
            echo "=========================================="
            echo "🏷️  Build: #${BUILD_NUMBER}"
            echo "📊 Statut: ${currentBuild.currentResult}"
            echo "🔌 Port d'application: ${APP_PORT}"
            echo "🌐 URL: http://192.168.136.129:${APP_PORT}"
            echo "🐳 Docker Hub: https://hub.docker.com/r/${DOCKER_REPO}"
            echo "📦 Images: ${DOCKER_REPO}:${BUILD_NUMBER} et :latest"
            echo "=========================================="
            
            // Nettoyage
            sh 'docker image prune -f 2>/dev/null || true'
        }
        success {
            echo "🎉🎉🎉 FÉLICITATIONS ! 🎉🎉🎉"
            echo "✅ PIPELINE COMPLÈTE RÉUSSIE"
            echo "✅ Tous les 8 stages exécutés"
            echo "✅ Application déployée avec succès"
            echo "✅ Images Docker disponibles sur Docker Hub"
        }
        failure {
            echo "❌❌❌ PIPELINE ÉCHOUÉE ❌❌❌"
            echo "🔍 Analyse des logs recommandée"
            echo "🔄 Relancez après correction des erreurs"
            
            // Affichage des erreurs courantes
            sh '''
                echo "🔧 Diagnostic rapide:"
                docker --version 2>/dev/null || echo "❌ Docker non installé"
                mvn --version 2>/dev/null || echo "❌ Maven non installé"
                java --version 2>/dev/null || echo "❌ Java non installé"
                curl --version 2>/dev/null || echo "❌ curl non installé"
            '''
        }
    }
}
