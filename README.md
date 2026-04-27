# CI/CD Pipeline - Complete Exercise Solution ✅

A complete **Continuous Integration and Continuous Delivery (CI/CD)** pipeline implementation using Jenkins, with automated code analysis, security scanning, and quality gates.

## ✅ Exercise Requirements Completed

| Requirement | Status | Details |
|-------------|--------|---------|
| **Static Analysis (20 min)** | ✅ Complete | SonarQube integration - Local server configured for code analysis |
| **Security Scanning (20 min)** | ✅ Complete | Trivy auto-downloaded and installed - Vulnerability scanning on Docker image |
| **Quality Gates (20 min)** | ✅ Complete | Automated gatekeeping - Pipeline fails if Security Hotspot detected or CRITICAL vulnerabilities found |
| **Infrastructure & Cleanup (30 min)** | ✅ Complete | Post stage configured - Full documentation in this README |

---

## 🔄 Pipeline Flow

```
┌─────────────────────────────────────────────────────────┐
│           JENKINS CI/CD PIPELINE FLOW                    │
└─────────────────────────────────────────────────────────┘

1. Docker Build & Push
   ├─ Download Maven 3.9.0 (if needed)
   ├─ Compile: mvn clean package -DskipTests
   ├─ Build Docker image
   └─ Tag: cicd-demo:BUILD_NUMBER
           │
           ↓
2. Static Code Analysis (SonarQube)
   ├─ Run mvn sonar:sonar
   ├─ Analyze: bugs, code smells, vulnerabilities
   └─ Report to http://localhost:9000
           │
           ↓
3. Quality Gate (SonarQube)
   ├─ Wait for SonarQube analysis
   ├─ FAIL if Security Hotspot detected ⛔
   └─ MAX timeout: 2 minutes
           │
           ↓
4. Docker Scan (Trivy)
   ├─ Download Trivy 0.70.0 (if needed)
   ├─ Scan image for vulnerabilities
   └─ Report: HIGH, CRITICAL issues
           │
           ↓
5. Security Gate (Trivy)
   ├─ Check for CRITICAL vulnerabilities
   ├─ FAIL if any CRITICAL found ⛔
   └─ Blocks deployment of insecure image
           │
           ↓
6. Build Info & Cleanup
   ├─ Show build number, URL, image tag
   ├─ Clean Docker dangling images
   └─ SUCCESS: All gates passed ✅
```

---

## 🚀 Quick Start

### 1. SonarQube Setup

**Start SonarQube locally:**
```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:latest
```

**Access:** http://localhost:9000
- Username: `admin`
- Password: `admin`

**Generate Token:**
1. My Account > Security > Generate Tokens
2. Name: `sonarqube-token`
3. Copy the token

### 2. Jenkins - Add SonarQube Credentials

**Via Jenkins UI:**
1. Manage Jenkins > Credentials > System > Global credentials
2. Add Credentials > Secret text
3. Paste your SonarQube token
4. ID: `sonarqube-token`
5. Save

**Via Jenkins Script Console:**
```groovy
import com.cloudbees.plugins.credentials.impl.*
import com.cloudbees.plugins.credentials.domains.*
import jenkins.model.Jenkins

store = Jenkins.instance.getExtensionList('com.cloudbees.plugins.credentials.SystemCredentialsProvider')[0].getStore()
domain = Domain.global()

cred = new StringCredentialsImpl(
  CredentialsScope.GLOBAL,
  "sonarqube-token",
  "SonarQube Token",
  Secret.fromString("YOUR_TOKEN_HERE")
)

store.addCredentials(domain, cred)
```

### 3. Create Jenkins Pipeline Job

1. New Item > Pipeline
2. Name: `mi-pipeline-demo`
3. Definition: Pipeline script from SCM
4. SCM: Git
5. Repository URL: `https://github.com/mcordoba12/cicd-demo`
6. Branch: `*/master`
7. Script Path: `Jenkinsfile`
8. Save and Build

---

## 📋 Project Components

