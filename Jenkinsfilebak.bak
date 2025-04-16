pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = "localhost:5000"
        APP_NAME = "myapp"
        Sonar_Url = "https://localhost"
        Sonar_Token = "sonarqube"
        security_scan_tool = "trivy"
        GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }
         stage('Setup PHP with Xdebug') {
            steps {
                sh 'sudo add-apt-repository ppa:ondrej/php -y'
                sh 'sudo apt-get update'
                sh 'sudo apt-get install -y php8.1 php8.1-cli php8.1-xml php8.1-xdebug'
                sh 'php -v'
            }
        }

        stage('Install dependencies with Composer') {
            steps {
                sh 'composer update --no-ansi --no-interaction --no-progress'
            }
        }

        stage('Run tests with PHPUnit') {
            steps {
                sh 'vendor/bin/phpunit --coverage-clover=coverage.xml'
            }
        }

        stage('Static Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'sonar-scanner'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK' && qg.status != 'NONE') {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
                }
            }
        }

        stage('Docker Build and Push') {
            steps {
                script {
                    sh """
                    docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA} .
                    docker push ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}
                    """
                }
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
                         sh '''
                         helm upgrade --install ${APP_NAME} ./helm --set image.repository=${DOCKER_REGISTRY}/${APP_NAME} --set image.tag=${GIT_SHA}
                         '''
                     } else if (fileExists('deployment.yaml')) {
                         sh '''
                         case "$(uname -s)" in
                             Darwin)
                                 sed -i '' 's#image: .*#image: ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}#' deployment.yaml
                                 ;;
                             *)
                                 sed -i 's#image: .*#image: ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_SHA}#' deployment.yaml
                                 ;;
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
            script {
                // Rollback mechanism
                sh """
                kubectl rollout undo deployment/${APP_NAME} -n default
                """
            }
        }
    }
}
