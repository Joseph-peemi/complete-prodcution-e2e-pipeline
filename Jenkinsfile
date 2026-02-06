pipeline {
    agent {
        label "jenkins-agent"
    }
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }
    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }
        stage("Checkout from SCM") {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/Joseph-peemi/complete-prodcution-e2e-pipeline.git',
                        credentialsId: 'github'
                    ]]
                ])
            }
        }
        stage("Build with Maven") {
            steps {
                sh "mvn clean package -DskipTests"
            }
        }
         stage("Run Tests with Maven") {
            steps {
                sh "mvn test"
            }
        }
        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv(credentialsId: 'jenkins-sonar-token') {
                    sh "mvn sonar:sonar"
                }
            }
        }
    }
}