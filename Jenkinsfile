pipeline {
    environment {
        DOCKER_REGISTRY = "localhost:5000"
        APP_NAME = "myapp"
        Sonar_Url = "https://localhost"
        Sonar_Token = "sonarqube"
        security_scan_tool = "trivy"
        GIT_SHA = "${env.GIT_COMMIT[0..6]}"
    }
    agent any

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'apt-get update && apt-get install -y php-cli php-mbstring unzip'
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA} .'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'echo "Running Unit Tests..."'
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
                    def qualityGate = sh(script: 'curl -s ${Sonar_Url}/api/qualitygates/project_status?projectKey=${APP_NAME}', returnStdout: true)
                    if (qualityGate.contains('ERROR')) {
                        error("Quality Gate failed")
                    }
                }
            }
        }

        stage('Docker Push') {
            steps {
                sh 'docker push ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}'
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
                        sh 'helm upgrade --install ${APP_NAME} ./helm --set image.repository=${DOCKER_REGISTRY}/${APP_NAME} --set image.tag=${GIT_SHA}'
                    } else if (fileExists('deployment.yaml')) {
                        sh '''
                        case "$(uname -s)" in
                            Darwin*) sed -i "" "s|image: .*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}|" deployment.yaml ;;
                            *) sed -i "s|image: .*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}|" deployment.yaml ;;
                        esac
                        '''
                        sh 'kubectl apply -f deployment.yaml'
                    } else {
                        error("No deployment configuration found")
                    }
                }
            }
        }

        stage('Rollback') {
            steps {
                script {
                    try {
                        sh 'kubectl rollout undo deployment/${APP_NAME}'
                    } catch (Exception e) {
                        echo "Rollback failed: ${e.message}"
                    }
                }
            }
        }
    }
}