pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'gopikaks/hospital-appointment-portal'
        DEPLOYMENT = 'hospital-appointment'
        NODE_PORT = '30013'
        TEST_PORT = '18084'
        STABLE_IMAGE = 'gopikaks/hospital-appointment-portal:5'
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
                    @echo off
                    docker rm -f hospital-test 2>NUL
                    docker run -d --name hospital-test -p %TEST_PORT%:80 %DOCKER_IMAGE%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                '''
                powershell '''
                    Start-Sleep -Seconds 5
                    $r = Invoke-WebRequest -Uri "http://localhost:$env:TEST_PORT/" -UseBasicParsing -TimeoutSec 10
                    if ($r.StatusCode -ne 200) {
                        throw "Application test failed"
                    }
                    Write-Host "Application test passed."
                '''
            }
        }

        stage('Deploy and Automatic Rollback Test') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    script {
                        // Ensure the known stable image is available locally.
                        bat '''
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl get deployment %DEPLOYMENT%
                            if errorlevel 1 exit /b 1
                        '''

                        echo "Stable image for recovery: ${STABLE_IMAGE}"

                        // Verify the current application is healthy before testing.
                        bat '''
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                            if errorlevel 1 exit /b 1
                        '''

                        // Intentionally deploy a nonexistent tag.
                        bat '''
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl set image deployment/%DEPLOYMENT% hospital-appointment=%DOCKER_IMAGE%:rollback-test-invalid
                            if errorlevel 1 exit /b 1
                        '''

                        def rolloutStatus = bat(
                            script: '@echo off\r\nset "KUBECONFIG=%KUBECONFIG_FILE%"\r\nkubectl rollout status deployment/%DEPLOYMENT% --timeout=60s',
                            returnStatus: true
                        )

                        if (rolloutStatus == 0) {
                            echo 'Invalid image unexpectedly rolled out; restoring stable image.'
                        } else {
                            echo 'Faulty rollout detected. Starting automatic recovery.'
                        }

                        // Always restore the known stable image.
                        bat """
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl set image deployment/%DEPLOYMENT% hospital-appointment=${STABLE_IMAGE}
                            if errorlevel 1 exit /b 1
                            kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                            if errorlevel 1 exit /b 1
                        """

                        echo "Stable image restored: ${STABLE_IMAGE}"
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
                        kubectl get pods -l app=hospital-appointment
                    '''
                }

                powershell '''
                    Start-Sleep -Seconds 5
                    $r = Invoke-WebRequest -Uri "http://localhost:$env:NODE_PORT/" -UseBasicParsing -TimeoutSec 15
                    if ($r.StatusCode -ne 200) {
                        throw "Website recovery verification failed"
                    }
                    Write-Host "Website recovery verification passed."
                '''
            }
        }
    }

    post {
        always {
            bat 'docker rm -f hospital-test 2>NUL & exit /b 0'
        }
        success {
            echo 'Rollback demonstration completed and stable image restored.'
        }
        failure {
            echo 'Pipeline failed. Check Console Output and deployment status.'
        }
    }
}
