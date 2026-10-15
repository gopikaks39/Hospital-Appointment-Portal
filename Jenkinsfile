pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'gopikaks/hospital-appointment-portal'
        DEPLOYMENT = 'hospital-appointment'
        SERVICE = 'hospital-appointment-service'
        NODE_PORT = '30013'
        TEST_PORT = '18084'
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}"
                }
                bat 'docker build -t %DOCKER_IMAGE%:%IMAGE_TAG% .'
            }
        }

        stage('Test Application') {
            steps {
                bat '''
                    docker rm -f hospital-test 2>NUL
                    docker run -d --name hospital-test -p %TEST_PORT%:80 %DOCKER_IMAGE%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                '''
                powershell '''
                    Start-Sleep -Seconds 5
                    try {
                        $r = Invoke-WebRequest -Uri "http://localhost:$env:TEST_PORT/" -UseBasicParsing -TimeoutSec 10
                        if ($r.StatusCode -ne 200) { exit 1 }
                        Write-Host "Hospital portal test passed."
                    } catch {
                        docker logs hospital-test
                        exit 1
                    }
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    bat '''
                        docker login -u "%DH_USER%" -p "%DH_TOKEN%"
                        if errorlevel 1 exit /b 1
                        docker push %DOCKER_IMAGE%:%IMAGE_TAG%
                        if errorlevel 1 exit /b 1
                        docker tag %DOCKER_IMAGE%:%IMAGE_TAG% %DOCKER_IMAGE%:latest
                        docker push %DOCKER_IMAGE%:latest
                        if errorlevel 1 exit /b 1
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    script {
                        def previousRevision = bat(
                            script: '@echo off\r\nset "KUBECONFIG=%KUBECONFIG_FILE%"\r\nkubectl rollout history deployment/%DEPLOYMENT% 2>NUL',
                            returnStatus: true
                        )

                        bat '''
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl get deployment %DEPLOYMENT% >NUL 2>&1
                            if errorlevel 1 kubectl apply -f deployment.yaml
                            kubectl rollout history deployment/%DEPLOYMENT%
                            if errorlevel 1 exit /b 1
                            kubectl set image deployment/%DEPLOYMENT% hospital-appointment=%DOCKER_IMAGE%:%IMAGE_TAG%
                            if errorlevel 1 exit /b 1
                        '''

                        try {
                            bat '''
                                @echo off
                                set "KUBECONFIG=%KUBECONFIG_FILE%"
                                kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                                if errorlevel 1 exit /b 1
                            '''
                        } catch (Exception e) {
                            echo 'Rollout failed. Automatically restoring the previous stable revision.'
                            bat '''
                                @echo off
                                set "KUBECONFIG=%KUBECONFIG_FILE%"
                                kubectl rollout undo deployment/%DEPLOYMENT%
                                kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                            '''
                            throw e
                        }
                    }
                }
            }
        }

        stage('Verify Application') {
            steps {
                powershell '''
                    Start-Sleep -Seconds 5
                    try {
                        $r = Invoke-WebRequest -Uri "http://localhost:$env:NODE_PORT/" -UseBasicParsing -TimeoutSec 15
                        if ($r.StatusCode -ne 200) { throw "HTTP health check failed" }
                        Write-Host "Hospital portal is responding."
                    } catch {
                        Write-Host "Service health check failed."
                        exit 1
                    }
                '''
            }
        }
    }

    post {
        always {
            bat 'docker rm -f hospital-test 2>NUL & exit /b 0'
        }
        success {
            echo 'Hospital Appointment Portal deployed successfully.'
            echo 'URL: http://localhost:30013'
        }
        failure {
            echo 'Pipeline failed. Inspect Console Output and Kubernetes rollout status.'
        }
    }
}
