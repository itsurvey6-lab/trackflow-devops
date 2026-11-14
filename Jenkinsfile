
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(
            numToKeepStr: '10'
        ))
    }

    environment {
        APP_REPO = 'https://github.com/itsurvey6-lab/trackflow.git'
        APP_BRANCH = 'main'
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
                        ls frontend/package.json

                        echo "===== BACKEND ====="
                        ls backend/package.json

                        echo "===== DOCKERFILES ====="
                        find . -maxdepth 3 -name Dockerfile \
                            -not -path '*/node_modules/*'
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'TrackFlow source checkout and inspection successful.'
        }

        failure {
            echo 'TrackFlow pipeline failed. Check Jenkins console output.'
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
