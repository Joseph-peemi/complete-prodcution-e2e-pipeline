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
        JENKINS_API_TOKEN_CRED = "jenkins-api-token"
        DOCKER_USER = "abuchijoe"
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
    }
    stages {
        stage("Cleanup Workspace") {
            steps { cleanWs() }
        }
        stage("Checkout App Repo") {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/Joseph-peemi/complete-production-e2e-pipeline.git',
                        credentialsId: env.GIT_CREDENTIALS
                    ]]
                ])
            }
        }
        stage("Build with Maven") {
            steps { sh "mvn clean package -DskipTests" }
        }
        stage("Run Tests") {
            steps { sh "mvn test" }
        }
        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv(env.SONAR_CREDENTIALS) {
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
                    sh "docker build -t ${IMAGE_NAME}:${RELEASE_VERSION} ."
                }
            }
        }
        stage("Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_CREDENTIALS) {
                        def img = docker.image("${IMAGE_NAME}:${RELEASE_VERSION}")
                        img.push("${RELEASE_VERSION}")
                        img.push("latest")
                    }
                }
            }
        }
        stage("Trigger GitOps Pipeline") {
            steps {
                withCredentials([string(credentialsId: env.JENKINS_API_TOKEN_CRED, variable: 'JENKINS_API_TOKEN')]) {
                    sh """
                        curl -v -k --user admin:${JENKINS_API_TOKEN} \
                          -H 'Cache-Control: no-cache' \
                          -H 'Content-Type: application/x-www-form-urlencoded' \
                          --data 'RELEASE_VERSION=${RELEASE_VERSION}' \
                          'https://myjenkins.ucheada.xyz/job/gitops-complete-pipeline/buildWithParameters'
                    """
                }
            }
        }
    }
    post {
        always { cleanWs() }
    }
}
