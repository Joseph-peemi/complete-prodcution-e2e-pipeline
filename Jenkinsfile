pipeline {
    agent {
        label "jenkins-agent"
    }
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }
    environment {
        APP_NAME = "complete-production-e2e-pipeline"
        RELEASE_VERSION = "1.0.0"
        DOCKER_USER = "abuchijoe"
        DOCKER_PASSWORD = "dockerhub"
        IMAGE_NAME = "$DOCKER_USER/$APP_NAME:$RELEASE_VERSION"
        IMAGE_TAG = "$DOCKER_USER/$APP_NAME:$RELEASE_VERSION"
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
                withSonarQubeEnv(installationName: 'sonarqube-server', credentialsId: 'jenkins-sonarqube-token') {
                    sh "mvn sonar:sonar"
                }
            }
        }
        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }
            }
        }
        stage("Build Docker Image") {
            steps {
                sh "docker build -t $IMAGE_NAME ."
            }
        }
        stage("Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASSWORD) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }
                    docker.withRegistry('', DOCKER_PASSWORD) {
                        docker_image.push("$RELEASE_VERSION")
                        docker_image.push("latest")
                    }
                }
            }
        }
    }
}