# Flatris CI/CD Implementation Guide

This document outlines the step-by-step process used to build a production-grade CI/CD pipeline and Kubernetes infrastructure for the Flatris project.

---

## Step 1: Jenkins Master and Slave (Agent) Setup

To maintain a secure and scalable environment, we separated the Jenkins Controller (Master) from the Build Executor (Slave/Agent). This allows the master to handle only management tasks while the agent handles all resource-heavy operations like Docker builds and security scanning.

### 1.1 Jenkins Master (Controller)
The master is the brain of the operation. It was provisioned as an EC2 instance in **ap-south-1**.

- **OS**: Ubuntu 22.04 LTS (Recommended)
- **Base Tools**:
  - `OpenJDK 17`: Required for the Jenkins engine.
  - `Jenkins`: Installed via the official Debian repository.
- **Initial Setup**:
  - Unlocked via the initial admin password.
  - Installed "Suggested Plugins" (Git, Pipeline, GitHub, SSH Agent).
  - Configured **System Credentials** for GitHub (`github-creds`) and SonarQube (`sonar-token`).

### 1.2 Jenkins Slave (Agent) Configuration
The agent is where the actual work happens. It was labeled as **`agent`** in Jenkins.

#### **A. Infrastructure Setup**
- **Instance Type**: `t3.medium` (Minimum recommended for Docker builds and security scans).
- **Communication Protocol**: SSH (Static IP or Private DNS).
- **Security Group**: Open port `22` for the Master to connect via SSH.

#### **B. Essential Software Installation**
The following tools were installed on the agent to support our pipeline:
1.  **Docker**: For building and pushing the multi-stage images.
2.  **AWS CLI v2**: For interacting with ACM, EKS, and other AWS services.
3.  **eksctl & kubectl**: To manage and deploy to the EKS cluster.
4.  **Trivy**: To perform container vulnerability scanning.
5.  **Java 17**: Required for the Jenkins agent process itself.
6.  **Node.js & Yarn**: For project dependency installation and testing.
7.  **SonarScanner**: For static code analysis.

### 1.3 Connecting the Agent
1.  In Jenkins Master, navigate to **Manage Jenkins > Nodes > New Node**.
2.  **Name**: `node-1`, **Labels**: `agent`.
3.  **Launch Method**: `Launch agent via SSH`.
4.  **Host**: IP/DNS of the agent instance.
5.  **Credentials**: SSH private key for the agent's user (e.g., `ubuntu` or `jenkins`).

### 1.4 Global Tools and Credentials
The following IDs were established for use in the `Jenkinsfile`:
- **`github-creds`**: GitHub Personal Access Token for SCM.
- **`sonar-token`**: Token for the SonarQube server.
- **`nvd-api-key`**: For the OWASP Dependency-Check plugin.
- **`dockerhub-creds`**: Docker Hub credentials for image pushing.
- **`aws-creds`**: AWS Access Key and Secret Access Key for EKS deployment.

---

## Step 2: SonarQube Setup and Jenkins Integration

Static Application Security Testing (SAST) is handled by **SonarQube**, which analyzes the code for bugs, vulnerabilities, and code smells.

### 2.1 SonarQube EC2 Instance
A dedicated instance was created to host the SonarQube server.

- **Instance Type**: `t3.medium` (SonarQube requires at least 4GB of RAM).
- **Security Group**: Open port **`9000`** (Default SonarQube Port) to the Jenkins Agent and Master IPs.

### 2.2 Installation and Setup
SonarQube was installed and configured on the EC2 instance to allow for static code analysis.

1.  **Access**: The SonarQube dashboard is accessible at `http://<SonarQube-EC2-IP>:9000`.
2.  **Initial Login**: Authenticated using default credentials (`admin` / `admin`) and prompted for a password change.

### 2.3 Jenkins Integration
To link Jenkins with SonarQube, we performed the following:

#### **A. Generate SonarQube Token**
1.  In SonarQube, go to **My Account > Security**.
2.  Generate a **Global Analysis Token** and save it.

#### **B. Configure Jenkins Master**
1.  **Install Plugin**: Installed `SonarQube Scanner` in Jenkins.
2.  **Manage Jenkins > System**:
    - Added a **SonarQube Server**.
    - **Name**: `SonarQube`.
    - **Server URL**: `http://<SonarQube-EC2-IP>:9000`.
    - **Server Authentication Token**: Added the token from Step A as a `Secret Text` credential (`sonar-token`).

#### **C. Set Up Webhook (Optional but Recommended)**
1.  In SonarQube, go to **Administration > Configuration > Webhooks**.
2.  Add a Webhook pointing to Jenkins: `http://<Jenkins-Master-IP>:8080/sonarqube-webhook/`.
    - This allows Jenkins to wait for the Quality Gate result before proceeding.

---

## Step 3: OWASP Dependency-Check Security Scanning

To identify known vulnerabilities in project dependencies (SCA), we integrated the **OWASP Dependency-Check** tool into Jenkins.

### 3.1 Plugin Installation
1.  Navigated to **Manage Jenkins > Plugins > Available Plugins**.
2.  Searched for and installed the **OWASP Dependency-Check** plugin.

