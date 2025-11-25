---
layout: post
comments: false
title:  "Deploying Python and FastAPI app to Azure App Service"
date:   2025-11-25 10:00:00
---

Most of us work on a Windows system and develop software that we eventually deploy on Linux OS (For eg., Linux based Azure App Service for Python).
While you should almost always use WSL, in the rare unfortunate event otherwise, some handy scripts always help.

In this post I describe how I develop Python/FastAPI backends with LangGraph, LangChain, etc, and deploy to Azure App Service from my local workstation for PoC/MVP demonstrations.
While I follow the same setup, project structure and build and deployment process beyond MVP, but instead of using PowerShell scritps to build and deploy, we use proper CI/CD pipelines in Azure DevOps, that triggers everytime a pull request (or some say merge request) is merged into the dev branch, or the dev branch is merged to master.

### Project setup

```powershell
> New-Item -ItemType Directory -Path 'app_folder'
> uv init
> .venv\Scripts\activate
> uv add <python packages you need>
```

### The project structure I use, mostly looks like this...

```md
📁 app_folder
├── .venv
├── 📁 src
│   ├── 📁 background_jobs
│   ├── 📁 cache_client
│   ├── 📁 db_client
│   ├── 📁 graph
│   ├── 📁 guardrails
│   ├── 📁 llm_client
│   ├── 📁 logger
│   ├── 📁 models
│   ├── 📁 settings
│   ├── 📁 local_tools
│   ├── 📁 utils
│   ├── 📁 vector_search
│   ├── 📁 worker_agents
│   ├── 📁 mcp_client
│   ├── __init__.py
│   └── app.py
├── 📁 tests
│   ├── __init__.py
│   ├── test_this.py
│   └── test_that.py
├── 📁 migrations
│   ├── V1__init_schema.sql
│   └── V1.1__add_columns.sql
├── 📁 win_scripts
│   ├── build.ps1
│   └── deploy.ps1
├── .env
├── .gitignore
├── .pythonversion
├── pyproject.toml
├── README.md
└── uv.lock
```

> all folders are modules, containing `__init__.py` inside.

### The build script (inside win_scripts folder)
This script creates a folder called `deployment` consolidating all the artifacts required for deployment into the App Service.

```powershell
$Folder = '.\deployment'

Write-Host "Preparing build"
Write-Host -ForegroundColor red "This will delete everything in the"$Folder" folder"

$Response = Read-Host "Press y to continue, any other key to abort."
$Should_Abort = $Response -ne "y"

if ($Should_Abort) {
    Write-Host "Aborted."
    exit
}

Write-Host "Deleting existing"$Folder" folder"

if (Test-Path -Path $Folder) {
    Remove-Item -Path $Folder -Recurse -Force
}

Write-Host "Creating artifacts..."

New-Item -ItemType Directory -Force -Path $Folder

Copy-item -Force -Recurse '.\src' -Destination $Folder

Copy-item -Force -Recurse '.\fonts' -Destination $Folder

Copy-item -Force '.\pyproject.toml' -Destination $Folder

Copy-item -Force '.\uv.lock' -Destination $Folder

$Start_Up_Script = @"
python3 -m uvicorn src.app:app --host 0.0.0.0
"@

$Start_Up_Script | Out-File -FilePath $Folder'\startup.sh'

$Deploy_Config = @"
[config]
SCM_DO_BUILD_DURING_DEPLOYMENT=true
"@

$Deploy_Config | Out-File -FilePath $Folder'\.deployment'

Write-Host "Done."
```

### The deployment script (inside win_scripts folder)
This script creates a zip with the contents of the `deployment` folder, and pushes it into the App Service - Replicating CI/CD.

```powershell
$Folder = '.\deployment'

Write-Host "Beginning deployment"
if (!(Test-Path -Path $Folder)) {
    Write-Host "No folder"
} else {
    Write-Host "Yes folder"
}

Write-Host "Creating zip"
Compress-Archive -Force -Path $Folder/* -DestinationPath $Folder"\build.zip"

$Resource_Group = '<your rg name>'
$App_Service = '<your app service name>'

Write-Host -ForegroundColor yellow "Beginning deployment into resource grp ["$Resource_Group"]; app service ["$App_Service"]"
Write-Host -ForegroundColor red "This will delete the existing app"

$Response = Read-Host "Press y to continue (you will select subscription ID later), any other key to abort."
$Should_Abort = $Response -ne "y"

if ($Should_Abort) {
    Write-Host "Aborted."
    exit
}

az login
az webapp deploy --resource-group $Resource_Group --name $App_Service --src-path $Folder"\build.zip" --type "zip" --async true

Write-Host "Done."
Write-Host "Check deployment logs: https://<your app service name>.scm.azurewebsites.net/api/deployments/latest"
```

