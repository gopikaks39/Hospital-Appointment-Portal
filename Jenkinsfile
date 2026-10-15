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
                        Write-Host "Application test passed."
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

        stage('Deploy and Auto Rollback') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    script {
                        bat '''
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl get deployment %DEPLOYMENT%
                            if errorlevel 1 exit /b 1
                        '''

                        def previousImage = bat(
                            script: '@echo off\r\nset "KUBECONFIG=%KUBECONFIG_FILE%"\r\nkubectl get deployment %DEPLOYMENT% -o jsonpath="{.spec.template.spec.containers[0].image}"',
                            returnStdout: true
                        ).trim()

                        echo "Previous stable image: ${previousImage}"

                        bat '''
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
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

                            powershell '''
                                Start-Sleep -Seconds 5
                                $r = Invoke-WebRequest -Uri "http://localhost:$env:NODE_PORT/" -UseBasicParsing -TimeoutSec 15
                                if ($r.StatusCode -ne 200) {
                                    throw "Application verification failed"
                                }
                            '''
                        } catch (Exception e) {
                            echo 'FAILED RELEASE DETECTED. Restoring previous stable image.'

                            withEnv(["STABLE_IMAGE=${previousImage}"]) {
                                bat '''
                                    @echo off
                                    set "KUBECONFIG=%KUBECONFIG_FILE%"
                                    kubectl set image deployment/%DEPLOYMENT% hospital-appointment=%STABLE_IMAGE%
                                    if errorlevel 1 exit /b 1
                                    kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                                    if errorlevel 1 exit /b 1
                                '''
                            }

                            echo 'Previous stable image restored.'
                            throw e
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f hospital-test 2>NUL & exit /b 0'
        }
        success {
            echo 'Hospital portal deployed successfully.'
            echo 'URL: http://localhost:30013'
        }
        failure {
            echo 'Pipeline failed. Check Console Output and rollback status.'
        }
    }
}
