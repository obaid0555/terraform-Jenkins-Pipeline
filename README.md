# Amazon-FE

# Jenkins Pipeline Overview

This pipeline automates the CI/CD process for a Node.js application. It integrates code quality checks, security scanning, and Docker-based deployment. Below is a breakdown of each stage and its purpose.

## 1. Clean Workspace

```groovy
stage('Clean Workspace') { ... }

Purpose: Cleans the Jenkins workspace to ensure a fresh environment.
Step: cleanWs()
2. Checkout Code
groovy
Copy code
stage('Checkout Code') { ... }
Purpose: Checks out the application code from the Git repository.
Step:
groovy
Copy code
git branch: 'main', url: 'https://github.com/Aj7Ay/Amazon-FE.git'
3. SonarQube Analysis
groovy
Copy code
stage('SonarQube Analysis') { ... }
Purpose: Performs static code analysis using SonarQube to identify code issues.
Steps:
groovy
Copy code
withSonarQubeEnv('sonar-server') {
    sh '''
    $SCANNER_HOME/bin/sonar-scanner \
    -Dsonar.projectKey=amazon \
    -Dsonar.sources=. \
    -Dsonar.login=$SONAR_TOKEN
    '''
}
4. Quality Gate
groovy
Copy code
stage('Quality Gate') { ... }
Purpose: Checks if the project passes the quality gate conditions set in SonarQube.
Step:
groovy
Copy code
waitForQualityGate abortPipeline: true
5. NPM Install
groovy
Copy code
stage('NPM Install') { ... }
Purpose: Installs Node.js dependencies required for the application.
Step:
bash
Copy code
npm install
6. OWASP Full Scan (FS) Scan
groovy
Copy code
stage('OWASP FS Scan') { ... }
Purpose: Performs a security vulnerability scan on the project's dependencies using OWASP Dependency-Check.
Steps:
groovy
Copy code
dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP'
dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
7. Trivy File System Scan
groovy
Copy code
stage('TRIVY FS') { ... }
Purpose: Scans the file system for vulnerabilities using Trivy.
Step:
bash
Copy code
trivy fs . > TRIVYFS.txt
8. Docker Build & Push (DB & DP)
groovy
Copy code
stage('DB & DP') { ... }
Purpose: Builds a Docker image of the application and pushes it to Docker Hub.
Steps:
groovy
Copy code
withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
    sh '''
    docker build -t amazon .
    docker tag amazon obaid0555/amazon:latest
    docker push obaid0555/amazon:latest
    '''
}
9. Trivy Image Scan
groovy
Copy code
stage('TRIVY IMAGE') { ... }
Purpose: Scans the Docker image for vulnerabilities using Trivy.
Step:
bash
Copy code
trivy image obaid0555/amazon:latest > TRIVYFSIMAGE.txt
10. Deploy Amazon App
groovy
Copy code
stage('AMAZON APP') { ... }
Purpose: Runs the Docker container of the application.
Step:
bash
Copy code
docker run -d --name amazon -p 3000:3000 obaid0555/amazon:latest
Notes and Considerations
Security Best Practices:
Credentials Management: Use Jenkins Credentials Plugin to store sensitive information such as SONAR_TOKEN and Docker credentials.
groovy
Copy code
withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) { ... }
Tool Installations:
Ensure JDK 17, Node.js 16, SonarQube Scanner, OWASP Dependency-Check, Trivy, and Docker are installed on the Jenkins agent.
Plugin Requirements:
SonarQube Plugin: Required for SonarQube integration.
OWASP Dependency-Check Plugin: Required for dependency scanning.
Trivy Installation: Ensure Trivy is installed or accessible via Docker.
Docker Registry Access:
Ensure valid Docker Hub credentials are stored in Jenkins under credentialsId: 'docker'.
Port Availability:
Ensure port 3000 is available on the Jenkins agent where the Docker container is run.
Error Handling:
The pipeline uses abortPipeline: true to fail fast if the quality gate fails, allowing for early issue detection.
Workspace Management:
The cleanWs() step ensures a clean workspace for every build.
Logging and Reports:
Trivy scan outputs are stored in TRIVYFS.txt and TRIVYFSIMAGE.txt.
OWASP Dependency-Check reports are published within Jenkins.
Pipeline Flow:
Preparation: Clean workspace and checkout code.
Code Quality: Perform SonarQube analysis and enforce quality gates.
Dependency Management: Install dependencies using NPM.
Security Scanning:
Run OWASP Dependency-Check.
Perform Trivy scans on file system and Docker image.
Build & Deployment:
Build Docker image and push to Docker Hub.
Deploy the application using Docker.
