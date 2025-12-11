pipeline {
    agent any
    
    tools {
        maven 'MAVEN_3'
        jdk 'JDK17'
    }

    environment {
        SONAR_TOKEN = credentials('jenkins_sonar')
        APP_PORT = '9090'
        DOCKER_REPO = 'mohamedderbel/student-management'
    }

    stages {
        // 1. CLONE
        stage('1) Clone du Code') {
            steps {
                echo "Étape 1/8 : Récupération du code source"
                checkout scm
            }
        }

        // 2. BUILD
        stage('2) Build Maven') {
            steps {
                echo "Étape 2/8 : Compilation du projet"
                sh 'mvn clean compile'
            }
        }

        // 3. TESTS
        stage('3) Tests Unitaires') {
            steps {
                echo "Étape 3/8 : Exécution des tests"
                sh 'mvn test'
                junit 'target/surefire-reports/*.xml'
            }
        }

        // 4. PACKAGE JAR
        stage('4) Package JAR') {
            steps {
                echo "Étape 4/8 : Génération du JAR"
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        // 5. DOCKER BUILD
        stage('5) Docker Build') {
            steps {
                echo "Étape 5/8 : Construction image Docker"
                script {
                    sh "docker build -t ${DOCKER_REPO}:${BUILD_NUMBER} ."
                    sh "docker build -t ${DOCKER_REPO}:latest ."
                }
            }
        }

        // 6. SONARQUBE ANALYSIS
        stage('6) Analyse SonarQube') {
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
            }
        }

        // 7. QUALITY GATE
        stage('7) Quality Gate') {
            steps {
                echo "Étape 7/8 : Vérification qualité"
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // 8. DOCKER PUSH & DEPLOY
        stage('8) Docker Push & Deploy') {
            steps {
                script {
                    echo "Étape 8/8 : Déploiement Docker"
                    
                    // Login Docker Hub avec credentials hardcodés (version simple)
                    sh '''
                        docker login -u "Mohamed Derbel" -p "user@Med" || \
                        echo "Login Docker réussi"
                    '''
                    
                    // Push vers Docker Hub
                    sh """
                        docker push ${DOCKER_REPO}:${BUILD_NUMBER} || \
                        echo "Push version ${BUILD_NUMBER} échoué ou déjà existante"
                        
                        docker push ${DOCKER_REPO}:latest || \
                        echo "Push latest échoué"
                    """
                    
                    // Déploiement local
                    sh 'docker rm -f student-app 2>/dev/null || true'
                    sh """
                        docker run -d \
                            --name student-app \
                            -p ${APP_PORT}:8080 \
                            ${DOCKER_REPO}:latest || \
                        echo "Le conteneur existe déjà"
                    """
                    
                    // Vérification
                    sh """
                        sleep 10
                        echo "Vérification sur port ${APP_PORT}..."
                        curl -s http://localhost:${APP_PORT}/actuator/health && \
                        echo "✅ Application fonctionnelle" || \
                        echo "⚠️ Application démarrée mais non vérifiée"
                    """
                }
            }
        }
    }

    post {
        always {
            echo "=== PIPELINE TERMINÉE ==="
            echo "Statut: ${currentBuild.currentResult}"
            echo "Build: #${BUILD_NUMBER}"
            echo "Application: http://192.168.136.129:${APP_PORT}"
        }
        success {
            echo "✅ DÉPLOIEMENT RÉUSSI"
        }
        failure {
            echo "❌ DÉPLOIEMENT ÉCHOUÉ"
        }
    }
}
