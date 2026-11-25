
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers {
        
    }

    environment {
        APP_REPO = 'https://github.com/itsurvey6-lab/trackflow.git'
        APP_BRANCH = 'main'
        FRONTEND_IMAGE = 'trackflow-frontend'
        BACKEND_IMAGE = 'trackflow-backend'
        FRONTEND_API_URL = 'http://trackflow.local'
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





        stage('SonarQube Code Analysis') {
            steps {
                dir('workspace/trackflow') {
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            set -e

                            echo "Starting TrackFlow SonarQube analysis"

                            sonar-scanner \
                                -Dsonar.projectKey=TrackFlow \
                                -Dsonar.projectName=TrackFlow \
                                -Dsonar.sources=frontend/src,backend/src \
                                -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/build/**,backend/uploads/**

                            echo "SonarQube analysis submitted"
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
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
			--pull \
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


	stage('Push Images to Docker Hub') {
	    steps {
	        withCredentials([usernamePassword(
	            credentialsId: 'dockerhub-credentials',
	            usernameVariable: 'DOCKERHUB_USER',
	            passwordVariable: 'DOCKERHUB_TOKEN'
	        )]) {
	            sh '''
	                set -e

	                echo "Logging in to Docker Hub"

	                echo "$DOCKERHUB_TOKEN" | docker login \
	                    -u "$DOCKERHUB_USER" \
	                    --password-stdin

	                echo "Tagging TrackFlow images"

	                docker tag "$FRONTEND_IMAGE:$BUILD_NUMBER" \
	                    "$DOCKERHUB_USER/trackflow-frontend:$BUILD_NUMBER"

	                docker tag "$BACKEND_IMAGE:$BUILD_NUMBER" \
	                    "$DOCKERHUB_USER/trackflow-backend:$BUILD_NUMBER"

	                echo "Pushing frontend image"

	                docker push "$DOCKERHUB_USER/trackflow-frontend:$BUILD_NUMBER"

	                echo "Pushing backend image"

	                docker push "$DOCKERHUB_USER/trackflow-backend:$BUILD_NUMBER"

	                docker logout

	                echo "Both TrackFlow images published successfully"
	            '''
	        }
	    }
	}

    }





    post {
    success {
        echo 'TrackFlow Docker images built and verified successfully.'

        build job: 'TrackFlow-CD',
            parameters: [
                string(name: 'IMAGE_TAG', value: env.BUILD_NUMBER)
            ],
            wait: false
    }

    failure {
        echo 'TrackFlow pipeline failed. Check Jenkins console output.'
    }

    always {
        echo 'Pipeline execution completed.'
    }
}



}
