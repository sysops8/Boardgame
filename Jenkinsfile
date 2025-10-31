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
    
    stages {
        stage('Checkout Source Code') {
            steps {
                echo "📥 Checking out source code..."
                checkout scm
            }
        }
        
        stage('Set Build Version') {
            steps {
                script {
                    echo "🏷️ Setting Maven version to 0.0.${env.BUILD_NUMBER}"
                    sh "mvn versions:set -DnewVersion=0.0.${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('Build Application') {
            steps {
                echo "🔨 Building Maven project..."
                sh "mvn clean package -DskipTests"
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
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo "📊 Running SonarQube analysis..."
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh "mvn sonar:sonar -Dsonar.host.url=${SONARQUBE_URL}"
                }
            }
        }
        stage('Quality Gate') {
            steps {
                echo "🚦 Waiting for Quality Gate..."
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false, credentialsId: SONARQUBE_CREDENTIALS
                }
            }
        }
        
        stage('Publish to Nexus') {
            steps {
                echo "📤 Publishing Maven artifacts to Nexus..."
                configFileProvider([configFile(fileId: 'maven-settings', variable: 'MAVEN_SETTINGS')]) {
                    withCredentials([usernamePassword(credentialsId: NEXUS_CREDENTIALS, usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PSW')]) {
                        sh """
                            mvn clean deploy -s $MAVEN_SETTINGS \
                                -DaltDeploymentRepository=nexus::default::${NEXUS_URL} \
                                -Dnexus.username=${NEXUS_USER} \
                                -Dnexus.password=${NEXUS_PSW}
                        """
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "🐳 Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    dockerImage = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
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
                            git clone https://${GIT_TOKEN}@github.com/sysops8/boardgame-gitops.git
                            cd boardgame-gitops
                            
                            # Обновление image tag в deployment.yaml
                            sed -i "s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g" base/overlays/production/kustomization.yaml
                            
                            # Или обновление в base/deployment.yaml
                            sed -i "s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g" base/deployment.yaml
                            
                            # Коммит и пуш
                            git config user.email "jenkins@local.lab"
                            git config user.name "Jenkins CI"
                            git add .
                            git commit -m "Update image to ${IMAGE_TAG} - Build #${env.BUILD_NUMBER}" || true
                            git push origin main
                            
                            cd ..
                            rm -rf boardgame-gitops
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
                                        argocd login argocd.local.lab --auth-token $ARGOCD_TOKEN --insecure  # Логин в ArgoCD
                                        argocd app sync boardgame --force                    # Синхронизация приложения
                                        argocd app wait boardgame --timeout 300              # Ожидание завершения
                            '''            
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo "✅ Verifying deployment in Kubernetes..."
                    
                    withCredentials([string(credentialsId: ARGOCD_CREDENTIALS, variable: 'ARGOCD_TOKEN')]) {
                        sh """
                            # Проверка статуса приложения в ArgoCD
                            argocd app get boardgame
                            
                            # Проверка подов в Kubernetes
                            kubectl get pods -n production -l app=boardgame
                            kubectl rollout status deployment/boardgame-deployment -n production --timeout=300s
                        """
                    }
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo "🩺 Performing health check..."
                    
                    retry(3) {
                        sleep(10)
                        sh """
                            # Проверка readiness подов
                            kubectl wait --for=condition=ready pod \
                                -l app=boardgame \
                                -n production \
                                --timeout=60s
                            
                            # HTTP health check
                            LOAD_BALANCER_IP=\$(kubectl get svc boardgame-service -n production -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                            curl -f http://\${LOAD_BALANCER_IP}/ || exit 1
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
        
        always {
            echo "🧹 Cleaning workspace..."
            cleanWs()
        }
    }
}
