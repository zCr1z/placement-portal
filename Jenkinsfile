
pipeline {
    agent any

    options {
        timestamps()
    }

    parameters {
        booleanParam(
            name: 'SIMULATE_FAULT',
            defaultValue: false,
            description: 'Deploy a deliberately missing image and test automatic rollback'
        )
    }

    stages {
        stage('1. Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('2. Build Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASS'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        $image = "$env:DH_USER/placement-portal:$env:BUILD_NUMBER"

                        docker build -t $image -t "$env:DH_USER/placement-portal:latest" .
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
                    '''
                }
            }
        }

        stage('3. Test Application') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASS'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        $image = "$env:DH_USER/placement-portal:$env:BUILD_NUMBER"

                        docker rm -f placement-test 2>$null
                        docker run -d --name placement-test -p 18080:80 $image
                        if ($LASTEXITCODE -ne 0) { exit 1 }

                        try {
                            $ok = $false
                            for ($i = 0; $i -lt 10; $i++) {
                                try {
                                    $r = Invoke-WebRequest `
                                        -Uri http://localhost:18080 `
                                        -UseBasicParsing
                                    if ($r.StatusCode -eq 200 -and
                                        $r.Content -match 'Student Placement Portal') {
                                        $ok = $true
                                        break
                                    }
                                } catch {
                                    Start-Sleep -Seconds 2
                                }
                            }

                            if (-not $ok) {
                                throw "Website smoke test failed"
                            }
                            Write-Host "PASS: Website returned HTTP 200"
                        } finally {
                            docker rm -f placement-test | Out-Null
                        }
                    '''
                }
            }
        }

        stage('4. Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASS'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        $image = "$env:DH_USER/placement-portal:$env:BUILD_NUMBER"

                        $env:DH_PASS | docker login `
                            --username $env:DH_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                        docker push $image
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                        docker push "$env:DH_USER/placement-portal:latest"
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
                    '''
                }
            }
        }

        stage('5. Deploy to Kubernetes') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASS'
                    ),
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"

                        $image = "$env:DH_USER/placement-portal:$env:BUILD_NUMBER"
                        $targetImage = $image

                        if ($env:SIMULATE_FAULT -eq "true") {
                            $targetImage = "$env:DH_USER/placement-portal:missing-$env:BUILD_NUMBER"
                            Write-Host "DEMO: Deploying deliberately missing image"
                        }

                        $manifest = Get-Content deployment.yaml -Raw
                        $manifest = $manifest.Replace(
                            'IMAGE_PLACEHOLDER', $targetImage
                        )
                        Set-Content deployment-rendered.yaml $manifest -Encoding ascii

                        kubectl --kubeconfig="$env:KUBECONFIG" `
                            apply -f deployment-rendered.yaml
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                        try {
                            kubectl --kubeconfig="$env:KUBECONFIG" `
                                rollout status deployment/placement-portal `
                                --timeout=45s

                            if ($LASTEXITCODE -ne 0) {
                                throw "Deployment rollout failed"
                            }

                            Write-Host "PASS: Deployment rollout successful"
                        } catch {
                            Write-Host "FAILED ROLLOUT DETECTED: $_"
                            Write-Host "Automatically rolling back..."

                            kubectl --kubeconfig="$env:KUBECONFIG" `
                                rollout undo deployment/placement-portal
                            if ($LASTEXITCODE -ne 0) {
                                throw "Rollback command failed"
                            }

                            kubectl --kubeconfig="$env:KUBECONFIG" `
                                rollout status deployment/placement-portal `
                                --timeout=120s
                            if ($LASTEXITCODE -ne 0) {
                                throw "Rollback verification failed"
                            }

                            Write-Host "PASS: Previous stable release restored"
                        }

                        kubectl --kubeconfig="$env:KUBECONFIG" `
                            get pods -l app=placement-portal -o wide
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
                    '''
                }
            }
        }

        stage('6. Verify Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    powershell '''
                        kubectl --kubeconfig="$env:KUBECONFIG" `
                            get deployment,service
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

                        kubectl --kubeconfig="$env:KUBECONFIG" `
                            get pods -l app=placement-portal -o wide
                        if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
                    '''
                }
            }
        }
    }
}
