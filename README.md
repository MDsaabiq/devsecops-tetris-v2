# DevSecOps Tetris V2

<video src="https://github.com/user-attachments/assets/b80a1246-2cff-4526-a8c5-f5264b1ca65a" controls></video>

End-to-end DevSecOps lab using Jenkins, SonarQube, Trivy, Docker, GitOps, ArgoCD, kOps, and HPA. This README contains a recruiter-ready project summary plus a student-friendly lab manual with step-by-step commands and expected outputs.

---

## Demo Video
[Demo Video](https://github.com/user-attachments/assets/1ae5d562-53e4-4051-946c-e1b98a370791)

## Recruiter-Ready Overview

**What I built**
- Production-style CI/CD pipeline for a React game app with code quality gates, container security scanning, GitOps-driven deployments, and Kubernetes autoscaling.
- Jenkins handles build and GitOps manifest updates only; ArgoCD is the sole deployer to Kubernetes.

**Key outcomes**
- Automated quality checks with SonarQube and security scanning with Trivy.
- Docker image builds and pushes to DockerHub.
- GitOps manifest updates trigger ArgoCD syncs for zero-downtime updates.
- HPA scales workloads based on CPU load during synthetic traffic tests.

**Tech stack**
- CI: Jenkins, SonarQube
- Security: Trivy
- Container: Docker, DockerHub
- GitOps: ArgoCD, GitHub
- Kubernetes: kOps, kubectl, HPA
- Cloud: AWS EC2, S3
- App: React (Tetris)

**Project structure**
- React app source in [src/](src/)
- Container build in [Dockerfile](Dockerfile)

**Screenshots**
- [assets/react-tetris-desktop-1.png](assets/react-tetris-desktop-1.png)
- [assets/react-tetris-mobile-1.png](assets/react-tetris-mobile-1.png)
- [assets/react-tetris-mobile-2.png](assets/react-tetris-mobile-2.png)

---

## DevSecOps End-to-End Lab Manual

### About This Lab
This is a clean, student-friendly lab manual derived from the final working project flow. It is a run-book, not reference notes.

**Lab Rules**
1. Jenkins only builds and updates Git.
2. Jenkins never touches Kubernetes.
3. ArgoCD is the only deployer.
4. HPA controls replicas, not you.
5. BusyBox load is temporary only.
6. No AWS-level deletion (student-safe).

---

### Phase 0: Accounts and Prerequisites

**Required accounts**
- AWS account (free tier is enough)
- GitHub: `MDsaabiq`
- DockerHub: `mdsaabiq`

**Local knowledge**
- Basic Linux commands
- Git basics (clone, commit, push)

---

### Phase 1: AWS Management Node (EC2)

**Step 1.1: Launch EC2 instance**

| Setting | Value |
| --- | --- |
| Name | devsecops-jenkins |
| AMI | Ubuntu 24.04 LTS |
| Instance | t2.large |
| Storage | 50 GB |
| IAM Role | AdministratorAccess |
| Security Group | SSH 22 to My IP, TCP 8080 to Anywhere (Jenkins), TCP 9000 to Anywhere (SonarQube) |

**Step 1.2: SSH into instance**

```bash
ssh -i key.pem ubuntu@<EC2_PUBLIC_IP>
```

**Expected output**
- Login success

---

### Phase 2: Tool Installation (One-Time Setup)

**Step 2.1: Create setup script**

```bash
vi setup.sh
```

Paste the full setup script exactly as provided in the reference.

**Step 2.2: Run script**

```bash
chmod +x setup.sh
./setup.sh
```

**Tools installed**
- Java 17
- Jenkins
- Docker
- AWS CLI
- kubectl
- kOps
- Trivy
- SonarQube (Docker container)

**Expected output**
- Jenkins running on port 8080
- SonarQube running on port 9000

---

### Phase 3: Jenkins Initial Configuration

**Step 3.1: Unlock Jenkins**

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open:
```
http://<EC2_PUBLIC_IP>:8080
```

Actions:
- Install suggested plugins
- Create admin user

---

### Phase 4: SonarQube Configuration

**Step 4.1: Access SonarQube**

```
http://<EC2_PUBLIC_IP>:9000
```

Login: `admin` / `admin`

**Step 4.2: Generate token**

Path: Profile -> My Account -> Security
Token name: `tetris-token`

**Step 4.3: Add token to Jenkins**

| Field | Value |
| --- | --- |
| Kind | Secret Text |
| ID | sonar-token |
| Value | Paste token |

**Step 4.4: Configure Sonar server**

Jenkins -> Manage Jenkins -> System

| Setting | Value |
| --- | --- |
| Name | sonar-server |
| URL | http://<EC2_PUBLIC_IP>:9000 |
| Credential | sonar-token |

**Step 4.5: Configure Sonar webhook (Mandatory)**

SonarQube -> Administration -> Webhooks

URL:
```
http://<EC2_PUBLIC_IP>:8080/sonarqube-webhook/
```

**Debug note**
- Without webhook, Quality Gate waits forever.

---

### Phase 5: Jenkins Credentials

**DockerHub credential**

| Field | Value |
| --- | --- |
| ID | docker |
| Username | mdsaabiq |
| Password | DockerHub password |

**GitHub token**

- Create a PAT (Classic) with all scopes
- Add as:

| Field | Value |
| --- | --- |
| ID | github |
| Kind | Secret Text |
| Value | GitHub PAT |

---

### Phase 6: Kubernetes Cluster (kOps)

**Step 6.1: AWS CLI config**

```bash
aws configure
```

**Step 6.2: Create kOps state store**

```bash
export BUCKET_NAME=tetris-kops-$(date +%s)
aws s3api create-bucket --bucket $BUCKET_NAME --region ap-south-1 --create-bucket-configuration LocationConstraint=ap-south-1
aws s3api put-bucket-versioning --bucket $BUCKET_NAME --versioning-configuration Status=Enabled
```

**Step 6.3: Create cluster**

```bash
export KOPS_STATE_STORE=s3://$BUCKET_NAME
export CLUSTER_NAME=tetris.k8s.local
kops create cluster --name=$CLUSTER_NAME --zones=ap-south-1a --node-count=2 --node-size=t3.medium --master-size=t3.medium --yes
```

**Step 6.4: Validate cluster**

```bash
kops validate cluster --wait 10m
kubectl get nodes
```

**Expected output**
- All nodes Ready

---

### Phase 7: CI Pipeline (Version 1)

**Purpose**
- Build app
- Scan code
- Push Docker image
- Update GitOps manifest
- Jenkins does not deploy

**Pipeline (V1)**

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
			steps {
				checkout scm
			}
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
					sh "rm -rf gitops"
					sh "git clone https://${GITHUB_TOKEN}@github.com/MDsaabiq/devsecops-Tetris-manifest.git gitops"
					sh "sed -i 's#image: .*#image: ${IMAGE_REPO}:${IMAGE_TAG}#' gitops/deployment.yaml"
					sh "cd gitops && git add deployment.yaml && git commit -m 'Update image to ${IMAGE_TAG}' && git push"
				}
			}
		}
	}
	post {
		always {
			sh "docker logout || true"
		}
	}
}
```

**Expected output**
- Pipeline is green
- Image pushed to DockerHub
- Manifest repo updated

**Debug notes**
- If Quality Gate hangs, check Sonar webhook.
- If Trivy fails, fix findings before proceeding.

---

### Phase 8: ArgoCD (GitOps Deployment)

**Install ArgoCD**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"LoadBalancer"}}'
```

