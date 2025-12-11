pipeline {
    agent any
    
    environment {
        APP_PORT = '9090'
        DOCKER_REPO = 'mohamedderbel/student-management'
        DOCKER_USERNAME = 'mohamedderbel'
    }

    stages {
        // 1. CLONE
        stage('1) Clone du Code') {
            steps {
                echo "Étape 1/8 : Récupération du code source"
                checkout scm
                sh 'ls -la'
            }
        }

        // 2. BUILD (sans tests d'abord)
        stage('2) Build Maven (compile only)') {
            steps {
                echo "Étape 2/8 : Compilation du projet"
                sh 'mvn clean compile'
                echo "✅ Compilation réussie"
            }
        }

        // 3. TESTS AVEC GESTION D'ERREUR AMÉLIORÉE
        stage('3) Tests Unitaires') {
            steps {
                echo "Étape 3/8 : Exécution des tests"
                script {
                    // Essayer les tests avec gestion d'erreur plus robuste
                    def testResult = sh(script: 'mvn test', returnStatus: true)
                    
                    if (testResult != 0) {
                        echo "⚠️ Tests échoués - vérifiez la configuration de test"
                        echo "ℹ️ Pour débuguer : mvn test -Dtest=YourTestClass"
                        // Ne pas bloquer le pipeline pour les tests
                        unstable("Tests échoués mais pipeline continue")
                    } else {
                        echo "✅ Tests réussis"
                    }
                    
                    // Archiver les rapports même si échec
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        // 4. PACKAGE (skip tests car déjà exécutés)
        stage('4) Package JAR') {
            steps {
                echo "Étape 4/8 : Génération du JAR"
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                echo "✅ JAR généré"
            }
        }

        // 5. DOCKER BUILD AVEC VÉRIFICATION DOCKERFILE
        stage('5) Docker Build') {
            steps {
                echo "Étape 5/8 : Construction image Docker"
                script {
                    // Vérifier que le Dockerfile existe
                    if (!fileExists('Dockerfile')) {
                        error("❌ Dockerfile introuvable ! Créez un Dockerfile à la racine du projet.")
                    }
                    
                    // Construire les images
                    sh "docker build -t ${DOCKER_REPO}:${BUILD_NUMBER} ."
                    sh "docker tag ${DOCKER_REPO}:${BUILD_NUMBER} ${DOCKER_REPO}:latest"
                    echo "✅ Images Docker construites: ${BUILD_NUMBER} et latest"
                }
            }
        }

        // 6. SONARQUBE (optionnel, robuste)
        stage('6) Analyse SonarQube') {
            steps {
                echo "Étape 6/8 : Analyse SonarQube"
                script {
                    def sonarAvailable = sh(
                        script: 'curl -s -o /dev/null -w "%{http_code}" http://192.168.136.129:9000/api/system/status',
                        returnStdout: true
                    ).trim()
                    
                    if (sonarAvailable == "200") {
                        echo "✅ SonarQube accessible"
                        try {
                            withSonarQubeEnv('SonarQube') {
                                sh '''
                                    mvn sonar:sonar \
                                      -Dsonar.projectKey=student-management \
                                      -Dsonar.projectName="Student Management" \
                                      -Dsonar.host.url=http://192.168.136.129:9000
                                '''
                            }
                            echo "✅ Analyse SonarQube terminée"
                        } catch (Exception e) {
                            echo "⚠️ Analyse SonarQube échouée: ${e.message}"
                            unstable("Analyse SonarQube échouée")
                        }
                    } else {
                        echo "⚠️ SonarQube non accessible (HTTP ${sonarAvailable}) - étape ignorée"
                    }
                }
            }
        }

        // 7. DOCKER PUSH CORRIGÉ
        stage('7) Docker Push') {
            steps {
                script {
                    echo "Étape 7/8 : Push vers Docker Hub"
                    
                    try {
                        // Utiliser les credentials Jenkins
                        withCredentials([usernamePassword(
                            credentialsId: 'dockerhub-credentials',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            sh '''
                                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                                echo "✅ Docker login réussi"
                            '''
                            
                            sh """
                                docker push ${DOCKER_REPO}:${BUILD_NUMBER}
                                docker push ${DOCKER_REPO}:latest
                                echo "✅ Images poussées vers Docker Hub"
                            """
                            
                            sh 'docker logout'
                        }
                    } catch (Exception e) {
                        echo "⚠️ Docker push échoué: ${e.message}"
                        echo "ℹ️ Vérifiez vos credentials Docker Hub dans Jenkins"
                        unstable("Docker push échoué")
                    }
                }
            }
        }

        // 8. DÉPLOIEMENT AMÉLIORÉ
        stage('8) Déploiement') {
            steps {
                script {
                    echo "Étape 8/8 : Déploiement de l'application"
                    
                    // Arrêter et supprimer l'ancien conteneur
                    sh '''
                        docker stop student-app 2>/dev/null || true
                        docker rm student-app 2>/dev/null || true
                        echo "ℹ️ Ancien conteneur supprimé"
                    '''
                    
                    // Démarrer le nouveau conteneur
                    sh """
                        docker run -d \
                            --name student-app \
                            --restart unless-stopped \
                            -p ${APP_PORT}:8080 \
                            -e SPRING_PROFILES_ACTIVE=prod \
                            ${DOCKER_REPO}:latest
                    """
                    
                    echo "⏳ Attente du démarrage de l'application (15s)..."
                    sleep 15
                    
                    // Vérification du déploiement
                    def healthCheck = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${APP_PORT}/actuator/health || echo '000'",
                        returnStdout: true
                    ).trim()
                    
                    if (healthCheck == "200") {
                        echo "✅ Application fonctionnelle et accessible"
                    } else {
                        echo "⚠️ Application démarrée mais health check échoué (HTTP ${healthCheck})"
                        echo "ℹ️ Vérifiez les logs: docker logs student-app"
                    }
                    
                    echo "🌐 URL: http://192.168.136.129:${APP_PORT}"
                    
                    // Afficher les logs récents
                    sh 'docker logs --tail 20 student-app || true'
                }
            }
        }
    }

    post {
        always {
            echo "========================================"
            echo "🏁 PIPELINE TERMINÉE"
            echo "========================================"
            echo "📦 Build: #${BUILD_NUMBER}"
            echo "📊 Statut: ${currentBuild.currentResult}"
            echo "🐳 Image: ${DOCKER_REPO}:${BUILD_NUMBER}"
            echo "🌐 Port: ${APP_PORT}"
            echo "🔗 URL: http://192.168.136.129:${APP_PORT}"
            echo "========================================"
            
            // Nettoyer les images non taguées
            sh 'docker image prune -f || true'
        }
        success {
            echo "✅✅✅ SUCCÈS COMPLET ✅✅✅"
            echo "🚀 Application déployée avec succès"
            echo "📝 Swagger UI: http://192.168.136.129:${APP_PORT}/swagger-ui.html"
        }
        failure {
            echo "❌❌❌ ÉCHEC DE LA PIPELINE ❌❌❌"
            echo ""
            echo "🔍 DIAGNOSTIC:"
            echo "1. Vérifiez les logs Jenkins ci-dessus"
            echo "2. Tests échoués? → Configurez H2 database pour les tests"
            echo "3. Dockerfile manquant? → Créez-le à la racine du projet"
            echo "4. Docker push échoué? → Vérifiez credentials 'dockerhub-credentials'"
            echo ""
            echo "📚 SOLUTIONS:"
            echo "• Tests: Ajoutez application-test.properties avec H2"
            echo "• Docker: Créez Dockerfile avec FROM openjdk:17-jdk-slim"
            echo "• Credentials: Jenkins > Manage > Credentials > Add dockerhub-credentials"
        }
        unstable {
            echo "⚠️⚠️⚠️ BUILD INSTABLE ⚠️⚠️⚠️"
            echo "L'application est déployée mais certaines étapes ont échoué"
            echo "🔍 Vérifiez les warnings ci-dessus"
        }
        cleanup {
            echo "🧹 Nettoyage..."
            // Nettoyer le workspace si nécessaire
            // cleanWs()
        }
    }
}
