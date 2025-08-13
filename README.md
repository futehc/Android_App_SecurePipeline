# Secure Android App 🚀

A secure Android application built with modern development practices and a DevSecOps-first approach.  
This project integrates a complete CI/CD pipeline using Jenkins with the following stages:

- **Software Composition Analysis (SCA)** — OWASP Dependency-Check
- **Secret Scanning** — Gitleaks
- **Static Application Security Testing (SAST)** — SonarQube
- **Android Lint & Unit Tests**
- **APK Build** — Debug/Release variants
- **Deployment** — Firebase App Distribution
- **Dynamic Application Security Testing (DAST)** — MobSF

---

## 📂 Project Structure

app/ # Main Android source code
gradle/ # Gradle wrapper files
reports/ # Security & test reports generated in CI/CD
Jenkinsfile # Jenkins pipeline definition
---

## 🛠️ CI/CD Pipeline
The Jenkins pipeline automates:
1. **Checkout & Gradle validation**
2. **Security scans**
3. **Quality gates**
4. **Build & deploy**
5. **Security report archiving**

---

## 🚀 Getting Started

### Prerequisites
- **Java 17** (JDK)
- **Android SDK**
- **Gradle** (via `gradlew`)
- **MobSF** (optional for local testing)

### Local Build
```bash
./gradlew clean assembleDebug
