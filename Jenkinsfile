pipeline {
    agent {
        docker {
            image 'maven:3.9.4-eclipse-temurin-17'
            args '-u root:root' // чтобы позволить установку trivy и sonar-scanner
        }
    }

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
    }

    stages {

        stage('Prepare Environment') {
            steps {
                sh '''
                    apt-get update -y
                    apt-get install -y curl wget jq git
                    if ! command -v trivy >/dev/null 2>&1; then
                        echo "Installing Trivy..."
                        wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor -o /usr/share/keyrings/trivy.gpg
                        echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" > /etc/apt/sources.list.d/trivy.list
                        apt-get update && apt-get install -y trivy
                    fi
                    if ! command -v sonar-scanner >/dev/null 2>&1; then
                        echo "Installing Sonar Scanner..."
                        apt-get install -y unzip
                        wget -q https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
                        unzip sonar-scanner-cli-5.0.1.3006-linux.zip -d /opt/
                        ln -s /opt/sonar-scanner-*/bin/sonar-scanner /usr/local/bin/sonar-scanner
                    fi
                '''
            }
        }

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
                        sonar-scanner \
                        -Dsonar.projectKey=${MY_APP} \
                        -Dsonar.projectName=${MY_APP} \
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
