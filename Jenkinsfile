pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = "localhost:5000"
        APP_NAME = "myapp"
        Sonar_Url = "https://localhost"
        Sonar_Token = "sonarqube"
        security_scan_tool = "trivy"
        GIT_SHA = "${sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()}"
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
                sh 'php build.php'
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh 'phpunit tests'
            }
        }

        stage('Static Code Analysis') {
            steps {
                sh "sonar-scanner -Dsonar.host.url=${Sonar_Url} -Dsonar.login=${Sonar_Token}"
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qualityGate = sh(returnStdout: true, script: "curl -u ${Sonar_Token}: ${Sonar_Url}/api/qualitygates/project_status?projectKey=${APP_NAME}").trim()
                    if (!qualityGate.contains('OK') && !qualityGate.contains('NONE')) {
                        error "Quality Gate failed: ${qualityGate}"
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
                sh "${security_scan_tool} image ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}"
            }
        }

        stage('Deploy') {
            steps {
                script {
                    if (fileExists('helm/values.yaml')) {
                        sh "helm upgrade --install ${APP_NAME} ./helm --set image.repository=${DOCKER_REGISTRY}/${APP_NAME} --set image.tag=${GIT_SHA}"
                    } else if (fileExists('deployment.yaml')) {
                        sh '''
                        case "$(uname -s)" in
                            Darwin*) sed -i '' "s|image: .*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}|" deployment.yaml ;;
                            *) sed -i "s|image: .*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}|" deployment.yaml ;;
                        esac
                        '''
                        sh 'kubectl apply -f deployment.yaml'
                    } else {
                        error "Neither helm/values.yaml nor deployment.yaml found. Deployment cannot proceed."
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "Build failed. Initiating rollback."
            sh '''
            if kubectl rollout status deployment/${APP_NAME} | grep -q "successfully rolled out"; then
                kubectl rollout undo deployment/${APP_NAME}
            else
                echo "No previous deployment to rollback to."
            fi
            '''
        }
    }
}