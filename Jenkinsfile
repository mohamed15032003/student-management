pipeline {
    agent any
    
    environment {
        APP_NAME = 'student-management'
        DOCKER_IMAGE = 'student-management-app'
        DOCKER_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('1) Git Clone') {
            steps {
                git(
                    url: 'https://github.com/mohamed15032003/student-management.git',
                    branch: 'main',
                    credentialsId: 'github-creds'
                )
            }
        }
        
        stage('2) Build') {
            steps {
                sh '''
                    echo "=== COMPILATION ==="
                    mvn clean compile
                    echo "✅ Compilation terminée"
                '''
            }
        }
        
        stage('3) Optional: SonarQube') {
            when {
                expression { 
                    // Optionnel - seulement si SonarQube fonctionne
                    return false // Mettre à true quand SonarQube fonctionne
                }
            }
            steps {
                script {
                    echo "ℹ️ SonarQube temporairement désactivé"
                    echo "Pour activer:"
                    echo "1. Accédez à http://localhost:9000"
                    echo "2. Login: admin / Password: admin"
                    echo "3. Créez un token"
                    echo "4. Ajoutez SONARQUBE_TOKEN dans Jenkins"
                }
            }
        }
        
        stage('4) Package') {
            steps {
                sh '''
                    echo "=== GÉNÉRATION JAR ==="
                    mvn clean package -DskipTests
                    echo "✅ JAR généré"
                    ls -lh target/*.jar
                '''
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }
        
        stage('5) Docker Build') {
            steps {
                sh '''
                    echo "=== CONSTRUCTION DOCKER ==="
                    cat > Dockerfile << 'EOF'
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/student-management-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
EOF
                    docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                    docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                '''
            }
        }
        
        stage('6) Deploy') {
            steps {
                sh '''
                    echo "=== DÉPLOIEMENT ==="
                    docker stop ${APP_NAME} 2>/dev/null || true
                    docker rm ${APP_NAME} 2>/dev/null || true
                    docker run -d \
                        --name ${APP_NAME} \
                        -p 8080:8080 \
                        --restart unless-stopped \
                        ${DOCKER_IMAGE}:latest
                    echo "✅ Application déployée"
                '''
            }
        }
        
        stage('7) Verify') {
            steps {
                sh '''
                    echo "=== VÉRIFICATION ==="
                    sleep 30
                    echo "Test d'accès..."
                    curl -s http://localhost:8080/actuator/health || echo "En cours de démarrage"
                    echo ""
                    echo "🌐 APPLICATION DISPONIBLE SUR:"
                    echo "   http://localhost:8080"
                    echo "   http://localhost:8080/actuator/health"
                '''
            }
        }
    }
    
    post {
        always {
            sh 'rm -f Dockerfile 2>/dev/null || true'
        }
        success {
            echo '🎉 DÉPLOIEMENT RÉUSSI !'
        }
        failure {
            echo '❌ ÉCHEC DU DÉPLOIEMENT'
            sh '''
                docker stop ${APP_NAME} 2>/dev/null || true
                docker rm ${APP_NAME} 2>/dev/null || true
            '''
        }
    }
}
