pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'localhost:5000'
        APP_NAME = 'myapp'
        Sonar_Url = 'https://localhost'
        Sonar_Token = 'sonarqube'
        security_scan_tool = 'trivy'
        GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim