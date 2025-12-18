pipeline {
    agent any

    stages {
        // 1) Git Clone
        stage('1) Git Clone') {
            steps {
                git branch: 'mohamed15032003',
                    url: 'https://github.com/mohamed15032003/student-management/'
                sh 'echo "Code source récupéré"'
            }
        }

        // 2) Build
        stage('2) Build') {
            steps {
                sh 'mvn clean compile'
                sh 'echo "Compilation réussie"'
            }
        }
        */

        // 4) Build JAR
        stage('4) Build JAR') {
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

        // 5) SonarQube Analysis
        stage('5) SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'jenkins_sonar', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        echo "Analyse SonarQube"
                        mvn sonar:sonar -Dsonar.projectKey=Devops \
                          -Dsonar.host.url=http://localhost:9000\
                          -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        // 6) Build & Push Docker Image
        stage('6) Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            echo "Build de l’image Docker..."
                            docker build -t $DOCKER_USER/alpine:1.0.0 .
                            echo "Connexion Docker Hub..."
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            echo "Push de l’image..."
                            docker push $DOCKER_USER/alpine:1.0.0
                            echo "Image Docker publiée avec succès !"
                        '''
                    }
                }
            }
        }
    }

    post {
        success { echo 'SUCCÈS TOTAL ! Pipeline vert sans les tests DB pour l’instant !' }
        failure { echo 'ÉCHEC du pipeline.' }
    }
}