**Access ArgoCD**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Login:
- Username: `admin`
- Password: output above

**Create application**

| Field | Value |
| --- | --- |
| Repo | devsecops-Tetris-manifest |
| Path | ./ |
| Sync | Automatic |
| Prune | Enabled |
| Self-Heal | Enabled |

**Expected output**
- App is Synced
- App is Healthy

---

### Phase 9: Zero Downtime Update (V2)

Change only:
- Docker tag
- App repo (V2)

**Expected output**
- Old pods terminate after new pods are ready
- No downtime

---

### Phase 10: Resource and HPA Configuration

**Deployment resources (Mandatory)**

```yaml
resources:
	requests:
		cpu: "100m"
		memory: "128Mi"
	limits:
		cpu: "500m"
		memory: "256Mi"
```

**HPA**

```yaml
minReplicas: 2
maxReplicas: 6
targetCPU: 50%
```

**Debug note**
- HPA overrides replica count, this is expected.

---

### Phase 11: Load Test (BusyBox)

**Fix kubeconfig expiry**

```bash
kops export kubeconfig --name tetris.k8s.local --admin
```

**Run load**

```bash
kubectl run load-generator --image=busybox --restart=Never -it -- /bin/sh
```

Inside the pod:

```bash
while true; do wget -q -O- http://tetris-service; sleep 0.3; done
```

Observe:

```bash
kubectl get hpa -w
kubectl get pods
```

---

### Phase 12: Safe Cleanup

```bash
kubectl delete pod load-generator --ignore-not-found
kubectl delete hpa tetris-hpa --ignore-not-found
kubectl scale deployment tetris --replicas=2
```

---

## Notes and Assumptions

- This lab uses kOps for cluster management and ArgoCD for deployment.
- Jenkins never applies manifests to Kubernetes.
- Adjust image tags and repo URLs as needed for your account.

## Local Development (Optional)

```bash
npm install
npm run dev
```

Open `http://localhost:5173`.
