pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'student-management:latest'
        K8S_NAMESPACE = 'devops'
        APP_VERSION = "${BUILD_NUMBER}"
    }
    stages {
        stage('Clone') {
            steps { checkout scm }
        }
        stage('Build') {
            steps { sh 'mvn clean package -DskipTests' }
        }
        stage('Docker Build') {
            steps {
                sh 'eval $(minikube docker-env)'
                sh "docker build -t ${DOCKER_IMAGE}:${APP_VERSION} ."
                sh 'eval $(minikube docker-env -u)'
            }
        }
        stage('Deploy') {
            steps {
                sh """
                    kubectl set image deployment/student-management \
                      student-app=${DOCKER_IMAGE}:${APP_VERSION} \
                      -n ${K8S_NAMESPACE}
                    kubectl rollout status deployment/student-management \
                      -n ${K8S_NAMESPACE} --timeout=300s
                """
            }
        }
    }
}
EOF
