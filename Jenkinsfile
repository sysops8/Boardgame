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

        // Kubernetes
        KUBECONFIG_CREDENTIALS = "k8s-kubeconfig"
        K8S_MANIFEST = "k8s_deployment-service.yaml"

        // Email
        EMAIL_RECIPIENTS = "almastvx@gmail.com"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "📦 Checking out source code..."
                checkout scm
            }
        }

        stage('Set Build Version') {
            steps {
                script {
                    echo "⚙️ Setting project version to 0.0.${env.BUILD_NUMBER}"
                    sh "mvn versions:set -DnewVersion=0.0.${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    IMAGE_TAG = "${HARBOR_URL}/${HARBOR_PROJECT}/myapp:${env.BUILD_NUMBER}"
                    echo "🐳 Building Docker image: ${IMAGE_TAG}"
                    dockerImage = docker.build(IMAGE_TAG)
                }
            }
        }

        stage('Push Docker Image to Harbor') {
            steps {
                script {
                    echo "📤 Pushing image to Harbor: ${IMAGE_TAG}"
                    docker.withRegistry("http://${HARBOR_URL}", HARBOR_CREDENTIALS) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Publish Artifacts to Nexus') {
            steps {
                echo "📦 Publishing Maven artifacts to Nexus..."
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
                echo "🔍 Running SonarQube analysis..."
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh "mvn sonar:sonar -Dsonar.host.url=${SONARQUBE_URL}"
                }
            }
        }

        stage('Update K8s Manifest') {
            steps {
                script {
                    echo "📝 Updating Kubernetes manifest with image: ${IMAGE_TAG}"

                    sh '''
                        set -e
                        FILE_PATH="${K8S_MANIFEST}"

                        if [ ! -f "$FILE_PATH" ]; then
                            echo "❌ ERROR: Kubernetes manifest not found: $FILE_PATH"
                            exit 1
                        fi

                        echo "🔍 Found manifest: $FILE_PATH"
                        echo "Replacing image with: ${IMAGE_TAG}"

                        # безопасная замена с сохранением отступов
                        awk -v new_image="${IMAGE_TAG}" '
                            /^[[:space:]]*image:[[:space:]]*/ && updated==0 {
                                indent = match($0, /[^ ]/)
                                printf "%*simage: %s\\n", indent-1, "", new_image
                                updated=1
                                next
                            }
                            { print }
                        ' "$FILE_PATH" > tmp.yaml && mv tmp.yaml "$FILE_PATH"

                        echo "✅ Manifest updated successfully:"
                        grep 'image:' "$FILE_PATH"

                        echo "🔍 Validating manifest syntax..."
                        kubectl apply --dry-run=client -f "$FILE_PATH"
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "🚀 Deploying to Kubernetes cluster..."
                    withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            set -e
                            export KUBECONFIG=${KUBECONFIG_FILE}
                            kubectl apply -f ${K8S_MANIFEST}
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    echo "🔄 Verifying rollout status of deployment..."
                    withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            set -e
                            export KUBECONFIG=${KUBECONFIG_FILE}
                            kubectl rollout status deployment/boardgame-deployment --timeout=120s
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build and deployment succeeded!"
            mail to: "${EMAIL_RECIPIENTS}",
                subject: "✅ SUCCESS: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                body: """The Jenkins job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' completed successfully.
Build URL: ${env.BUILD_URL}
Image deployed: ${HARBOR_URL}/${HARBOR_PROJECT}/myapp:${env.BUILD_NUMBER}
"""
        }
        failure {
            echo "❌ Build or deployment failed!"
            mail to: "${EMAIL_RECIPIENTS}",
                subject: "❌ FAILURE: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                body: "The Jenkins job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' failed.\nBuild URL: ${env.BUILD_URL}"
        }
        unstable {
            echo "⚠️ Build marked as unstable."
            mail to: "${EMAIL_RECIPIENTS}",
                subject: "⚠️ UNSTABLE: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                body: "The Jenkins job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' is unstable.\nBuild URL: ${env.BUILD_URL}"
        }
        always {
            echo "🧹 Cleaning workspace..."
            cleanWs()
        }
    }
}
