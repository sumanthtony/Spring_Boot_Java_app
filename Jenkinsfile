@Library('my-shared-library') _

pipeline {
    agent any
    environment {
    NEXUS_REPO = "maven-snapshots"
    GROUP_ID   = "com.minikube.sample"
    ARTIFACT   = "kubernetes-configmap-reload"
    VERSION    = "0.0.1-SNAPSHOT"
}

    parameters {
        choice(name: 'action', choices: 'create\ndelete', description: 'Choose create/Destroy')
        string(name: 'ImageName', description: "name of the docker build", defaultValue: 'javapp')
        string(name: 'ImageTag', description: "tag of the docker build", defaultValue: 'latest')
        string(name: 'DockerHubUser', description: "name of the Application", defaultValue: 'sumanthtony')
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Git Checkout') {
            when { expression { params.action == 'create' } }
            steps {
                gitCheckout(
                    branch: "main",
                    url: "https://github.com/sumanthtony/Spring_Boot_Java_app.git",
                    gitTool: 'javagit'
                )
            }
        }

        stage('Unit Test maven') {
            when { expression { params.action == 'create' } }
            steps {
                script {
                    mvnTest()
                }
            }
        }

        stage('Integration Test maven') {
            when { expression { params.action == 'create' } }
            steps {
                script {
                    mvnIntegrationTest()
                }
            }
        }
        stage('Upload JAR to Nexus') {
    when { expression { params.action == 'create' } }
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'nexus-creds',
                usernameVariable: 'NEXUS_USER',
                passwordVariable: 'NEXUS_PASS'
            ),
            string(credentialsId: 'nexus-url', variable: 'NEXUS_URL')
        ]) {
            sh """
            JAR_FILE=\$(ls target/*.jar | head -n 1)

            mvn deploy:deploy-file \
              -DgroupId=${GROUP_ID} \
              -DartifactId=${ARTIFACT} \
              -Dversion=${VERSION} \
              -Dpackaging=jar \
              -Dfile=\$JAR_FILE \
              -DrepositoryId=nexus \
              -Durl=\${NEXUS_URL}/repository/${NEXUS_REPO}
            """
        }
    }
}

        stage('Static code analysis: Sonarqube') {
            when { expression { params.action == 'create' } }
            steps {
                withSonarQubeEnv('sonar-api') {
                    sh "mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.1.2184:sonar"
                }
            }
        }

        stage('Quality Gate Status Check : Sonarqube') {
            when { expression { params.action == 'create' } }
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Maven Build : maven') {
            when { expression { params.action == 'create' } }
            steps {
                script {
                    mvnBuild()
                }
            }
        }

        stage('Docker Image Build') {
            when { expression { params.action == 'create' } }
            steps {
                script {
                    sh """
                    cd ${env.WORKSPACE}
                    docker build -t ${params.DockerHubUser}/${params.ImageName}:${params.ImageTag} .
                    """
                }
            }
        }

        stage('Docker Image Scan: trivy ') {
            when { expression { params.action == 'create' } }
            steps {
                script {
                    dockerImageScan("${params.ImageName}", "${params.ImageTag}", "${params.DockerHubUser}")
                }
            }
        }

        stage('Docker Image Push : DockerHub ') {
            when { expression { params.action == 'create' } }
            steps {
                script {
                    dockerImagePush("${params.ImageName}", "${params.ImageTag}", "${params.DockerHubUser}")
                }
            }
        }

        stage('Docker Image Cleanup : DockerHub ') {
            when { expression { params.action == 'create' } }
            steps {
                script {
                    dockerImageCleanup("${params.ImageName}", "${params.ImageTag}", "${params.DockerHubUser}")
                }
            }
        }
    }
}
