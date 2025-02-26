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
                sh 'vendor/bin/phpunit tests'
            }
        }
        stage('Static Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'sonar-scanner -Dsonar.projectKey=$APP_NAME -Dsonar.sources=.'
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Docker Build and Push') {
            steps {
                script {
                    sh '''
                    docker build -t $DOCKER_REGISTRY/$APP_NAME:$GIT_SHA .
                    docker push $DOCKER_REGISTRY/$APP_NAME:$GIT_SHA
                    '''
                }
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
                        helm upgrade --install $APP_NAME ./helm --set image.repository=$DOCKER_REGISTRY/$APP_NAME --set image.tag=$GIT_SHA
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
                        error 'No deployment file found!'
                    }
                }
            }
        }
    }
    post {
        failure {
            script {
                echo 'Deployment failed, initiating rollback...'
                // Add rollback steps here
            }
        }
    }
}