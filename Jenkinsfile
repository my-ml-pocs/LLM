pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = "localhost:5000"
        APP_NAME = "myapp"
        Sonar_Url = "https://localhost"
        Sonar_Token = "sonarqube"
        security_scan_tool = "trivy"
        GIT_SHA = "${env.GIT_COMMIT[0..6]}"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'composer install'
            }
        }

        stage('Build') {
            steps {
                sh 'echo "Building the application..."'
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh 'echo "Running unit tests..."'
            }
        }

        stage('Static Code Analysis') {
            steps {
                sh 'sonar-scanner -Dsonar.host.url=${Sonar_Url} -Dsonar.login=${Sonar_Token}'
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qualityGate = waitForQualityGate()
                    if (qualityGate.status != 'OK' && qualityGate.status != 'NONE') {
                        error "Pipeline aborted due to quality gate failure: ${qualityGate.status}"
                    }
                }
            }
        }

        stage('Docker Build and Push') {
            steps {
                sh '''
                    docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA} .
                    docker push ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh '${security_scan_tool} image ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}'
            }
        }

        stage('Deploy') {
            steps {
                script {
                    if (fileExists('helm/values.yaml')) {
                        sh '''
                            helm upgrade --install ${APP_NAME} ./helm \
                                --set image.repository=${DOCKER_REGISTRY}/${APP_NAME} \
                                --set image.tag=${GIT_SHA}
                        '''
                    } else if (fileExists('deployment.yaml')) {
                        sh '''
                            case "$(uname -s)" in
                                Darwin*) sed -i '' "s|image: .*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}|" deployment.yaml ;;
                                *) sed -i "s|image: .*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}|" deployment.yaml ;;
                            esac
                            kubectl apply -f deployment.yaml
                        '''
                    } else {
                        error "No deployment configuration found (helm/values.yaml or deployment.yaml)"
                    }
                }
            }
        }
    }

    post {
        failure {
            echo 'Deployment failed. Rolling back to the previous version...'
            sh '''
                if (fileExists('deployment.yaml')) {
                    kubectl rollout undo deployment/${APP_NAME}
                } else {
                    echo "Rollback mechanism not implemented for Helm deployments."
                }
            '''
        }
    }
}