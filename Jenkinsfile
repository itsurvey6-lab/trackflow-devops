
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        APP_REPO = 'https://github.com/itsurvey6-lab/trackflow.git'
        APP_BRANCH = 'main'
        FRONTEND_IMAGE = 'trackflow-frontend'
        BACKEND_IMAGE = 'trackflow-backend'
        FRONTEND_API_URL = 'http://192.168.1.20:5000'
    }

    stages {

        stage('Checkout Application') {
            steps {
                echo 'Checking out TrackFlow application source'

                dir('workspace/trackflow') {
                    git branch: "${APP_BRANCH}",
                        url: "${APP_REPO}"
                }
            }
        }

        stage('Inspect Application') {
            steps {
                dir('workspace/trackflow') {
                    sh '''
                        echo "===== APPLICATION STRUCTURE ====="
                        ls -la

                        echo "===== FRONTEND ====="
                        test -f frontend/package.json

                        echo "===== BACKEND ====="
                        test -f backend/package.json

                        echo "===== LOCK FILES ====="
                        test -f frontend/package-lock.json
                        test -f backend/package-lock.json

                        echo "Application inspection successful"
                    '''
                }
            }
        }

        stage('Validate Backend') {
            steps {
                dir('workspace/trackflow/backend') {
                    sh '''
                        echo "Checking backend JavaScript syntax"

                        find src -name "*.js" -type f \
                            -exec node --check {} \\;

                        echo "Backend syntax validation successful"
                    '''
                }
            }
        }

        stage('Build Frontend Docker Image') {
            steps {
                sh '''
                    set -e

                    echo "Preparing frontend Docker build context"

                    cp deployment/docker/frontend/nginx.conf \
                        workspace/trackflow/frontend/nginx.conf

                    echo "Building frontend image"

                    docker build \
                        --build-arg VITE_API_URL="$FRONTEND_API_URL" \
                        -t "$FRONTEND_IMAGE:$BUILD_NUMBER" \
                        -t "$FRONTEND_IMAGE:latest" \
                        -f deployment/docker/frontend/Dockerfile \
                        workspace/trackflow/frontend
                '''
            }
        }

        stage('Build Backend Docker Image') {
            steps {
                sh '''
                    set -e

                    echo "Building backend image"

                    docker build \
                        -t "$BACKEND_IMAGE:$BUILD_NUMBER" \
                        -t "$BACKEND_IMAGE:latest" \
                        -f deployment/docker/backend/Dockerfile \
                        workspace/trackflow/backend
                '''
            }
        }

        stage('Verify Docker Images') {
            steps {
                sh '''
                    echo "===== BUILT DOCKER IMAGES ====="

                    docker image inspect \
                        "$FRONTEND_IMAGE:$BUILD_NUMBER"

                    docker image inspect \
                        "$BACKEND_IMAGE:$BUILD_NUMBER"

                    docker images | grep trackflow
                '''
            }
        }


        stage('Trivy Security Scan') {
            steps {
                sh '''
                    set -e

                    echo "Scanning Backend Image..."
                    trivy image --exit-code 1 --severity HIGH,CRITICAL \
                        ${BACKEND_IMAGE}:${BUILD_NUMBER}

                    echo "Scanning Frontend Image..."
                    trivy image --exit-code 1 --severity HIGH,CRITICAL \
                        ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                '''
            }
        }


	stage('Deploy TrackFlow') {
            steps {
                sh '''
                    set -e

                    echo "Deploying TrackFlow build $BUILD_NUMBER"

                    IMAGE_TAG="$BUILD_NUMBER" docker compose \
                        -p trackflow \
                        --project-directory /home/ubuntu/trackflow \
                        -f deployment/docker-compose.yml \
                        up -d \
                        --no-build \
                        --pull never \
                        --wait

                    echo "Deployment completed successfully"
                '''
            }
        }



    }





    post {
        success {
            echo 'TrackFlow Docker images built and verified successfully.'
        }

        failure {
            echo 'TrackFlow pipeline failed. Check Jenkins console output.'
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
