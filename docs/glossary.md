---
title: DIAL Local Setup - Glossary
description: Definitions of key terms, abbreviations, and domain-specific terminology
version: 1.0.0
last_updated: 2025-12-30
related: [README.md, architecture.md]
tags: [glossary, terminology, definitions]
---

# Glossary

## Table of Contents
- [Core Concepts](#core-concepts)
- [Components](#components)
- [Configuration Terms](#configuration-terms)
- [Development Terms](#development-terms)
- [Protocols & Standards](#protocols--standards)
- [Abbreviations](#abbreviations)

## Core Concepts

### DIAL (AI Distributed Application Layer)
A platform for orchestrating AI models and applications through a unified gateway. Provides API routing, authentication, rate limiting, and conversation management.

**Related:** DIAL Core, DIAL Chat, DIAL SDK

---

### Model
An AI language model (LLM) accessible through DIAL Core. Examples: GPT-4, Claude, Gemini.

**Configuration:** Defined in `core/config.json` under `models` section.

**Example:**
```json
{
  "models": {
    "gpt-4o": {
      "displayName": "GPT-4o",
      "endpoint": "http://adapter-dial:5000/...",
      "type": "chat"
    }
  }
}
```

---

### Application
Custom AI-powered application built with DIAL SDK. Examples: Echo App, Essay Assistant.

**Configuration:** Defined in `core/config.json` under `applications` section.

**Example:**
```json
{
  "applications": {
    "echo": {
      "displayName": "Echo App",
      "endpoint": "http://host.docker.internal:5022/..."
    }
  }
}
```

---

### Deployment Name
Unique identifier for a model or application in DIAL. Used in API requests and configuration.

**Example:** `gpt-4o`, `echo`, `essay-assistant`

**Usage:**
```
POST /openai/deployments/{deployment-name}/chat/completions
```

---

### Upstream
External AI model provider (OpenAI, Anthropic, Google) that DIAL calls to fulfill requests.

**Configuration:**
```json
{
  "upstreams": [
    {
      "endpoint": "https://api.openai.com/v1/chat/completions",
      "key": "sk-..."
    }
  ]
}
```

---

### Streaming
Incremental response delivery where content arrives chunk-by-chunk instead of all at once.

**Benefits:** Faster perceived latency, better UX for long responses.

**API:** Set `"stream": true` in request.

---

### Chat Completion
API endpoint for conversational AI interactions. Takes message history, returns assistant response.

**Standard:** OpenAI Chat Completion API format.

## Components

### DIAL Core
Java-based API gateway that routes requests to models and applications.

**Port:** 8080  
**Image:** `epam/ai-dial-core:development`  
**Responsibilities:** Authentication, routing, rate limiting, storage

---

### DIAL Chat
Next.js web UI for interacting with models and applications.

**Port:** 3000  
**Image:** `epam/ai-dial-chat:development`  
**Features:** Conversations, marketplace, prompts, file attachments

---

### DIAL Adapter
Service that translates between DIAL protocol and upstream provider APIs.

**Port:** Internal (no host exposure)  
**Image:** `epam/ai-dial-adapter-dial:development`  
**Purpose:** Protocol translation, streaming, error handling

---

### Themes Service
Static asset server for DIAL Chat UI themes and icons.

**Port:** 3001  
**Image:** `epam/ai-dial-chat-themes:development`

---

### Redis
In-memory cache for sessions, rate limits, and temporary conversation storage.

**Port:** 6379  
**Image:** `redis:7.2.4-alpine3.19`  
**Configuration:** 2GB maxmemory, volatile-lfu eviction

## Configuration Terms

### config.json
Main configuration file defining models, applications, keys, and roles.

**Location:** `core/config.json`  
**Format:** JSON  
**Tracked:** Yes (in Git)

---

### keys.json
Secret configuration file containing API keys for upstream providers.

**Location:** `core/keys.json`  
**Format:** JSON  
**Tracked:** No (gitignored)  
**Purpose:** Keep secrets out of version control

---

### settings.json
DIAL Core server settings (authentication, encryption, port).

**Location:** `settings/settings.json`  
**Tracked:** Yes

---

### API Key
Authentication token for accessing DIAL Core API.

**Example:** `dial_api_key`  
**Usage:** `Authorization: Bearer dial_api_key`  
**Configuration:** Defined in `keys` section of config.json

---

### Role
Authorization profile with associated rate limits.

**Example:** `default`, `premium`  
**Configuration:**
```json
{
  "roles": {
    "default": {
      "limits": {
        "minute": "256000",
        "day": "1000000"
      }
    }
  }
}
```

---

### Rate Limit
Maximum number of tokens allowed per time period (minute/day/week/month).

**Purpose:** Prevent API abuse, control costs.  
**Unit:** Token count (not request count).

---

### Endpoint
URL where a model or application receives requests.

**Model endpoint:** Points to adapter  
**Application endpoint:** Points to application server

**Example:**
```
http://host.docker.internal:5022/openai/deployments/echo/chat/completions
```

## Development Terms

### host.docker.internal
Special hostname that resolves to the Docker host machine's IP address from within containers.

**Purpose:** Allow Core (in Docker) to reach applications running on host (in IDE).

**Platform Support:**
- macOS: Built-in ✅
- Windows: Built-in ✅
- Linux: Requires configuration ⚠️

**Usage:**
```json
{
  "endpoint": "http://host.docker.internal:5022/..."
}
```

---

### Hot Reload
Development pattern where code changes take effect immediately without full rebuild/restart.

**In DIAL:**
- Edit Python app
- Restart Python process (Ctrl+C, `python app.py`)
- No Docker rebuild needed

---

### Container
Isolated runtime environment for a service (Core, Chat, Redis, etc.).

**Management:**
- Start: `docker compose up -d`
- Stop: `docker compose stop`
- Logs: `docker compose logs -f`

---

### Volume Mount
Mapping between host filesystem and container filesystem.

**Example:**
```yaml
volumes:
  - ./core:/opt/config  # Host:Container
```

**Purpose:** Edit config on host, visible inside container.

---

### Docker Compose
Tool for defining and running multi-container Docker applications.

**File:** `docker-compose.yml`  
**Commands:** `docker compose up`, `docker compose down`

---

### Task (T1-T5)
Sequential learning module in DIAL Local Setup curriculum.

**Tasks:**
- T1: Basic setup
- T2: Containerized app
- T3: Local development
- T4: Add models
- T5: Composite applications

## Protocols & Standards

### OpenAI API
De facto standard API format for chat completions.

**Request:**
```json
{
  "messages": [{"role": "user", "content": "..."}],
  "stream": false
}
```

**Response:**
```json
{
  "choices": [{
    "message": {"role": "assistant", "content": "..."}
  }]
}
```

**Adopted by:** OpenAI, Anthropic (via adapter), Google (via adapter), many others

---

### SSE (Server-Sent Events)
Protocol for streaming responses from server to client.

**Format:**
```
data: {"chunk": "content"}
data: {"chunk": "more content"}
data: [DONE]
```

**Usage:** Streaming chat completions

---

### JWT (JSON Web Token)
Authentication token format (optional in local setup).

**Configuration:** `settings.json` → `identityProviders`  
**Local Setup:** JWT verification disabled for simplicity

---

### HTTP/REST
Communication protocol between services.

**Example:** Chat → Core, Core → Adapter, Core → Application

## Abbreviations

### AI
**Artificial Intelligence** - Computer systems that perform tasks requiring human intelligence.

---

### API
**Application Programming Interface** - Interface for software components to communicate.

---

### CLI
**Command-Line Interface** - Text-based user interface (e.g., `docker compose`).

---

### CRUD
**Create, Read, Update, Delete** - Basic database operations.

---

### IDE
**Integrated Development Environment** - Software for writing code (VS Code, PyCharm).

---

### JSON
**JavaScript Object Notation** - Data format used for configuration and API payloads.

---

### LLM
**Large Language Model** - AI model trained on text data (GPT-4, Claude, Gemini).

---

### REST
**Representational State Transfer** - Architectural style for web APIs.

---

### SDK
**Software Development Kit** - Libraries and tools for building applications.

**Example:** `aidial-sdk` for building DIAL applications.

---

### UI
**User Interface** - Visual interface for user interaction (DIAL Chat).

---

### URL
**Uniform Resource Locator** - Web address (e.g., `http://localhost:3000`).

---

### YAML
**YAML Ain't Markup Language** - Human-readable data format (used in `docker-compose.yml`).

## DIAL-Specific Terms

### DIAL SDK
Python library for building DIAL-compatible applications.

**Package:** `aidial-sdk`  
**Version:** 0.27.0  
**Purpose:** Abstracts DIAL protocol, provides helper classes

**Key Classes:**
- `DIALApp` - FastAPI wrapper
- `ChatCompletion` - Abstract base for handlers
- `Request` - Input data structure
- `Response` - Output builder

---

### AsyncDial Client
Python client library for calling DIAL APIs.

**Package:** `aidial-client`  
**Version:** 0.3.0  
**Purpose:** Make requests to DIAL Core from applications

**Usage:**
```python
from aidial_client import AsyncDial

client = AsyncDial(
    base_url="http://localhost:8080",
    api_key="dial_api_key"
)
```

---

### Choice
Individual response option in chat completion (usually only one choice).

**Structure:**
```json
{
  "choices": [
    {
      "index": 0,
      "message": {"role": "assistant", "content": "..."},
      "finish_reason": "stop"
    }
  ]
}
```

---

### Finish Reason
Indicator of why model stopped generating.

**Values:**
- `stop` - Natural completion
- `length` - Max tokens reached
- `content_filter` - Content policy violation
- `null` - Still generating (streaming)

---

### Message Role
Identifier for message sender in conversation.

**Roles:**
- `system` - Instructions to model
- `user` - User input
- `assistant` - Model response

---

### Token
Unit of text processed by models (roughly 0.75 words in English).

**Usage:** Rate limiting, pricing, context limits

**Example:** "Hello world" ≈ 2 tokens

---

### Marketplace
UI view in DIAL Chat showing available models and applications.

**Access:** [http://localhost:3000/marketplace](http://localhost:3000/marketplace)

---

### Conversation
Series of messages between user and model/application.

**Storage:** Redis (temporary) + filesystem (persistent)

---

### Prompt
Input message or system instructions sent to model.

**System Prompt:** Instructions about behavior (e.g., "You are a helpful assistant")

**User Prompt:** Actual user question

## Related Documentation

- [Architecture Guide](./architecture.md) - System design and components
- [API Reference](./api.md) - Detailed API documentation
- [Setup Guide](./setup.md) - Installation and configuration
- [Roadmap](./roadmap.md) - Learning tasks

---

**Didn't find a term?** Check the [Architecture Guide](./architecture.md) or [API Reference](./api.md) for technical details.
