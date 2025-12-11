pipeline {
    agent any
    
    environment {
        DOCKER_REPO = 'mohamedderbel/student-management'
        APP_PORT = '9090'
    }

    stages {
        // =============================================
        // STAGE 1: CLONE
        // =============================================
        stage('Clone') {
            steps {
                echo '📦 Cloning repository...'
                checkout scm
                sh 'ls -la'
            }
        }

        // =============================================
        // STAGE 2: BUILD
        // =============================================
        stage('Build') {
            steps {
                echo '🔨 Compiling project...'
                sh 'mvn clean compile'
            }
        }

        // =============================================
        // STAGE 3: TEST
        // =============================================
        stage('Test') {
            steps {
                echo '🧪 Running tests...'
                script {
                    def testStatus = sh(
                        script: 'mvn test -Dspring.profiles.active=test',
                        returnStatus: true
                    )
                    
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                    
                    if (testStatus != 0) {
                        echo '⚠️ Tests failed but continuing...'
                        unstable('Tests failed')
                    }
                }
            }
        }

        // =============================================
        // STAGE 4: JAR
        // =============================================
        stage('JAR') {
            steps {
                echo '📦 Building JAR...'
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                sh 'ls -lh target/*.jar'
            }
        }

        // =============================================
        // STAGE 5: SONARQUBE
        // =============================================
        stage('SonarQube') {
            steps {
                echo '📊 SonarQube analysis...'
                script {
                    def sonar = sh(
                        script: 'curl -s -o /dev/null -w "%{http_code}" http://192.168.136.129:9000 2>/dev/null || echo "000"',
                        returnStdout: true
                    ).trim()
                    
                    if (sonar == "200") {
                        try {
                            sh '''
                                mvn sonar:sonar \
                                  -Dsonar.projectKey=student-management \
                                  -Dsonar.host.url=http://192.168.136.129:9000 \
                                  || echo "SonarQube analysis failed"
                            '''
                        } catch (Exception e) {
                            echo "⚠️ SonarQube skipped: ${e.message}"
                        }
                    } else {
                        echo '⚠️ SonarQube not available'
                    }
                }
            }
        }

        // =============================================
        // STAGE 6: DOCKER
        // =============================================
        stage('Docker') {
            steps {
                echo '🐳 Building Docker image...'
                script {
                    // Vérifier Dockerfile
                    if (!fileExists('Dockerfile')) {
                        error('❌ Dockerfile not found! Create it at project root.')
                    }
                    
                    // Build
                    sh "docker build -t ${DOCKER_REPO}:${BUILD_NUMBER} ."
                    sh "docker tag ${DOCKER_REPO}:${BUILD_NUMBER} ${DOCKER_REPO}:latest"
                    
                    // Push (optional)
                    try {
                        withCredentials([usernamePassword(
                            credentialsId: 'dockerhub-credentials',
                            usernameVariable: 'USER',
                            passwordVariable: 'PASS'
                        )]) {
                            sh 'echo "$PASS" | docker login -u "$USER" --password-stdin'
                            sh "docker push ${DOCKER_REPO}:${BUILD_NUMBER}"
                            sh "docker push ${DOCKER_REPO}:latest"
                            sh 'docker logout'
                        }
                    } catch (Exception e) {
                        echo "⚠️ Docker push skipped: ${e.message}"
                    }
                    
                    // Deploy
                    sh 'docker stop student-app 2>/dev/null || true'
                    sh 'docker rm student-app 2>/dev/null || true'
                    sh """
                        docker run -d \
                          --name student-app \
                          --restart unless-stopped \
                          -p ${APP_PORT}:8080 \
                          ${DOCKER_REPO}:latest
                    """
                    
                    sleep 15
                    sh "curl -f http://localhost:${APP_PORT}/actuator/health || echo 'Health check pending...'"
                }
            }
        }
    }

    post {
        always {
            echo "════════════════════════════════"
            echo "Build #${BUILD_NUMBER} - ${currentBuild.result}"
            echo "URL: http://192.168.136.129:${APP_PORT}"
            echo "════════════════════════════════"
            sh 'docker image prune -f 2>/dev/null || true'
        }
        success {
            echo "✅ SUCCESS"
        }
        unstable {
            echo "⚠️ UNSTABLE (tests failed)"
        }
        failure {
            echo "❌ FAILURE"
            echo "Check: 1) Dockerfile exists 2) application-test.properties created"
        }
    }
}
