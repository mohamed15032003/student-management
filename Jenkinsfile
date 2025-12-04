pipeline {
    agent any  // Tourne sur n'importe quelle machine Jenkins
    
    tools {
        jdk 'jdk11'  // Utilise Java 11 (configure dans Jenkins)
    }
    
    stages {
        // ÉTAPE 1 : CLONE - Prendre le code depuis GitHub
        stage('📥 Clone du Code') {
            steps {
                echo '1. Je prends le code depuis GitHub...'
                git branch: 'main',
                    url: 'https://github.com/mohamed15032003/student-management.git'
                echo '✅ Code téléchargé !'
            }
        }
        
        // ÉTAPE 2 : COMPILATION - Vérifier que le code compile
        stage('🔨 Compilation') {
            steps {
                echo '2. Je vérifie que le code compile...'
                sh 'mvn clean compile'
                echo '✅ Code compilé avec succès !'
            }
        }
        
        // ÉTAPE 3 : TESTS - Exécuter les tests (SANS -DskipTests)
        stage('🧪 Tests Automatiques') {
            steps {
                echo '3. Je lance les tests automatiques...'
                sh 'mvn test'
                echo '✅ Tests terminés !'
            }
        }
        
        // ÉTAPE 4 : SONARQUBE - Analyse de qualité
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
        
        // ÉTAPE 5 : PACKAGE - Créer le fichier JAR
        stage('📦 Création du JAR') {
            steps {
                echo '5. Je crée le fichier JAR executable...'
                sh 'mvn package -DskipTests'  // Tests déjà faits, on peut sauter
                echo '✅ JAR créé avec succès !'
            }
        }
        
        // ÉTAPE 6 : SAUVEGARDE - Garder une copie du JAR
        stage('💾 Sauvegarde') {
            steps {
                echo '6. Je sauvegarde le fichier JAR...'
                archiveArtifacts 'target/*.jar'
                echo '✅ JAR sauvegardé dans Jenkins !'
            }
        }
    }
    
    // CE QUI SE PASSE APRÈS LE PIPELINE
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
