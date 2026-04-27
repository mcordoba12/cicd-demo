# CICD-DEMO

A skeleton project to apply **Continuous Integration (CI)** and **Continuous Delivery (CD)** with a complete pipeline that includes security analysis, vulnerability scanning, and automated deployments.

## 🏗️ Complete Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          AUTOMATED CI/CD FLOW                                 │
└──────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐         ┌──────────────┐        ┌──────────────┐
  │  Checkout   │    ─→   │ Build & Test │   ─→   │ Docker Build │
  │  (SCM)      │         │  (Maven)     │        │  & Push      │
  └─────────────┘         └──────────────┘        └──────────────┘
                                                          │
                                                          ↓
  ┌──────────────┐        ┌─────────────────┐    ┌──────────────┐
  │ Deploy QA    │   ←─   │ Quality Gate ✓  │←─  │ SonarQube    │
  │ (Kubernetes) │        │ (Master/Release)│    │ Analysis     │
  └──────────────┘        └─────────────────┘    └──────────────┘
        ↑                                                 ↑
        │                                                 │
  ┌──────────────┐        ┌─────────────────┐
  │ Deploy Dev   │   ←─   │ Security Gate ✓ │←─  Trivy Scan
  │ (Kubernetes) │        │ (CRITICAL-only) │    (Vulnerabilities)
  └──────────────┘        └─────────────────┘
                                │
                                ↓
                        ┌──────────────┐
                        │ Docker Scan  │
                        │  (Trivy)     │
                        └──────────────┘
```

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

## 🛡️ Interpreting Trivy Security Report

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