pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh '''
                    mvn clean package -DskipTests
                    echo "✅ Build réussi"
                '''
            }
        }
        
        stage('Test Locally') {
            steps {
                sh '''
                    # Démarrer l'application localement pour test
                    java -jar target/*.jar &
                    APP_PID=$!
                    sleep 30
                    
                    # Tester
                    curl -s http://localhost:8080/actuator/health
                    
                    # Arrêter
                    kill $APP_PID
                    echo "✅ Tests locaux OK"
                '''
            }
        }
        
        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar'
                echo "✅ Artéfacts archivés"
            }
        }
    }
    
    post {
        success {
            echo '🎉 Pipeline de base réussie !'
        }
    }
}
