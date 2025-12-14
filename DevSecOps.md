Perfect! Here is a **GitHub-ready, professional DevSecOps with Jenkins README.md** — covering:

✔ Trivy
✔ SonarQube
✔ Docker Scout
✔ OWASP Dependency-Check
✔ OWASP ZAP
✔ Jenkinsfile pipeline
✔ Commands with explanations
✔ Clean formatting

Copy → paste directly into your GitHub repo.

---

```markdown
# 🔐 DevSecOps with Jenkins – Complete CI/CD + Security Pipeline

This repository contains a complete **DevSecOps Pipeline** using Jenkins.  
It integrates security tools into each stage of CI/CD:

- **SonarQube** → Code Quality & Static Code Analysis  
- **Trivy** → Vulnerability, Misconfiguration & Secret Scanning  
- **Docker Scout** → Container insights & CVE analysis  
- **OWASP Dependency-Check** → Library & dependency CVE scan  
- **OWASP ZAP** → Dynamic Application Security Testing (DAST)

This README explains every setup step and every command clearly.

---

# 📚 Table of Contents

1. [DevSecOps Workflow Overview](#-devsecops-workflow-overview)
2. [Tool Installation](#-tool-installation)
   - Trivy
   - SonarQube
   - Docker Scout
   - Dependency-Check
   - OWASP ZAP
3. [Folder Structure](#-folder-structure)
4. [Jenkins Pipeline (Jenkinsfile)](#-jenkins-pipeline-jenkinsfile)
5. [Pipeline Stage Breakdown](#-pipeline-stage-breakdown)
6. [Commands Explained](#-commands-explained)
7. [Reports & Artifacts](#-reports--artifacts)
8. [How to Run This Pipeline](#-how-to-run-this-pipeline)

---

# 🚀 DevSecOps Workflow Overview

```

```
       ┌─────────────┐
       │   Developer  │
       └──────┬──────┘
              │  Push Code
              ▼
 ┌──────────────────────────┐
 │       Jenkins CI/CD      │
 └──────────────────────────┘
```

┌─────────────┬──────────────┬──────────────┬──────────────┐
│ SonarQube    │ Trivy FS     │ Docker Scout │ DependencyChk │
│ Code Scan    │ File Scan    │ Image Scan   │ CVE Scan      │
└─────────────┴──────────────┴──────────────┴──────────────┘
│
▼
OWASP ZAP DAST
│
▼
Deploy Artifact

````

Security is applied at **every stage** of the pipeline.

---

# 🛠 Tool Installation

## 1️⃣ Install Trivy (FS + Docker Image Scanner)

```bash
sudo apt-get install wget -y
wget https://aquasecurity.github.io/trivy-repo/deb/pool/main/t/trivy/trivy_0.50.1_amd64.deb
sudo dpkg -i trivy_0.50.1_amd64.deb
````

Test:

```bash
trivy --version
```

---

## 2️⃣ Install SonarQube (Docker Recommended)

```bash
docker network create sonar-net

docker run -d --name postgres-sonar \
  -e POSTGRES_USER=sonar \
  -e POSTGRES_PASSWORD=sonar \
  -e POSTGRES_DB=sonar \
  --network sonar-net postgres:13

docker run -d --name sonarqube \
  -p 9000:9000 \
  --network sonar-net \
  sonarqube:lts
```

Access:

```
http://localhost:9000
```

Default login: **admin / admin**

---

## 3️⃣ Docker Scout Installation

Scout comes with Docker Desktop or Docker Engine 24+.

Test:

```bash
docker scout
```

Scan image:

```bash
docker scout quickview myapp:latest
```

---

## 4️⃣ Install OWASP Dependency-Check

```bash
wget https://github.com/jeremylong/DependencyCheck/releases/download/v10.0.3/dependency-check-10.0.3-release.zip
unzip dependency-check-10.0.3-release.zip
```

Run scan:

```bash
./dependency-check/bin/dependency-check.sh --scan . --format HTML
```

---

## 5️⃣ Install OWASP ZAP (DAST)

```bash
docker run -u root -p 8081:8080 -i owasp/zap2docker-stable zap.sh
```

Run baseline scan:

```bash
docker run owasp/zap2docker-stable \
  zap-baseline.py -t http://localhost:8080 -r zap-report.html
