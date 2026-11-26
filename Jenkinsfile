
pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub_credentials'
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm

                script {
                    def branch = env.BRANCH_NAME ?: 'manual'
                    env.SAFE_BRANCH = branch.replaceAll(/[^A-Za-z0-9_.-]/, '-')
                    env.IMAGE_TAG = "${env.SAFE_BRANCH}-${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASS'
                    )
                ]) {
                    script {
                        env.DOCKERHUB_USER = DH_USER
                    }

                    sh '''
                        echo "$DH_PASS" | docker login \
                            -u "$DH_USER" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Build & Tag Images') {
            steps {
                script {
                    env.FRONTEND_TAG_DH = "${env.DOCKERHUB_USER}/three-tier-app-frontend:${env.IMAGE_TAG}"
                    env.BACKEND_TAG_DH = "${env.DOCKERHUB_USER}/three-tier-app-backend:${env.IMAGE_TAG}"
                }

                sh '''
                    docker build -t "$BACKEND_TAG_DH" ./backend
                    docker build -t "$FRONTEND_TAG_DH" ./frontend
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh '''
                    docker push "$BACKEND_TAG_DH"
                    docker push "$FRONTEND_TAG_DH"
                '''
            }
        }

        stage('Prepare .env for Compose') {
            steps {
                script {
                    writeFile(
                        file: '.env',
                        text: """BACKEND_IMAGE=${env.BACKEND_TAG_DH}
FRONTEND_IMAGE=${env.FRONTEND_TAG_DH}
"""
                    )
                }
            }
        }

        stage('Approval for Staging / Prod Deploy') {
            when {
                anyOf {
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                input(
                    message: "Deploy to ${env.BRANCH_NAME} environment?",
                    ok: 'Yes, Deploy'
                )
            }
        }

        stage('Deploy Environment') {
            when {
                anyOf {
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                sh '''
                    docker compose --env-file .env down
                    docker compose --env-file .env pull
                    docker compose --env-file .env up -d --remove-orphans
                '''
            }
        }

        stage('Cleanup Local Images') {
            steps {
                sh '''
                    docker rmi "$BACKEND_TAG_DH" "$FRONTEND_TAG_DH" || true
                '''
            }
        }
    }

    post {
        success {
            echo "SUCCESS: ${env.BRANCH_NAME} pipeline completed successfully!"
        }

        failure {
            echo "FAILURE: Pipeline failed. Check Jenkins console output."
        }

        always {
            sh 'docker logout || true'
        }
    }
}

