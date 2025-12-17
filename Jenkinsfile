pipeline {
    agent any

    environment {
        APP_NAME = "student-management"
        IMAGE_TAG = "1.0.0"
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        DOCKER_IMAGE = "mohamedderbel/student-management"
        SONAR_URL = "http://192.168.132.129:9000"
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

        // 2) Build Maven
        stage('2) Build Maven') {
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
                    echo "=== Vérification JAR ==="
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
                        mvn sonar:sonar \
                        -Dsonar.projectKey=student-management \
                        -Dsonar.host.url=$SONAR_URL \
                        -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        // 5) Docker Build & Push
        stage('5) Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "=== Vérification JAR avant Docker ==="
                        ls -lh target/*.jar

                        echo "🐳 Build Docker"
                        docker build -t $DOCKER_USER/$APP_NAME:$IMAGE_TAG .

                        echo "🔐 Login DockerHub"
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        echo "📤 Push Docker"
                        docker push $DOCKER_USER/$APP_NAME:$IMAGE_TAG
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '🎉 PIPELINE RÉUSSIE – Build Maven + SonarQube + Docker push OK'
        }
        failure {
            echo '❌ PIPELINE EN ÉCHEC'
        }
    }
}

