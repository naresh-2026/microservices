pipeline {
    agent any
    triggers {
        githubPush() // Triggers the pipeline automatically on GitHub push events
    }
    environment {
        DOCKERHUB_USERNAME = credentials('dockerhub-username')
        DOCKERHUB_TOKEN = credentials('dockerhub-token')
    }

    options {
        timestamps()
        ansiColor('xterm')
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "📦 Starting CI/CD Pipeline"

                    // Record trigger and pipeline start times
                    def triggerTime = env.GIT_COMMITTER_DATE ?: new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'", TimeZone.getTimeZone('UTC'))
                    def triggerEpoch = sh(script: "date -d '${triggerTime}' +%s", returnStdout: true).trim()
                    def pipelineStartEpoch = sh(script: "date +%s", returnStdout: true).trim()

                    env.PIPELINE_START = pipelineStartEpoch
                    env.TRIGGER_TO_START_DELAY = (pipelineStartEpoch.toInteger() - triggerEpoch.toInteger()).toString()

                    echo "⏱️ Trigger-to-start delay: ${env.TRIGGER_TO_START_DELAY} seconds"
                }
            }
        }

        // stage('Set up Go') {
        //     steps {
        //         sh 'go version || sudo apt-get update && sudo apt-get install -y golang'
        //     }
        // }

        stage('Build and Push Microservices') {
            steps {
                script {
                    // Path to the folder containing all services
                    def servicesDir = "microservices"
                    def services = sh(
                        script: "ls -d ${servicesDir}/*/ | xargs -n 1 basename",
                        returnStdout: true
                    ).trim().split("\n")

                    echo "🔍 Detected services: ${services.join(', ')}"

                    for (serviceName in services) {
                        echo "🚀 Building and deploying service: ${serviceName}"
                        def startTime = sh(script: "date +%s", returnStdout: true).trim()

                        dir("${servicesDir}/${serviceName}") {
                            // Build Go binary
                            sh """
                                echo "🛠 Building Go binary for ${serviceName}"
                                go mod tidy
                                go build -o app .
                            """

                            // Build Docker image
                            sh """
                                echo "🐳 Building Docker image for ${serviceName}"
                                docker build -t ${DOCKERHUB_USERNAME}/githubactions:${serviceName} .
                            """

                            // Push Docker image
                            sh """
                                echo "📤 Pushing Docker image for ${serviceName}"
                                echo "${DOCKERHUB_TOKEN}" | docker login -u "${DOCKERHUB_USERNAME}" --password-stdin
                                docker push ${DOCKERHUB_USERNAME}/githubactions:${serviceName}
                            """
                        }

                        def endTime = sh(script: "date +%s", returnStdout: true).trim()
                        def duration = endTime.toInteger() - startTime.toInteger()
                        echo "✅ Deployment of ${serviceName} took ${duration} seconds"
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                def pipelineEndEpoch = sh(script: "date +%s", returnStdout: true).trim()
                def totalDuration = pipelineEndEpoch.toInteger() - env.PIPELINE_START.toInteger()

                echo "🎯 Pipeline completed successfully!"
                echo "⏱️ Total trigger-to-start delay: ${env.TRIGGER_TO_START_DELAY} seconds"
                echo "🕒 Total pipeline duration: ${totalDuration} seconds"
            }
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
