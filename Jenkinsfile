pipeline {
    agent any
    
    tools {
        maven 'MAVEN_3'
        jdk 'JDK17'
    }

    environment {
        SONAR_TOKEN = credentials('jenkins_sonar')
        DOCKERHUB_CREDS = credentials('docker-hub-credentials')
        APP_PORT = '9090'
        BUILD_VERSION = "${BUILD_NUMBER}"
        DOCKER_REPO = 'mohamed15032003/student-management'
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
                    sh "docker build -t ${DOCKER_REPO}:${BUILD_VERSION} ."
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
        // Si vous avez supprimé le stage 8, assurez-vous qu'il n'y a pas de virgule après le stage 7
    }

    post {
        always {
            echo "=== PIPELINE TERMINÉE ==="
            echo "Statut: ${currentBuild.currentResult}"
            echo "Build: #${BUILD_NUMBER}"
            echo "Application: http://192.168.136.129:${APP_PORT}"
            script {
                def repoParts = "${DOCKER_REPO}".split('/')
                echo "Docker Hub: https://hub.docker.com/r/${repoParts[0]}/student-management"
            }
        }
        success {
            echo "✅ DÉPLOIEMENT RÉUSSI"
        }
        failure {
            echo "❌ DÉPLOIEMENT ÉCHOUÉ"
        }
    }
}
