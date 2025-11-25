---
layout: post
comments: false
title:  "Deploying Python and FastAPI app to Azure App Service"
date:   2025-11-25 10:00:00
---

### Project setup

```powershell
New-Item -ItemType Directory -Path 'app_folder'
uv init
.venv\Scripts\activate
uv add <python packages you need>
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
