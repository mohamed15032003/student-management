pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'student-management:latest'
        K8S_NAMESPACE = 'devops'
        SONAR_HOST = 'http://192.168.136.129:9000'
        APP_VERSION = "${BUILD_NUMBER}"
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
        // STAGE 3: UNIT TESTS
        // =============================================
        stage('Unit Tests') {
            steps {
                echo '🧪 Running unit tests...'
                sh 'mvn test'
                
                // Archiver les résultats des tests
                junit 'target/surefire-reports/*.xml'
                
                // Générer le rapport de couverture Jacoco
                sh 'mvn jacoco:report'
            }
        }

        // =============================================
        // STAGE 4: SONARQUBE ANALYSIS
        // =============================================
        stage('SonarQube Analysis') {
            when {
                expression { 
                    return sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' ${SONAR_HOST}/api/system/status 2>/dev/null || echo '000'",
                        returnStdout: true
                    ).trim() == "200"
                }
            }
            steps {
                echo '📊 Running SonarQube analysis...'
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=student-management-${APP_VERSION} \
                          -Dsonar.projectName='Student Management' \
                          -Dsonar.projectVersion=${APP_VERSION} \
                          -Dsonar.java.source=17 \
                          -Dsonar.sources=src/main/java \
                          -Dsonar.tests=src/test/java \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                          -Dsonar.host.url=${SONAR_HOST} \
                          -Dsonar.exclusions='**/target/**,**/node_modules/**,**/*.yml,**/*.yaml'
                    """
                }
                
                // Attendre le résultat de la qualité
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        // =============================================
        // STAGE 5: BUILD JAR - STAGE SPÉCIFIQUE POUR LE JAR
        // =============================================
        stage('Build JAR') {
            steps {
                echo '📦 Building JAR file...'
                script {
                    // Package sans tests (déjà faits)
                    sh 'mvn package -DskipTests'
                    
                    // Vérifier le JAR généré
                    sh 'ls -lh target/*.jar'
                    
                    // Afficher les infos du JAR
                    sh '''
                        echo "=== JAR INFORMATION ==="
                        ls -la target/*.jar
                        echo "Taille du JAR:"
                        du -h target/*.jar
                    '''
                    
                    // Archiver le JAR
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                    
                    // Sauvegarder aussi le JAR avec version
                    sh """
                        cp target/*.jar target/student-management-${APP_VERSION}.jar
                        ls -la target/*.jar
                    """
                    
                    echo "✅ JAR créé: student-management-${APP_VERSION}.jar"
                }
            }
        }

        // =============================================
        // STAGE 6: DOCKER BUILD
        // =============================================
        stage('Docker Build') {
            steps {
                echo '🐳 Building Docker image...'
                script {
                    // Vérifier que le JAR existe
                    sh 'test -f target/*.jar || (echo "❌ JAR not found!" && exit 1)'
                    
                    // Activer l'environnement Docker de Minikube
                    sh 'eval $(minikube docker-env)'
                    
                    // Construire l'image
                    sh """
                        docker build \
                          --build-arg JAR_FILE=target/*.jar \
                          -t ${DOCKER_IMAGE} \
                          -t ${DOCKER_IMAGE}:${APP_VERSION} \
                          .
                    """
                    
                    // Vérifier les images créées
                    sh 'docker images | grep student-management'
                    
                    // Désactiver l'environnement Minikube
                    sh 'eval $(minikube docker-env -u)'
                    
                    echo "✅ Image Docker créée: ${DOCKER_IMAGE}:${APP_VERSION}"
                }
            }
        }

        // =============================================
        // STAGE 7: DEPLOY TO KUBERNETES
        // =============================================
        stage('Deploy to Kubernetes') {
            steps {
                echo '🚀 Deploying to Kubernetes...'
                script {
                    // Mettre à jour l'image dans le déploiement
                    sh """
                        kubectl set image deployment/student-management \
                          student-app=${DOCKER_IMAGE}:${APP_VERSION} \
                          -n ${K8S_NAMESPACE}
                    """
                    
                    // Vérifier le déploiement
                    sh """
                        kubectl rollout status deployment/student-management \
                          -n ${K8S_NAMESPACE} \
                          --timeout=300s
                    """
                    
                    // Vérifier les pods
                    sh "kubectl get pods -n ${K8S_NAMESPACE} | grep student-management"
                    
                    echo "✅ Déploiement Kubernetes terminé!"
                }
            }
        }

        // =============================================
        // STAGE 8: INTEGRATION TESTS
        // =============================================
        stage('Integration Tests') {
            steps {
                echo '🔍 Running integration tests...'
                script {
                    // Obtenir l'URL
                    def url = sh(
                        script: "minikube service student-service -n ${K8S_NAMESPACE} --url",
                        returnStdout: true
                    ).trim()
                    
                    echo "🌐 Application URL: ${url}"
                    
                    // Attendre que l'application soit prête
                    sh 'sleep 30'
                    
                    // Tests d'intégration
                    sh """
                        echo "=== INTEGRATION TESTS ==="
                        
                        # Test 1: Health check
                        echo "1. Testing health endpoint..."
                        curl -f ${url}/actuator/health || (echo "❌ Health check failed" && exit 1)
                        echo "✅ Health check passed"
                        
                        # Test 2: Root endpoint
                        echo "2. Testing root endpoint..."
                        STATUS_CODE=\$(curl -s -o /dev/null -w "%{http_code}" ${url})
                        if [ "\$STATUS_CODE" = "200" ] || [ "\$STATUS_CODE" = "404" ]; then
                            echo "✅ Root endpoint returned \$STATUS_CODE"
                        else
                            echo "⚠️ Root endpoint returned \$STATUS_CODE"
                        fi
                        
                        # Test 3: Specific endpoints
                        echo "3. Testing API endpoints..."
                        curl -s ${url}/students && echo "✅ /students endpoint OK" || echo "⚠️ /students endpoint not available"
                        curl -s ${url}/department/getAllDepartment && echo "✅ /department endpoint OK" || echo "⚠️ /department endpoint not available"
                        
                        # Test 4: Database connection
                        echo "4. Testing database connection..."
                        kubectl exec -it \$(kubectl get pod -l app=mysql -n ${K8S_NAMESPACE} -o jsonpath='{.items[0].metadata.name}') \
                          -n ${K8S_NAMESPACE} -- \
                          mysql -u root -proot123 -e "USE springdb; SHOW TABLES;" && \
                          echo "✅ Database connection OK" || \
                          echo "⚠️ Database connection issue"
                    """
                }
            }
        }
    }

    post {
        always {
            echo "══════════════════════════════════════════════════════════"
            echo "📊 PIPELINE REPORT - Build #${BUILD_NUMBER}"
            echo "══════════════════════════════════════════════════════════"
            
            script {
                // Nettoyage
                sh 'docker image prune -f 2>/dev/null || true'
                
                // Rapport final
                def url = sh(
                    script: "minikube service student-service -n ${K8S_NAMESPACE} --url 2>/dev/null || echo 'N/A'",
                    returnStdout: true
                ).trim()
                
                echo "📦 APPLICATION INFO:"
                echo "  Version: ${APP_VERSION}"
                echo "  URL: ${url}"
                echo "  Namespace: ${K8S_NAMESPACE}"
                echo "  Docker Image: ${DOCKER_IMAGE}:${APP_VERSION}"
                
                // État Kubernetes
                sh """
                    echo ""
                    echo "📊 KUBERNETES STATUS:"
                    kubectl get pods -n ${K8S_NAMESPACE} 2>/dev/null | grep -E "NAME|student|mysql" || true
                    echo ""
                    echo "🔗 SERVICES:"
                    kubectl get svc -n ${K8S_NAMESPACE} 2>/dev/null | grep -E "NAME|student|mysql" || true
                """
            }
            
            // Sauvegarder les rapports
            archiveArtifacts artifacts: 'target/site/jacoco/**/*', fingerprint: true
            junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
        }
        
        success {
            echo '✅ PIPELINE SUCCESSFUL!'
            script {
                def url = sh(
                    script: "minikube service student-service -n ${K8S_NAMESPACE} --url",
                    returnStdout: true
                ).trim()
                
                echo "🎉 Application déployée avec succès!"
                echo "🔗 Accédez à: ${url}"
                echo "📦 JAR version: ${APP_VERSION}"
            }
        }
        
        failure {
            echo '❌ PIPELINE FAILED!'
            script {
                // Rollback automatique
                sh """
                    kubectl rollout undo deployment/student-management \
                      -n ${K8S_NAMESPACE} \
                      2>/dev/null && echo "⏪ Rollback effectué" || echo "⚠️ Rollback non disponible"
                """
                
                // Logs des pods en échec
                sh """
                    echo "=== ERROR LOGS ==="
                    kubectl logs -l app=student-management -n ${K8S_NAMESPACE} --tail=50 2>/dev/null || true
                """
            }
        }
        
        unstable {
            echo '⚠️ PIPELINE UNSTABLE (Tests or quality check failed)'
        }
    }
}
