pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'localhost:5000'
        APP_NAME = 'myapp'
        Sonar_Url = 'https://localhost'
        Sonar_Token = 'sonarqube'
        security_scan_tool = 'trivy'
        GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }

    stages {
        stage('User Confirmation') {
            steps {
                script {
                    input message: 'Do you want to proceed with the pipeline execution?'
                }
            }
        }
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
                sh 'composer build'
            }
        }
        stage('Run Unit Tests') {
            steps {
                sh 'composer test'
            }
        }
        stage('Static Code Analysis') {
            steps {
                sh "sonar-scanner -Dsonar.projectKey=${APP