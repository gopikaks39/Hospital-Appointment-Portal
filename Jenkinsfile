pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'gopikaks/hospital-appointment-portal'
        DEPLOYMENT = 'hospital-appointment'
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
                    @echo off
                    docker rm -f hospital-test 2>NUL
                    docker run -d --name hospital-test -p %TEST_PORT%:80 %DOCKER_IMAGE%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                '''

                powershell '''
                    Start-Sleep -Seconds 5
                    try {
                        $r = Invoke-WebRequest -Uri "http://localhost:$env:TEST_PORT/" -UseBasicParsing -TimeoutSec 10
                        if ($r.StatusCode -ne 200) {
                            throw "Application returned an unexpected status."
                        }
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
                        @echo off
                        echo %DH_TOKEN% | docker login -u "%DH_USER%" --password-stdin
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

                        if (!stableImage.contains(DOCKER_IMAGE + ':')) {
                            error("Could not identify a valid stable image. Stopping to protect the deployment.")
                        }

                        // Deploy and verify the newly built release first.
                        bat """
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl set image deployment/%DEPLOYMENT% hospital-appointment=${DOCKER_IMAGE}:${IMAGE_TAG}
                            if errorlevel 1 exit /b 1
                            kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                            if errorlevel 1 exit /b 1
                        """

                        echo 'New release deployed. Starting controlled rollback demonstration.'

                        // Use an invalid image to simulate a failed release.
                        bat """
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl set image deployment/%DEPLOYMENT% hospital-appointment=${DOCKER_IMAGE}:rollback-test-invalid
                            if errorlevel 1 exit /b 1
                        """

                        def failedRollout = bat(
                            script: '@echo off\r\nset "KUBECONFIG=%KUBECONFIG_FILE%"\r\nkubectl rollout status deployment/%DEPLOYMENT% --timeout=60s',
                            returnStatus: true
                        )

                        if (failedRollout == 0) {
                            echo 'The test image unexpectedly rolled out successfully.'
                            echo 'Restoring the stable release anyway.'

                            bat """
                                @echo off
                                set "KUBECONFIG=%KUBECONFIG_FILE%"
                                kubectl set image deployment/%DEPLOYMENT% hospital-appointment=${stableImage}
                                if errorlevel 1 exit /b 1
                                kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                                if errorlevel 1 exit /b 1
                            """

                            error('Rollback test inconclusive: faulty rollout did not fail.')
                        }

                        echo 'Faulty rollout detected. Restoring previous stable image.'

                        // Insert the saved image directly into the Windows command.
                        bat """
                            @echo off
                            set "KUBECONFIG=%KUBECONFIG_FILE%"
                            kubectl set image deployment/%DEPLOYMENT% hospital-appointment=${stableImage}
                            if errorlevel 1 exit /b 1
                            kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                            if errorlevel 1 exit /b 1
                        """

                        echo "Rollback successful. Restored image: ${stableImage}"
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

                    try {
                        $r = Invoke-WebRequest -Uri "http://localhost:$env:NODE_PORT/" -UseBasicParsing -TimeoutSec 15
                        if ($r.StatusCode -ne 200) {
                            throw "Website recovery verification failed."
                        }

                        Write-Host "Website recovery verification passed."
                    } catch {
                        Write-Host "Website verification failed."
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
            echo 'Deployment and automatic rollback demonstration completed successfully.'
            echo 'Hospital portal: http://localhost:30013'
        }

        failure {
            echo 'Pipeline failed. Check Console Output and verify the deployment image.'
        }
    }
}
