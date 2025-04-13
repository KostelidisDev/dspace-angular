pipeline {
    agent {
        label 'docker-builder'
    }

    environment {
        DOCKER_PROJECT = 'repository'
        ARCHS = "linux/amd64,linux/arm64"
    }

    stages {
        stage('Pipeline') {
            parallel {
                stage('UI') {
                    steps {
                        script {
                            dockerBuilder(
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
    }
}

def dockerBuilder(imageName, contextPath, dockerfileName, extraBuildContext = []) {
    def timestamp = sh(script: "date +%Y.%m.%d%H", returnStdout: true).trim()

    def imageTagLatest = "${DOCKER_REGISTRY}/${DOCKER_PROJECT}/${imageName}:latest"
    def imageTagTimestamped = "${DOCKER_REGISTRY}/${DOCKER_PROJECT}/${imageName}:${timestamp}"

    def buildContextArgs = ""

    if (extraBuildContext) {
        extraBuildContext.each { context ->
            buildContextArgs += " --build-context ${context}"
        }
    }

    withCredentials([
        usernamePassword(
            credentialsId: DOCKER_CREDENTIALS_ID, 
            usernameVariable: 'DOCKER_USER', 
            passwordVariable: 'DOCKER_PASS'
        )
    ]) {
        sh """
            echo \$DOCKER_PASS | docker login ${DOCKER_REGISTRY} -u \$DOCKER_USER --password-stdin
        """
        sh """
            docker buildx create --use --platform=$ARCHS --name multi-platform-builder-${imageName}
        """
        sh """
            docker buildx inspect --bootstrap
        """
        sh """
            docker \
                buildx \
                build \
                --push \
                --platform=$ARCHS \
                ${buildContextArgs} \
                    -f ${dockerfileName} \
                    -t ${imageTagLatest} \
                    -t ${imageTagTimestamped} \
                ${contextPath}
        """
    }
}