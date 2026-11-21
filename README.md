<div align="center">

# 🎮 React Tetris V1 — DevSecOps CI/CD & GitOps

**A React Tetris game deployed through a security-focused CI/CD workflow with Jenkins, Docker, Kubernetes and Argo CD.**

[![GitHub Repo](https://img.shields.io/badge/GitHub-Repository-181717?logo=github)](https://github.com/abhisheksaste31-source/devsecops-Tetris-V1)
![React](https://img.shields.io/badge/Frontend-React-61DAFB?logo=react&logoColor=black)
![Jenkins](https://img.shields.io/badge/CI-Jenkins-D24939?logo=jenkins)
![Kubernetes](https://img.shields.io/badge/Deployment-Kubernetes-326CE5?logo=kubernetes)
![Argo CD](https://img.shields.io/badge/GitOps-Argo%20CD-EF7B4D)

</div>

---

## 📌 Project Overview

**React Tetris V1** is a browser-based Tetris game built with React. This project demonstrates how to integrate automated code quality checks, dependency vulnerability scanning, containerization, and GitOps-based Kubernetes deployment into a DevSecOps workflow.

> **Note:** The stages and snippets below describe the intended CI/CD implementation. Configure the tool names, credentials, Docker image, and manifest repository to match your own environment.

## 🖼️ Application Preview

![Tetris Game](./architucture/Tetris.png)

## 🏗️ Architecture

### Architecture Diagram

![DevSecOps Architecture](./architucture/Architecture.png)

### Animated Architecture

![DevSecOps Architecture Animation](./architucture/architecture.gif)

### CI/CD and GitOps Flow

```mermaid
flowchart TD
    A[Developer pushes React code] --> B[GitHub application repository]
    B --> C[Jenkins CI/CD pipeline]
    C --> D[SonarQube code analysis]
    D --> E[OWASP Dependency-Check]
    E --> F[Build Docker image]
    F --> G[Push image to container registry]
    G --> H[Update Kubernetes deployment image]
    H --> I[GitHub manifest repository]
    I --> J[Argo CD sync]
    J --> K[Kubernetes cluster]
    K --> L[React Tetris application]
```

## 🛠️ Technology Stack

| Tool | Purpose |
| --- | --- |
| React | Tetris game frontend |
| Git & GitHub | Version control and source hosting |
| Jenkins | CI/CD automation |
| SonarQube | Static code analysis and quality checks |
| OWASP Dependency-Check | Dependency vulnerability scanning |
| Docker | Application containerization |
| Kubernetes | Container orchestration |
| Argo CD | GitOps continuous delivery |

## 🚀 CI/CD Workflow

1. **Checkout:** Jenkins retrieves the React source code from GitHub.
2. **Code analysis:** SonarQube analyzes code quality and maintainability.
3. **Dependency scanning:** OWASP Dependency-Check inspects dependencies for known vulnerabilities.
4. **Build:** The React application is built and packaged into a Docker image.
5. **Publish:** The image is pushed to the configured container registry.
6. **Manifest update:** Jenkins updates the container image reference in the Kubernetes manifest repository.
7. **GitOps delivery:** Argo CD detects the manifest change and synchronizes it to Kubernetes.

## 🔍 SonarQube Analysis — Jenkins Snippet

Configure **Manage Jenkins → Tools** with a SonarScanner installation named `sonar-scanner`, and configure a SonarQube server named `sonar-server`.

```groovy
environment {
    SCANNER_HOME = tool 'sonar-scanner'
}

stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('sonar-server') {
            sh '''
                "$SCANNER_HOME/bin/sonar-scanner" \
                  -Dsonar.projectName=React-Tetris-V1 \
                  -Dsonar.projectKey=react-tetris-v1
            '''
        }
    }
}
```

## 🛡️ OWASP Dependency-Check — Jenkins Snippet

Install and configure the OWASP Dependency-Check Jenkins plugin and a tool installation named `DP-Check`.

```groovy
stage('OWASP FS Scan') {
    steps {
        dependencyCheck \
            additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', \
            odcInstallation: 'DP-Check'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
    }
}
```

> Disabling Yarn and Node audit avoids those specific audit mechanisms; it does **not** disable Dependency-Check itself. Review scan reports and configure build failure thresholds according to your security requirements.

## 🐳 Docker & Kubernetes Deployment

Build and push your application image using the image name configured for your own container registry:

```bash
docker build -t YOUR_DOCKERHUB_USERNAME/react-tetris-v1:latest .
docker login
docker push YOUR_DOCKERHUB_USERNAME/react-tetris-v1:latest
```

Example Kubernetes deployment image field:

```yaml
spec:
  template:
    spec:
      containers:
        - name: react-tetris
          image: YOUR_DOCKERHUB_USERNAME/react-tetris-v1:latest
```

For reproducible deployments, prefer unique version tags or image digests over a mutable `latest` tag.

## 🔄 GitOps — Update Kubernetes Manifest from Jenkins

This example updates a **separate manifest repository**. Replace `YOUR_GITHUB_USERNAME`, `YOUR_MANIFEST_REPO`, `YOUR_DOCKERHUB_USERNAME`, and the image tag with your actual values. The Jenkins credential `github` must contain a token with permission to push to the manifest repository.

```groovy
environment {
    GIT_REPO_NAME = 'YOUR_MANIFEST_REPO'
    GIT_USER_NAME = 'YOUR_GITHUB_USERNAME'
}

stage('Checkout Manifest Repository') {
    steps {
        dir('manifest-repo') {
            git branch: 'main', url: "https://github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git"
        }
    }
}

stage('Update Deployment Manifest') {
    steps {
        dir('manifest-repo') {
            withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                sh '''
                    set -eu
                    NEW_IMAGE_NAME="YOUR_DOCKERHUB_USERNAME/react-tetris-v1:${BUILD_NUMBER}"
                    sed -i "s|image: .*|image: ${NEW_IMAGE_NAME}|" deployment.yml
                    git config user.name "jenkins-ci"
                    git config user.email "jenkins-ci@example.invalid"
                    git add deployment.yml
                    if ! git diff --cached --quiet; then
                        git commit -m "Update Tetris image to ${NEW_IMAGE_NAME}"
                        git -c credential.helper= \
                            -c core.askPass=true \
                            push "https://x-access-token:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git" HEAD:main
                    fi
                '''
            }
        }
    }
}
```

**Security note:** Avoid embedding tokens in command-line URLs in production, where they may appear in process arguments or logs. Prefer Jenkins Git credentials binding with a credential helper or SSH credentials. Ensure the Docker build pushes the same `${BUILD_NUMBER}` tag before updating the manifest. Adjust the file path if `deployment.yml` is in a subdirectory.

## ☸️ Argo CD Setup

Argo CD watches the Kubernetes manifest repository and reconciles the desired state with the cluster.

1. Install Argo CD on your Kubernetes cluster.
2. Register your Kubernetes manifest repository in Argo CD.
3. Create an Argo CD Application with the correct repository URL, branch, manifest path, and destination namespace.
4. Enable automatic synchronization if desired.
5. Push a new image reference to the manifest repository and verify the deployment in Argo CD.

**Reference:** [Argo CD installation walkthrough — EKS Workshop (archived)](https://archive.eksworkshop.com/intermediate/290_argocd/install/)

## 💻 Run Locally

```bash
git clone https://github.com/abhisheksaste31-source/devsecops-Tetris-V1.git
cd devsecops-Tetris-V1
npm install
npm start
```

> The start command assumes the project defines an `npm start` script. If it uses Vite or another build tool, use the script listed in `package.json` (for example, `npm run dev`).

## 📂 Repository & Assets

- **Application repository:** [devsecops-Tetris-V1](https://github.com/abhisheksaste31-source/devsecops-Tetris-V1)
- **Architecture assets:** [architucture folder](https://github.com/abhisheksaste31-source/devsecops-Tetris-V1/tree/main/architucture)
- **Static architecture:** `architucture/Architecture.png`
- **Animated architecture:** `architucture/architecture.gif`
- **Application screenshot:** `architucture/Tetris.png`

> GitHub paths are case-sensitive. Keep the folder spelling `architucture` and filenames exactly as shown, or update these links if your repository differs.

## 🎯 Key Learning Outcomes

- Building automated CI/CD pipelines with Jenkins
- Integrating code quality and dependency security checks
- Containerizing a React application with Docker
- Managing Kubernetes application manifests
- Automating deployments through Argo CD and GitOps

## 👨‍💻 Author

**Abhishek Saste**  
[GitHub](https://github.com/abhisheksaste31-source) · [LinkedIn](https://www.linkedin.com/in/abhisheksaste)

---

<div align="center">

⭐ **If this project is useful, consider starring the repository!**

</div>
