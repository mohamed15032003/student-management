pipeline {
    agent any

    tools {
        maven 'MAVEN_3'
        jdk 'JDK17'
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

        stage('3️⃣ Test & Package (Tests Sautés)') {
            steps {
                echo '📦 Packaging du projet...'
                sh 'mvn package -DskipTests'
            }
        }

        stage('4️⃣ Package JAR') {
            steps {
                echo '📦 Packaging en JAR...'
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('5️⃣ Archive Artifact') {
            steps {
                echo '📁 Archivage du fichier JAR...'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
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

