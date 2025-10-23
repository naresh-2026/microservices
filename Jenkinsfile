pipeline {
    agent any
    triggers {
        githubPush() // Triggers the pipeline automatically on GitHub push events
    }

    options {
        timestamps()
        ansiColor('xterm') // Requires AnsiColor plugin
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "📦 Starting CI/CD Pipeline"

                    // Record trigger and pipeline start times
                    //def triggerTime = env.GIT_COMMITTER_DATE //?: 
                    def triggerTime = sh(
                        script: "git log -1 --pretty=format:'%cI'",
                        returnStdout: true
                    ).trim()
                    if (!triggerTime) {
                        echo "⚠️ Latest Git commit not found"
                        triggerTime = new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'", TimeZone.getTimeZone('UTC'))
                    } else {
                        echo "ℹ️ Latest Git commit date: ${triggerTime}"
                    }
                    def triggerEpoch = sh(script: "date -d '${triggerTime}' +%s", returnStdout: true).trim()
                    def pipelineStartEpoch = sh(script: "date +%s", returnStdout: true).trim()

                    env.PIPELINE_START = pipelineStartEpoch
                    env.TRIGGER_TO_START_DELAY = (pipelineStartEpoch.toInteger() - triggerEpoch.toInteger()).toString()

                    echo "⏱️ Trigger-to-start delay: ${env.TRIGGER_TO_START_DELAY} seconds"
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Push Microservices') {
            steps {
                script {
                    def servicesDir = "microservices"
                    // Safely list directories
                    def services = sh(
                        script: "ls -d ${servicesDir}/*/ 2>/dev/null | xargs -n 1 basename || true",
                        returnStdout: true
                    ).trim()

                    if (!services) {
                        echo "⚠️ No services found in ${servicesDir}"
                        services = []
                    } else {
                        services = services.split("\n")
                    }

                    echo "🔍 Detected services: ${services.join(', ')}"

                    // Loop through services
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

                            // Secure Docker login and build/push
                            withCredentials([
                                usernamePassword(credentialsId: 'dockerhub-username', 
                                                 usernameVariable: 'DOCKERHUB_USERNAME', 
                                                 passwordVariable: 'DOCKERHUB_TOKEN')
                            ]) {
                                sh """
                                    echo "🐳 Building Docker image for ${serviceName}"
                                    docker build -t ${DOCKERHUB_USERNAME}/githubactions:${serviceName} .

                                """
                                // Delay after building but before pushing for testing
                                sh 'sleep 0.4'

                                sh """
                                    echo "📤 Logging into Docker Hub"
                                    echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin

                                    echo "📤 Pushing Docker image for ${serviceName}"
                                    docker push ${DOCKERHUB_USERNAME}/githubactions:${serviceName}
                                """
                            }
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

                echo "🎯 Pipeline completed!"
                echo "⏱️ Total trigger-to-start delay: ${env.TRIGGER_TO_START_DELAY} seconds"
                echo "🕒 Total pipeline duration: ${totalDuration} seconds"
            }
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
