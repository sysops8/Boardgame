pipeline {
    agent any

    environment {
        MYAPP = "boardgame"
        // Harbor
        HARBOR_URL = "harbor.local.lab"
        HARBOR_PROJECT = "library"
        HARBOR_CREDENTIALS = "harbor-creds"
        
        // Harbor image name
        IMAGE_NAME = 'boardgame'
        IMAGE_TAG = "${BUILD_NUMBER}"
        FULL_IMAGE_NAME = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"
        LATEST_IMAGE_NAME = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest"
        
        // Nexus
        NEXUS_URL = "http://nexus.local.lab:8081/repository/maven-releases/"
        NEXUS_CREDENTIALS = "nexus-creds"

        // SonarQube
        SONARQUBE_SERVER = "SonarQube"
        SONARQUBE_URL = "http://sonar.local.lab:9000"
        SONARQUBE_CREDENTIALS = "sonar-token"

        // Kubernetes
        KUBECONFIG_CREDENTIALS = "k8s-kubeconfig"

        // Email
        EMAIL_RECIPIENTS = "almastvx@gmail.com"


    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out source code..."
                checkout scm
            }
        }
    stage('Set Build Version') {
        steps {
            script {
                sh "mvn versions:set -DnewVersion=0.0.${env.BUILD_NUMBER}"                
            }
        }
    }
        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${HARBOR_URL}/${HARBOR_PROJECT}/${MYAPP}:${env.BUILD_NUMBER}")
                    
                }
            }
        }

        stage('Push Docker Image to Harbor') {
            steps {
                script {
                    docker.withRegistry("http://${HARBOR_URL}", HARBOR_CREDENTIALS) {
                        dockerImage.push()
                        dockerImage.push('latest') // optional
                    }
                }
            }
        }

        stage('Publish Artifacts to Nexus') {
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

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh "mvn sonar:sonar -Dsonar.host.url=${SONARQUBE_URL}"
                }
            }
        }

        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false, credentialsId: "${SONARQUBE_CREDENTIALS}"
                }
            }
        }
        
        stage('Update K8s Manifest') {
            steps {
                script {
                    echo "📝 Updating Kubernetes manifest with image: ${HARBOR_URL}/${HARBOR_PROJECT}/${MYAPP}:${env.BUILD_NUMBER}"
        
                    // Используем безопасную оболочку без Groovy-интерполяции
                    sh '''
                        IMAGE_TAG="${HARBOR_URL}/${HARBOR_PROJECT}/${MYAPP}:${BUILD_NUMBER}"
                        if [ ! -f k8s_deployment-service.yaml ]; then
                            echo "❌ File k8s_deployment-service.yaml not found!"
                            exit 1
                        fi
                        echo "Updating image to: $IMAGE_TAG"
                        sed -i "0,/image:/s|image: .*|image: $IMAGE_TAG|" k8s_deployment-service.yaml
                        #sed -i 's|newTag:.*|newTag: "'${BUILD_NUMBER}'"|g' k8s_deployment-service.yaml
                        echo "✅ Manifest updated successfully:"
                        grep "image:" k8s_deployment-service.yaml
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG_FILE')]) {
                    sh """
                        export KUBECONFIG=${KUBECONFIG_FILE}
                        kubectl apply -f k8s_deployment-service.yaml
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo "🩺 Checking application health..."
                    withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            export KUBECONFIG=${KUBECONFIG_FILE}
                            kubectl wait --for=condition=available --timeout=60s deployment/boardgame-deployment
                            kubectl get pods -o wide | grep boardgame
                            kubectl wait --for=condition=ready pod -l app=boardgame -n default --timeout=60s  
                        '''
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG_FILE')]) {
                    sh """
                        export KUBECONFIG=${KUBECONFIG_FILE}
                        kubectl rollout status deployment/boardgame-deployment
                    """
                }
            }
        }
    }

        post {
                always {
                    // Архивация артефактов
                    archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
                    archiveArtifacts artifacts: 'trivy-*-report.html', allowEmptyArchive: true
                    
                    // Очистка workspace
                    cleanWs()
                }
                
                success {
                    echo "🎉 Pipeline completed successfully!"
                    echo "📧 Sending success notification..."              
                    
                    emailext(
                        subject: "✅ Pipeline Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
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
                                        <td style="padding: 8px; border: 1px solid #ddd;"><strong>Build Number:</strong></td>
                                        <td style="padding: 8px; border: 1px solid #ddd;">${env.BUILD_NUMBER}</td>
                                    </tr>
                                    <tr>
                                        <td style="padding: 8px; border: 1px solid #ddd;"><strong>Docker Image:</strong></td>
                                        <td style="padding: 8px; border: 1px solid #ddd;">${FULL_IMAGE_NAME}</td>
                                    </tr>
                                    <tr>
                                        <td style="padding: 8px; border: 1px solid #ddd;"><strong>Application URL:</strong></td>
                                        <td style="padding: 8px; border: 1px solid #ddd;">
                                            <a href="https://boardgame.apps.your-domain.com">https://boardgame.apps.your-domain.com</a>
                                        </td>
                                    </tr>
                                    <tr>
                                        <td style="padding: 8px; border: 1px solid #ddd;"><strong>Build URL:</strong></td>
                                        <td style="padding: 8px; border: 1px solid #ddd;">
                                            <a href="${env.BUILD_URL}">${env.BUILD_URL}</a>
                                        </td>
                                    </tr>
                                </table>
                                <p style="margin-top: 20px;">Check the attached Trivy security report for details.</p>
                            </body>
                            </html>
                        """,
                        to: EMAIL_RECIPIENTS,
                        mimeType: 'text/html',
                        attachmentsPattern: 'trivy-image-report.html'
                    )
                }
                
                failure {
                    emailext(
                        subject: "❌ Pipeline Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: """
                            <html>
                            <body style="font-family: Arial, sans-serif;">
                                <h2 style="color: #dc3545;">❌ Pipeline Execution Failed!</h2>
                                <table style="border-collapse: collapse; width: 100%;">
                                    <tr>
                                        <td style="padding: 8px; border: 1px solid #ddd;"><strong>Job:</strong></td>
                                        <td style="padding: 8px; border: 1px solid #ddd;">${env.JOB_NAME}</td>
                                    </tr>
                                    <tr>
                                        <td style="padding: 8px; border: 1px solid #ddd;"><strong>Build Number:</strong></td>
                                        <td style="padding: 8px; border: 1px solid #ddd;">${env.BUILD_NUMBER}</td>
                                    </tr>
                                    <tr>
                                        <td style="padding: 8px; border: 1px solid #ddd;"><strong>Failed Stage:</strong></td>
                                        <td style="padding: 8px; border: 1px solid #ddd;">${env.STAGE_NAME}</td>
                                    </tr>
                                    <tr>
                                        <td style="padding: 8px; border: 1px solid #ddd;"><strong>Console Output:</strong></td>
                                        <td style="padding: 8px; border: 1px solid #ddd;">
                                            <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a>
                                        </td>
                                    </tr>
                                </table>
                                <p style="margin-top: 20px; color: #dc3545;">Please check the console output for detailed error information.</p>
                            </body>
                            </html>
                        """,
                        to: EMAIL_RECIPIENTS,
                        mimeType: 'text/html'
                    )
                }
            }
}
