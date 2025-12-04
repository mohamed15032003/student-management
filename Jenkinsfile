pipeline {
    agent any
    
    tools {
        jdk 'JDK17'  // ← CORRECTION ICI
    }
    
    stages {
        // ÉTAPE 1 : CLONE
        stage('📥 Clone du Code') {
            steps {
                echo '1. Je prends le code depuis GitHub...'
                git branch: 'main',
                    url: 'https://github.com/mohamed15032003/student-management.git'
                echo '✅ Code téléchargé !'
            }
        }
        
        // ÉTAPE 2 : COMPILATION
        stage('🔨 Compilation') {
            steps {
                echo '2. Je vérifie que le code compile...'
                sh 'mvn clean compile'
                echo '✅ Code compilé avec succès !'
            }
        }
        
        // ÉTAPE 3 : TESTS
        stage('🧪 Tests Automatiques') {
            steps {
                echo '3. Je lance les tests automatiques...'
                sh 'mvn test'
                echo '✅ Tests terminés !'
            }
        }
        
        // ÉTAPE 4 : SONARQUBE
        stage('🔍 Analyse SonarQube') {
            steps {
                echo '4. Je vérifie la qualité du code avec SonarQube...'
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=student-management \
                          -Dsonar.host.url=http://192.168.136.129:9000
                    '''
                }
                echo '✅ Analyse qualité terminée !'
            }
        }
        
        // ÉTAPE 5 : PACKAGE
        stage('📦 Création du JAR') {
            steps {
                echo '5. Je crée le fichier JAR executable...'
                sh 'mvn package -DskipTests'
                echo '✅ JAR créé avec succès !'
            }
        }
        
        // ÉTAPE 6 : SAUVEGARDE
        stage('💾 Sauvegarde') {
            steps {
                echo '6. Je sauvegarde le fichier JAR...'
                archiveArtifacts 'target/*.jar'
                echo '✅ JAR sauvegardé dans Jenkins !'
            }
        }
    }
    
    post {
        always {
            echo '📊 Résumé du pipeline terminé !'
        }
        success {
            echo '🎉 FÉLICITATIONS ! Tout a fonctionné !'
            echo '👉 Voir SonarQube : http://192.168.136.129:9000'
        }
        failure {
            echo '❌ Oups, quelque chose a échoué.'
            echo '🔍 Regarde les logs pour comprendre.'
        }
    }
}
