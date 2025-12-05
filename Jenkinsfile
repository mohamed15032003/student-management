pipeline {
    agent any

    tools {
        maven 'MAVEN_3'
        jdk 'JDK17'
    }

    environment {
        SONAR_TOKEN = credentials('SONARQUBE_TOKEN')  
    }

    stages {
        stage('1️⃣ Clone Repository') {
            steps {
                echo '📥 Clonage du repository Git...'
                git branch: 'main', url: 'https://github.com/mohamed15032003/student-management.git'
                echo '✅ Clonage terminé'
            }
        }

        stage('2️⃣ Build Project') {
            steps {
                echo '🔨 Compilation du projet avec Maven...'
                sh 'mvn clean compile -DskipTests'
                echo '✅ Build terminé'
            }
        }

        stage('3️⃣ Run Tests') {
            steps {
                echo '🧪 Exécution des tests...'
                sh 'mvn test -DskipTests'
                echo '✅ Tests terminés'
            }
        }

        stage('4️⃣ Package JAR') {
            steps {
                echo '📦 Packaging du projet en JAR...'
                sh 'mvn package -DskipTests'
                echo '✅ Package JAR terminé'
            }
        }

        stage('5️⃣ SonarQube Analysis') {
            steps {
                echo '🔍 Analyse de qualité du code avec SonarQube...'
                withSonarQubeEnv('sonarqube') {
                    sh """
                    mvn sonar:sonar \
                        -Dsonar.projectKey=student-management \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.login=${SONAR_TOKEN} \
                        -DskipTests
                    """
                }
                echo '✅ Analyse SonarQube terminée'
            }
        }

        stage('6️⃣ Archive Artifact') {
            steps {
                echo '📁 Archivage du fichier JAR...'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                echo '✅ Archivage terminé'
            }
        }
    }

    post {
        failure {
            echo '❌ Le pipeline a échoué'
        }
        success {
            echo '🎉 Pipeline terminé avec succès'
        }
    }
}
