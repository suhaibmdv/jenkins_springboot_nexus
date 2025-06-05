pipeline {
    agent any

    tools {
        maven 'Maven-3.9.0'
        jdk 'OpenJDK-17'
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "localhost:8090"
        NEXUS_REPOSITORY = "maven-snapshots"
        NEXUS_CREDENTIAL_ID = "nexus-credentials"
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building the application...'
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'mvn test'
            }
            post {
                always {
                    publishTestResults(
                        testResultsPattern: 'target/surefire-reports/*.xml'
                    )
                }
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging the application...'
                sh 'mvn package -DskipTests'
            }
        }

        stage('Deploy to Nexus') {
            steps {
                echo 'Deploying artifacts to Nexus...'
                script {
                    pom = readMavenPom file: "pom.xml"
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}")

                    if(filesByGlob.length == 0) {
                        error "No artifacts found to deploy"
                    }

                    artifactPath = filesByGlob[0].path
                    artifactExists = fileExists artifactPath

                    if(artifactExists) {
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: pom.version.endsWith("SNAPSHOT") ? "maven-snapshots" : "maven-releases",
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                 classifier: '',
                                 file: artifactPath,
                                 type: pom.packaging],
                                [artifactId: pom.artifactId,
                                 classifier: '',
                                 file: "pom.xml",
                                 type: "pom"]
                            ]
                        )
                    }
                }
            }
        }

        stage('Integration Tests') {
            steps {
                echo 'Running integration tests...'
                sh 'mvn verify -DskipUTs'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline executed successfully!'
            emailext(
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Good news! The build ${env.BUILD_URL} completed successfully.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
        failure {
            echo 'Pipeline failed!'
            emailext(
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Build failed. Check console output at ${env.BUILD_URL}",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