- ✅ Spring Boot Java application
- ✅ Jenkinsfile for pipeline automation
- ✅ Optimized Dockerfile for Java applications
- ✅ Makefile and docker-compose simplifying pipeline steps
- ✅ Kubernetes deployment files
- ✅ SonarQube integration (code analysis)
- ✅ Trivy integration (vulnerability scanning)

---

## 🚀 Prerequisites

### Running Jenkins and SonarQube with Docker

#### Option 1: Using docker-compose (Recommended)

If you have a `docker-compose.yml` with Jenkins and SonarQube services:

```bash
# Download and start containers
docker-compose up -d

# Verify they are running
docker ps

# Access the services
# Jenkins:    http://localhost:8080
# SonarQube:  http://localhost:9000
```

#### Option 2: Running containers individually

**Jenkins:**
```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

**SonarQube:**
```bash
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  sonarqube:lts
```

#### Verify accessibility

```bash
# Jenkins (wait ~2 minutes)
curl http://localhost:8080

# SonarQube (wait ~1 minute)
curl http://localhost:9000
```

---

## 🔧 Pipeline Stages Detailed

### Stage 1: Docker Build & Push
**Purpose:** Compile Java code and build Docker image

```bash
# Downloads Maven 3.9.0 (if not exists)
# Compiles: mvn clean package -DskipTests
# Builds: docker build -t cicd-demo:latest .
# Tags: cicd-demo:${BUILD_NUMBER}
```

**Fails if:**
- Maven compilation fails
- Docker build fails
- Dockerfile is invalid

**Tools Used:**
- Maven 3.9.0
- Docker

---

### Stage 2: Static Code Analysis (SonarQube)
**Purpose:** Analyze code quality and identify issues

```bash
mvn sonar:sonar \
    -Dsonar.projectKey=cicd-demo \
    -Dsonar.sources=src \
    -Dsonar.host.url=http://localhost:9000 \
    -Dsonar.login=${SONARQUBE_TOKEN}
```

**Analyzes:**
- 🐛 Bugs
- 💨 Code Smells
- 🔒 Vulnerabilities
- 📊 Code Coverage
- 🎯 Maintainability

**Requirements:**
- SonarQube running at http://localhost:9000
- Valid `sonarqube-token` credential
- Network connectivity

**View Results:**
- Dashboard: http://localhost:9000/dashboard?id=cicd-demo

---

### Stage 3: Quality Gate (SonarQube)
**Purpose:** Enforce code quality standards ⛔

```bash
# Polls SonarQube Quality Gate status
# Timeout: 2 minutes (24 attempts, 5sec each)
```

**FAILS if:**
- ❌ Security Hotspot detected
- ❌ Coverage threshold not met
- ❌ Quality Gate status = ERROR

**PASSES if:**
- ✅ All quality criteria met
- ✅ No Security Hotspots
- ✅ Quality Gate status = OK

---

### Stage 4: Docker Scan (Trivy)
**Purpose:** Scan Docker image for vulnerabilities

```bash
# Downloads Trivy 0.70.0 (if not exists)
# Runs: trivy image --severity HIGH,CRITICAL cicd-demo:latest
```

**Reports:**
- Vulnerable dependencies
- OS package vulnerabilities
- CVE details and fix recommendations

**Output Example:**
```
Target Image: cicd-demo:latest

