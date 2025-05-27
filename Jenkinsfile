pipeline {
    agent any

    environment {
        VM_HOST = '20.2.8.241'
        VM_DEPLOY_DIR = 'C:\\inetpub\\wwwroot\\MyDotnetApp'
        APP_POOL = 'DefaultAppPool'
        IIS_SITE = 'MyDotnetApp'
        PUBLISH_DIR = "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\jenkins_publish"
    }

    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'GithubAccess', url: 'https://github.com/mdsajjad217/JenkinsCiCdApp.git'
            }
        }

        stage('Restore') {
            steps {
                bat 'dotnet restore JenkinsCiCdApp.csproj'
            }
        }

        stage('Build') {
            steps {
                bat 'dotnet build JenkinsCiCdApp.csproj --configuration Release'
            }
        }

        stage('Publish') {
			steps {
				bat '''
					IF EXIST "%PUBLISH_DIR%" RMDIR /S /Q "%PUBLISH_DIR%"
					dotnet publish JenkinsCiCdApp.csproj --configuration Release --output "%PUBLISH_DIR%"
				'''
			}
		}

        stage('Deploy to Azure VM via WinRM') {
			steps {
				withCredentials([usernamePassword(credentialsId: 'azure-vm-creds', usernameVariable: 'VM_USER', passwordVariable: 'VM_PASS')]) {
					powershell '''
						$ErrorActionPreference = "Stop"

						$username = "$env:VM_USER"
						$password = ConvertTo-SecureString "$env:VM_PASS" -AsPlainText -Force
						$cred = New-Object System.Management.Automation.PSCredential ($username, $password)
						$session = New-PSSession -ComputerName $env:VM_HOST -Credential $cred

						Invoke-Command -Session $session -ScriptBlock {
							param($siteName, $appPool, $deployDir)

							Import-Module WebAdministration

							if (!(Test-Path "IIS:\\AppPools\\$appPool")) {
								New-WebAppPool -Name $appPool
							}

							if (!(Test-Path $deployDir)) {
								New-Item -ItemType Directory -Path $deployDir -Force
							}

							if (!(Get-Website -Name $siteName -ErrorAction SilentlyContinue)) {
								New-Website -Name $siteName -Port 5001 -PhysicalPath $deployDir -ApplicationPool $appPool
							}

							if ((Get-Website -Name $siteName).State -eq "Started") {
								Stop-Website -Name $siteName
							}
						} -ArgumentList "$env:IIS_SITE", "$env:APP_POOL", "$env:VM_DEPLOY_DIR"

						Copy-Item -ToSession $session -Path "$env:PUBLISH_DIR\\*" -Destination "$env:VM_DEPLOY_DIR" -Recurse -Force

						Invoke-Command -Session $session -ScriptBlock {
							param($siteName)
							Start-Website -Name $siteName
						} -ArgumentList "$env:IIS_SITE"

						Remove-PSSession $session
						Write-Host "✅ Deployment completed successfully."
					'''
				}
			}
}
    }

    post {
        always {
            echo "✅ Pipeline execution completed."
        }
    }
}
