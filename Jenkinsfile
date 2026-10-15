pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'gopikaks/hospital-appointment-portal'
        DEPLOYMENT = 'hospital-appointment'
        NODE_PORT = '30013'
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
                    docker run -d --name hospital-test -p 18084:80 %DOCKER_IMAGE%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                '''
                powershell '''
                    Start-Sleep -Seconds 5
                    $r = Invoke-WebRequest -Uri "http://localhost:18084/" -UseBasicParsing -TimeoutSec 10
                    if ($r.StatusCode -ne 200) { exit 1 }
                    Write-Host "Application test passed."
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
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy and Automatic Rollback Test') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    script {
                        def stableImage = bat(
                            script: '@echo off\r\nset "KUBECONFIG=%KUBECONFIG_FILE%"\r\nkubectl get deployment %DEPLOYMENT% -o jsonpath="{.spec.template.spec.containers[0].image}"',
                            returnStdout: true
                        ).trim()

                        echo "Stable image before deployment: ${stableImage}"

                        // Deploy the real image built by this Jenkins run.
                        bat '''
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl set image deployment/%DEPLOYMENT% hospital-appointment=%DOCKER_IMAGE%:%IMAGE_TAG%
                            if errorlevel 1 exit /b 1
                            kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                            if errorlevel 1 exit /b 1
                        '''

                        echo 'Stable release deployed. Starting controlled rollback demonstration.'

                        // Apply an intentionally invalid image to trigger a failed rollout.
                        bat '''
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl set image deployment/%DEPLOYMENT% hospital-appointment=%DOCKER_IMAGE%:rollback-test-invalid
                            if errorlevel 1 exit /b 1
                        '''

                        def failedRollout = bat(
                            script: '@echo off\r\nset "KUBECONFIG=%KUBECONFIG_FILE%"\r\nkubectl rollout status deployment/%DEPLOYMENT% --timeout=60s',
                            returnStatus: true
                        )

                        if (failedRollout == 0) {
                            error('The deliberately faulty rollout unexpectedly succeeded; rollback test cannot be demonstrated.')
                        }

                        echo 'Faulty rollout detected. Automatically restoring the stable image.'

                        bat '''
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl set image deployment/%DEPLOYMENT% hospital-appointment=%STABLE_IMAGE%
                            if errorlevel 1 exit /b 1
                            kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                            if errorlevel 1 exit /b 1
                        '''

                        echo 'Rollback successful. Previous stable image restored.'
                    }
                }
            }
        }

        stage('Verify Recovery') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    bat '''
                        @echo off
                        set "KUBECONFIG=%KUBECONFIG_FILE%"
                        kubectl get deployment %DEPLOYMENT%
                        if errorlevel 1 exit /b 1
                        kubectl get pods -l app=hospital-appointment
                        if errorlevel 1 exit /b 1
                    '''
                }
                powershell '''
                    Start-Sleep -Seconds 5
                    $r = Invoke-WebRequest -Uri "http://localhost:$env:NODE_PORT/" -UseBasicParsing -TimeoutSec 15
                    if ($r.StatusCode -ne 200) {
                        throw "Website recovery verification failed."
                    }
                    Write-Host "Website recovered successfully."
                '''
            }
        }
    }

    post {
        always {
            bat 'docker rm -f hospital-test 2>NUL & exit /b 0'
        }
        success {
            echo 'Deployment and automatic rollback demonstration completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Inspect Console Output and deployment status.'
        }
    }
}
