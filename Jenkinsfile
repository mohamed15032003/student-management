pipeline {
    agent any
    
    environment {
        // Variables d'environnement
        DOCKER_IMAGE = 'mohamedderbel15032003/student-management-app'
        DOCKER_TAG = 'latest'
    }
    
    stages {
        // Étape 1: Récupération du code
        stage('1) Git Clone') {
            steps {
                git(
                    url: 'https://github.com/mohamed15032003/student-management.git',
                    branch: 'main',
                    credentialsId: 'github-creds'  // Utilisation de vos identifiants GitHub
                )
                sh 'echo "Code source récupéré avec succès !"'
            }
        }
        
        // Étape 2: Compilation avec Maven
        stage('2) Build Maven') {
            steps {
                sh 'mvn clean compile'
                sh 'echo "Compilation Maven réussie !"'
            }
        }
        
        // Étape 3: Génération du JAR
        stage('3) Build JAR') {
            steps {
                sh 'mvn package -DskipTests'
                sh '''
                    echo "=== ARTEFACTS ==="
                    ls -la target/*.jar
                    echo "=== TAILLE ==="
                    du -h target/*.jar
                '''
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                sh 'echo "JAR archivé dans Jenkins"'
            }
        }
        
        // Étape 4: Analyse SonarQube - CORRIGÉE avec le bon ID
        stage('4) SonarQube Analysis') {
            steps {
                withCredentials([
                    string(credentialsId: 'SONARQUBE_TOKEN', variable: 'SONAR_TOKEN')
                ]) {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=student-management \
                        -Dsonar.host.url=http://54.37.78.131:9000 \
                        -Dsonar.login=${SONAR_TOKEN}
                    '''
                }
            }
        }
        
        // Étape 5: Construction et push Docker
        stage('5) Docker Build & Push') {
            environment {
                // Utilisation de vos identifiants Docker Hub
                DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'
            }
            steps {
                script {
                    // Construction de l'image Docker
                    sh """
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    """
                    
                    // Connexion à Docker Hub
                    withCredentials([
                        usernamePassword(
                            credentialsId: DOCKER_CREDENTIALS_ID,
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )
                    ]) {
                        sh '''
                            echo "Connexion à Docker Hub..."
                            echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin
                        '''
                    }
                    
                    // Push des images
                    sh """
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    """
                    
                    sh 'echo "Images Docker poussées avec succès !"'
                }
            }
        }
        
        // Étape optionnelle: Déploiement
        stage('6) Déploiement') {
            steps {
                sh 'echo "Étape de déploiement - À configurer selon votre environnement"'
                // Exemple de déploiement:
                // sh 'kubectl apply -f k8s/deployment.yaml'
                // sh 'kubectl rollout status deployment/student-management'
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline réussie !'
            // Nettoyage des images locales pour économiser l'espace
            sh '''
                docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true
                docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER} || true
            '''
        }
        failure {
            echo '❌ Pipeline échouée.'
            // Actions en cas d'échec (notification, logs, etc.)
        }
        always {
            echo '🔧 Nettoyage...'
            // Nettoyage des conteneurs arrêtés
            sh 'docker container prune -f || true'
            // Nettoyage des images sans tag
            sh 'docker image prune -f || true'
        }
    }
}