```

---

# 📁 Folder Structure

```
myapp/
│── src/
│── Dockerfile
│── pom.xml (or package.json)
│── Jenkinsfile
│── README.md
```

---

# 🧩 Jenkins Pipeline (Jenkinsfile)

Below is a **complete DevSecOps Jenkins pipeline** including all tools.

```groovy
pipeline {
    agent any

    environment {
        SONARQUBE = 'MySonarQube'
        IMAGE = 'myapp:latest'
        DEP_DIR = 'dependency-check-report'
    }

    stages {

        stage('Checkout Code') {
            steps { checkout scm }
        }

        stage('Build') {
            steps { sh 'mvn clean install' }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv(SONARQUBE) {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=myapp \
                        -Dsonar.projectName=myapp \
                        -Dsonar.projectVersion=1.0
                    '''
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                    trivy fs --exit-code 1 --severity HIGH,CRITICAL .
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t myapp:latest .'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:latest
                '''
            }
        }

        stage('Docker Scout Scan') {
            steps {
                sh '''
                    docker scout quickview myapp:latest || true
                    docker scout cves myapp:latest || true
                '''
            }
        }

        stage('Dependency Check') {
            steps {
                sh '''
                    mkdir -p ${DEP_DIR}
                    dependency-check.sh --scan . --format HTML --out ${DEP_DIR}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${DEP_DIR}/dependency-check-report.html", fingerprint: true
                }
            }
        }

        stage('Run App for ZAP') {
            steps {
                sh 'docker run -d -p 8080:8080 --name myapp myapp:latest'
            }
        }

        stage('OWASP ZAP Scan') {
            steps {
                sh '''
                    docker run --rm --network host \
                    -v $PWD:/zap/wrk \
                    owasp/zap2docker-stable zap-baseline.py \
                    -t http://localhost:8080 \
                    -r zap-report.html || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap-report.html', fingerprint: true
                }
            }
        }
    }

    post {
        always {
            sh "docker stop myapp || true"
            sh "docker rm myapp || true"
        }
    }
}
```

---

# 🧵 Pipeline Stage Breakdown

| Stage            | Tool           | Purpose                         |
| ---------------- | -------------- | ------------------------------- |
| Checkout         | Git            | Pull latest code                |
| Build            | Maven/NPM      | Compile & package               |
| SonarQube        | SAST           | Code quality & security         |
| Trivy FS         | SCA            | Scan source & IaC               |
| Docker Build     | Docker         | Build container                 |
| Trivy Image      | Image Scan     | Vulnerabilities in image layers |
| Docker Scout     | Image Insights | CVEs & recommendations          |
| Dependency-Check | SCA            | Vulnerable libraries            |
| Run App          | Docker         | Start target for ZAP            |
| ZAP Scan         | DAST           | Attack running app              |

---

# 🧪 Commands Explained

### Trivy FS Scan

```bash
trivy fs .
```

Scans your **source code** for vulnerabilities, secrets, and misconfigurations.

---

### Trivy Image Scan

```bash
trivy image myapp:latest
```

Scans Docker image layers for CVEs.

---

### Docker Scout Scan

```bash
docker scout quickview myapp:latest
```

Shows quick vulnerability overview.

---

### Dependency-Check

```bash
dependency-check.sh --scan .
```

Finds CVEs in project dependencies (Java, Node.js, Python, etc).

---

### ZAP Baseline Scan

```bash
zap-baseline.py -t http://localhost:8080
```

Attacks your **running application** for OWASP Top 10 issues.

---

# 📊 Reports & Output Location

| Tool             | Output                         |
| ---------------- | ------------------------------ |
| SonarQube        | Web UI at port 9000            |
| Trivy            | Console output + JSON optional |
| Docker Scout     | Console output                 |
| Dependency-Check | `dependency-check-report.html` |
| OWASP ZAP        | `zap-report.html`              |
| Jenkins          | Artifacts stored per build     |

---

# 🚀 How to Run This Pipeline

### 1️⃣ Add Jenkinsfile to repository

Commit the file named **`Jenkinsfile`** at root.

### 2️⃣ Create Jenkins Pipeline Job

* New Item → Pipeline → SCM → Git repository
* Jenkins will auto-detect Jenkinsfile

### 3️⃣ Ensure Tools Are Installed on Jenkins Agent

* Docker
* Trivy
* dependency-check
* sonar-scanner or Maven
* Docker Scout
* ZAP (Docker)

### 4️⃣ Run the Pipeline

Click **Build Now**.

---

# 🎉 DevSecOps Pipeline Ready!

This README is fully GitHub-friendly and includes:

* Complete DevSecOps workflow
* Setup commands
* Jenkinsfile
* Explanations for every tool
* Clean formatting

If you want, I can also generate:

🔥 **A Kubernetes DevSecOps README**
🔥 **A Jenkins + GitHub Webhook README**
🔥 **A Jenkins + Docker + Sonar + Trivy Diagram (SVG for GitHub)**

Just say: **“create Kubernetes DevSecOps README”** or what you need next.
