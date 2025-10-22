pipeline {
    agent {
        tools {
            maven 'M3' 
            jdk 'JDK17'
        }
    }
    stages {
        stage('1. Checkout Code') {
            steps {
                checkout scm
            }
        }
        stage('2. Build & Test') {
            steps {
                sh 'mvn clean install -DskipTests=false'
            }
        }
        stage('3. Code Analysis') {
            steps {
                withSonarQubeEnv('MySonarServer') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        stage('4. Quality Gate Check') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('5. Artifact Upload (Nexus)') {
            steps {
                // Use Nexus credential ID 'nexus-admin' for secure push
                withCredentials([usernamePassword(credentialsId: 'nexus-admin', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')]) {
                    sh 'mvn deploy -DskipTests=true' 
                }
            }
        }
    }
}