Vulnerabilities
┌────────────────────────────────────────┐
│ Severity   │ Count                      │
├────────────┼────────────────────────────┤
│ CRITICAL   │ 2      (⚠️ BLOCKS DEPLOY) │
│ HIGH       │ 10     (Review soon)       │
│ MEDIUM     │ 25     (Track & monitor)   │
│ LOW        │ 50     (Informational)     │
└────────────────────────────────────────┘
```

---

### Stage 5: Security Gate (Trivy)
**Purpose:** Block deployment if CRITICAL vulnerabilities exist ⛔

```bash
# Strict check: exit-code 1 if CRITICAL found
trivy image --exit-code 1 --severity CRITICAL cicd-demo:latest
```

**FAILS if:**
- ❌ Any CRITICAL vulnerability detected
- ❌ Trivy returns exit code 1

**PASSES if:**
- ✅ No CRITICAL vulnerabilities
- ✅ Exit code = 0

**Impact:**
- 🛑 **Stops entire pipeline** - No deployment of insecure image
- Security-first approach

---

## 📊 Pipeline Stages

Each stage has a specific purpose and conditions that may cause it to fail:

### 1️⃣ **Docker Build & Push**
- **What it does**: Authenticates to Docker registry, builds the Docker image, and pushes it to the repository
- **Command**: `make dockerLogin build dockerBuild dockerPush`
- **Fails if**:
  - Registry credentials are invalid
  - Image build fails
  - No connectivity to Docker repository

### 2️⃣ **Docker Scan** (Trivy)
- **What it does**: Scans the Docker image for known vulnerabilities
- **Command**: `make dockerScan`
- **Fails if**:
  - Trivy is not installed
  - No connectivity to vulnerability database
- **Note**: This stage does NOT automatically stop the pipeline. Failure is determined by "Security Gate"

### 3️⃣ **Security Gate** ⛔ (Critical)
- **What it does**: Validates that no CRITICAL vulnerabilities exist in the Docker image
- **Command**: `trivy image --exit-code 1 --severity CRITICAL cicd-demo:latest`
- **Fails if**:
  - CRITICAL severity vulnerabilities are found
  - Trivy returns exit code 1 (failure)
- **Impact**: **Stops the entire pipeline** if critical vulnerabilities exist (no tests or deployment continues)

### 4️⃣ **Static Code Analysis (SonarQube)**
- **What it does**: Analyzes source code for quality issues, bugs, and vulnerabilities
- **Branch**: Only executes on `master` and `release`
- **Command**: `mvn sonar:sonar -Dsonar.projectKey=cicd-demo`
- **Fails if**:
  - No connectivity to SonarQube
  - Analysis finds critical errors
- **Note**: Runs in parallel with "Integration Tests"

### 5️⃣ **Integration Tests**
- **What it does**: Executes integration tests with databases and external services
- **Command**: `make integrationTest`
- **Fails if**:
  - Tests fail
  - No connectivity to dependent services
- **Note**: Runs in parallel with "Static Code Analysis"

### 6️⃣ **Quality Gate** (SonarQube) ⛔ (Critical)
- **What it does**: Validates that code meets quality thresholds defined in SonarQube
- **Branch**: Only on `master` and `release`
- **Timeout**: 2 minutes maximum
- **Fails if**:
  - Code quality doesn't meet criteria (coverage, bugs, security hotspots)
  - Quality gate is configured as "FAILED" in SonarQube
- **Impact**: **Stops deployment** if quality requirements are not met

### 7️⃣ **Push Latest Tag**
- **What it does**: Tags the image as `latest` in the repository
- **Branch**: Only on `master`
- **Command**: `make dockerPushLatest`
- **Fails if**: No permissions in Docker repository

### 8️⃣ **Deploy to Dev**
- **What it does**: Deploys the application to the development Kubernetes cluster
- **Always executes** (even if not master)
- **Variables**: Requires `KUBE_API_SERVER` and `KUBE_DEV_TOKEN`
- **Fails if**: No cluster connectivity or invalid tokens

### 9️⃣ **Deploy to QA**
- **What it does**: Deploys the application to the QA Kubernetes cluster
- **Branch**: Only on `master` and `release-*`
- **Variables**: Requires `KUBE_API_SERVER` and `KUBE_QA_TOKEN`
- **Fails if**: No connectivity to QA cluster

---

## 🔍 Viewing SonarQube Results

### 1. Access the web interface

Open your browser and go to:
```
http://localhost:9000
```

### 2. Default credentials (first time)
- **Username**: `admin`
- **Password**: `admin`

### 3. View the CICD-Demo project

Once authenticated:
1. Click **"Projects"** in the top navigation bar
2. Search for the **`cicd-demo`** project
3. You'll see a dashboard with:
   - **Coverage**: Percentage of code covered by tests
   - **Bugs**: Errors found
   - **Vulnerabilities**: Security vulnerabilities
   - **Code Smells**: Code quality issues
   - **Security Hotspots**: Critical security points

### 4. Detailed breakdown by category

- Click each category to see detailed issues
- Filter by severity: CRITICAL, MAJOR, MINOR, INFO
- Review the "Quality Gate" status in the overview section

---

## 🚨 Troubleshooting Guide

### Issue: Quality Gate Fails - "Security Hotspot Detected"

**Error Message:**
```
❌ Quality Gate FALHOU - Security Hotspot detectado
```

**Root Cause:**
SonarQube found security issues in your code.

**Solution:**
1. Visit: http://localhost:9000/dashboard?id=cicd-demo
2. Click "Security Hotspots" section
3. Review each issue and fix the code
4. Commit changes: `git push origin master`
5. Trigger pipeline again

**Common Fixes:**
- Fix hardcoded credentials
- Remove SQL injection vulnerabilities
- Fix XSS vulnerabilities
- Remove insecure serialization

---

### Issue: Security Gate Fails - "CRITICAL Vulnerabilities Found"

**Error Message:**
```
❌ FALLO: Se encontraron vulnerabilidades CRITICAL
```

**Root Cause:**
Trivy detected CRITICAL severity vulnerabilities in dependencies.

**Solution:**

**Option 1: Update Dependencies**
```bash
# Check vulnerable library
trivy image cicd-demo:latest | grep -i critical

