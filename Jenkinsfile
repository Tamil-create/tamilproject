// Employee Management System - ECS CI/CD Jenkinsfile
// Flow: GitHub -> Webhook -> Jenkins -> Build/Test -> Docker -> ECR -> ECS

pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = credentials('ecr-registry-url')
        ECS_CLUSTER_NAME = 'employee-management-cluster'
        ECS_BACKEND_SERVICE = "${params.ECS_BACKEND_SERVICE ?: 'backend-service'}"
        ECS_FRONTEND_SERVICE = "${params.ECS_FRONTEND_SERVICE ?: 'frontend-service'}"
        ECS_BACKEND_CONTAINER = 'backend'
        ECS_FRONTEND_CONTAINER = 'frontend'
        DOCKER_BUILDKIT = '1'
        IMAGE_TAG = "${env.GIT_COMMIT.take(8)}"
        BACKEND_IMAGE = "${ECR_REGISTRY}/employee-management-backend:${IMAGE_TAG}"
        FRONTEND_IMAGE = "${ECR_REGISTRY}/employee-management-frontend:${IMAGE_TAG}"
    }

    parameters {
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip running tests'
        )
        string(
            name: 'ECS_BACKEND_SERVICE',
            defaultValue: 'backend-service',
            description: 'ECS backend service name'
        )
        string(
            name: 'ECS_FRONTEND_SERVICE',
            defaultValue: 'frontend-service',
            description: 'ECS frontend service name'
        )
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "========================================="
                    echo "Employee Management System - CI/CD Pipeline"
                    echo "========================================="
                    echo "Build Number: ${env.BUILD_NUMBER}"
                    echo "Git Commit: ${env.GIT_COMMIT}"
                    echo "Git Branch: ${env.GIT_BRANCH}"
                    echo "ECS Cluster: ${ECS_CLUSTER_NAME}"
                    echo "Image Tag: ${IMAGE_TAG}"
                    echo "========================================="

                    // Clean workspace
                    cleanWs()
                    checkout scm
                }
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Backend Dependencies') {
                    steps {
                        dir('backend') {
                            script {
                                echo 'Installing backend dependencies...'
                                sh '''
                                    if [ -f pom.xml ]; then
                                        mvn clean install -DskipTests
                                    elif [ -f package.json ]; then
                                        npm ci --production=false
                                    fi
                                '''
                            }
                        }
                    }
                }
                stage('Frontend Dependencies') {
                    steps {
                        dir('frontend') {
                            script {
                                echo 'Installing frontend dependencies...'
                                sh 'npm ci --production=false'
                            }
                        }
                    }
                }
            }
        }

        stage('Code Quality & Security') {
            parallel {
                stage('Lint') {
                    steps {
                        script {
                            echo 'Running linters...'
                            dir('frontend') {
                                sh 'npm run lint || true'
                            }
                        }
                    }
                }
                stage('Security Scan') {
                    steps {
                        script {
                            echo 'Running security scans...'
                            sh '''
                                # NPM audit
                                cd frontend && npm audit --audit-level=moderate || true

                                # Trivy vulnerability scanner (if available)
                                if command -v trivy &> /dev/null; then
                                    trivy fs --severity HIGH,CRITICAL --exit-code 0 .
                                fi
                            '''
                        }
                    }
                }
            }
        }

        stage('Run Tests') {
            when {
                expression { return !params.SKIP_TESTS }
            }
            parallel {
                stage('Backend Tests') {
                    steps {
                        dir('backend') {
                            script {
                                echo 'Running backend tests...'
                                sh '''
                                    if [ -f pom.xml ]; then
                                        mvn test
                                    elif [ -f package.json ]; then
                                        npm test || true
                                    fi
                                '''
                            }
                        }
                    }
                }
                stage('Frontend Tests') {
                    steps {
                        dir('frontend') {
                            script {
                                echo 'Running frontend tests...'
                                sh 'CI=true npm test -- --coverage --watchAll=false || true'
                            }
                        }
                    }
                }
            }
        }

        stage('Build Applications') {
            parallel {
                stage('Build Backend') {
                    steps {
                        dir('backend') {
                            script {
                                echo 'Building backend application...'
                                sh '''
                                    if [ -f pom.xml ]; then
                                        mvn clean package -DskipTests
                                    elif [ -f package.json ]; then
                                        npm run build
                                    fi
                                '''
                            }
                        }
                    }
                }
                stage('Build Frontend') {
                    steps {
                        dir('frontend') {
                            script {
                                echo 'Building frontend application...'
                                sh 'npm run build'
                            }
                        }
                    }
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                script {
                    echo 'Authenticating with ECR...'
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """

                    parallel(
                        'Backend Image': {
                            dir('backend') {
                                echo "Building backend image: ${BACKEND_IMAGE}"
                                sh """
                                    docker build \
                                        --build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
                                        --build-arg VCS_REF=${env.GIT_COMMIT} \
                                        --build-arg VERSION=${IMAGE_TAG} \
                                        -t ${BACKEND_IMAGE} \
                                        -t ${ECR_REGISTRY}/employee-management-backend:latest \
                                        .

                                    docker push ${BACKEND_IMAGE}
                                    docker push ${ECR_REGISTRY}/employee-management-backend:latest
                                """
                            }
                        },
                        'Frontend Image': {
                            dir('frontend') {
                                echo "Building frontend image: ${FRONTEND_IMAGE}"
                                sh """
                                    docker build \
                                        --build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
                                        --build-arg VCS_REF=${env.GIT_COMMIT} \
                                        --build-arg VERSION=${IMAGE_TAG} \
                                        -t ${FRONTEND_IMAGE} \
                                        -t ${ECR_REGISTRY}/employee-management-frontend:latest \
                                        .

                                    docker push ${FRONTEND_IMAGE}
                                    docker push ${ECR_REGISTRY}/employee-management-frontend:latest
                                """
                            }
                        }
                    )
                }
            }
        }

        stage('Image Security Scan') {
            steps {
                script {
                    echo 'Scanning Docker images for vulnerabilities...'
                    sh """
                        if command -v trivy &> /dev/null; then
                            trivy image --severity HIGH,CRITICAL --exit-code 0 ${BACKEND_IMAGE}
                            trivy image --severity HIGH,CRITICAL --exit-code 0 ${FRONTEND_IMAGE}
                        else
                            echo 'Trivy not installed, skipping image scan'
                        fi
                    """
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    echo "Deploying version ${IMAGE_TAG} to ECS cluster ${ECS_CLUSTER_NAME}"
                    sh """
                        set -e

                        deploy_ecs_service() {
                            SERVICE=\$1
                            CONTAINER=\$2
                            NEW_IMAGE=\$3
                            FILE=\$4

                            echo "Preparing ECS service: \${SERVICE}"

                            CURRENT_TASK_DEF=\$(aws ecs describe-services \\
                                --cluster ${ECS_CLUSTER_NAME} \\
                                --services "\${SERVICE}" \\
                                --region ${AWS_REGION} \\
                                --query 'services[0].taskDefinition' \\
                                --output text)

                            if [ "\${CURRENT_TASK_DEF}" = "None" ] || [ -z "\${CURRENT_TASK_DEF}" ]; then
                                echo "ERROR: ECS service \${SERVICE} was not found."
                                exit 1
                            fi

                            echo "Current task definition: \${CURRENT_TASK_DEF}"

                            aws ecs describe-task-definition \\
                                --task-definition "\${CURRENT_TASK_DEF}" \\
                                --region ${AWS_REGION} \\
                                --query 'taskDefinition' > "\${FILE}.source.json"

                            jq --arg IMAGE "\${NEW_IMAGE}" --arg CONTAINER "\${CONTAINER}" '
                                if any(.containerDefinitions[]; .name == \$CONTAINER)
                                then del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy)
                                     | .containerDefinitions |= map(if .name == \$CONTAINER then .image = \$IMAGE else . end)
                                else error("Container name not found in task definition")
                                end
                            ' "\${FILE}.source.json" > "\${FILE}.json"

                            NEW_TASK_DEF=\$(aws ecs register-task-definition \\
                                --cli-input-json file://"\${FILE}.json" \\
                                --region ${AWS_REGION} \\
                                --query 'taskDefinition.taskDefinitionArn' \\
                                --output text)

                            echo "Registered task definition: \${NEW_TASK_DEF}"

                            aws ecs update-service \\
                                --cluster ${ECS_CLUSTER_NAME} \\
                                --service "\${SERVICE}" \\
                                --task-definition "\${NEW_TASK_DEF}" \\
                                --region ${AWS_REGION} \\
                                --query 'service.serviceName' \\
                                --output text

                            echo "\${NEW_TASK_DEF}" > "\${FILE}.new-td"
                        }  

                        deploy_ecs_service \\
                            "${ECS_BACKEND_SERVICE}" \\
                            "${ECS_BACKEND_CONTAINER}" \\
                            "${BACKEND_IMAGE}" \\
                            "/tmp/backend-task-definition"

                        deploy_ecs_service \\
                            "${ECS_FRONTEND_SERVICE}" \\
                            "${ECS_FRONTEND_CONTAINER}" \\
                            "${FRONTEND_IMAGE}" \\
                            "/tmp/frontend-task-definition"

                        echo "Waiting for ECS services to become stable..."
                        aws ecs wait services-stable \\
                            --cluster ${ECS_CLUSTER_NAME} \\
                            --services ${ECS_BACKEND_SERVICE} ${ECS_FRONTEND_SERVICE} \\
                            --region ${AWS_REGION}
                    """
                }
            }
        }

        stage('Post-Deployment Verification') {
            steps {
                sh """
                    echo "Checking ECS service status..."

                    aws ecs describe-services \\
                        --cluster ${ECS_CLUSTER_NAME} \\
                        --services ${ECS_BACKEND_SERVICE} ${ECS_FRONTEND_SERVICE} \\
                        --region ${AWS_REGION} \\
                        --query 'services[].{Service:serviceName,Status:status,Desired:desiredCount,Running:runningCount,Pending:pendingCount}' \\
                        --output table

                    echo "Running backend tasks:"
                    aws ecs list-tasks \\
                        --cluster ${ECS_CLUSTER_NAME} \\
                        --service-name ${ECS_BACKEND_SERVICE} \\
                        --region ${AWS_REGION} \\
                        --desired-status RUNNING

                    echo "Running frontend tasks:"
                    aws ecs list-tasks \\
                        --cluster ${ECS_CLUSTER_NAME} \\
                        --service-name ${ECS_FRONTEND_SERVICE} \\
                        --region ${AWS_REGION} \\
                        --desired-status RUNNING
                """
            }
        }
    }

    post {
        always {
            script {
                echo 'Cleaning up...'
                sh """
                    # Remove local Docker images
                    docker rmi ${BACKEND_IMAGE} || true
                    docker rmi ${FRONTEND_IMAGE} || true
                    docker rmi ${ECR_REGISTRY}/employee-management-backend:latest || true
                    docker rmi ${ECR_REGISTRY}/employee-management-frontend:latest || true

                    # Clean up docker system
                    docker system prune -f || true
                """
            }
        }
        success {
            echo 'Pipeline completed successfully!'
            script {
                sh """
                    echo "========================================="
                    echo "Deployment Summary"
                    echo "========================================="
                    echo "ECS Cluster: ${ECS_CLUSTER_NAME}"
                    echo "Backend Image: ${BACKEND_IMAGE}"
                    echo "Frontend Image: ${FRONTEND_IMAGE}"
                    echo "========================================="

                    aws ecs describe-services \\
                        --cluster ${ECS_CLUSTER_NAME} \\
                        --services ${ECS_BACKEND_SERVICE} ${ECS_FRONTEND_SERVICE} \\
                        --region ${AWS_REGION} \\
                        --query 'services[].{Service:serviceName,Status:status,Desired:desiredCount,Running:runningCount}' \\
                        --output table
                """
            }
        }
        failure {
            echo 'Pipeline failed!'
            echo 'Check the Jenkins console log and ECS service events for the failed deployment.'
        }
    }
}
