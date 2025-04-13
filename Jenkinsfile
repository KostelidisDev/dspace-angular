pipeline {
    agent {
        label 'docker-agent'
    }

    environment {
        DOCKER_PROJECT = 'repository'
        ARCHS = "linux/amd64,linux/arm64"
        BUILDER_NAME = "multiarch-builder-${BUILD_ID}"
    }

    stages {
        stage('Init') {
            steps {
                script {
                    createDockerBuilder()
                }
            }
        }

        stage('Pipeline') {
            parallel {
                stage('UI') {
                    steps {
                        script {
                            dockerBuild(
                                'repository-ui', 
                                './',
                                './Dockerfile.dist'
                            )
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Build or push failed."
        }
        success {
            echo "✅ All images built and pushed successfully."
        }
        always {
            sh "docker buildx rm ${BUILDER_NAME} || true"
        }
    }
}

def createDockerBuilder() {
    sh """
        docker buildx create --use --name ${env.BUILDER_NAME}
    """
    sh """
        docker buildx inspect --bootstrap
    """
}

def dockerBuild(imageName, contextPath, dockerfileName, extraBuildContext = []) {
    def timestamp = sh(script: "date +%Y.%m.%d%H", returnStdout: true).trim()

    def imageTagLatest = "${env.DOCKER_REGISTRY}/${env.DOCKER_PROJECT}/${imageName}:latest"
    def imageTagTimestamped = "${env.DOCKER_REGISTRY}/${env.DOCKER_PROJECT}/${imageName}:${timestamp}"

    def buildContextArgs = ""
    if (extraBuildContext) {
        extraBuildContext.each { context ->
            buildContextArgs += " --build-context ${context}"
        }
    }

    withCredentials([
        usernamePassword(
            credentialsId: env.DOCKER_CREDENTIALS_ID,
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )
    ]) {
        sh """
            echo \$DOCKER_PASS | docker login ${env.DOCKER_REGISTRY} -u \$DOCKER_USER --password-stdin
        """

        sh """
            docker buildx build \
                --platform=${env.ARCHS} \
                ${buildContextArgs} \
                -f ${dockerfileName} \
                -t ${imageTagLatest} \
                -t ${imageTagTimestamped} \
                --push \
                ${contextPath}
        """
    }
}