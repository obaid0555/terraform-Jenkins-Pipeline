# Jenkins Pipeline Configuration

This Jenkins pipeline script automates the build, testing, security analysis, and deployment of a Node.js application with integrated SonarQube, OWASP, and Trivy scanning.

## Pipeline Structure

```groovy
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner' // Points to the configured SonarQube scanner tool
        SONAR_TOKEN = 'sqp_43f47efa8cf2195df972883b66970c6d7b5838b8' // SonarQube token (use Jenkins Credentials for security)
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs() // Clean workspace before starting the build
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/obaid0555/terraform-Jenkins-Pipeline.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('sonar-server') {
                        sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=amazon \
                        -Dsonar.sources=. \
                        -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: true // Abort the pipeline if SonarQube quality gate fails
                }
            }
        }

        stage('NPM Install') {
            steps {
                script {
                    sh 'npm install' // Install Node.js dependencies
                }
            }
        }

        stage('OWASP FS Scan') {
            steps {
                script {
                    // Run OWASP Dependency Check with additional arguments
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP'
                    
                    // Publish the Dependency Check report
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
            }
        }

        stage('TRIVY FS') {
            steps {
                sh 'trivy fs . > TRIVYFS.txt' // Run Trivy file system scan and output to TRIVYFS.txt
            }
        }

        stage('DB & DP') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                        docker build -t amazon . // Build Docker image
                        docker tag amazon obaid0555/amazon:latest // Tag the image
                        docker push obaid0555/amazon:latest // Push the image to Docker Hub
                        '''
                    }
                }
            }
        }

        stage('TRIVY IMAGE') {
            steps {
                sh 'trivy image obaid0555/amazon:latest > TRIVYFSIMAGE.txt' // Run Trivy scan on Docker image
            }
        }

        stage('AMAZON APP') {
            steps {
                sh 'docker run -d --name amazon -p 3000:3000 obaid0555/amazon:latest' // Deploy the application in a Docker container
            }
        }
    }
}

Key Stages Overview
Clean Workspace:

Cleans the Jenkins workspace to ensure no leftover files from previous builds.
Checkout Code:

Retrieves the code from the GitHub repository.
SonarQube Analysis:

Static code analysis using SonarQube to check code quality.
Quality Gate:

Ensures that the project passes the quality standards defined in SonarQube.
NPM Install:

Installs Node.js dependencies for the project.
OWASP Full Scan (FS):

Scans the project for known vulnerabilities in dependencies using OWASP Dependency-Check.
Trivy File System (FS) Scan:

Scans the file system for vulnerabilities using Trivy.
Docker Build & Push (DB & DP):

Builds the Docker image, tags it, and pushes it to Docker Hub.
Trivy Image Scan:

Scans the Docker image for vulnerabilities using Trivy.
Deploy Amazon App:

Runs the Docker container for the application, exposing it on port 3000.