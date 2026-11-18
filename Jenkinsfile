pipeline {
    agent any

    environment {
        IMAGE_NAME = 'shariaa/fa23-bds-036-midterm'
        DOCKER_CREDENTIALS = 'dockerhub-creds'
        CONTAINER_NAME = 'midterm-api'
        TEST_CONTAINER = 'midterm-api-test'
        TEST_PORT = '18000'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_SHA = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    env.BUILD_TAG_IMAGE = "${IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv .venv
                    .venv/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('Validate Dataset') {
            steps {
                sh '''
                    test -f config.json
                    test -f dataset/train.csv

                    .venv/bin/python - <<'PYTHON'
import json
import pandas as pd

with open("config.json") as f:
    config = json.load(f)

df = pd.read_csv("dataset/train.csv")

assert config["student_id"] == "FA23-BDS-036"
assert len(df) >= 100
assert "label" in df.columns
assert df["label"].nunique() == 2
assert set(config["features"]).issubset(df.columns)

print("Dataset validation passed:", len(df), "rows")
print("Missing values:", int(df[config["features"]].isna().sum().sum()))
PYTHON
                '''
            }
        }

        stage('Train Model') {
            steps {
                sh '''
                    GIT_COMMIT="$GIT_SHA" .venv/bin/python train.py
                    test -s artifacts/model.joblib
                    test -s artifacts/metrics.json
                    cat artifacts/metrics.json
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      --build-arg GIT_COMMIT="$GIT_SHA" \
                      -t "$BUILD_TAG_IMAGE" .
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                sh '''
                    docker rm -f "$TEST_CONTAINER" 2>/dev/null || true

                    docker run -d \
                      --name "$TEST_CONTAINER" \
                      -p "$TEST_PORT:8000" \
                      "$BUILD_TAG_IMAGE"

                    READY=false
                    for i in $(seq 1 30); do
                        if curl -fsS "http://127.0.0.1:$TEST_PORT/health" >/dev/null; then
                            READY=true
                            break
                        fi
                        sleep 2
                    done

                    if [ "$READY" != "true" ]; then
                        docker logs "$TEST_CONTAINER" || true
                        exit 1
                    fi

                    curl -fsS "http://127.0.0.1:$TEST_PORT/health"
                    curl -fsS "http://127.0.0.1:$TEST_PORT/metrics"

                    curl -fsS -X POST "http://127.0.0.1:$TEST_PORT/predict" \
                      -H "Content-Type: application/json" \
                      -d '{"records":[{"motor_heat":0.7,"vibration":-0.2,"wear_index":0.4,"load_factor":0.8}]}'

                    docker rm -f "$TEST_CONTAINER"
                '''
            }
        }

        stage('Push Docker Hub Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        set +x
                        echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin
                        docker tag "$BUILD_TAG_IMAGE" "$IMAGE_NAME:latest"
                        docker push "$BUILD_TAG_IMAGE"
                        docker push "$IMAGE_NAME:latest"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy on EC2') {
            steps {
                sh '''
                    docker rm -f midterm-api-previous 2>/dev/null || true

                    if docker inspect "$CONTAINER_NAME" >/dev/null 2>&1; then
                        docker rename "$CONTAINER_NAME" midterm-api-previous
                        docker stop midterm-api-previous
                    fi

                    docker run -d \
                      --name "$CONTAINER_NAME" \
                      --restart unless-stopped \
                      -p 8000:8000 \
                      "$BUILD_TAG_IMAGE"

                    READY=false
                    for i in $(seq 1 30); do
                        if curl -fsS http://127.0.0.1:8000/health >/dev/null; then
                            READY=true
                            break
                        fi
                        sleep 2
                    done

                    if [ "$READY" != "true" ]; then
                        docker logs "$CONTAINER_NAME" || true
                        docker rm -f "$CONTAINER_NAME" || true

                        if docker inspect midterm-api-previous >/dev/null 2>&1; then
                            docker rename midterm-api-previous "$CONTAINER_NAME"
                            docker start "$CONTAINER_NAME"
                        fi

                        exit 1
                    fi

                    docker rm -f midterm-api-previous 2>/dev/null || true
                    curl -fsS http://127.0.0.1:8000/health
                    curl -fsS http://127.0.0.1:8000/metrics
                '''
            }
        }
    }

    post {
        always {
            sh "docker rm -f midterm-api-test 2>/dev/null || true"
        }
        success {
            script {
                def causes = currentBuild.getBuildCauses()
                def description = causes.collect { it.shortDescription ?: "" }
                    .find { it.contains("GitHub push by") } ?: ""
                def pusher = description.replace("Started by GitHub push by ", "").trim()

                def recipients = [
                    "sharianoor": "sharianoor2@gmail.com",
                    "malik-qasim": "qasimalik@gmail.com"
                ]

                def recipient = recipients[pusher]

                if (recipient) {
                    emailext(
                        to: recipient,
                        from: "sharianoor2@gmail.com",
                        subject: "FA23-BDS-036 - SUCCESS - Build #${BUILD_NUMBER}",
                        mimeType: "text/plain",
                        body: """Pipeline completed successfully.

Student ID: FA23-BDS-036
Git Commit: ${GIT_SHA}
Pusher: ${pusher}
Build URL: ${BUILD_URL}
"""
                    )
                } else {
                    echo "Email skipped: unrecognized or missing GitHub pusher: ${pusher}"
                }
            }
        }
        failure {
            script {
                def causes = currentBuild.getBuildCauses()
                def description = causes.collect { it.shortDescription ?: "" }
                    .find { it.contains("GitHub push by") } ?: ""
                def pusher = description.replace("Started by GitHub push by ", "").trim()

                def recipients = [
                    "sharianoor": "sharianoor2@gmail.com",
                    "malik-qasim": "qasimalik@gmail.com"
                ]

                def recipient = recipients[pusher]

                if (recipient) {
                    emailext(
                        to: recipient,
                        from: "sharianoor2@gmail.com",
                        subject: "FA23-BDS-036 - FAILURE - Build #${BUILD_NUMBER}",
                        mimeType: "text/plain",
                        body: """Pipeline failed.

Student ID: FA23-BDS-036
Pusher: ${pusher}
Build URL: ${BUILD_URL}
"""
                    )
                } else {
                    echo "Email skipped: unrecognized or missing GitHub pusher: ${pusher}"
                }
            }
        }
    }
}
