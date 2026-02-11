# DevSecOps Tetris V2

A production-grade **DevSecOps CI/CD Pipeline** for a React-based Tetris application, featuring automated security gates, GitOps-driven deployments, and high-availability infrastructure on AWS.

---

## 📽️ Demo Video

[![Demo Video](https://github.com/user-attachments/assets/b80a1246-2cff-4526-a8c5-f5264b1ca65a)](https://github.com/user-attachments/assets/b80a1246-2cff-4526-a8c5-f5264b1ca65a)

---

## 🚀 Project Overview

### 🌟 Key Highlights
- **GitOps Implementation**: Leveraged **ArgoCD** for declarative, automated, and zero-downtime Kubernetes deployments.
- **Security-First Approach**: Integrated **SonarQube** (SAST) and **Trivy** (SCA) to enforce strict code quality and container security gates.
- **Scalable Infrastructure**: Deployed to a **Kubernetes** cluster managed by **kOps**, with **Horizontal Pod Autoscaler (HPA)** for dynamic resource management.
- **Full Automation**: Engineered a seamless Jenkins pipeline that triggers from code push to production, including manifest updates in a dedicated GitOps repository.

### 🛠️ Tech Stack
- **Infrastructure**: AWS (EC2, S3), kOps, Kubernetes
- **CI/CD**: Jenkins, ArgoCD (GitOps)
- **Security**: SonarQube, Trivy
- **DevOps**: Docker, DockerHub, Git
- **App**: React, Node.js

---

## 🏗️ CI/CD Pipeline Architecture

The pipeline follows a strict separation of concerns where Jenkins handles the build and manifest updates, while ArgoCD manages the deployment state.

### Jenkins Pipeline (Declarative)
```groovy
pipeline {
    agent any
    environment {
        APP_NAME = "tetris"
        IMAGE_REPO = "mdsaabiq/devsecops-tetris"
        IMAGE_TAG = "v1-${BUILD_NUMBER}"
        SONARQUBE_ENV = "sonar-server"
        GITOPS_REPO = "https://github.com/MDsaabiq/devsecops-Tetris-manifest.git"
    }
    stages {
        stage("Checkout") {
            steps { checkout scm }
        }
        stage("SonarQube Scan") {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh "sonar-scanner -Dsonar.projectKey=${APP_NAME} -Dsonar.sources=."
                }
            }
        }
        stage("Quality Gate") {
            steps {
                timeout(time: 5, unit: "MINUTES") {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage("Trivy Scan") {
            steps {
                sh "trivy fs --exit-code 1 --severity HIGH,CRITICAL ."
            }
        }
        stage("Docker Build and Push") {
            steps {
                withCredentials([usernamePassword(credentialsId: "docker", usernameVariable: "DOCKER_USER", passwordVariable: "DOCKER_PASS")]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker build -t ${IMAGE_REPO}:${IMAGE_TAG} ."
                    sh "docker push ${IMAGE_REPO}:${IMAGE_TAG}"
                }
            }
        }
        stage("Update GitOps Manifest") {
            steps {
                withCredentials([string(credentialsId: "github", variable: "GITHUB_TOKEN")]) {
                    sh "rm -rf gitops && git clone https://${GITHUB_TOKEN}@github.com/MDsaabiq/devsecops-Tetris-manifest.git gitops"
                    sh "sed -i 's#image: .*#image: ${IMAGE_REPO}:${IMAGE_TAG}#' gitops/deployment.yaml"
                    sh "cd gitops && git add deployment.yaml && git commit -m 'Update image to ${IMAGE_TAG}' && git push"
                }
            }
        }
    }
}
```

---

## ☸️ Kubernetes & GitOps Configuration

### ArgoCD Deployment
- **Repo**: `devsecops-Tetris-manifest`
- **Sync Policy**: Automatic with Self-Heal and Prune enabled.
- **Strategy**: Zero-downtime rolling updates.

### High Availability (HPA)
The application scales dynamically based on CPU utilization to handle traffic spikes.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: tetris-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: tetris
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

---

## 🔄 V1 to V2: The GitOps Transition

Updating the application from V1 to V2 is fully automated through the GitOps workflow:

1.  **Code Change**: Push new code (V2) to the application repository.
2.  **Jenkins Build**: Jenkins builds the new V2 image and pushes it to DockerHub.
3.  **Manifest Update**: Jenkins automatically updates the image tag in the **GitOps Manifest Repository**.
4.  **ArgoCD Sync**: ArgoCD detects the change and triggers a **Zero-Downtime Rolling Update** in Kubernetes, seamlessly replacing V1 pods with V2.

---

## 💻 Local Development

```bash
npm install
npm run dev
```
Open `http://localhost:5173` to play.
