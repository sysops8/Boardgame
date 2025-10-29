# 🚀 Jenkins Multibranch CI/CD Pipeline

Этот проект использует **Jenkins Multibranch Pipeline** для автоматической сборки, тестирования, анализа качества кода и деплоя приложения в Kubernetes.

---

## 🧩 Общая структура

```
├── Jenkinsfile
├── pom.xml
├── src/
├── k8s/
│ ├── deployment-dev.yaml
│ ├── deployment-stage.yaml
│ └── deployment-prod.yaml
└── README.md
```



---

## ⚙️ Jenkins Multibranch Pipeline

### 🔹 Что делает Pipeline

| Этап | Описание |
|------|-----------|
| **Checkout** | Клонирует текущую ветку из Git |
| **Build & Test** | Собирает Maven-проект и выполняет тесты |
| **SonarQube Analysis** | Анализирует код и отправляет результаты в SonarQube |
| **Docker Build & Push** | Собирает Docker-образ и публикует в Nexus или DockerHub |
| **Deploy to Kubernetes** | Разворачивает приложение в соответствующее окружение (`dev`, `stage`, `prod`) |
| **Verify Deployment** | Проверяет статус rollout в Kubernetes |

---

## 🌿 Ветки и окружения

Pipeline автоматически определяет окружение по названию ветки Git:

| Ветка Git | Namespace в Kubernetes | Файл манифеста |
|------------|------------------------|----------------|
| `develop`  | `dev`                  | `k8s/deployment-dev.yaml` |
| `main`     | `prod`                 | `k8s/deployment-prod.yaml` |
| `feature/*`| `stage`                | `k8s/deployment-stage.yaml` |

💡 Для каждой ветки создаётся отдельная сборка, Jenkins подставляет соответствующий namespace и деплой-файл.

---

## 🔐 Credentials

| ID | Тип | Назначение |
|----|-----|-------------|
| `nexus-cred` | Username + Password | Доступ к Nexus Repository |
| `sonar-token` | Secret Text | Токен для SonarQube |
| `dockerhub-cred` | Username + Password | Доступ к Docker Hub |
| `k8s-kubeconfig` | Secret File | Файл `~/.kube/config` из K3s master |

---

## 🏗️ Jenkinsfile (основные фрагменты)

```groovy
pipeline {
    agent any

    environment {
        KUBECONFIG_CREDENTIALS = 'k8s-kubeconfig'
    }

    stages {
        stage('Build & Test') {
            steps {
                sh 'mvn clean package -DskipTests=false'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                sh 'docker build -t myrepo/boardgame:${BUILD_NUMBER} .'
                sh 'docker push myrepo/boardgame:${BUILD_NUMBER}'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: 'unknown'
                    def namespace = (branch == 'main') ? 'prod' :
                                    (branch == 'develop') ? 'dev' : 'stage'

                    def deploymentFile = "k8s/deployment-${namespace}.yaml"

                    withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG_FILE')]) {
                        withEnv(["KUBECONFIG=${KUBECONFIG_FILE}"]) {
                            sh """
                                kubectl apply -f ${deploymentFile} -n ${namespace}
                                kubectl rollout status deployment/boardgame-deployment -n ${namespace}
                            """
                        }
                    }
                }
            }
        }
    }
}
```
**🧠 Как работает деплой**
- Jenkins определяет ветку (env.BRANCH_NAME)
- Выбирает нужный namespace и манифест
- Применяет манифест командой kubectl apply -f ...
- Проверяет статус деплоя через kubectl rollout status
- В случае ошибки уведомляет по почте или в Slack

🚀 Пример деплоя вручную
```bash

# Для dev окружения
kubectl apply -f k8s/deployment-dev.yaml -n dev

# Для prod окружения
kubectl apply -f k8s/deployment-prod.yaml -n prod
```
**🧩 Требования**
- Jenkins 2.440+
- Плагин Pipeline: Multibranch
- Плагин Kubernetes CLI

**Установленные утилиты:**
- kubectl
- maven
- docker

✅ Рекомендации
Каждый namespace (dev, stage, prod) должен существовать в Kubernetes:

```bash
Копировать код
kubectl create namespace dev
kubectl create namespace stage
kubectl create namespace prod
```
Обновляйте k3s.yaml при изменении конфигурации кластера


Проверяйте права у Jenkins: kubectl auth can-i apply -A --as system:serviceaccount:default:jenkins

