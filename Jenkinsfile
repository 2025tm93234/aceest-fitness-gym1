pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    environment {
        IMAGE_NAME = 'aceest-fitness-ci'
    }

    stages {
        stage('Clean Checkout') {
            steps {
                deleteDir()
                checkout scm
            }
        }

        stage('Build & Lint') {
            steps {
                sh '''
                    set -eu
                    python3 --version
                    python3 -m venv .venv
                    . .venv/bin/activate
                    python -m pip install -r requirements.txt
                    python -m compileall -q app.py tests
                    python -m flake8 app.py tests/
                    python -m pytest -v --cov=app
                '''
            }
        }

        stage('Docker Image Assembly') {
            steps {
                sh 'docker build --no-cache -t ${IMAGE_NAME}:${BUILD_NUMBER} .'
            }
        }

        stage('Automated Testing inside Container') {
            steps {
                sh 'docker run --rm ${IMAGE_NAME}:${BUILD_NUMBER} python -m pytest -v --cov=app'
            }
        }

        stage('Container Health Check') {
            steps {
                sh '''
                    set -eu
                    CONTAINER_NAME="aceest-smoke-${BUILD_NUMBER}"
                    docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true
                    docker run -d --name "$CONTAINER_NAME" "$IMAGE_NAME:$BUILD_NUMBER" >/dev/null
                    cleanup() {
                        docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true
                    }
                    trap cleanup EXIT
                    ATTEMPT=0
                    until docker exec "$CONTAINER_NAME" python -c "from urllib.request import urlopen; print(urlopen('http://127.0.0.1:5000/health', timeout=2).read().decode())"; do
                        ATTEMPT=$((ATTEMPT + 1))
                        if [ "$ATTEMPT" -ge 20 ]; then
                            docker logs "$CONTAINER_NAME"
                            exit 1
                        fi
                        sleep 1
                    done
                '''
            }
        }
    }
}
