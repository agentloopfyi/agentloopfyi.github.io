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

#### Project setup

```powershell
> New-Item -ItemType Directory -Path 'app_folder'
> uv python install 3.11
> uv python pin 3.11
> uv init
> .venv\Scripts\activate
> uv add <python packages you need>
```

#### The project structure I use, mostly looks like this...

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

> All folders are modules, containing `__init__.py` inside.

Locally we can use a `.env` file and list env-vars there. Using the `python-dotenv` package along with `pydantic` provides a clever way to intialize a `Settings` class, that contains all env-vars as class member variables. So instead of doing `os.getenv(...)` all over the place we can import the settings.

The contents of the `settings\settings.py` file looks like below:
(This example is from a project where I used, OpenAI models for LLM and embedding, Postgres, Redis, etc.)

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    APP_ENV: str = 'LOCAL'
    DEBUG: bool = False
    OPENAI_API_KEY: str
    OPENAI_API_VERSION: str
    OPENAI_DEPLOYMENT: str
    AZURE_OPENAI_ENDPOINT: str 
    AZURE_SUBSCRIPTION: str
    AZURE_BEARER_TOKEN_PROVIDER_ENDPOINT: str
    USE_REMOTE_MCP: str
    COMMON_TOOLS_MCP_URI: str
    REDIS_URI: str
    REDIS_PORT: str
    REDIS_PASSWORD: str
    PGVECTOR_URI: str
    EMBEDDING_MODEL: str 
    EMBEDDING_API_VERSION: str
    USE_DEEP_RESEARCH: str
    AZURE_CLIENT_ID: str

    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

# Create an instance of your settings
settings = Settings()

if settings.DEBUG:
    print(f"""
    Showing env settings for debugging purposes.
    Please remove this in production by setting DEBUG=False.

    APP_ENV: {settings.APP_ENV}
    DEBUG: {settings.DEBUG}

    OPENAI_API_KEY: {settings.OPENAI_API_KEY}    
    OPENAI_API_VERSION: {settings.OPENAI_API_VERSION}
    OPENAI_DEPLOYMENT: {settings.OPENAI_DEPLOYMENT}
    AZURE_OPENAI_ENDPOINT: {settings.AZURE_OPENAI_ENDPOINT}

    AZURE_SUBSCRIPTION: {settings.AZURE_SUBSCRIPTION}
    AZURE_BEARER_TOKEN_PROVIDER_ENDPOINT: {settings.AZURE_BEARER_TOKEN_PROVIDER_ENDPOINT}

    USE_REMOTE_MCP: {settings.USE_REMOTE_MCP}
    COMMON_TOOLS_MCP_URI: {settings.COMMON_TOOLS_MCP_URI}

    REDIS_URI: {settings.REDIS_URI}
    REDIS_PORT: {settings.REDIS_PORT}
    REDIS_PASSWORD: {settings.REDIS_PASSWORD}

    PGVECTOR_URI: {settings.PGVECTOR_URI}

    EMBEDDING_MODEL: {settings.EMBEDDING_MODEL}
    EMBEDDING_API_VERSION: {settings.EMBEDDING_API_VERSION}
    
    USE_DEEP_RESEARCH: {settings.USE_DEEP_RESEARCH}

    AZURE_CLIENT_ID: {settings.AZURE_CLIENT_ID} # Client ID of your managed identity
    """)
```

The `settings\settings.py` contains...

```python
from .settings import settings
```

..to exports the settings.

In case you are interested about managed identities and keyless authentication with Azure OpenAI, check my <a href="/2025/05/06/keyless-auth-openai/">other post</a>.

#### The build script (inside win_scripts folder)
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

#### The deployment script (inside win_scripts folder)
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

The script will deploy the Python app and poll the App Service's start up status. The script will exit once the App Service has started.

#### However, the App Service might not start and the script might terminate with error in the following events:

❌ One or more env-var required for the app to start, is/are missing. You need to add the env-var in the App Service's settings

❌ During start up, the app is creating a connection to Redis or Postgres or any other service, and the service is not reachable. You need to check the log stream to identify why it is failing. The services must be reachable from the App Service. Check if they should be added to a virtual network, or IP whitelising is required

❌ For Python apps, `requirements.txt` or `uv.lock` or similar files must be at the root of the project folder that was zipped and uploaded. Otherwise the App service will not be able to detect it's a Python app, and everything after, like `pip install` that the App Service internally does, will fail

❌ You will also need to configure the startup script in the App Service. Go to Settings -> Configuration -> Stack Setting -> Startup command: Write `python3 -m uvicorn src.app:app --host 0.0.0.0` and save changes. Note, this is the same content that we have in the `startup.sh` script that gets generated by running the `win_scripts\build.ps1` script