# Update in pom.xml
mvn versions:use-latest-versions
mvn dependency:update-dependencies

git add pom.xml
git commit -m "chore: update vulnerable dependencies"
git push origin master
```

**Option 2: Use Secure Base Image**
```dockerfile
# Old (less secure)
FROM openjdk:11-jre-slim-buster

# New (more secure)
FROM eclipse-temurin:17-jre-alpine
```

**Option 3: Add Trivy Exceptions** (Temporary, not recommended)
```bash
# Create .trivyignore
echo "CVE-2021-XXXX" >> .trivyignore
```

---

### Issue: Maven Download Fails

**Error Message:**
```
mvn: not found
```

**Solutions:**
1. Check internet connectivity
2. Verify Apache Archive URL is accessible:
   ```bash
   curl -I https://archive.apache.org/dist/maven/maven-3/3.9.0/binaries/
   ```
3. Try manual download:
   ```bash
   cd /tmp && curl -sL https://archive.apache.org/dist/maven/maven-3/3.9.0/binaries/apache-maven-3.9.0-bin.tar.gz | tar xz
   ```

---

### Issue: SonarQube Not Responding

**Error Message:**
```
Connection refused or timeout
```

**Solutions:**
```bash
# Check if SonarQube is running
curl http://localhost:9000/api/system/status

# Start SonarQube
docker run -d --name sonarqube -p 9000:9000 sonarqube:latest

# Restart if already running
docker restart sonarqube

# Check logs
docker logs sonarqube
```

---

### Issue: Trivy Download Fails

**Error Message:**
```
error downloading Trivy archive
```

**Solutions:**
```bash
# Verify GitHub is accessible
curl -I https://github.com/aquasecurity/trivy/releases/download/

# Manual download and setup
cd /tmp
mkdir -p trivy
cd trivy
curl -sL https://github.com/aquasecurity/trivy/releases/download/v0.70.0/trivy_0.70.0_Linux-64bit.tar.gz -o trivy.tar.gz
tar xzf trivy.tar.gz
export PATH="/tmp/trivy:$PATH"
trivy version
```

---

## 📊 Interpreting Results

### Report Structure

Trivy generates a detailed vulnerability report with this format:

```
Target Image: cicd-demo:latest

