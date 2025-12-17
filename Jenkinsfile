pipeline {
    agent any
    environment {
        K8S_NAMESPACE = 'devops'
    }
    stages {
        stage('Clone and Build') {
            steps {
                checkout scm
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Option 1: Utiliser kubectl depuis Jenkins
                    sh '''
                        # Construire l'image localement (si on peut accéder à Docker)
                        docker build -t student-management-jenkins:latest . || echo "Docker build skipped"
                        
                        # Mettre à jour l'image dans Kubernetes
                        kubectl set image deployment/student-management \
                          student-app=student-management:latest \
                          -n devops
                        
                        # Vérifier le déploiement
                        kubectl rollout status deployment/student-management \
                          -n devops --timeout=300s
                    '''
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                script {
                    // Attendre
                    sleep 30
                    
                    // Vérifier les pods
                    sh 'kubectl get pods -n devops | grep student-management'
                    
                    // Obtenir l'URL Minikube
                    sh '''
                        echo "=== APPLICATION INFO ==="
                        minikube service student-service -n devops --url 2>/dev/null || echo "Use: http://192.168.49.2:30080"
                    '''
                }
            }
        }
    }
    post {
        success {
            echo '✅ Déploiement réussi!'
            sh 'echo "Application: http://192.168.49.2:30080"'
        }
        failure {
            echo '❌ Déploiement échoué'
            sh 'kubectl logs -l app=student-management -n devops --tail=50'
        }
    }
}
