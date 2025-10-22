pipeline {
    // 1. Define the execution environment (agent) and make it available globally
    agent any

    // 2. Define the build tools (JDK and Maven) globally
    tools {
        jdk 'JDK17' // Tool name from Jenkins Global Tool Configuration
        maven 'M3'  // Tool name from Jenkins Global Tool Configuration
    }

    stages {
        stage('1. Code Checkout (GitHub)') {
            steps {
                // Fetches the code from the repository configured in the job
                checkout scm 
            }
        }

        stage('2. Build & Unit Test') {
            steps {
                echo "Building and running unit tests..."
                // Runs Maven commands using the configured M3 and JDK17
                sh 'mvn clean install -DskipTests=false'
            }
        }

        stage('3. Code Analysis (SonarQube)') {
            steps {
                // Uses the SonarQube server named 'MySonarServer' and its token
                withSonarQubeEnv('MySonarServer') {
                    echo "Running SonarQube analysis..."
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('4. Quality Gate Check') {
            steps {
                echo "Waiting for SonarQube Quality Gate status..."
                // Pipeline waits up to 5 minutes for SonarQube results
                timeout(time: 5, unit: 'MINUTES') {
                    // Aborts the pipeline if quality gate fails
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('5. Artifact Upload (Nexus)') {
            steps {
                // Securely injects Nexus admin credentials for the deployment step
                withCredentials([usernamePassword(credentialsId: 'nexus-admin', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')]) {
                    echo "Deploying artifact to Nexus Repository..."
                    // Maven deploy pushes the built artifact to Nexus (requires configuration in pom.xml)
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }

        stage('6. Continuous Delivery Success') {
            steps {
                echo "SUCCESS: CI/CD Pipeline completed. Artifact available in Nexus."
            }
        }
    }
}