Vulnerabilities
───────────────────────────────────────
Severity   Count   Description
───────────────────────────────────────
CRITICAL      2    ⚠️ Requires immediate action
HIGH         10    ⚠️ Review and patch soon
MEDIUM       25    ⚠️ Monitor and plan fixes
LOW          50    ℹ️ Informational, low risk
───────────────────────────────────────
Total Vulnerabilities: 87
```

### How to Interpret

| Severity | Required Action |
|----------|-----------------|
| **CRITICAL** | ❌ Pipeline FAILS. Blocks deployment until patched |
| **HIGH** | ⚠️ Review urgently, plan fix for next release |
| **MEDIUM** | ℹ️ Track in technical backlog |
| **LOW** | 📋 Document but not blocking |

### Fixing Vulnerabilities

1. **Identify the cause**
   ```bash
   # Example: vulnerability in openssl library
   trivy image --severity CRITICAL cicd-demo:latest | grep openssl
   ```

2. **Update dependencies**
   ```bash
   # Maven
   mvn dependency:tree | grep [vulnerable-lib]
   mvn versions:use-latest-versions
   ```

3. **Use newer base image**
   ```dockerfile
   # Change in Dockerfile
   FROM openjdk:11-jre-slim-buster  # Old version
   FROM openjdk:11-jre-slim         # Newer version
   ```

4. **Re-run the pipeline**
   ```bash
   # After committing changes
   git push origin feature-branch
   # Jenkins will automatically execute the pipeline
   ```

---

## 📦 Project Deliverables

This project must deliver the following artifacts:

### 1. **Jenkins Job Export**
- XML configuration of the job running the Jenkinsfile
- Location: Jenkins → [Job Name] → Configure → Download configuration
- Expected file: `jenkins-job-config.xml`

### 2. **Configuration Screenshots**
Capture screenshots of:
- ✅ Jenkins - Job overview
- ✅ Jenkins - Pipeline configuration
- ✅ Jenkins - Configured credentials (without sensitive values)
- ✅ SonarQube - Server configured in Jenkins
- ✅ SonarQube - `cicd-demo` project visible in interface

### 3. **Pipeline Execution Results**
Capture from at least one successful execution:
- ✅ Jenkins - Complete build log
- ✅ Jenkins - Stage timeline summary
- ✅ SonarQube - Static analysis results
- ✅ SonarQube - Quality Gate (PASS/FAIL)
- ✅ Trivy - Security report (Security Gate)
- ✅ Jenkins - Archived artifacts (JARs and test results)

### 4. **Modified Source Code**
- ✅ Jenkinsfile (configured and working)
- ✅ Dockerfile (with optimized stages)
- ✅ pom.xml (with SonarQube properties)
- ✅ docker-compose.yml (auxiliary services)
- ✅ Makefile (targets for each stage)
- ✅ README.md (complete documentation)

---

## 🧪 Running Tests Locally

Tests are separated by category using [JUnit Categories][].

[JUnit Categories]: https://maven.apache.org/surefire/maven-surefire-plugin/examples/junit.html

### Unit Tests
```bash
mvn test -Dgroups=UnitTest
```

Or using Docker:
```bash
make build
```

### Integration Tests
```bash
mvn integration-test -Dgroups=IntegrationTests
```

Or using Docker:
```bash
make integrationTest
```

### System Tests (Selenium)
System tests run with Selenium using docker-compose to run a [Selenium standalone container][] with Chrome.

[Selenium standalone container]: https://github.com/SeleniumHQ/docker-selenium

```bash
# Ensure $APP_URL points to a valid application instance
APP_URL=http://localhost:8080 make systemTest
```

---

## 📚 Additional Resources

- [Jenkinsfile Documentation](https://www.jenkins.io/doc/book/pipeline/)
- [SonarQube Quality Gates](https://docs.sonarqube.org/latest/user-guide/quality-gates/)
- [Trivy Vulnerability Scanner](https://github.com/aquasecurity/trivy)
- [Spring Boot Maven Plugin](https://docs.spring.io/spring-boot/docs/current/maven-plugin/reference/)