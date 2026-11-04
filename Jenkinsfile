pipeline {
    agent { label 'build' }

    options {
        timestamps()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        SOLUTION_NAME = 'CIDI-Tofayel'
        GIT_URL       = 'https://github.com/2fael/CIDI-Tofayel.git'
        GIT_BRANCH    = 'master'
        REGISTRY      = 'node5.naasbd.com'
        REPO_IMAGE    = 'node5.naasbd.com/cicdtofael/cicddummy'
        DOCKER_HOST   = 'unix:///var/run/docker.sock'
        BUILDER_NAME  = 'hld'
    }

    stages {
        stage('Prep workspace') {
            steps {
                cleanWs()
                sh 'set -euo pipefail; echo "Workspace prepared."'
            }
        }

        stage('Clone') {
            steps {
                retry(2) {
                     
                        sh '''
                            set -euo pipefail
                            rm -rf "$SOLUTION_NAME"
                            git clone --single-branch --branch "$GIT_BRANCH" "$GIT_URL" "$SOLUTION_NAME"
                            ls -alh "$SOLUTION_NAME"
                        '''
                }
            }
        }

        stage('Read project version') {
            steps {
                script {
                    def projectFile = sh(script: "find '${env.SOLUTION_NAME}/' -type f -name '*.csproj' | head -n1",
                                         returnStdout: true).trim()
                    if (!projectFile) { error 'Project file not found (pattern *.csproj).' }
                    def ver = sh(returnStdout: true, script: """
                        set -euo pipefail
                        xmllint --xpath "string(//PropertyGroup/Version)" '${projectFile}'
                    """).trim()
                    if (!ver) { error "Version element not found in ${projectFile}." }
                    env.VERSION = ver
                    echo "Detected version: ${env.VERSION}"
                }
            }
        }

        stage('Verify version') {
            steps {
                script {
                    if (!env.VERSION?.trim()) { error('Project version is empty - aborting.') }
                    echo "Using VERSION=${env.VERSION}"
                }
            }
        }

        stage('Login registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'HARBOR-CRED',
                                                  usernameVariable: 'HARBOR_USER',
                                                  passwordVariable: 'HARBOR_PASS')]) {
                    sh '''
                        set -euo pipefail
                        echo "$HARBOR_PASS" | docker login "$REGISTRY" -u "$HARBOR_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Setup Docker Buildx') {
            steps {
                sh """
                    set -euo pipefail
                    export DOCKER_BUILDKIT=1
                    if ! docker buildx inspect ${BUILDER_NAME} >/dev/null 2>&1; then
                      docker buildx create --name ${BUILDER_NAME} --driver docker-container --use
                    else
                      docker buildx use ${BUILDER_NAME}
                    fi
                    docker buildx inspect --bootstrap
                """
            }
        }

        stage('Build image (buildx)') {
            steps {
                sh """
                    set -euo pipefail
                    export DOCKER_BUILDKIT=1
                    docker buildx use ${BUILDER_NAME}
                    docker buildx build \\
                      --platform linux/amd64 \\
                      --progress=plain \\
                      -t '${REPO_IMAGE}:${VERSION}' \\
                      -f '${SOLUTION_NAME}/Dockerfile' \\
                      '${SOLUTION_NAME}/' \\
                      --load
                """
            }
        }

        stage('Push image') {
            steps {
                retry(2) {
                    sh """
                        set -euo pipefail
                        docker push '${REPO_IMAGE}:${VERSION}'
                    """
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker image prune -f || true'
                cleanWs()
            }
        }
    }

    post {
        success { echo "Build & push OK: ${env.REPO_IMAGE}:${env.VERSION}" }
        failure { echo 'Build failed.' }
    }
}
