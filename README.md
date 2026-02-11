# DevSecOps Tetris V2
<!-- 
<video src="https://github.com/user-attachments/assets/b80a1246-2cff-4526-a8c5-f5264b1ca65a" controls></video> -->

End-to-end DevSecOps pipeline implementation using Jenkins, SonarQube, Trivy, Docker, GitOps, ArgoCD, kOps, and HPA.

---

## Demo Video
[Demo Video](https://github.com/user-attachments/assets/1ae5d562-53e4-4051-946c-e1b98a370791)

## 🚀 Project Overview

A production-grade **DevSecOps CI/CD Pipeline** for a React-based Tetris application, featuring automated security gates, GitOps-driven deployments, and high-availability infrastructure on AWS.

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

## Local Development

To run the application locally:

```bash
# Install dependencies
npm install

# Start development server
npm run dev
```

The application will be available at `http://localhost:5173`.
