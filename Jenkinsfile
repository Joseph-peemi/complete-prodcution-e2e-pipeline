pipeline {
    agent { label "jenkins-agent" }
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }
    environment {
        APP_NAME = "complete-production-e2e-pipeline"
        RELEASE_VERSION = "1.0.0"
        GIT_CREDENTIALS = "github"
        SONAR_CREDENTIALS = "jenkins-sonarqube-token"
        DOCKER_CREDENTIALS = "dockerhub"
        // NEW: Credential ID for the remote trigger token (create this in Jenkins)
        CD_TRIGGER_TOKEN_CRED = "cd-pipeline-remote-token"
    }
    stages {
        stage("Cleanup Workspace") {
            steps { cleanWs() }
        }
        stage("Checkout from SCM") {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/Joseph-peemi/complete-prodcution-e2e-pipeline.git',
                        credentialsId: env.GIT_CREDENTIALS
                    ]]
                ])
            }
        }
        stage("Build with Maven") {
            steps { sh "mvn clean package -DskipTests" }
        }
        stage("Run Tests with Maven") {
            steps { sh "mvn test" }
        }
        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh "mvn sonar:sonar"
                }
            }
        }
        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage("Build Docker Image") {
            steps {
                script {
                    IMAGE_NAME = "abuchijoe/complete-production-e2e-pipeline:${env.RELEASE_VERSION}"
                    sh "docker build -t ${IMAGE_NAME} ."
                }
            }
        }
        stage("Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_CREDENTIALS) {
                        def img = docker.image("abuchijoe/complete-production-e2e-pipeline:${env.RELEASE_VERSION}")
                        img.push("${env.RELEASE_VERSION}")
                        img.push("latest")
                    }
                }
            }
        }
        stage("Trigger CD Pipeline") {
            steps {
                withCredentials([string(credentialsId: env.CD_TRIGGER_TOKEN_CRED, variable: 'REMOTE_TOKEN')]) {
                    script {
                        echo "Triggering CD pipeline with IMAGE_TAG = ${APP_NAME}:${RELEASE_VERSION}"
                        try {
                            build job: 'gitops-complete-pipeline',
                                  parameters: [string(name: 'IMAGE_TAG', value: "${APP_NAME}:${RELEASE_VERSION}")],
                                  wait: false,
                                  propagate: false,
                                  token: "${REMOTE_TOKEN}"
                            echo "CD pipeline triggered successfully"
                        } catch (Exception e) {
                            echo "Failed to trigger CD pipeline: ${e.getMessage()}"
                            error "Trigger failed"
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}