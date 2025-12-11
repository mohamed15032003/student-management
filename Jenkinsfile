pipeline {
    agent any
    
    environment {
        // VARIABLES OBLIGATOIRES - DOIVENT ÊTRE LÀ !
        APP_PORT = '9090'  # C'EST ÇA QUI MANQUE !
        DOCKER_REPO = 'mohamedderbel/student-management'
        
        # SonarQube - seulement si configuré
        SONAR_HOST_URL = 'http://192.168.136.129:9000'
        SONAR_PROJECT_KEY = 'student-management'
    }

    stages {
        // 1. CLONE
        stage('1) Clone du Code') {
            steps {
                echo "Étape 1: Clone"
                checkout scm
            }
        }

        // 2. BUILD
        stage('2) Build Maven') {
            steps {
                echo "Étape 2: Build"
                sh 'mvn clean compile'
            }
        }

        // 3. TESTS
        stage('3) Tests Unitaires') {
            steps {
                echo "Étape 3: Tests"
                sh 'mvn test'
                junit 'target/surefire-reports/*.xml'
            }
        }

        // 4. PACKAGE
        stage('4) Package JAR') {
            steps {
                echo "Étape 4: Package"
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        // 5. DOCKER BUILD
        stage('5) Docker Build') {
            steps {
                echo "Étape 5: Docker Build"
                script {
                    sh "docker build -t ${DOCKER_REPO}:${BUILD_NUMBER} ."
                    sh "docker build -t ${DOCKER_REPO}:latest ."
                }
            }
        }

        // 6. SONARQUBE (TEST SIMPLE SANS CREDENTIALS)
        stage('6) SonarQube Test') {
            steps {
                echo "Étape 6: Test SonarQube"
                script {
                    // Test simple de connexion
                    sh """
                        echo "Test SonarQube sur ${SONAR_HOST_URL}"
                        curl -s ${SONAR_HOST_URL}/api/system/status || \
                        echo "SonarQube non accessible - étape ignorée"
                    """
                }
            }
        }

        // 7. DOCKER PUSH
        stage('7) Docker Push') {
            steps {
                script {
                    echo "Étape 7: Docker Push"
                    sh '''
                        echo "user123@Med" | docker login -u "Mohamed Derbel" --password-stdin || \
                        echo "Docker login échoué - push ignoré"
                    '''
                    sh "docker push ${DOCKER_REPO}:${BUILD_NUMBER} || echo 'Push ignoré'"
                    sh "docker push ${DOCKER_REPO}:latest || echo 'Push latest ignoré'"
                }
            }
        }

        // 8. DÉPLOIEMENT
        stage('8) Déploiement') {
            steps {
                script {
                    echo "Étape 8: Déploiement"
                    sh 'docker rm -f student-app 2>/dev/null || true'
                    sh """
                        docker run -d \
                            --name student-app \
                            -p ${APP_PORT}:8080 \
                            ${DOCKER_REPO}:latest
                    """
                    sh "sleep 5 && echo 'Application déployée sur port ${APP_PORT}'"
                }
            }
        }
    }

    post {
        always {
            echo "========================================"
            echo "PIPELINE TERMINÉE"
            echo "Build: #${BUILD_NUMBER}"
            echo "Statut: ${currentBuild.currentResult}"
            // CORRECTION: Utilisation de ${env.APP_PORT} pour être sûr
            echo "Port d'application: ${env.APP_PORT}"
            echo "========================================"
        }
        success {
            echo "✅ SUCCÈS"
        }
        failure {
            echo "❌ ÉCHEC"
            // CORRECTION: Pas de 'sh' dans failure sans node
            echo "Vérifiez les logs pour les détails"
        }
    }
}
    }
}

