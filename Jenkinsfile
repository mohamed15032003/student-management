pipeline {
    agent any

    environment {
        APP_NAME = "student-management"
        IMAGE_TAG = "1.0.0"
    }

    stages {

        // 1) Git Clone
        stage('1) Git Clone') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/mohamedderbel/student-management.git'
                echo "✅ Code source récupéré depuis GitHub"
            }
        }

        // 2) Build
        stage('2) Build') {
            steps {
                sh 'mvn clean compile'
                echo "✅ Compilation réussie"
            }
        }

        // 3) Build JAR
        stage('3) Build JAR') {
            steps {
                sh 'mvn package -DskipTests'
                sh '''
                    echo "=== ARTEFACT JAR ==="
                    ls -lh target/*.jar
                '''
                archiveArtifacts artifacts: 'target/*.jar'
                echo "✅ JAR archivé dans Jenkins"
            }
        }

        // 4) SonarQube Analysis
        stage('4) SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'jenkins_sonar', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        echo "🔍 Analyse SonarQube"
                        mvn sonar:sonar \
                        -Dsonar.projectKey=student-management \
                        -Dsonar.host.url=http://192.168.132.129:9000 \
                        -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        // 5) Build & Push Docker Image
        stage('5) Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "🐳 Build image Docker"
                        docker build -t $DOCKER_USER/student-management:$IMAGE_TAG .

                        echo "🔐 Login DockerHub"
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        echo "📤 Push image Docker"
                        docker push $DOCKER_USER/student-management:$IMAGE_TAG

                        echo "✅ Image Docker publiée avec succès"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '🎉 PIPELINE RÉUSSIE – BUILD + SONAR + DOCKER OK'
        }
        failure {
            echo '❌ PIPELINE EN ÉCHEC'
        }
    }
}
