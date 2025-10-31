pipeline {
    agent any
    
    environment {
        // Harbor
        HARBOR_URL = "harbor.local.lab"
        HARBOR_PROJECT = "library"
        HARBOR_CREDENTIALS = "harbor-creds"
        
        // Nexus
        NEXUS_URL = "http://nexus.local.lab:8081/repository/maven-releases/"
        NEXUS_CREDENTIALS = "nexus-cred"
        
        // SonarQube
        SONARQUBE_SERVER = "SonarQube"
        SONARQUBE_URL = "http://sonar.local.lab:9000"
        SONARQUBE_CREDENTIALS = "sonar-token"
        
        // ArgoCD
        ARGOCD_SERVER = "argocd.local.lab"
        ARGOCD_CREDENTIALS = "argocd-token"
        GITOPS_REPO = "https://github.com/sysops8/Boardgame-gitops.git"
        GITOPS_CREDENTIALS = "github-token"
        
        // Email
        EMAIL_RECIPIENTS = "almastvx@gmail.com"
        
        // Image
        IMAGE_NAME = "${HARBOR_URL}/${HARBOR_PROJECT}/myapp"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Checkout Source Code') {
            steps {
                echo "📥 Checking out source code..."
                checkout scm
            }
        }
        
        stage('Environment Check') {
            steps {
                script {
                    echo "🔧 Checking required tools..."
                    sh '''
                        which mvn || echo "❌ Maven not found"
                        which docker || echo "❌ Docker not found"
                        which trivy || echo "❌ Trivy not found"
                        which kubectl || echo "❌ kubectl not found"
                        which argocd || echo "❌ ArgoCD CLI not found"
                        git --version || echo "❌ Git not found"
                    '''
                }
            }
        }
        
        stage('Set Build Version') {
            steps {
                script {
                    echo "🏷️ Setting Maven version to 0.0.${env.BUILD_NUMBER}"
                    sh "mvn versions:set -DnewVersion=0.0.${env.BUILD_NUMBER} -DgenerateBackupPoms=false"
                }
            }
        }
        
        stage('Build Application') {
            steps {
                echo "🔨 Building Maven project..."
                sh "mvn clean compile -DskipTests"
            }
        }
        
        stage('Run Unit Tests') {
            steps {
                echo "🧪 Running unit tests..."
                sh "mvn test"
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: '**/target/surefire-reports/*.xml', allowEmptyArchive: true
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo "📊 Running SonarQube analysis..."
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh "mvn sonar:sonar -Dsonar.host.url=${SONARQUBE_URL} -Dsonar.projectVersion=0.0.${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                echo "🚦 Waiting for Quality Gate..."
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true, credentialsId: SONARQUBE_CREDENTIALS
                }
            }
        }
        
        stage('Package Application') {
            steps {
                echo "📦 Packaging application..."
                sh "mvn package -DskipTests"
            }
        }
        
        stage('Publish to Nexus') {
            steps {
                echo "📤 Publishing Maven artifacts to Nexus..."
                configFileProvider([configFile(fileId: 'maven-settings', variable: 'MAVEN_SETTINGS')]) {
                    withCredentials([usernamePassword(credentialsId: NEXUS_CREDENTIALS, usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PSW')]) {
                        sh """
                            mvn deploy -s $MAVEN_SETTINGS \
                                -DaltDeploymentRepository=nexus::default::${NEXUS_URL} \
                                -Dnexus.username=${NEXUS_USER} \
                                -Dnexus.password=${NEXUS_PSW} \
                                -DskipTests
                        """
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "🐳 Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    // Убедитесь, что Dockerfile существует в корне проекта
                    dockerImage = docker.build("${IMAGE_NAME}:${IMAGE_TAG}", "--build-arg JAR_FILE=target/*.jar .")
                    dockerImage.tag('latest')
                }
            }
        }
        
        stage('Security Scan with Trivy') {
            steps {
                echo "🔍 Scanning Docker image for vulnerabilities..."
                sh """
                    trivy image --severity HIGH,CRITICAL \
                        --format table \
                        --output trivy-report.txt \
                        --exit-code 0 \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
                    // Читаем отчет для логирования
                    sh 'cat trivy-report.txt || true'
                }
            }
        }
        
        stage('Push to Harbor') {
            steps {
                script {
                    echo "📦 Pushing Docker image to Harbor..."
                    docker.withRegistry("https://${HARBOR_URL}", HARBOR_CREDENTIALS) {
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push('latest')
                    }
                }
            }
        }
        
        stage('Update GitOps Repo') {
            steps {
                script {
                    echo "📝 Updating GitOps repository with new image tag..."
                    
                    withCredentials([string(credentialsId: GITOPS_CREDENTIALS, variable: 'GIT_TOKEN')]) {
                        sh """
                            # Клонирование GitOps репозитория
                            rm -rf boardgame-gitops
                            git clone https://${GIT_TOKEN}@github.com/sysops8/Boardgame-gitops.git
                            cd Boardgame-gitops
                            
                            # Обновление image tag в base/deployment.yaml
                            sed -i "s|image: ${HARBOR_URL}/${HARBOR_PROJECT}/myapp:[0-9]*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g" base/deployment.yaml
                            
                            # Проверяем изменения
                            echo "🔍 Changes made:"
                            git diff || true
                            
                            # Коммит и пуш
                            git config user.email "jenkins@local.lab"
                            git config user.name "Jenkins CI"
                            git add base/deployment.yaml
                            git commit -m "Update image to ${IMAGE_TAG} - Build #${env.BUILD_NUMBER}" || echo "No changes to commit"
                            git push origin main
                            
                            echo "✅ GitOps repository updated successfully"
                            
                            cd ..
                            rm -rf Boardgame-gitops
                        """
                    }
                }
            }
        }
        
        stage('Sync ArgoCD Application') {
            steps {
                script {
                    echo "🔄 Triggering ArgoCD sync..."
        
                    withCredentials([string(credentialsId: ARGOCD_CREDENTIALS, variable: 'ARGOCD_TOKEN')]) {
                        sh '''
                            set +e  # Разрешаем продолжение при ошибках для лучшей диагностики
                            
                            # Логин в ArgoCD
                            echo "🔐 Logging into ArgoCD..."
                            argocd login ${ARGOCD_SERVER} \
                                --auth-token $ARGOCD_TOKEN \
                                --username "api" \
                                --insecure \
                                --grpc-web
                            
                            if [ $? -ne 0 ]; then
                                echo "❌ ArgoCD login failed"
                                exit 1
                            fi
                            
                            # Проверяем статус приложения перед синхронизацией
                            echo "📊 Current application status:"
                            argocd app get boardgame || echo "⚠️ Cannot get application details"
                            
                            # Синхронизация приложения
                            echo "🔄 Starting sync..."
                            argocd app sync boardgame --force --prune
                            
                            SYNC_EXIT_CODE=$?
                            if [ $SYNC_EXIT_CODE -eq 0 ]; then
                                echo "✅ Sync initiated successfully"
                            else
                                echo "⚠️ Sync completed with exit code: $SYNC_EXIT_CODE"
                            fi
                            
                            # Ожидание завершения синхронизации
                            echo "⏳ Waiting for sync to complete..."
                            argocd app wait boardgame --health --timeout 600
                            
                            WAIT_EXIT_CODE=$?
                            if [ $WAIT_EXIT_CODE -eq 0 ]; then
                                echo "🎉 Application synced and healthy"
                            else
                                echo "⚠️ Wait completed with exit code: $WAIT_EXIT_CODE"
                                # Показываем статус для диагностики
                                argocd app get boardgame
                            fi
                            
                            set -e
                        '''
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    echo "✅ Verifying deployment in Kubernetes..."
                    
                    sh """
                        # Проверка подов в Kubernetes
                        echo "📋 Checking pods..."
                        kubectl get pods -n production -l app=boardgame -o wide
                        
                        # Проверка готовности deployment
                        echo "🔄 Checking deployment rollout status..."
                        kubectl rollout status deployment/boardgame-deployment -n production --timeout=300s
                        
                        # Проверка сервисов
                        echo "🌐 Checking services..."
                        kubectl get svc -n production -l app=boardgame
                        
                        # Проверка реплик
                        echo "🔢 Checking replica status..."
                        kubectl get deployment boardgame-deployment -n production -o jsonpath='{.status.availableReplicas}/{.status.replicas} pods available'
                        echo ""
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo "🩺 Performing health check..."
                    
                    retry(3) {
                        sleep(15)  # Увеличиваем задержку для полного старта
                        sh """
                            # Проверка readiness подов
                            kubectl wait --for=condition=ready pod \
                                -l app=boardgame \
                                -n production \
                                --timeout=120s
                            
                            # Получаем IP для health check
                            echo "🌐 Performing HTTP health check..."
                            
                            # Способ 1: через Service (если есть LoadBalancer)
                            if kubectl get svc boardgame-service -n production &>/dev/null; then
                                APP_URL=\$(kubectl get svc boardgame-service -n production -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                                if [ -n "\$APP_URL" ]; then
                                    echo "Testing http://\${APP_URL}/"
                                    curl -f http://\${APP_URL}/ || exit 1
                                else
                                    echo "⚠️ LoadBalancer IP not available, using port-forward method"
                                fi
                            fi
                            
                            # Способ 2: через port-forward (fallback)
                            echo "🔧 Using port-forward for health check..."
                            kubectl port-forward svc/boardgame-service -n production 8080:8080 &
                            PF_PID=\$!
                            sleep 5
                            curl -f http://localhost:8080/ || exit 1
                            kill \$PF_PID
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            script {
                def appUrl = "http://boardgame.local.lab"
                def argocdUrl = "https://${ARGOCD_SERVER}/applications/boardgame"
                
                echo "🎉 Pipeline completed successfully!"
                echo "📧 Sending success notification..."
                
                emailext(
                    subject: "✅ SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <html>
                        <body style="font-family: Arial, sans-serif;">
                            <h2 style="color: #28a745;">🎉 Pipeline Executed Successfully!</h2>
                            <table style="border-collapse: collapse; width: 100%;">
                                <tr>
                                    <td style="padding: 8px; border: 1px solid #ddd;"><strong>Job:</strong></td>
                                    <td style="padding: 8px; border: 1px solid #ddd;">${env.JOB_NAME}</td>
                                </tr>
                                <tr>
                                    <td style="padding: 8px; border: 1px solid #ddd;"><strong>Build:</strong></td>
                                    <td style="padding: 8px; border: 1px solid #ddd;">#${env.BUILD_NUMBER}</td>
                                </tr>
                                <tr>
                                    <td style="padding: 8px; border: 1px solid #ddd;"><strong>Docker Image:</strong></td>
                                    <td style="padding: 8px; border: 1px solid #ddd;">${IMAGE_NAME}:${IMAGE_TAG}</td>
                                </tr>
                                <tr>
                                    <td style="padding: 8px; border: 1px solid #ddd;"><strong>Application:</strong></td>
                                    <td style="padding: 8px; border: 1px solid #ddd;">
                                        <a href="${appUrl}">${appUrl}</a>
                                    </td>
                                </tr>
                                <tr>
                                    <td style="padding: 8px; border: 1px solid #ddd;"><strong>ArgoCD:</strong></td>
                                    <td style="padding: 8px; border: 1px solid #ddd;">
                                        <a href="${argocdUrl}">${argocdUrl}</a>
                                    </td>
                                </tr>
                                <tr>
                                    <td style="padding: 8px; border: 1px solid #ddd;"><strong>Build URL:</strong></td>
                                    <td style="padding: 8px; border: 1px solid #ddd;">
                                        <a href="${env.BUILD_URL}">${env.BUILD_URL}</a>
                                    </td>
                                </tr>
                            </table>
                            <p style="margin-top: 20px;">
                                <strong>Deployed by ArgoCD via GitOps</strong><br>
                                GitOps Repo: ${GITOPS_REPO}
                            </p>
                        </body>
                        </html>
                    """,
                    to: EMAIL_RECIPIENTS,
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-report.txt'
                )
            }
        }
        
        failure {
            script {
                echo "❌ Pipeline failed!"
                emailext(
                    subject: "❌ FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <html>
                        <body style="font-family: Arial, sans-serif;">
                            <h2 style="color: #dc3545;">❌ Pipeline Failed!</h2>
                            <table style="border-collapse: collapse; width: 100%;">
                                <tr>
                                    <td style="padding: 8px; border: 1px solid #ddd;"><strong>Job:</strong></td>
                                    <td style="padding: 8px; border: 1px solid #ddd;">${env.JOB_NAME}</td>
                                </tr>
                                <tr>
                                    <td style="padding: 8px; border: 1px solid #ddd;"><strong>Build:</strong></td>
                                    <td style="padding: 8px; border: 1px solid #ddd;">#${env.BUILD_NUMBER}</td>
                                </tr>
                                <tr>
                                    <td style="padding: 8px; border: 1px solid #ddd;"><strong>Failed Stage:</strong></td>
                                    <td style="padding: 8px; border: 1px solid #ddd;">${env.STAGE_NAME}</td>
                                </tr>
                                <tr>
                                    <td style="padding: 8px; border: 1px solid #ddd;"><strong>Console:</strong></td>
                                    <td style="padding: 8px; border: 1px solid #ddd;">
                                        <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a>
                                    </td>
                                </tr>
                            </table>
                        </body>
                        </html>
                    """,
                    to: EMAIL_RECIPIENTS,
                    mimeType: 'text/html'
                )
            }
        }
        
        always {
            echo "🧹 Cleaning workspace..."
            // Сохраняем важные артефакты перед очисткой
            archiveArtifacts artifacts: '**/target/*.jar,trivy-report.txt', allowEmptyArchive: true
            cleanWs()
        }
    }
}
