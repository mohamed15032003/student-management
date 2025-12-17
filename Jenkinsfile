pipeline {
    agent any
    environment {
        K8S_NAMESPACE = 'devops'
        DOCKER_IMAGE = 'student-management:latest'
    }
    stages {
        stage('Clone & Build') {
            steps {
                checkout scm
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Construire l'image
                    sh '''
                        # Construire dans Minikube
                        eval $(minikube docker-env)
                        docker build -t student-management:latest .
                        eval $(minikube docker-env -u)
                    '''
                    
                    // Déployer
                    sh """
                        kubectl set image deployment/student-management \\
                          student-app=student-management:latest \\
                          -n ${K8S_NAMESPACE}
                        
                        kubectl rollout status deployment/student-management \\
                          -n ${K8S_NAMESPACE} --timeout=300s
                    """
                }
            }
        }
        stage('Verify') {
            steps {
                script {
                    sh 'sleep 30'
                    sh "kubectl get pods -n ${K8S_NAMESPACE}"
                    sh "minikube service student-service -n ${K8S_NAMESPACE} --url"
                }
            }
        }
    }
    post {
        success {
            echo '✅ Pipeline réussie!'
            sh 'echo "Application déployée avec succès"'
        }
        failure {
            echo '❌ Pipeline échouée'
            sh 'kubectl logs -l app=student-management -n devops --tail=50'
        }
    }
}
