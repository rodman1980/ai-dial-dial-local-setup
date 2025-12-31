---
title: DIAL Local Setup - Setup Guide
description: Complete installation, configuration, and troubleshooting guide for DIAL local development environment
version: 1.0.0
last_updated: 2025-12-30
related: [README.md, architecture.md, roadmap.md]
tags: [setup, installation, docker, configuration, troubleshooting]
---

# Setup Guide

## Table of Contents
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Verification](#verification)
- [Common Tasks](#common-tasks)
- [Troubleshooting](#troubleshooting)
- [Platform-Specific Notes](#platform-specific-notes)
- [Uninstallation](#uninstallation)

## Prerequisites

### System Requirements

**Minimum:**
- CPU: 2.7 GHz dual-core
- RAM: 8 GB
- Disk: 5 GB free space
- OS: macOS 10.15+, Windows 10+, or Linux (Ubuntu 20.04+)

**Recommended:**
- CPU: 3.0 GHz quad-core
- RAM: 16 GB
- Disk: 10 GB free space
- SSD for better Docker performance

### Required Software

#### 1. Docker & Docker Compose

**macOS:**
```bash
# Install Docker Desktop
# Download from: https://www.docker.com/products/docker-desktop

# Verify installation
docker --version
docker compose version
```

**Windows:**
```powershell
# Install Docker Desktop with WSL2 backend
# Download from: https://www.docker.com/products/docker-desktop

# Verify installation
docker --version
docker compose version
```

**Linux (Ubuntu/Debian):**
```bash
# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

**Version Requirements:**
- Docker: 20.10.0+
- Docker Compose: 2.0.0+

#### 2. Python 3.11+

**macOS (Homebrew):**
```bash
brew install python@3.11
python3.11 --version
```

**Windows:**
```powershell
# Download from: https://www.python.org/downloads/
# Or use winget:
winget install Python.Python.3.11

# Verify
python --version
```

**Linux:**
```bash
sudo apt-get update
sudo apt-get install -y python3.11 python3.11-venv python3-pip
python3.11 --version
```

#### 3. Git (Optional, for cloning)

```bash
# macOS
brew install git

# Windows
winget install Git.Git

# Linux
sudo apt-get install -y git
```

### Optional Tools

- **curl** - For API testing
- **jq** - For JSON processing
- **VS Code** - Recommended IDE with Python extension

## Installation

### Step 1: Get the Project

**Option A: Clone with Git**
```bash
git clone https://github.com/epam/ai-dial-dial-local-setup.git
cd ai-dial-dial-local-setup
```

**Option B: Download ZIP**
```bash
# Download and extract manually
cd ai-dial-dial-local-setup
```

### Step 2: Start Core Services

```bash
# From project root
docker compose up -d
```

**Expected Output:**
```
[+] Running 5/5
 ✔ Network ai-dial-dial-local-setup_default  Created
 ✔ Container redis                           Started
 ✔ Container themes                          Started
 ✔ Container core                            Started
 ✔ Container chat                            Started
```

### Step 3: Verify Services

```bash
docker compose ps -a
```

**Expected Output:**
```
NAME      IMAGE                                  STATUS
chat      epam/ai-dial-chat:development          Up (healthy)
core      epam/ai-dial-core:development          Up (healthy)
redis     redis:7.2.4-alpine3.19                 Up (healthy)
themes    epam/ai-dial-chat-themes:development   Up (healthy)
```

**All services should show "Up" status.**

### Step 4: Access DIAL Chat

Open browser: [http://localhost:3000](http://localhost:3000)

**Expected:** DIAL Chat interface loads with marketplace view.

**Note:** You won't see any models or applications yet - this is normal! Continue to configuration.

## Configuration

### Basic Configuration

#### Minimal Working Config

Edit `core/config.json`:

```json
{
  "routes": {},
  "applications": {},
  "models": {},
  "keys": {
    "dial_api_key": {
      "project": "TEST-PROJECT",
      "role": "default"
    }
  },
  "roles": {
    "default": {
      "limits": {}
    }
  }
}
```

This creates an API key `dial_api_key` with no rate limits.

### Adding Your First Application

#### Create Echo Application

```bash
# Navigate to task 3 echo example
cd tasks/t3/echo

# Create virtual environment
python3.11 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**Contents of requirements.txt:**
```
aidial-sdk==0.27.0
```

**Run the application:**
```bash
python app.py
```

**Expected output:**
```
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:5022
```

#### Register in Core Config

Edit `core/config.json`, add to `applications` section:

```json
{
  "applications": {
    "echo": {
      "displayName": "My Echo App",
      "description": "Simple application that repeats user's message",
      "endpoint": "http://host.docker.internal:5022/openai/deployments/echo/chat/completions"
    }
  }
}
```

**Key Point:** Use `host.docker.internal` for locally-running apps.

#### Restart Core

```bash
# From project root
docker compose stop core
docker compose up -d core
```

Or use quick restart:
```bash
docker compose stop && docker compose up -d --build
```

#### Test Echo App

1. Open [http://localhost:3000/marketplace](http://localhost:3000/marketplace)
2. Find "My Echo App"
3. Start conversation
4. Type: `Hello, Echo!`
5. Expected response: `Hello, Echo!`

### Adding Models

#### Add GPT-4o Model

Edit `core/config.json`, add to `models` section:

```json
{
  "models": {
    "gpt-4o": {
      "displayName": "GPT-4o",
      "endpoint": "http://adapter-dial:5000/openai/deployments/gpt-4o/chat/completions",
      "iconUrl": "http://localhost:3001/gpt4.svg",
      "type": "chat",
      "upstreams": [
        {
          "endpoint": "https://ai-proxy.lab.epam.com/openai/deployments/gpt-4o/chat/completions",
          "key": "YOUR_API_KEY_HERE"
        }
      ]
    }
  }
}
```

**⚠️ Replace `YOUR_API_KEY_HERE` with actual API key.**

#### Add DIAL Adapter

Edit `docker-compose.yml`, add service:

```yaml
services:
  # ... existing services ...
  
  adapter-dial:
    image: epam/ai-dial-adapter-dial:development
    # Uncomment for macOS ARM (M1/M2/M3):
    # platform: linux/amd64
    environment:
      DIAL_URL: "http://core:8080"
      LOG_LEVEL: "INFO"
```

**Restart services:**
```bash
docker compose stop && docker compose up -d --build
```

**Verify adapter is running:**
```bash
docker compose ps -a | grep adapter-dial
```

#### Test Model

1. Open [http://localhost:3000](http://localhost:3000)
2. Select "GPT-4o" from dropdown
3. Type: `Hi, what can you do?`
4. Expected: Response from GPT-4o

### Securing API Keys

#### Create keys.json (Recommended)

```bash
# Create gitignored secrets file
touch core/keys.json
```

**Split configuration:**

**core/config.json** (public, tracked in Git):
```json
{
  "models": {
    "gpt-4o": {
      "displayName": "GPT-4o",
      "endpoint": "http://adapter-dial:5000/openai/deployments/gpt-4o/chat/completions",
      "type": "chat"
      // NO upstreams here
    }
  }
}
```

**core/keys.json** (secret, gitignored):
```json
{
  "models": {
    "gpt-4o": {
      "upstreams": [
        {
          "endpoint": "https://ai-proxy.lab.epam.com/openai/deployments/gpt-4o/chat/completions",
          "key": "YOUR_API_KEY"
        }
      ]
    }
  }
}
```

Core automatically merges both files at startup.

**Verify .gitignore:**
```bash
cat .gitignore
# Should contain: core/keys.json
```

## Verification

### Health Checks

**Check all services:**
```bash
docker compose ps -a
```

All should show "Up" status.

**Check Core health:**
```bash
curl http://localhost:8080/health
```

Expected: `200 OK` or health status JSON.

**Check Chat health:**
```bash
curl http://localhost:3000/api/health
```

**Check Redis:**
```bash
docker compose exec redis redis-cli PING
```

Expected: `PONG`

### Test API Directly

**Test with curl:**
```bash
curl -X POST http://localhost:8080/openai/deployments/echo/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dial_api_key" \
  -d '{
    "messages": [
      {"role": "user", "content": "Test message"}
    ],
    "stream": false
  }'
```

**Expected response:**
```json
{
  "id": "chatcmpl-...",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Test message"
      }
    }
  ]
}
```

### Check Logs

**Core logs:**
```bash
docker compose logs -f core
```

**Application logs:**
```bash
# If running locally
tail -f logs/app.log  # Or check console output
```

**Check for errors:**
```bash
docker compose logs core | grep ERROR
```

## Common Tasks

### Start/Stop Services

**Start all:**
```bash
docker compose up -d
```

**Stop all:**
```bash
docker compose stop
```

**Stop and remove:**
```bash
docker compose down
```

**Remove with volumes (full cleanup):**
```bash
docker compose down -v --remove-orphans
```

### Restart After Config Changes

```bash
docker compose stop core
docker compose up -d core
```

Or full restart:
```bash
docker compose stop && docker compose up -d --build
```

### Update Docker Images

```bash
docker compose pull
docker compose up -d --force-recreate
```

### View Logs

**All services:**
```bash
docker compose logs -f
```

**Specific service:**
```bash
docker compose logs -f core
docker compose logs -f chat
```

**Last 100 lines:**
```bash
docker compose logs --tail=100 core
```

### Clean Up Disk Space

**Remove unused images:**
```bash
docker image prune -a
```

**Remove unused volumes:**
```bash
docker volume prune
```

**Full cleanup:**
```bash
docker system prune -a --volumes
```

### Port Management

**Check port usage:**
```bash
# macOS/Linux
lsof -i :3000
lsof -i :8080

# Windows
netstat -ano | findstr :3000
```

**Kill process on port:**
```bash
# macOS/Linux
kill -9 $(lsof -t -i:3000)

# Windows
taskkill /PID <PID> /F
```

## Troubleshooting

### Issue: Containers Won't Start

**Symptom:** `docker compose up` fails or containers immediately exit.

**Solution 1: Check Docker is running**
```bash
docker ps
# If error: Start Docker Desktop
```

**Solution 2: Check logs**
```bash
docker compose logs core
docker compose logs chat
```

**Solution 3: Port conflicts**
```bash
# Check if ports are in use
lsof -i :3000
lsof -i :8080
lsof -i :6379

# Change ports in docker-compose.yml
```

**Solution 4: Platform issues (macOS ARM)**
```yaml
# Uncomment in docker-compose.yml for each service:
platform: linux/amd64
```

### Issue: Chat UI Shows "Not available agent selected"

**Symptom:** Chat loads but can't select models/apps.

**Cause:** No models or applications configured.

**Solution:**
1. Check `core/config.json` has models or applications
2. Restart Core: `docker compose stop core && docker compose up -d core`
3. Verify config syntax is valid JSON

### Issue: "Connection refused" from Local App

**Symptom:** Core can't reach locally-running application.

**Cause:** Wrong endpoint configuration.

**Solution:**
```json
// ❌ Wrong
"endpoint": "http://localhost:5022/..."

// ✅ Correct
"endpoint": "http://host.docker.internal:5022/..."
```

**Linux-specific:**
```bash
# Add to docker-compose.yml core service:
extra_hosts:
  - "host.docker.internal:host-gateway"
```

### Issue: Model Returns 401 Unauthorized

**Symptom:** Model configured but returns auth error.

**Cause:** Invalid API key in upstreams.

**Solution:**
1. Check `upstreams[0].key` in config
2. Verify key is valid with provider
3. Check adapter logs: `docker compose logs adapter-dial`

### Issue: Rate Limit Errors

**Symptom:** 429 Too Many Requests errors.

**Solution:**
```json
{
  "roles": {
    "default": {
      "limits": {
        "minute": "1000000",  // Increase limits
        "day": "10000000"
      }
    }
  }
}
```

Or remove limits entirely:
```json
"limits": {}
```

### Issue: Slow Performance

**Cause 1: Insufficient resources**
```bash
# Check Docker resource limits
docker stats
```

**Solution:** Increase Docker Desktop resources:
- Settings → Resources → Memory: 8GB+
- Settings → Resources → CPUs: 4+

**Cause 2: Redis memory**
```bash
docker compose exec redis redis-cli INFO memory
```

**Solution:** Increase Redis maxmemory in docker-compose.yml:
```yaml
command: >
  redis-server
  --maxmemory 4000mb  # Increase from 2000mb
```

### Issue: "Module not found" in Python App

**Symptom:** Application crashes with import errors.

**Solution:**
```bash
# Ensure virtual environment is activated
source venv/bin/activate  # macOS/Linux
venv\Scripts\activate     # Windows

# Reinstall dependencies
pip install -r requirements.txt

# Verify installation
pip list | grep aidial
```

### Issue: Config Changes Not Applied

**Symptom:** Modified config.json but changes don't appear.

**Solution:**
1. Verify JSON syntax: `cat core/config.json | jq .`
2. Restart Core: `docker compose stop core && docker compose up -d core`
3. Check Core logs: `docker compose logs core | tail -50`
4. Clear browser cache and reload

### Issue: Docker Compose Version Mismatch

**Symptom:** `docker-compose` command not found or syntax errors.

**Old syntax (v1):**
```bash
docker-compose up -d
```

**New syntax (v2+):**
```bash
docker compose up -d  # No hyphen
```

**Solution:** Update Docker Desktop or install Compose V2 plugin.

## Platform-Specific Notes

### macOS

**ARM (M1/M2/M3) Users:**

Add to all services in docker-compose.yml:
```yaml
services:
  chat:
    platform: linux/amd64
    # ...
  core:
    platform: linux/amd64
    # ...
```

**host.docker.internal:** Works natively, no configuration needed.

**File Permissions:**
- Core logs: `./core-logs/` owned by Docker user
- May need: `sudo chown -R $USER core-logs`

### Windows

**WSL2 Backend Required:**
- Docker Desktop → Settings → General → Use WSL 2 based engine

**Path Separators:**
```powershell
# Use forward slashes in docker-compose.yml
./core:/opt/config  # Not .\core:/opt/config
```

**Line Endings:**
```bash
# If cloning from Git, ensure LF not CRLF
git config --global core.autocrlf input
```

**host.docker.internal:** Works in recent Docker Desktop versions.

### Linux

**host.docker.internal Setup:**

```yaml
# Add to core service in docker-compose.yml
services:
  core:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

**Or use Docker bridge IP:**
```json
{
  "endpoint": "http://172.17.0.1:5022/..."
}
```

**Permissions:**
```bash
# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

**File Permissions:**
```bash
# Fix volume mount permissions
sudo chown -R $USER:$USER core-logs core-data
```

## Uninstallation

### Stop and Remove Everything

```bash
# Stop services
docker compose down

# Remove volumes (conversations, cache)
docker compose down -v --remove-orphans

# Remove images
docker rmi epam/ai-dial-chat:development
docker rmi epam/ai-dial-core:development
docker rmi epam/ai-dial-chat-themes:development
docker rmi epam/ai-dial-adapter-dial:development
docker rmi redis:7.2.4-alpine3.19

# Remove local files
rm -rf core-logs core-data
```

### Remove Docker Desktop (Full Cleanup)

**macOS:**
```bash
# Uninstall Docker Desktop
rm -rf ~/Library/Group\ Containers/group.com.docker
rm -rf ~/Library/Containers/com.docker.docker
```

**Windows:**
```powershell
# Uninstall Docker Desktop via Settings → Apps
# Then delete: %APPDATA%\Docker
```

## Environment Variables Reference

### Docker Compose Environment

Create `.env` file in project root:

```bash
# Chat timeouts
CHAT_KEEP_ALIVE_TIMEOUT=20000

# Core data directory
DIAL_DIR=.

# User ID for file permissions (Linux)
UID=1000
```

### Core Environment Variables

```yaml
services:
  core:
    environment:
      AIDIAL_SETTINGS: '/opt/settings/settings.json'
      JAVA_OPTS: '-Dgflog.config=/opt/settings/gflog.xml'
      LOG_DIR: '/app/log'
      STORAGE_DIR: '/app/data'
      'aidial.config.files': '["/opt/config/config.json"]'
      'aidial.storage.overrides': '{ "jclouds.filesystem.basedir": "data" }'
      'aidial.redis.singleServerConfig.address': 'redis://redis:6379'
```

### Chat Environment Variables

```yaml
services:
  chat:
    environment:
      NEXTAUTH_SECRET: "secret"  # Change in production
      THEMES_CONFIG_HOST: "http://themes:8080"
      DIAL_API_HOST: "http://core:8080"
      DIAL_API_KEY: "dial_api_key"
      ENABLED_FEATURES: "conversations-section,prompts-section,..."
```

## Next Steps

After successful setup:

1. **Learn the basics:** Follow [Roadmap - Task 1](./roadmap.md#task-1-basic-dial-setup)
2. **Build your first app:** Complete [Task 3](./roadmap.md#task-3-local-development-workflow)
3. **Add models:** Follow [Task 4](./roadmap.md#task-4-integrating-models)
4. **Create advanced apps:** Try [Task 5](./roadmap.md#task-5-composite-applications)

**Explore:**
- [Architecture Guide](./architecture.md) - Understand the system
- [API Reference](./api.md) - Build applications
- [Glossary](./glossary.md) - Learn terminology

---

**Need Help?**
- Check [Troubleshooting](#troubleshooting) section
- Review [Common Tasks](#common-tasks)
- See [Platform-Specific Notes](#platform-specific-notes)
