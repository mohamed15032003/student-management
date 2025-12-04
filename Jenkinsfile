pipeline {
    agent any

    tools {
        maven 'MAVEN_3'
        jdk 'JDK17'
    }

    stages {

        stage('1️⃣ Clone') {
            steps {
                echo '📥 Cloning repository...'
                git branch: 'main', url: 'https://github.com/mohamedderbel317/student-management.git'
                echo '✅ Clone done'
            }
        }

        stage('2️⃣ Build') {
            steps {
                echo '🔨 Building the project...'
                sh 'mvn clean compile -DskipTests'
                echo '✅ Build completed'
            }
        }

        stage('3️⃣ Test') {
            steps {
                echo '🧪 Running tests...'
                sh 'mvn test'
                echo '📊 Tests completed'
            }
        }

        stage('4️⃣ JAR Packaging') {
            steps {
                echo '📦 Creating JAR file...'
                sh 'mvn package -DskipTests'
                echo '🎉 JAR generated in target/'
            }
        }

    }

    post {
        success {
            echo '🎉 Pipeline executed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}


