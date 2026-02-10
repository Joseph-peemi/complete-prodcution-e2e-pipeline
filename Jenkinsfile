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
        DOCKER_CREDENTIALS = "dockerhub" // Jenkins credentials id for Docker Hub (username/password)
        JENKINS_API_TOKEN_CRED = "${JENKINS_API_TOKEN}"     // Jenkins token credential id (string) for API authentication  
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
                    sh "mvn sonar:sonar -Dsonar.login=${SONAR_CREDENTIALS}"
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
                    IMAGE_NAME = "${env.APP_NAME}:${env.RELEASE_VERSION}"
                    sh "docker build -t ${IMAGE_NAME} ."
                }
            }
        }
        stage("Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_CREDENTIALS) {
                        def img = docker.image("${env.APP_NAME}:${env.RELEASE_VERSION}")
                        img.push("${env.RELEASE_VERSION}")
                        img.push("latest")
                    }
                }
            }
        }
        stage("Trigger CD Pipeline") {
            steps {
                withCredentials([string(credentialsId: env.JENKINS_API_TOKEN_CRED, variable: 'JENKINS_API_TOKEN')]) {
                    sh """
                        curl -v -k --user admin:${JENKINS_API_TOKEN} \
                          -H 'Cache-Control: no-cache' \
                          -H 'Content-Type: application/x-www-form-urlencoded' \
                          --data 'IMAGE_TAG=${env.APP_NAME}:${env.RELEASE_VERSION}' \
                          'https://myjenkins.ucheada.xyz/job/gitops-complete-pipeline/buildWithParameters'
                    """
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