### 3.2 Global Tool Configuration
The tool itself must be configured so Jenkins can invoke it.

1.  Went to **Manage Jenkins > Appearance > Global Tool Configuration**.
2.  Scrolled to **Dependency-Check**.
3.  Added a new installation:
    - **Name**: `DP-Check`
    - **Install automatically**: Selected `Install from github.com`.

### 3.3 NVD API Key Integration (Required)
Starting in 2023, the National Vulnerability Database (NVD) requires an API key for high-frequency updates.

1.  Requested an API key from the [NVD website](https://nvd.nist.gov/developers/request-an-api-key).
2.  Added the key value to Jenkins as a **Secret Text** credential with the ID: **`nvd-api-key`**.
3.  This key is critical as it prevents rate-limiting during project dependency scans in the pipeline.

---

## Step 4: SonarQube Analysis and Quality Gates

In this step, the pipeline performs a deep scan of the source code and checks it against pre-defined safety standards (Quality Gates).

### 4.1 Static Analysis Stage
The pipeline invokes the `sonar-scanner` to upload the code and test coverage reports to the SonarQube server configured in Step 2.

### 4.2 Quality Gate Configuration (Non-Blocking)
For this project, we have configured the Quality Gate to be **non-blocking**:

```groovy
waitForQualityGate abortPipeline: false
```

- **Reason**: Currently, the project has several vulnerabilities and code smells (as seen in the Dependency-Check results). To ensure we can complete the full CI/CD lifecycle (Docker build, EKS deployment), we allow the pipeline to proceed even if the Quality Gate fails.

> [!WARNING]
> **Production Best Practice**: In a real-world enterprise environment, you should set `abortPipeline: true`. This ensures that any code containing "Critical" or "High" vulnerabilities is automatically blocked from moving toward production.

---

## Step 5: Docker Build, Trivy Scan, and Docker Push

This stage transforms the source code into a portable container image, verifies its security, and publishes it to the registry.

### 5.1 Multi-Stage Docker Build
Using the project's **`Dockerfile`**, we perform a multi-stage build:
- **Build Stage**: Compiles the React/Next.js assets and optimizes the production bundle.
- **Production Stage**: Copies only the necessary artifacts into a slim Node.js environment to reduce the attack surface.

The image is tagged twice:
1.  **`saikiran8050/flatris:${BUILD_NUMBER}`**: For immutable versioning.
2.  **`saikiran8050/flatris:latest`**: For easy deployment of the most recent stable version.

### 5.2 Trivy Container Scan
Before pushing to the registry, the image is scanned by **Trivy** for OS-level vulnerabilities and package-level threats.
- The scan results are archived as **JSON reports** in Jenkins for audit and compliance.

### 5.3 Docker Push
The final images are pushed to **Docker Hub** using the **`dockerhub-creds`**. 
- Tagged version (`${BUILD_NUMBER}`)
- Latest version (`latest`)

---

## Step 6: Deploy to Amazon EKS

The final stage of the pipeline automates the deployment of the application to a production-grade Amazon EKS cluster.

### 6.1 Infrastructure Overview
- **Cluster Name**: `flatris`
- **Region**: `ap-south-1` (Mumbai)
- **Namespace**: `flatris` (Used for resource isolation)

### 6.2 Automated Deployment Process
The pipeline uses the **`aws-creds`** globally configured in Jenkins to securely interact with the cluster.

1.  **Namespace Creation**: The pipeline ensures the `flatris` namespace exists before deployment.
2.  **Dynamic Image Update**: 
    - The deployment manifest (`k8s/02-deployment.yaml`) is dynamically updated with the current `${BUILD_NUMBER}` tag.
    - This ensures that every successful build results in a fresh rollout of the latest container image.
3.  **Applying Manifests**:
    - All manifests from the **`k8s/`** directory are applied to the cluster:
      - Resource Quotas & Limits
      - Service & Ingress (ACM SSL Enabled)
      - Horizontal Pod Autoscaler (HPA)
      - Pod Disruption Budget (PDB)

### 6.3 Rolling Updates
Kubernetes performs a **Rolling Update**, ensuring that new pods are healthy (passing Liveness/Readiness probes) before the old ones are terminated. This results in **Zero Downtime Deploys**.

---

## Step 7: Slack Notifications

To keep the team informed of the build status in real-time, we integrated **Slack Notifications** into the pipeline.

### 7.1 Slack App Setup
1.  Created a Slack App and enabled **Incoming Webhooks**.
2.  Generated a unique Webhook URL for the **`#jenkins-notifications`** channel.

### 7.2 Jenkins Credential Configuration
For security, the Webhook URL is not hardcoded in the `Jenkinsfile`.

1.  Went to **Manage Jenkins > Credentials**.
2.  Added a new **Secret Text** credential:
    - **ID**: `slack-webhook`
    - **Secret**: The full Slack Webhook URL.
    - **Description**: `slack webhook`

### 7.3 Pipeline Integration
The pipeline uses a `post` block to send notifications after every execution. It dynamically changes the notification color based on the build result:
- 🟢 **Green**: Success
- 🟡 **Yellow**: Unstable (Test failures)
- 🔴 **Red**: Failure (Build or Deploy error)

---
