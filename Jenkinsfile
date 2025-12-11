pipeline {
    agent any

    tools {
        maven 'MAVEN_3'
        jdk 'JDK17'
    }

    environment {
        SONAR_TOKEN = credentials('jenkins_sonar')
    }

    stages {

        /* ------------------------------------------------------------------ */
        stage('📥 1) Git Clone') {
            steps {
                echo "📥 Récupération du code source..."
                git branch: 'MohamedYoussefMellouli', 
                    url: 'https://github.com/MohamedYoussefMellouli/Devops.git'
                echo "✅ Code source récupéré"
            }
        }

        /* ------------------------------------------------------------------ */
        stage('🏗️ 2) Build Maven') {
            steps {
                echo "🏗️ Compilation du projet..."
                sh 'mvn clean compile -DskipTests'
                echo "✅ Compilation réussie"
            }
        }

        /* ------------------------------------------------------------------ */
        stage('🧪 3) Tests avec Base de Données (Docker MySQL)') {
            steps {
                script {
                    echo "🐬 Lancement de MySQL dans Docker..."

                    def mysqlID = sh(script: """
                        docker run -d \
                            --name mysql-test \
                            -e MYSQL_ROOT_PASSWORD=root \
                            -e MYSQL_DATABASE=student_db \
                            -P \
                            mysql:8.0
                    """, returnStdout: true).trim()

                    echo "🐋 MySQL ID : ${mysqlID}"

                    def port = sh(script: "docker port mysql-test 3306 | cut -d':' -f2", returnStdout: true).trim()
                    echo "🔌 MySQL disponible sur le port : ${port}"

                    echo "⏳ Attente de MySQL..."
                    sh "sleep 25"

                    sh """
                        mvn test \
                        -Dspring.datasource.url=jdbc:mysql://localhost:${port}/student_db \
                        -Dspring.datasource.username=root \
                        -Dspring.datasource.password=root
                    """

                    echo "🧹 Nettoyage du conteneur..."
                    sh "docker stop mysql-test"
                    sh "docker rm mysql-test"
                }
            }
        }

        /* ------------------------------------------------------------------ */
        stage('📦 4) Build JAR + Archivage') {
            steps {
                echo "📦 Génération du JAR..."
                sh 'mvn package -DskipTests'

                sh '''
                    echo "=== ARTEFACTS ==="
                    ls -la target/*.jar
                    du -h target/*.jar
                '''

                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                echo "✅ JAR archivé"
            }
        }

        /* ------------------------------------------------------------------ */
        stage('🐳 5) Docker Build & Run') {
            steps {
                script {
                    echo "🐳 Construction de l'image Docker..."

                    sh """
                        docker build -t devops-app:latest .
                    """

                    echo "🧹 Suppression ancien conteneur..."
                    sh "docker rm -f devops-container || true"

                    echo "🚀 Lancement du conteneur..."
                    sh """
                        docker run -d \
                            --name devops-container \
                            -p 8085:8080 \
                            devops-app:latest
                    """

                    echo "🐋 Docker exécuté sur : http://localhost:8085"
                }
            }
        }

        /* ------------------------------------------------------------------ */
        stage('🔍 6) Analyse SonarQube') {
            steps {
                echo "🔍 Analyse SonarQube..."

                withSonarQubeEnv('sonarqube') {
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=Devops \
                            -Dsonar.host.url=http://192.168.132.129:9000 \
                            -Dsonar.login=${SONAR_TOKEN}
                    """
                }

                echo "✅ Analyse SonarQube terminée"
            }
        }
    }

    /* ---------------------------------------------------------------------- */
    post {
        always {
            echo "📊 Statut final : ${currentBuild.result ?: 'UNKNOWN'}"
        }
        success {
            echo "🎉 Pipeline exécutée avec SUCCÈS !"
        }
        failure {
            echo "❌ Le pipeline a ÉCHOUÉ."
        }
    }
}

