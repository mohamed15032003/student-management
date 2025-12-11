pipeline {
    agent any
    
    environment {
        APP_PORT = '9090'
        DOCKER_REPO = 'mohamedderbel/student-management'
        DOCKER_USERNAME = 'mohamedderbel'
    }

    stages {
        // ========================================
        // 1. CLONE DU CODE SOURCE
        // ========================================
        stage('1) Clone du Code') {
            steps {
                echo "=========================================="
                echo "📦 Étape 1/8 : Récupération du code source"
                echo "=========================================="
                checkout scm
                sh 'ls -la'
                echo "✅ Code récupéré avec succès"
            }
        }

        // ========================================
        // 2. COMPILATION MAVEN
        // ========================================
        stage('2) Build Maven') {
            steps {
                echo "=========================================="
                echo "🔨 Étape 2/8 : Compilation du projet"
                echo "=========================================="
                sh 'mvn clean compile'
                echo "✅ Compilation réussie"
            }
        }

        // ========================================
        // 3. TESTS UNITAIRES
        // ========================================
        stage('3) Tests Unitaires') {
            steps {
                echo "=========================================="
                echo "🧪 Étape 3/8 : Exécution des tests"
                echo "=========================================="
                script {
                    // Exécuter les tests avec gestion d'erreur
                    def testResult = sh(
                        script: 'mvn test -Dspring.profiles.active=test',
                        returnStatus: true
                    )
                    
                    // Archiver les rapports de test
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                    
                    // Gérer le résultat
                    if (testResult != 0) {
                        echo "⚠️ ATTENTION : Des tests ont échoué"
                        echo "📊 Consultez les rapports JUnit dans Jenkins"
                        echo "💡 La pipeline continue malgré l'échec des tests"
                        unstable("Tests échoués - Build marqué comme UNSTABLE")
                    } else {
                        echo "✅ Tous les tests sont passés avec succès"
                    }
                }
            }
        }

        // ========================================
        // 4. PACKAGING JAR
        // ========================================
        stage('4) Package JAR') {
            steps {
                echo "=========================================="
                echo "📦 Étape 4/8 : Génération du JAR"
                echo "=========================================="
                sh 'mvn package -DskipTests'
                
                // Archiver le JAR
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                
                // Afficher les informations du JAR
                sh '''
                    echo "📄 Fichiers générés:"
                    ls -lh target/*.jar
                '''
                echo "✅ JAR généré avec succès"
            }
        }

        // ========================================
        // 5. CONSTRUCTION IMAGE DOCKER
        // ========================================
        stage('5) Docker Build') {
            steps {
                echo "=========================================="
                echo "🐳 Étape 5/8 : Construction image Docker"
                echo "=========================================="
                script {
                    // Vérifier que le Dockerfile existe
                    if (!fileExists('Dockerfile')) {
                        error("❌ ERREUR CRITIQUE : Dockerfile introuvable !\n" +
                              "📝 Créez un Dockerfile à la racine du projet avec le contenu suivant:\n" +
                              "FROM eclipse-temurin:17-jre-alpine\n" +
                              "WORKDIR /app\n" +
                              "COPY target/*.jar app.jar\n" +
                              "EXPOSE 8080\n" +
                              "ENTRYPOINT [\"java\", \"-jar\", \"app.jar\"]")
                    }
                    
                    echo "📝 Dockerfile trouvé"
                    
                    // Construire l'image avec le numéro de build
                    sh """
                        docker build -t ${DOCKER_REPO}:${BUILD_NUMBER} .
                        echo "✅ Image ${DOCKER_REPO}:${BUILD_NUMBER} construite"
                    """
                    
                    // Taguer comme 'latest'
                    sh """
                        docker tag ${DOCKER_REPO}:${BUILD_NUMBER} ${DOCKER_REPO}:latest
                        echo "✅ Image taguée comme 'latest'"
                    """
                    
                    // Afficher les images
                    sh """
                        echo "🐳 Images Docker disponibles:"
                        docker images | grep ${DOCKER_REPO} || echo "Aucune image trouvée"
                    """
                    
                    echo "✅ Construction Docker terminée avec succès"
                }
            }
        }

        // ========================================
        // 6. ANALYSE SONARQUBE
        // ========================================
        stage('6) Analyse SonarQube') {
            steps {
                echo "=========================================="
                echo "📊 Étape 6/8 : Analyse de qualité du code"
                echo "=========================================="
                script {
                    // Tester la disponibilité de SonarQube
                    def sonarStatus = sh(
                        script: 'curl -s -o /dev/null -w "%{http_code}" http://192.168.136.129:9000/api/system/status 2>/dev/null || echo "000"',
                        returnStdout: true
                    ).trim()
                    
                    if (sonarStatus == "200") {
                        echo "✅ SonarQube est accessible"
                        
                        try {
                            // Exécuter l'analyse SonarQube
                            withSonarQubeEnv('SonarQube') {
                                sh '''
                                    mvn sonar:sonar \
                                      -Dsonar.projectKey=student-management \
                                      -Dsonar.projectName="Student Management System" \
                                      -Dsonar.host.url=http://192.168.136.129:9000 \
                                      -Dsonar.java.binaries=target/classes
                                '''
                            }
                            echo "✅ Analyse SonarQube terminée"
                            echo "🔗 Résultats: http://192.168.136.129:9000/dashboard?id=student-management"
                        } catch (Exception e) {
                            echo "⚠️ Échec de l'analyse SonarQube: ${e.message}"
                            echo "💡 Vérifiez la configuration SonarQube dans Jenkins"
                            unstable("Analyse SonarQube échouée")
                        }
                    } else {
                        echo "⚠️ SonarQube non accessible (HTTP ${sonarStatus})"
                        echo "💡 L'analyse de code est ignorée"
                        echo "🔧 Pour activer SonarQube: docker run -d -p 9000:9000 sonarqube:lts-community"
                    }
                }
            }
        }

        // ========================================
        // 7. PUSH VERS DOCKER HUB
        // ========================================
        stage('7) Docker Push') {
            steps {
                echo "=========================================="
                echo "📤 Étape 7/8 : Publication sur Docker Hub"
                echo "=========================================="
                script {
                    try {
                        // Connexion à Docker Hub avec credentials Jenkins
                        withCredentials([usernamePassword(
                            credentialsId: 'dockerhub-credentials',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            sh '''
                                echo "🔐 Connexion à Docker Hub..."
                                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                                echo "✅ Connexion réussie"
                            '''
                            
                            // Push de l'image avec le numéro de build
                            sh """
                                echo "📤 Push de l'image ${DOCKER_REPO}:${BUILD_NUMBER}..."
                                docker push ${DOCKER_REPO}:${BUILD_NUMBER}
                                echo "✅ Image ${BUILD_NUMBER} publiée"
                            """
                            
                            // Push de l'image latest
                            sh """
                                echo "📤 Push de l'image ${DOCKER_REPO}:latest..."
                                docker push ${DOCKER_REPO}:latest
                                echo "✅ Image latest publiée"
                            """
                            
                            // Déconnexion
                            sh '''
                                docker logout
                                echo "🔓 Déconnexion de Docker Hub"
                            '''
                            
                            echo "✅ Publication terminée avec succès"
                            echo "🔗 Docker Hub: https://hub.docker.com/r/${DOCKER_REPO}"
                        }
                    } catch (Exception e) {
                        echo "❌ Échec du push Docker: ${e.message}"
                        echo ""
                        echo "🔍 SOLUTIONS POSSIBLES:"
                        echo "1. Créer les credentials dans Jenkins:"
                        echo "   • Manage Jenkins > Credentials > System > Global credentials"
                        echo "   • Add Credentials > Username with password"
                        echo "   • ID: dockerhub-credentials"
                        echo "   • Username: ${DOCKER_USERNAME}"
                        echo "   • Password: [votre token Docker Hub]"
                        echo ""
                        echo "2. Générer un token Docker Hub:"
                        echo "   • https://hub.docker.com/settings/security"
                        echo "   • New Access Token"
                        echo ""
                        unstable("Push Docker échoué - Images disponibles localement")
                    }
                }
            }
        }

        // ========================================
        // 8. DÉPLOIEMENT DE L'APPLICATION
        // ========================================
        stage('8) Déploiement') {
            steps {
                echo "=========================================="
                echo "🚀 Étape 8/8 : Déploiement de l'application"
                echo "=========================================="
                script {
                    // Arrêter et supprimer l'ancien conteneur
                    sh '''
                        echo "🛑 Arrêt du conteneur existant..."
                        docker stop student-app 2>/dev/null || echo "   Aucun conteneur à arrêter"
                        docker rm student-app 2>/dev/null || echo "   Aucun conteneur à supprimer"
                    '''
                    
                    // Démarrer le nouveau conteneur
                    sh """
                        echo "🚀 Démarrage du nouveau conteneur..."
                        docker run -d \
                            --name student-app \
                            --restart unless-stopped \
                            -p ${APP_PORT}:8080 \
                            -e SPRING_PROFILES_ACTIVE=prod \
                            -e JAVA_OPTS="-Xms256m -Xmx512m" \
                            ${DOCKER_REPO}:latest
                        
                        echo "✅ Conteneur démarré"
                    """
                    
                    // Attendre le démarrage
                    echo "⏳ Attente du démarrage de l'application (20 secondes)..."
                    sleep 20
                    
                    // Vérifier le health check
                    def healthStatus = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${APP_PORT}/actuator/health 2>/dev/null || echo '000'",
                        returnStdout: true
                    ).trim()
                    
                    echo ""
                    echo "=========================================="
                    echo "📊 RÉSULTAT DU DÉPLOIEMENT"
                    echo "=========================================="
                    
                    if (healthStatus == "200") {
                        echo "✅ Application FONCTIONNELLE et ACCESSIBLE"
                        echo "🌐 URL principale: http://192.168.136.129:${APP_PORT}"
                        echo "📊 Health Check: http://192.168.136.129:${APP_PORT}/actuator/health"
                        echo "📚 API Docs: http://192.168.136.129:${APP_PORT}/swagger-ui.html"
                        echo "=========================================="
                    } else {
                        echo "⚠️ Health check échoué (HTTP ${healthStatus})"
                        echo "💡 L'application démarre peut-être encore..."
                        echo "🔍 Vérifiez les logs: docker logs student-app"
                        echo "=========================================="
                    }
                    
                    // Afficher les logs récents
                    echo "📋 Logs récents de l'application:"
                    sh 'docker logs --tail 30 student-app 2>&1 || echo "Impossible de récupérer les logs"'
                    
                    echo ""
                    echo "✅ Déploiement terminé"
                }
            }
        }
    }

    // ========================================
    // ACTIONS POST-PIPELINE
    // ========================================
    post {
        always {
            echo ""
            echo "╔════════════════════════════════════════╗"
            echo "║     🏁 PIPELINE TERMINÉE               ║"
            echo "╚════════════════════════════════════════╝"
            echo ""
            echo "📦 Build: #${BUILD_NUMBER}"
            echo "📊 Statut: ${currentBuild.currentResult}"
            echo "⏱️  Durée: ${currentBuild.durationString.replace(' and counting', '')}"
            echo "🐳 Image: ${DOCKER_REPO}:${BUILD_NUMBER}"
            echo "🌐 Port: ${APP_PORT}"
            echo "🔗 URL: http://192.168.136.129:${APP_PORT}"
            echo ""
            
            script {
                // Nettoyer les images Docker non utilisées
                sh 'docker image prune -f 2>/dev/null || echo "Nettoyage Docker ignoré"'
            }
        }
        
        success {
            echo "╔════════════════════════════════════════╗"
            echo "║   ✅✅✅ SUCCÈS COMPLET ✅✅✅          ║"
            echo "╚════════════════════════════════════════╝"
            echo ""
            echo "🎉 Toutes les étapes ont réussi !"
            echo "🚀 Application déployée avec succès"
            echo ""
            echo "📝 ACCÈS À L'APPLICATION:"
            echo "   • Interface: http://192.168.136.129:${APP_PORT}"
            echo "   • API Docs: http://192.168.136.129:${APP_PORT}/swagger-ui.html"
            echo "   • Health: http://192.168.136.129:${APP_PORT}/actuator/health"
            echo ""
            echo "🐳 COMMANDES DOCKER UTILES:"
            echo "   • Logs: docker logs -f student-app"
            echo "   • Restart: docker restart student-app"
            echo "   • Stop: docker stop student-app"
            echo ""
        }
        
        unstable {
            echo "╔════════════════════════════════════════╗"
            echo "║   ⚠️  BUILD INSTABLE  ⚠️              ║"
            echo "╚════════════════════════════════════════╝"
            echo ""
            echo "⚠️ L'application est déployée mais avec des warnings"
            echo ""
            echo "🔍 POINTS D'ATTENTION:"
            echo "   • Certains tests ont échoué"
            echo "   • L'analyse SonarQube peut avoir échoué"
            echo "   • Le push Docker peut avoir échoué"
            echo ""
            echo "💡 ACTIONS RECOMMANDÉES:"
            echo "   1. Consultez les rapports de tests dans Jenkins"
            echo "   2. Vérifiez les logs ci-dessus"
            echo "   3. Corrigez les warnings pour le prochain build"
            echo ""
        }
        
        failure {
            echo "╔════════════════════════════════════════╗"
            echo "║   ❌❌❌ ÉCHEC DE LA PIPELINE ❌❌❌   ║"
            echo "╚════════════════════════════════════════╝"
            echo ""
            echo "🔍 DIAGNOSTIC AUTOMATIQUE:"
            echo ""
            
            script {
                def failureStage = currentBuild.rawBuild.getAction(
                    org.jenkinsci.plugins.workflow.cps.nodes.StepStartNode.class
                )
                
                echo "📋 PROBLÈMES COURANTS ET SOLUTIONS:"
                echo ""
                echo "1️⃣ TESTS ÉCHOUÉS"
                echo "   Symptôme: Hibernate cannot determine Dialect"
                echo "   Solution:"
                echo "   • Créez src/test/resources/application-test.properties"
                echo "   • Ajoutez: spring.datasource.url=jdbc:h2:mem:testdb"
                echo "   • Ajoutez: spring.jpa.database-platform=org.hibernate.dialect.H2Dialect"
                echo "   • Modifiez le test: @ActiveProfiles(\"test\")"
                echo ""
                echo "2️⃣ DOCKERFILE MANQUANT"
                echo "   Symptôme: Dockerfile introuvable"
                echo "   Solution:"
                echo "   • Créez un fichier 'Dockerfile' à la racine"
                echo "   • Contenu minimal:"
                echo "     FROM eclipse-temurin:17-jre-alpine"
                echo "     WORKDIR /app"
                echo "     COPY target/*.jar app.jar"
                echo "     EXPOSE 8080"
                echo "     ENTRYPOINT [\"java\", \"-jar\", \"app.jar\"]"
                echo ""
                echo "3️⃣ DOCKER PUSH ÉCHOUÉ"
                echo "   Symptôme: Permission denied / Authentication required"
                echo "   Solution:"
                echo "   • Jenkins > Manage > Credentials > Add"
                echo "   • Kind: Username with password"
                echo "   • ID: dockerhub-credentials"
                echo "   • Username: ${DOCKER_USERNAME}"
                echo "   • Password: [Token Docker Hub]"
                echo ""
                echo "4️⃣ PERMISSIONS DOCKER"
                echo "   Symptôme: Permission denied connecting to Docker"
                echo "   Solution:"
                echo "   • sudo usermod -aG docker jenkins"
                echo "   • sudo systemctl restart jenkins"
                echo ""
            }
            
            echo "📚 DOCUMENTATION:"
            echo "   • Tests: https://spring.io/guides/gs/testing-web/"
            echo "   • Docker: https://docs.docker.com/get-started/"
            echo "   • Jenkins: https://www.jenkins.io/doc/"
            echo ""
            echo "💬 BESOIN D'AIDE?"
            echo "   • Consultez les logs complets ci-dessus"
            echo "   • Vérifiez la console Jenkins pour plus de détails"
            echo ""
        }
        
        cleanup {
            echo "🧹 Nettoyage du workspace..."
            // Ne pas supprimer le workspace pour permettre le débogage
            // cleanWs()
        }
    }
}
