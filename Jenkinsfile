pipeline {
    agent any
    triggers {
        githubPush() // Triggers the pipeline automatically on GitHub push events
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
                    // List services ignoring @tmp directories
                    def services = sh(
                        script: "ls -d ${servicesDir}/*/ 2>/dev/null | grep -v '@tmp' | xargs -n 1 basename || true",
                        returnStdout: true
                    ).trim().split("\n")

                    if (!services) {
                        echo "⚠️ No services found in ${servicesDir}"
                        services = []
                    }

                    echo "🔍 Detected services: ${services.join(', ')}"

                    // Build a parallel map for each service
                    def parallelSteps = [:]

                    for (serviceName in services) {
                        // Need to wrap in closure to avoid groovy scoping issues
                        def svc = serviceName
                        parallelSteps[svc] = {
                            stage("Deploy ${svc}") {
                                def startTime = sh(script: "date +%s", returnStdout: true).trim()
                                dir("${servicesDir}/${svc}") {
                                    // Build Go binary
                                    sh """
                                        echo "🛠 Building Go binary for ${svc}"
                                        go mod tidy
                                        go build -o app .
                                    """

                                    // Docker build and push
                                    withCredentials([
                                        usernamePassword(credentialsId: 'dockerhub-username', 
                                                         usernameVariable: 'DOCKERHUB_USERNAME', 
                                                         passwordVariable: 'DOCKERHUB_TOKEN')
                                    ]) {
                                        sh """
                                            echo "🐳 Building Docker image for ${svc}"
                                            docker build -t ${DOCKERHUB_USERNAME}/githubactions:${svc} .
                                        """
                                        sh 'sleep 0.4'
                                        sh """
                                            echo "📤 Logging into Docker Hub"
                                            echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin

                                            echo "📤 Pushing Docker image for ${svc}"
                                            docker push ${DOCKERHUB_USERNAME}/githubactions:${svc}
                                        """
                                    }
                                }
                                def endTime = sh(script: "date +%s", returnStdout: true).trim()
                                def duration = endTime.toInteger() - startTime.toInteger()
                                echo "✅ Deployment of ${svc} took ${duration} seconds"
                            }
                        }
                    }

                    // Execute all services in parallel
                    parallel parallelSteps
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
