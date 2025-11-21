# Practice with DIAL Core, Chat, Themes and Adapter setup

## Project Structure

```
├── core/
│   └── config.json               🚧 TODO: Follow instructions in tasks. DIAL Core configuration with routes, applications, models, and keys
├── settings/
│   └── settings.json             ✅ Complete - Core server settings and identity providers
├── tasks/                        
│   ├── t1/
│   │   └── start.md              🚧 TODO: Follow instructions 
│   ├── t2/
│   │   ├── .env                  ✅ Complete - Environment configuration
│   │   ├── core/
│   │   │   └── config.json       🚧 TODO: Follow instructions
│   │   ├── docker-compose.yml    ✅ Complete - Extended compose with echo service
│   │   ├── echo/
│   │   │   ├── Dockerfile        ✅ Complete - Echo app containerization
│   │   │   ├── app.py            ✅ Complete - Simple echo application
│   │   │   └── requirements.txt  ✅ Complete - Python dependencies
│   │   └── task_2.md             🚧 TODO: Follow instructions
│   ├── t3/
│   │   ├── echo/
│   │   │   ├── app.py            ✅ Complete - Modified echo for local development
│   │   │   └── requirements.txt  ✅ Complete - Updated dependencies
│   │   └── task_3.md             🚧 TODO: Follow instructions 
│   ├── t4/
│   │   └── task_4.md             🚧 TODO: Follow instructions 
│   └── t5/
│       ├── essay_assistant/
│       │   ├── app.py            🚧 TODO: Complete implementation with AsyncDial client
│       │   └── requirements.txt  ✅ Complete - Dependencies for essay assistant
│       └── task_5.md             🚧 TODO: Follow instructions 
└── docker-compose.yml            🚧 TODO: - Main compose file (Add NASA_API_KEY)
```

## Services Architecture

### Core Services
- **themes** (port 3001) - DIAL Chat themes service
- **chat** (port 3000) - Main DIAL Chat interface
- **core** (port 8080) - DIAL Core API gateway
- **redis** (port 6379) - Cache and session storage
- **adapter-dial** - DIAL adapter for upstream model communication

### Development Applications
- **echo** (port 5000/5022) - Simple echo application for testing
- **essay-assistant** (port 5025) - Essay-focused AI assistant

## Configuration Files
- **core/config.json** - Main DIAL configuration with applications, models, and API keys
- **settings/settings.json** - Core server settings and security configuration
- **docker-compose.yml** - Service orchestration and networking

## Environment Requirements
- Docker and Docker Compose
- Python 3.11+ for local development
- DIAL API key for model access

## Learning Path
1. **T1** - Basic DIAL Chat setup
2. **T2** - Optional, First Echo application in container
3. **T3** - Local development workflow
4. **T4** - Models and adapter integration
5. **T5** - Advanced application with streaming

## AFTER ALL THE TASKS DONE - DON'T FORGET TO REMOVE API KEYs FROM core/config.json

---

## How to hide API keys:

Importunately, there is no option to fetch them from env variables, but you can hide them in project:

1. Create `keys.json` in [core config folder](core). `keys.json` is added to the [.gitignore](.gitignore) and will be ignored by git
2. Segregate [core config](core/config.json):
   Original config:
    ```json
    {
      "routes": {},
      "applications": {},
      "models": {
        "gpt-4o": {
          "displayName": "GPT 4o",
          "overrideName": "gpt-4o",
          "endpoint": "http://adapter-dial-openai:5000/openai/deployments/gpt-4o/chat/completions",
          "iconUrl": "http://localhost:3001/gpt4.svg",
          "type": "chat",
          "upstreams": [
            {
              "endpoint": "https://api.openai.com/v1/chat/completions",
              "key": "${OPENAI_API_KEY}"
            }
          ]
        }
      },
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
   Updated [core config](core/config.json):
    ```json
    {
      "routes": {},
      "applications": {},
      "models": {
        "gpt-4o": {
          "displayName": "GPT 4o",
          "overrideName": "gpt-4o",
          "endpoint": "http://adapter-dial-openai:5000/openai/deployments/gpt-4o/chat/completions",
          "iconUrl": "http://localhost:3001/gpt4.svg",
          "type": "chat"
        }
      },
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
   Moved `upstreams` to [keys config](core/keys.json):
    ```json
    {
      "models": {
        "gpt-4o": {
          "upstreams": [
            {
              "endpoint": "https://api.openai.com/v1/chat/completions",
              "key": "${OPENAI_API_KEY}"
            }
          ]
        }
      }
    }
    ```
3. Update `core` service env variable adn provide path to `keys.json` config:
   `'aidial.config.files': '["/opt/config/config.json", "/opt/config/keys.json"]'`
4. Delete `core` container and start it from scratch

---
# <img src="dialx-banner.png">