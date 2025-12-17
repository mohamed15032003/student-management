pipeline {
    agent any

    environment {
        SONAR_HOST_URL = 'http://192.168.132.129:9000' // adapte si nécessaire
        SONAR_PROJECT_KEY = 'student-management'
    }

    stages {
        stage('1) Git Clone') {
    steps {
        git branch: 'main',
            url: 'https://github.com/mohamed15032003/student-management.git',
            credentialsId: 'clone-creds'
        sh 'echo "Code source récupéré avec succès !"'
    }
}


        // 2) Build Maven
        stage('2) Build Maven') {
            steps {
                sh 'mvn clean compile'
                sh 'echo "Compilation Maven réussie !"'
            }
        }

        // 3) Build JAR
        stage('3) Build JAR') {
            steps {
                sh 'mvn package -DskipTests'
                sh '''
                    echo "=== ARTEFACTS ==="
                    ls -la target/*.jar
                    echo "=== TAILLE ==="
                    du -h target/*.jar
                '''
                archiveArtifacts 'target/*.jar'
                sh 'echo "JAR archivé dans Jenkins"'
            }
        }

        // 4) SonarQube Analysis
        stage('4) SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'jenkins_sonar', variable: 'SONAR_TOKEN')]) {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.host.url=${SONAR_HOST_URL} \
                          -Dsonar.login=$SONAR_TOKEN
                    """
                }
            }
        }

        // 5) Docker Build & Push
        stage('5) Docker Build & Push') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            docker build -t $DOCKER_USER/student-management:1.0.0 .
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker push $DOCKER_USER/student-management:1.0.0
                        """
                    }
                }
            }
        }
    }

    post {
        success { echo '✅ Pipeline exécuté avec succès !' }
        failure { echo '❌ Pipeline échoué.' }
    }
}

