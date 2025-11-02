pipeline {
    agent any
    environment { 
        // AppName
        MY_APP = 'boardgame'
        
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
        GITOPS_REPO = "https://github.com/sysops8/Boardgame-gitops.git"  
        GITOPS_CREDENTIALS = "github-gitops-token"
        GITOPS_KUSTOMIZATION_PATH = "apps/boardgame/kustomization.yaml"

        // Tools
        SCANNER_HOME = tool 'sonar-scanner'
    }

    tools {
        jdk 'java17'
        maven 'maven3.6'
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out source code..."
                checkout scm
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                    trivy fs \
                        --format table \
                        --output trivy-fs-report.html \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        .
                '''
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
                    dockerImage = docker.build("${HARBOR_URL}/${HARBOR_PROJECT}/${MY_APP}:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image \
                        --format table \
                        --output trivy-image-report.html \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        ${HARBOR_URL}/${HARBOR_PROJECT}/${MY_APP}:${env.BUILD_NUMBER}
                """
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
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=${MY_APP} \
                        -Dsonar.projectKey=${MY_APP} \
                        -Dsonar.host.url=${SONARQUBE_URL} \
                        -Dsonar.java.binaries=target/classes
                    """
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

        stage('Update GitOps Repository') {
            steps {
                script {
                    echo "🚀 Updating GitOps repository with new image version..."
                    
                    withCredentials([usernamePassword(credentialsId: GITOPS_CREDENTIALS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                        sh '''
                            git clone https://$GIT_USER:$GIT_TOKEN@github.com/sysops8/Boardgame-gitops.git gitops-repo
                            cd gitops-repo

                            echo "=== Updating image version ==="
                            sed -i 's|newTag:.*|newTag: "'${BUILD_NUMBER}'"|g' ${GITOPS_KUSTOMIZATION_PATH}
                            git config user.email "jenkins@local.lab"
                            git config user.name "Jenkins CI"
                            git add ${GITOPS_KUSTOMIZATION_PATH}
                            git commit -m "Deploy ${MY_APP} version '${BUILD_NUMBER}'"
                            git push origin main
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
                            argocd app sync $MY_APP --server ${ARGOCD_SERVER} --auth-token ${ARGOCD_TOKEN} --grpc-web --insecure
                            argocd app get "$MY_APP" --server ${ARGOCD_SERVER} --auth-token ${ARGOCD_TOKEN} --grpc-web --insecure
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
                            #kubectl wait --for=condition=available --timeout=60s deployment/boardgame-deployment
                            #kubectl get pods -o wide | grep boardgame
                            # Короткое ожидание готовности
                            kubectl wait --for=condition=ready \
                            pod -l app=boardgame,managed-by=argocd \
                            -n production \
                            --timeout=60s                            
                        '''
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG_FILE')]) {
                    echo "✅ Verifying deployment in Kubernetes..."
                    sh """
                        #export KUBECONFIG=${KUBECONFIG_FILE}
                        #kubectl rollout status deployment/boardgame-deployment    
                        echo "Checking pods..."
                        kubectl get pods -n production -l app=boardgame -o wide
                        
                        echo "Checking deployment rollout status..."
                        kubectl rollout status deployment/boardgame-deployment -n production --timeout=300s
                        
                        echo "Checking services..."
                        kubectl get svc -n production -l app=boardgame
                        
                        echo "Checking replica status..."
                        kubectl get deployment boardgame-deployment -n production -o jsonpath='{.status.availableReplicas}/{.status.replicas} pods available'
                        echo ""
                    """

                        
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
                - Image: ${HARBOR_URL}/${HARBOR_PROJECT}/${MY_APP}:${env.BUILD_NUMBER}
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
            archiveArtifacts artifacts: 'trivy-*-report.html', allowEmptyArchive: true
            cleanWs()
        }
    }
}
