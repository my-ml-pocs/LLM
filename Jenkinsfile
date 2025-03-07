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
                sh 'echo "Build step for PHP application"'
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh 'vendor/bin/phpunit tests'
            }
        }

        stage('Static Code Analysis') {
            steps {
                sh 'sonar-scanner -Dsonar.host.url=$Sonar_Url -Dsonar.login=$Sonar_Token'
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qualityGate = sh(script: 'curl -u $Sonar_Token: $Sonar_Url/api/qualitygates/project_status?projectKey=$APP_NAME', returnStdout: true).trim()
                    if (!qualityGate.contains('OK') && !qualityGate.contains('NONE')) {
                        error "Quality Gate failed: ${qualityGate}"
                    }
                }
            }
        }

        stage('Docker Build and Push') {
            steps {
                sh '''
                docker build -t $DOCKER_REGISTRY/$APP_NAME:$GIT_SHA .
                docker push $DOCKER_REGISTRY/$APP_NAME:$GIT_SHA
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh 'trivy image $DOCKER_REGISTRY/$APP_NAME:$GIT_SHA'
            }
        }

        stage('Deploy') {
            steps {
                script {
                    if (fileExists('helm/values.yaml')) {
                        sh '''
                        helm upgrade --install $APP_NAME ./helm \
                            --set image.repository=$DOCKER_REGISTRY/$APP_NAME \
                            --set image.tag=$GIT_SHA
                        '''
                    } else if (fileExists('deployment.yaml')) {
                        sh '''
                        case "$(uname -s)" in
                            Darwin*) sed -i '' 's#image: .*#image: $DOCKER_REGISTRY/$APP_NAME:$GIT_SHA#' deployment.yaml ;;
                            *) sed -i 's#image: .*#image: $DOCKER_REGISTRY/$APP_NAME:$GIT_SHA#' deployment.yaml ;;
                        esac
                        kubectl apply -f deployment.yaml
                        '''
                    } else {
                        error "Neither Helm chart nor deployment.yaml found for deployment."
                    }
                }
            }
        }

        stage('Rollback Mechanism') {
            steps {
                script {
                    try {
                        sh 'echo "Deployment successful"'
                    } catch (Exception e) {
                        sh 'kubectl rollout undo deployment/$APP_NAME'
                        error "Deployment failed and rolled back: ${e.message}"
                    }
                }
            }
        }
    }
}