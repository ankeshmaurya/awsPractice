pipeline {
    agent any

    parameters {
        string(
            name: 'IMAGE_NAME',
            defaultValue: 'cloudly-dashboard',
            description: 'Docker image name'
        )
        string(
            name: 'REGISTRY',
            defaultValue: '',
            description: 'Optional registry host, for example registry.example.com/team'
        )
        credentials(
            name: 'REGISTRY_CREDENTIALS',
            credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl',
            description: 'Optional Docker registry credentials',
            required: false
        )
        booleanParam(
            name: 'PUSH_IMAGE',
            defaultValue: false,
            description: 'Push the image to the configured registry'
        )
        booleanParam(
            name: 'DEPLOY_LOCALLY',
            defaultValue: false,
            description: 'Run the image on the Jenkins Docker host'
        )
    }

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        CONTAINER_NAME = 'cloudly-dashboard'
        DEPLOY_PORT = '8080'
    }

    stages {
        stage('Validate') {
            steps {
                script {
                    if (!fileExists('index.html')) {
                        error('index.html is required to build the dashboard image')
                    }
                    if (!fileExists('Dockerfile')) {
                        error('Dockerfile is required to build the dashboard image')
                    }
                    if (params.PUSH_IMAGE && !params.REGISTRY?.trim()) {
                        error('Set REGISTRY before enabling PUSH_IMAGE')
                    }
                    if (params.PUSH_IMAGE && !params.REGISTRY_CREDENTIALS?.trim()) {
                        error('Select REGISTRY_CREDENTIALS before enabling PUSH_IMAGE')
                    }
                    sh 'grep -q "<!DOCTYPE html>" index.html'
                }
            }
        }

        stage('Build image') {
            steps {
                script {
                    env.FULL_IMAGE = params.REGISTRY?.trim()
                        ? "${params.REGISTRY.trim()}/${params.IMAGE_NAME}:${env.IMAGE_TAG}"
                        : "${params.IMAGE_NAME}:${env.IMAGE_TAG}"
                    sh "docker build --pull --tag ${env.FULL_IMAGE} ."
                }
            }
        }

        stage('Smoke test') {
            steps {
                sh '''
                    set -eu
                    docker rm --force cloudly-dashboard-test >/dev/null 2>&1 || true
                    docker run --detach --name cloudly-dashboard-test -p 18080:80 "$FULL_IMAGE" >/dev/null
                    trap 'docker rm --force cloudly-dashboard-test >/dev/null 2>&1 || true' EXIT
                    for attempt in 1 2 3 4 5; do
                        if curl --fail --silent http://127.0.0.1:18080/ | grep -q "Cloudly"; then
                            exit 0
                        fi
                        sleep 2
                    done
                    echo "Dashboard smoke test failed"
                    exit 1
                '''
            }
        }

        stage('Push image') {
            when {
                expression {
                    return params.PUSH_IMAGE && params.REGISTRY?.trim()
                }
            }
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: params.REGISTRY_CREDENTIALS,
                            usernameVariable: 'REGISTRY_USER',
                            passwordVariable: 'REGISTRY_PASSWORD'
                        )
                    ]) {
                        sh '''
                            set -eu
                            echo "$REGISTRY_PASSWORD" | docker login "$REGISTRY" \
                                --username "$REGISTRY_USER" --password-stdin
                            docker push "$FULL_IMAGE"
                            docker logout "$REGISTRY"
                        '''
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                expression {
                    return params.DEPLOY_LOCALLY
                }
            }
            steps {
                sh '''
                    set -eu
                    docker rm --force "$CONTAINER_NAME" >/dev/null 2>&1 || true
                    docker run --detach \
                        --restart unless-stopped \
                        --name "$CONTAINER_NAME" \
                        --publish "$DEPLOY_PORT:80" \
                        "$FULL_IMAGE"
                '''
            }
        }
    }

    post {
        always {
            sh 'docker image prune --force --filter "label=org.opencontainers.image.title=Cloudly dashboard" || true'
        }
    }
}
