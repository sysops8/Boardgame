pipeline {
    agent any

    environment {
        MYAPP = "boardgame"
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

        // Kubernetes
        KUBECONFIG_CREDENTIALS = "k8s-kubeconfig"

        // Email
        EMAIL_RECIPIENTS = "almastvx@gmail.com"

        // ArgoCD
        ARGOCD_SERVER = "argocd.local.lab"
        ARGOCD_CREDENTIALS = "argocd-token"
        GITOPS_REPO = "https://github.com/sysops8/Boardgame-gitops.git"  // Добавлен https://
        GITOPS_CREDENTIALS = "github-gitops-token"
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
                    dockerImage = docker.build("${HARBOR_URL}/${HARBOR_PROJECT}/myapp:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Push Docker Image to Harbor') {
            steps {
                script {
                    docker.withRegistry("http://${HARBOR_URL}", HARBOR_CREDENTIALS) {
                        dockerImage.push()
                        dockerImage.push('latest')
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

            stage('Update GitOps Repository') {
                steps {
                    script {
                        echo "🚀 Updating GitOps repository with new image version..."
                        
                        withCredentials([usernamePassword(credentialsId: GITOPS_CREDENTIALS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                            sh '''
                                # Клонируем GitOps репозиторий
                                git clone https://$GIT_USER:$GIT_TOKEN@github.com/sysops8/Boardgame-gitops.git gitops-repo
                                cd gitops-repo
                                
                                echo "=== Проверка структуры репозитория ==="
                                find . -name "*.yaml" -type f
                                
                                echo "=== Текущее содержимое base/boardgame/kustomization.yaml ==="
                                cat base/boardgame/kustomization.yaml
                                
                                # Обновляем версию образа в базовых манифестах
                                echo "=== Обновляем версию образа ==="
                                #sed -i 's|newTag:.*|newTag: ''' + env.BUILD_NUMBER + '''|g' base/boardgame/kustomization.yaml
                                #sed -i 's|newTag:.*|newTag: "${BUILD_NUMBER}"|g' base/boardgame/kustomization.yaml
                                #sed -i `s|newTag:.*|newTag: "${BUILD_NUMBER}"|g` base/boardgame/kustomization.yaml
                                sed -i 's|newTag:.*|newTag: "'${BUILD_NUMBER}'"|g' base/boardgame/kustomization.yaml
                                
                                echo "=== Обновленное содержимое base/boardgame/kustomization.yaml ==="
                                cat base/boardgame/kustomization.yaml
                                
                                echo "=== Статус git ==="
                                git status
                                
                                # Коммитим и пушим изменения
                                git config user.email "jenkins@local.lab"
                                git config user.name "Jenkins CI"
                                git add base/boardgame/kustomization.yaml
                                git commit -m "Deploy myapp version ''' + env.BUILD_NUMBER + '''"
                                git push origin main
                                
                                echo "✅ GitOps repository updated successfully"
                            '''
                        }
                    }
                }
            }

        stage('Sync ArgoCD Application') {
            steps {
                script {
                    echo "🔄 Synchronizing ArgoCD application..."
                    
                    withCredentials([string(credentialsId: ARGOCD_CREDENTIALS, variable: 'ARGOCD_TOKEN')]) {
                        sh '''                    
                            # Логин в ArgoCD
                            # argocd login ''' + env.ARGOCD_SERVER + ''' --username admin --password "$ARGOCD_TOKEN" --insecure
                            argocd app sync boardgame --server ${ARGOCD_SERVER} --auth-token "$ARGOCD_TOKEN" --insecure

                            # Синхронизируем приложение
                            argocd app sync "$MY_APP"
                            
                            # Ждем завершения синхронизации
                            argocd app wait "$MY_APP" --health --timeout 300 
                            
                            # Проверяем статус
                            argocd app get "$MY_APP" 
                            
                            echo "✅ ArgoCD synchronization completed"
                        '''
                    }
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
        success {
            mail to: "${EMAIL_RECIPIENTS}",
                subject: "✅ SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
The Jenkins job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' completed successfully.

📦 Deployment Details:
- Image: ${HARBOR_URL}/${HARBOR_PROJECT}/myapp:${env.BUILD_NUMBER}
- ArgoCD Application: ${MY_APP}
- GitOps Repository: ${GITOPS_REPO}

🔗 Links:
- Build URL: ${env.BUILD_URL}
- ArgoCD UI: https://${ARGOCD_SERVER}
- Application: https://boardgame.local.lab
"""
        }
        failure {
            mail to: "${EMAIL_RECIPIENTS}",
                subject: "❌ FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "The Jenkins job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' failed.\nBuild URL: ${env.BUILD_URL}"
        }
        unstable {
            mail to: "${EMAIL_RECIPIENTS}",
                subject: "⚠️ UNSTABLE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "The Jenkins job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' is unstable.\nBuild URL: ${env.BUILD_URL}"
        }
        always {
            cleanWs()
        }
    }
}
