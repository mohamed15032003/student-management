pipeline {
    agent any
    
    environment {
        APP_NAME = 'student-management'
        DOCKER_IMAGE = 'student-management-app'
        K8S_NAMESPACE = 'student-namespace'
    }
    
    stages {
        stage('1) Build') {
            steps {
                git url: 'https://github.com/mohamed15032003/student-management.git', branch: 'main'
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('2) Docker Build') {
            steps {
                sh '''
                    cat > Dockerfile << EOF
FROM openjdk:17-jdk-slim
COPY target/student-management-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
EOF
                    docker build -t ${DOCKER_IMAGE}:latest .
                '''
            }
        }
        
        stage('3) Deploy to Minikube') {
            steps {
                sh '''
                    # Charger l'image dans Minikube
                    minikube image load ${DOCKER_IMAGE}:latest
                    
                    # Déployer avec kubectl
                    cat > deploy.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${K8S_NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      containers:
      - name: ${APP_NAME}
        image: ${DOCKER_IMAGE}:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-service
  namespace: ${K8S_NAMESPACE}
spec:
  type: NodePort
  selector:
    app: ${APP_NAME}
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
EOF
                    
                    # Créer namespace et déployer
                    kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                    kubectl apply -f deploy.yaml
                '''
            }
        }
        
        stage('4) Verify') {
            steps {
                sh '''
                    sleep 45
                    IP=$(minikube ip)
                    echo "🌐 Application disponible sur:"
                    echo "   http://${IP}:30080/"
                    echo "Test d'accès..."
                    curl http://${IP}:30080/actuator/health || echo "En cours de démarrage"
                '''
            }
        }
    }
    
    post {
        always {
            sh 'rm -f Dockerfile deploy.yaml 2>/dev/null || true'
        }
    }
}
