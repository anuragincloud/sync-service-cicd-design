pipeline {
    agent any

    environment {
        PROJECT_ID = "gcp-project-id"
        IMAGE_NAME = "sync-service"
        REGISTRY = "gcr.io/${PROJECT_ID}"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME}", url: 'https://github.com/anuragincloud/sync-service.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
                """
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([file(credentialsId: 'gcp-service-account', variable: 'GCP_KEY')]) {
                    sh """
                    gcloud auth activate-service-account --key-file=$GCP_KEY
                    gcloud auth configure-docker
                    docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'staging'
                    branch 'main'
                }
            }
            steps {
                script {
                    if (env.BRANCH_NAME == 'develop') {
                        deployEnv("qa")
                    } else if (env.BRANCH_NAME == 'staging') {
                        deployEnv("staging")
                    } else if (env.BRANCH_NAME == 'main') {
                        input message: "Approve Production Deployment?"
                        deployEnv("prod")
                    }
                }
            }
        }

        stage('Smoke Test') {
            steps {
                sh 'curl -f http://service-url/health || exit 1'
            }
        }
    }

    post {
        failure {
            echo "Deployment failed! Initiating rollback..."
            sh "./rollback.sh"
        }
    }
}

def deployEnv(envName) {
    sh """
    ssh user@${envName}-vm '
    docker pull gcr.io/${PROJECT_ID}/${IMAGE_NAME}:${IMAGE_TAG}
    docker stop sync-service || true
    docker rm sync-service || true
    docker run -d \
        --name sync-service \
        -e SPRING_PROFILES_ACTIVE=${envName} \
        -p 8080:8080 \
        gcr.io/${PROJECT_ID}/${IMAGE_NAME}:${IMAGE_TAG}
    '
    """
}
