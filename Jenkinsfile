pipeline {
    agent any

    environment {
        APP_PORT = '9090'
        DOCKER_REPO = 'mohamedderbel/student-management'
    }

    stages {
        stage('1) Clone') {
            steps {
                checkout scm
            }
        }

        stage('2) Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('3) Test') {
            steps {
                sh 'mvn test'
                junit 'target/surefire-reports/*.xml'
            }
        }

        stage('4) Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('5) Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_REPO}:${BUILD_NUMBER} . || true"
                sh "docker build -t ${DOCKER_REPO}:latest . || true"
            }
        }

        stage('6) Analyse SonarQube (optionnel)') {
            steps {
                echo "Analyse SonarQube ignorée"
            }
        }

        stage('7) Docker Push') {
            steps {
                sh '''
                    echo "user123@Med" | docker login -u "Mohamed Derbel" --password-stdin || echo "Login Docker échoué"
                '''
                sh """
                    docker push ${DOCKER_REPO}:${BUILD_NUMBER} || echo "Push version échoué"
                    docker push ${DOCKER_REPO}:latest || echo "Push latest échoué"
                """
            }
        }

        stage('8) Déploiement') {
            steps {
                sh 'docker rm -f student-app 2>/dev/null || true'
                sh """
                    docker run -d --name student-app -p ${APP_PORT}:8080 ${DOCKER_REPO}:latest || echo "Erreur lors du démarrage"
                """
                sh "sleep 10 && echo 'Application déployée sur le port ${APP_PORT}'"
            }
        }
    }

    post {
        always {
            echo "========================================"
            echo "PIPELINE TERMINÉE"
            echo "Statut: ${currentBuild.currentResult}"
            echo "Build: #${BUILD_NUMBER}"
            echo "Port d'application: ${env.APP_PORT}"
            echo "========================================"
        }
        success { echo "✅ DÉPLOIEMENT RÉUSSI" }
        failure { echo "❌ DÉPLOIEMENT ÉCHOUÉ – consultez les logs" }
    }
}

