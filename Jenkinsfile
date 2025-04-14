def nativeArch = 'amd64'

def builds = [
    [arch: 'arm64', name: 'repository-ui',  path: './',  dockerfile: './Dockerfile.dist', extraContext: []],
    [arch: 'amd64', name: 'repository-ui',  path: './',  dockerfile: './Dockerfile.dist', extraContext: []],
]

def merges = builds.findAll { it.arch == nativeArch }

pipeline {
    agent {
        label nativeArch
    }

    environment {
        DOCKER_PROJECT = 'repository'
    }

    stages {
        stage('Build Multi-Arch Images') {
            steps {
                script {
                    def parallelStages = builds.collectEntries { build ->
                        def stageName = "${build.name}-${build.arch}"
                        [(stageName): {
                            node(build.arch) {
                                stage(stageName) {
                                    dockerBuildImage(
                                        build.arch,
                                        build.name,
                                        build.path,
                                        build.dockerfile,
                                        build.extraContext
                                    )
                                }
                            }
                        }]
                    }
                    parallel parallelStages
                }
            }
        }

        stage('Merge Multi-Arch Images') {
            steps {
                script {
                    def parallelStages = merges.collectEntries { build ->
                        def stageName = "${build.name}"
                        [(stageName): {
                            node(build.arch) {
                                stage(stageName) {
                                    dockerMergeImages(
                                        build.name
                                    )
                                }
                            }
                        }]
                    }
                    parallel parallelStages
                }
            }
        }
    }

    post {
        failure {
            echo "❌"
        }
        success {
            echo "✅"
        }
    }
}

def dockerBuildImage(imageArch, imageName, contextPath, dockerfileName, extraBuildContext = []) {
    def imageBase = "${env.DOCKER_REGISTRY}/${env.DOCKER_PROJECT}/${imageName}"
    def imageTagLatest = "${imageBase}:latest-${imageArch}"

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

            docker build ${buildContextArgs} \
                -f ${dockerfileName} \
                -t ${imageTagLatest} \
                ${contextPath}
            docker push ${imageTagLatest}
        """
    }
}

def dockerMergeImages(imageName) {
    def timestamp = sh(script: "date +%Y.%m.%d%H", returnStdout: true).trim()

    def base = "${env.DOCKER_REGISTRY}/${env.DOCKER_PROJECT}/${imageName}"
    def timestampedManifest = "${base}:${timestamp}"
    def latestManifest = "${base}:latest"

    def latestArchTags = [
        "${base}:latest-amd64", "${base}:latest-arm64"
    ]

    withCredentials([
        usernamePassword(
            credentialsId: env.DOCKER_CREDENTIALS_ID,
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )
    ]) {
        sh """
            echo \$DOCKER_PASS | docker login ${env.DOCKER_REGISTRY} -u \$DOCKER_USER --password-stdin

            docker manifest create ${latestManifest} ${latestArchTags.join(' ')}
            docker manifest annotate ${latestManifest} ${latestArchTags[0]} --arch amd64
            docker manifest annotate ${latestManifest} ${latestArchTags[1]} --arch arm64
            docker manifest push ${latestManifest}

            docker manifest create ${timestampedManifest} ${latestArchTags.join(' ')}
            docker manifest annotate ${timestampedManifest} ${latestArchTags[0]} --arch amd64
            docker manifest annotate ${timestampedManifest} ${latestArchTags[1]} --arch arm64
            docker manifest push ${timestampedManifest}
        """
    }
}