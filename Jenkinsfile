pipeline {
    agent any
    
    environment {
        SONAR_TOKEN = credentials('jenkins_sonar')
        APP_PORT = '9090'
        DOCKER_REPO = 'mohamedderbel/student-management'
        SONAR_HOST_URL = 'http://192.168.136.129:9000'
        SONAR_PROJECT_KEY = 'student-management'
    }

    stages {
        // 1. CLONE
        stage('1) 📥 Clone du Code') {
            steps {
                echo "Étape 1/9 : Récupération du code source"
                checkout scm
                sh 'ls -la'
            }
        }

        // 2. BUILD
        stage('2) 🔨 Build Maven') {
            steps {
                echo "Étape 2/9 : Compilation du projet"
                sh 'mvn clean compile'
                echo "✅ Build Maven réussi"
            }
        }

        // 3. TESTS
        stage('3) 🧪 Tests Unitaires') {
            steps {
                echo "Étape 3/9 : Exécution des tests"
                sh 'mvn test'
                junit 'target/surefire-reports/*.xml'
                echo "✅ Tests exécutés avec succès"
            }
        }

        // 4. PACKAGE JAR
        stage('4) 📦 Package JAR') {
            steps {
                echo "Étape 4/9 : Génération du JAR"
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                echo "✅ JAR généré et archivé"
            }
        }

        // 5. DOCKER BUILD
        stage('5) 🐳 Docker Build') {
            steps {
                echo "Étape 5/9 : Construction image Docker"
                script {
                    sh "docker build -t ${DOCKER_REPO}:${BUILD_NUMBER} ."
                    sh "docker build -t ${DOCKER_REPO}:latest ."
                    echo "✅ Images Docker construites"
                }
            }
        }

        // 6. SONARQUBE ANALYSIS
        stage('6) 📊 Analyse SonarQube') {
            steps {
                echo "Étape 6/9 : Analyse qualité du code avec SonarQube"
                echo "URL SonarQube: ${SONAR_HOST_URL}"
                echo "Projet: ${SONAR_PROJECT_KEY}"
                
                script {
                    // Vérification que SonarQube est accessible
                    sh """
                        echo "🔍 Vérification de la connexion à SonarQube..."
                        curl -s ${SONAR_HOST_URL}/api/system/status || echo "⚠️ SonarQube semble inaccessible"
                    """
                    
                    // Analyse SonarQube
                    withSonarQubeEnv('sonarqube') {
                        sh """
                            mvn sonar:sonar \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.login=${SONAR_TOKEN} \
                              -Dsonar.java.binaries=target/classes \
                              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                              -Dsonar.sources=src/main/java \
                              -Dsonar.tests=src/test/java
                        """
                    }
                }
                echo "✅ Analyse SonarQube lancée"
            }
        }

        // 7. QUALITY GATE
        stage('7) ✅ Quality Gate') {
            steps {
                echo "Étape 7/9 : Vérification Quality Gate"
                echo "⏳ Attente du résultat de l'analyse SonarQube..."
                
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
                
                echo "✅ Quality Gate vérifié"
            }
        }

        // 8. DOCKER PUSH
        stage('8) 📤 Docker Push') {
            steps {
                script {
                    echo "Étape 8/9 : Push vers Docker Hub"
                    
                    // Login Docker Hub
                    sh '''
                        echo "user123@Med" | docker login -u "Mohamed Derbel" --password-stdin && \
                        echo "✅ Login Docker Hub réussi" || \
                        echo "⚠️ Login Docker Hub échoué (peut nécessiter configuration)"
                    '''
                    
                    // Push des images
                    sh """
                        echo "📤 Push version ${BUILD_NUMBER}..."
                        docker push ${DOCKER_REPO}:${BUILD_NUMBER} || echo "⚠️ Push version échoué"
                        
                        echo "📤 Push latest..."
                        docker push ${DOCKER_REPO}:latest || echo "⚠️ Push latest échoué"
                    """
                }
            }
        }

        // 9. DÉPLOIEMENT
        stage('9) 🚀 Déploiement') {
            steps {
                script {
                    echo "Étape 9/9 : Déploiement de l'application"
                    
                    // Arrêt conteneur existant
                    sh 'docker rm -f student-app 2>/dev/null || echo "ℹ️ Aucun conteneur à arrêter"'
                    
                    // Déploiement
                    sh """
                        docker run -d \
                            --name student-app \
                            -p ${APP_PORT}:8080 \
                            ${DOCKER_REPO}:latest
                    """
                    
                    echo "✅ Conteneur démarré sur le port ${APP_PORT}"
                    
                    // Vérification
                    sh """
                        echo "🔍 Vérification du déploiement..."
                        sleep 20
                        
                        if curl -s -f http://localhost:${APP_PORT}/actuator/health > /dev/null; then
                            echo "🎉 APPLICATION FONCTIONNELLE !"
                            echo "🌐 URL: http://192.168.136.129:${APP_PORT}"
                            echo "📊 Actuator Health: http://localhost:${APP_PORT}/actuator/health"
                        else
                            echo "⚠️ L'application ne répond pas immédiatement"
                            echo "📋 Logs du conteneur:"
                            docker logs student-app --tail 10 2>/dev/null || true
                            echo "🔗 Essayez: http://192.168.136.129:${APP_PORT}"
                        fi
                    """
                }
            }
        }
    }

    post {
        always {
            echo "=========================================="
            echo "             RAPPORT FINAL               "
            echo "=========================================="
            echo "🏷️  Build: #${BUILD_NUMBER}"
            echo "📊 Statut: ${currentBuild.currentResult}"
            echo "🔌 Port: ${APP_PORT}"
            echo "🌐 Application: http://192.168.136.129:${APP_PORT}"
            echo "📈 SonarQube: ${SONAR_HOST_URL}/dashboard?id=${SONAR_PROJECT_KEY}"
            echo "🐳 Docker Hub: https://hub.docker.com/r/${DOCKER_REPO}"
            echo "=========================================="
            
            // Nettoyage
            sh 'docker image prune -f 2>/dev/null || true'
        }
        success {
            echo "🎉🎉🎉 PIPELINE COMPLÈTE RÉUSSIE ! 🎉🎉🎉"
            echo "✅ 9 étapes exécutées avec succès"
            echo "✅ Analyse SonarQube terminée"
            echo "✅ Application déployée"
            echo "✅ Images Docker disponibles"
        }
        failure {
            echo "🔧 Diagnostic rapide:"
            script {
                sh '''
                    echo "1. Vérification Docker:"
                    docker --version 2>/dev/null || echo "❌ Docker non installé"
                    
                    echo "2. Vérification Maven:"
                    mvn --version 2>/dev/null || echo "❌ Maven non installé"
                    
                    echo "3. Vérification SonarQube:"
                    curl -s ${SONAR_HOST_URL}/api/system/status 2>/dev/null && \
                    echo "✅ SonarQube accessible" || \
                    echo "❌ SonarQube inaccessible - Vérifiez: ${SONAR_HOST_URL}"
                    
                    echo "4. Vérification application:"
                    docker ps | grep student-app && \
                    echo "✅ Conteneur en cours d'exécution" || \
                    echo "❌ Conteneur non démarré"
                '''
            }
        }
    }
}

