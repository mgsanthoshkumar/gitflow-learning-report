DevOps CI/CD Pipeline Implementation & Troubleshooting Report

This report documents the implementation of a full Continuous Integration/Continuous Delivery (CI/CD) pipeline integrating GitHub, Jenkins, SonarQube, and Nexus. It serves as a successful roadmap and a detailed troubleshooting manual based on the entire learning process.

1. Pipeline Overview and Goal

Tool

Purpose in Pipeline

Host Access

Internal Access

GitHub

Source Code Management (SCM) & Webhook Trigger.

https://github.com/...

N/A

Jenkins

CI/CD Orchestration & Job Execution.

http://192.168.2.61:8080

http://jenkins:8080

SonarQube

Code Quality Analysis & Quality Gatekeeping.

http://192.168.2.61:9000

http://sonarqube:9000

Nexus

Artifact Repository (Storage for built .jar files).

http://192.168.2.61:8081

http://nexus:8081

Goal: Implement a pipeline that automatically builds a Java application, verifies its quality, and stores the tested artifact in a repository.

2. Infrastructure Setup (Docker Compose)

All services were deployed locally using Docker Compose, leveraging the host machine's resources while maintaining network isolation.

Status Confirmation

Initial checks confirmed all prerequisites were installed on the host machine (sfty-lab261):

git --version $\checkmark$

docker --version $\checkmark$

docker compose version $\checkmark$

Initial Troubleshooting: Connectivity (Fatal Error #1)

Error

Cause

Fix

localhost refused to connect

The host firewall (FirewallD was inactive) or a Docker networking issue prevented access to the published ports (8080, 8081, 9000).

Confirmed host IP (192.168.2.61), but ultimately resolved by forcefully restarting Docker services: docker compose down then docker compose up -d.

Nexus/Jenkins passwords

Passwords are dynamically generated.

Used docker exec <container-name> cat <path/to/password> to retrieve initial admin passwords for Jenkins and Nexus.

3. Jenkins & Tool Integration

After successful deployment of the Docker stack, the focus shifted to configuring cross-tool authentication and environment paths in Jenkins.

A. Credential Setup (Manage Credentials)

Credential

Type

ID Used

Purpose

Nexus Admin

Username/Password

nexus-admin

Authenticates Jenkins for deploying artifacts (Stage 5).

SonarQube Token

Secret Text

sonarqube-token

Authenticates Jenkins to submit analysis reports (Stage 3).

GitHub PAT

(Implicit/Not explicitly shown)

github-pat

Authenticates Jenkins for checking out private GitHub repositories.

B. Global Configuration (System & Tools)

SonarQube Server Link (System):

Required Fix: The SonarQube Scanner Plugin needed to be manually installed in Jenkins before the configuration section appeared.

Configuration: Added a server named MySonarServer, pointing to the internal Docker service URL: http://sonarqube:9000, using the sonarqube-token.

Global Tool Configuration:

JDK: Configured JDK17 (Install Automatically).

Maven: Configured M3 (Install Automatically).

4. Pipeline Development and Execution (The Real Challenges)

The primary complexity involved tuning the pipeline Groovy script and ensuring the underlying Maven project was correctly structured.

A. Final Jenkinsfile Script (Successful Version)

pipeline {
    agent any
    tools {
        jdk 'JDK17'
        maven 'M3' 
    }
    stages {
        stage('1. Code Checkout (GitHub)') { steps { checkout scm } }
        stage('2. Build & Unit Test') { steps { sh 'mvn clean install -DskipTests=false' } }
        stage('3. Code Analysis (SonarQube)') { 
            steps { withSonarQubeEnv('MySonarServer') { sh 'mvn sonar:sonar' } } 
        }
        stage('4. Quality Gate Check') {
            steps { timeout(time: 15, unit: 'MINUTES') { waitForQualityGate abortPipeline: true } }
        }
        stage('5. Artifact Deployment (Nexus)') {
            steps { withCredentials([usernamePassword(credentialsId: 'nexus-admin', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')]) { sh 'mvn deploy -DskipTests=true' } }
        }
        stage('6. Workflow Complete') { steps { echo "SUCCESS: Pipeline validation complete." } }
    }
}


B. Execution and Troubleshooting (Fatal Errors #2 - #4)

Error

Stage

Cause

Fix

#2 fatal: couldn't find remote ref refs/heads/main

Checkout

The GitHub repository used the master branch, but the Jenkins job defaulted to looking for main.

Fix: Changed the Branch Specifier in the Jenkins job configuration from */main to */master.

#3 ERROR: Unable to find Jenkinsfile from git...

Checkout

The required Jenkinsfile was not in the root directory of the repository.

Fix: Created the Jenkinsfile locally, committed it, and pushed it to GitHub using git push origin master.

#4 Invalid agent type "tools" specified

Pipeline Script

The tools directive was incorrectly nested inside the agent block (Groovy syntax error).

Fix: Rewrote the top of the Jenkinsfile to place tools as a sibling of agent.

#5 The goal you specified requires a project... no POM

Build & Test

The repository lacked the mandatory pom.xml file required by Maven.

Fix: Created a minimal pom.xml file and the src/main/java/App.java file, committed, and pushed.

#6 Timeout has been exceeded

Quality Gate Check

The default 5-minute timeout was too short for the resource-constrained SonarQube container to process the analysis report and notify Jenkins.

Fix: Increased the timeout in the Jenkinsfile from 5 minutes to 15 minutes.

5. Final Validation & Diagram

Upon fixing the timeout, the pipeline achieved SUCCESS, confirming the following flow: 

Code from GitHub is pulled by Jenkins.

Maven builds and installs the artifact.

SonarQube scans the code and confirms the quality gate pass.

The final, verified artifact is deployed and written to the Nexus Repository.

The entire process, driven by the principles of reducing deployment time and improving release stability (via the quality gate), is fully implemented.
