pipeline {
    agent any

    tools {
        maven 'Maven-3.9'
    }

    environment {

        // Application Details
        APP_NAME           = "sync-service"
        PROJECT_ID         = "cloudeagle-devops-prod"
        REGION             = "asia-south1"

        // Artifact Registry
        REGISTRY_REPO      = "sync-service-repo"
        IMAGE_NAME         = "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_REPO}/${APP_NAME}"

        // Dynamic Build Tag
        BUILD_TAG_VERSION  = "v${BUILD_NUMBER}"

        // Kubernetes Cluster
        GKE_CLUSTER        = "sync-service-cluster"
        GKE_ZONE           = "asia-south1-a"

        // SonarQube
        SONARQUBE_SERVER   = "SonarQube"

        // Credentials IDs configured in Jenkins
        GCP_CREDENTIALS    = "gcp-service-account"
        DOCKER_CREDENTIALS = "artifact-registry-creds"

        // Environment Namespaces
        QA_NAMESPACE       = "qa"
        STAGE_NAMESPACE    = "staging"
        PROD_NAMESPACE     = "prod"
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "Checking out source code..."
                git branch: "${env.BRANCH_NAME}",
                credentialsId: 'github-creds',
                url: 'https://github.com/company/sync-service.git'
            }
        }

        stage('Build Application') {
            steps {
                echo "Building Spring Boot application..."
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Run Unit Tests') {
            steps {
                echo "Running unit tests..."
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "Running SonarQube analysis..."

                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                    mvn sonar:sonar \
                    -Dsonar.projectKey=sync-service \
                    -Dsonar.projectName=sync-service
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."

                sh """
                docker build -t ${IMAGE_NAME}:${BUILD_TAG_VERSION} .
                docker tag ${IMAGE_NAME}:${BUILD_TAG_VERSION} ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Authenticate with GCP') {
            steps {

                withCredentials([
                    file(credentialsId: "${GCP_CREDENTIALS}", variable: 'GCP_KEY')
                ]) {

                    sh '''
                    gcloud auth activate-service-account --key-file=$GCP_KEY
                    gcloud config set project ${PROJECT_ID}

                    gcloud auth configure-docker asia-south1-docker.pkg.dev --quiet
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {

                echo "Pushing Docker image to Artifact Registry..."

                sh """
                docker push ${IMAGE_NAME}:${BUILD_TAG_VERSION}
                docker push ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy to QA') {

            when {
                branch 'develop'
            }

            steps {

                echo "Deploying application to QA environment..."

                sh """
                kubectl set image deployment/${APP_NAME} \
                ${APP_NAME}=${IMAGE_NAME}:${BUILD_TAG_VERSION} \
                -n ${QA_NAMESPACE}

                kubectl rollout status deployment/${APP_NAME} -n ${QA_NAMESPACE}
                """
            }
        }

        stage('Deploy to Staging') {

            when {
                branch 'staging'
            }

            steps {

                echo "Deploying application to Staging environment..."

                sh """
                kubectl set image deployment/${APP_NAME} \
                ${APP_NAME}=${IMAGE_NAME}:${BUILD_TAG_VERSION} \
                -n ${STAGE_NAMESPACE}

                kubectl rollout status deployment/${APP_NAME} -n ${STAGE_NAMESPACE}
                """
            }
        }

        stage('Production Approval') {

            when {
                branch 'main'
            }

            steps {

                timeout(time: 15, unit: 'MINUTES') {

                    input message: "Approve Production Deployment?",
                    ok: "Deploy to Production"
                }
            }
        }

        stage('Deploy to Production') {

            when {
                branch 'main'
            }

            steps {

                echo "Deploying application to Production..."

                sh """
                kubectl set image deployment/${APP_NAME} \
                ${APP_NAME}=${IMAGE_NAME}:${BUILD_TAG_VERSION} \
                -n ${PROD_NAMESPACE}

                kubectl rollout status deployment/${APP_NAME} -n ${PROD_NAMESPACE}
                """
            }
        }

        stage('Health Check Validation') {

            steps {

                echo "Performing health check..."

                sh '''
                sleep 30

                curl -f http://sync-service.company.com/actuator/health
                '''
            }
        }
    }

    post {

        success {

            echo "Pipeline executed successfully."

            echo "Deployed Docker Image:"
            echo "${IMAGE_NAME}:${BUILD_TAG_VERSION}"
        }

        failure {

            echo "Deployment failed. Starting rollback..."

            sh """
            kubectl rollout undo deployment/${APP_NAME} -n ${PROD_NAMESPACE}
            """
        }

        always {

            cleanWs()
        }
    }
}


