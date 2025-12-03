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

