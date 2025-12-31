---
title: DIAL Local Setup - API Reference
description: Complete API reference for DIAL SDK, Core endpoints, and client libraries
version: 1.0.0
last_updated: 2025-12-30
related: [architecture.md, setup.md, README.md]
tags: [api, dial-sdk, asyncdial, python, rest]
---

# API Reference

## Table of Contents
- [DIAL Core API](#dial-core-api)
- [DIAL SDK (aidial-sdk)](#dial-sdk-aidial-sdk)
- [AsyncDial Client (aidial-client)](#asyncdial-client-aidial-client)
- [Configuration API](#configuration-api)
- [Code Examples](#code-examples)
- [Error Handling](#error-handling)

## DIAL Core API

DIAL Core exposes OpenAI-compatible endpoints for models and applications.

### Base URL
```
http://localhost:8080
```

### Authentication

**Header:**
```http
Authorization: Bearer dial_api_key
```

**API Key Configuration:**
```json
{
  "keys": {
    "dial_api_key": {
      "project": "TEST-PROJECT",
      "role": "default"
    }
  }
}
```

### Chat Completions

#### Endpoint
```http
POST /openai/deployments/{deployment-name}/chat/completions
```

#### Request Schema

```json
{
  "messages": [
    {
      "role": "system" | "user" | "assistant",
      "content": "string"
    }
  ],
  "stream": true | false,
  "temperature": 0.0-2.0,
  "max_tokens": integer,
  "top_p": 0.0-1.0,
  "frequency_penalty": -2.0-2.0,
  "presence_penalty": -2.0-2.0
}
```

#### Response Schema (Non-Streaming)

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "deployment-name",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Response text"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 20,
    "total_tokens": 30
  }
}
```

#### Response Schema (Streaming)

```jsonlines
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1234567890,"model":"deployment-name","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1234567890,"model":"deployment-name","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1234567890,"model":"deployment-name","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

#### Example: cURL

```bash
curl -X POST http://localhost:8080/openai/deployments/gpt-4o/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dial_api_key" \
  -d '{
    "messages": [
      {"role": "user", "content": "Hello!"}
    ],
    "stream": false
  }'
```

#### Example: Python requests

```python
import requests

response = requests.post(
    "http://localhost:8080/openai/deployments/gpt-4o/chat/completions",
    headers={
        "Authorization": "Bearer dial_api_key",
        "Content-Type": "application/json"
    },
    json={
        "messages": [{"role": "user", "content": "Hello!"}],
        "stream": False
    }
)

print(response.json())
```

### Rate Limiting

**Rate limit headers in response:**
```http
X-RateLimit-Limit-Requests: 1000
X-RateLimit-Remaining-Requests: 999
X-RateLimit-Reset-Requests: 1234567890
```

**Rate limit error (429):**
```json
{
  "error": {
    "message": "Rate limit exceeded",
    "type": "rate_limit_error",
    "code": "rate_limit_exceeded"
  }
}
```

## DIAL SDK (aidial-sdk)

Python SDK for building DIAL-compatible applications and model adapters.

### Installation

```bash
pip install aidial-sdk==0.27.0
```

### Core Classes

#### DIALApp

FastAPI-based application wrapper.

```python
from aidial_sdk import DIALApp

app = DIALApp()
```

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `add_chat_completion` | `(deployment_name: str, impl: ChatCompletion)` | Register a chat completion handler |
| `add_embeddings` | `(deployment_name: str, impl: Embeddings)` | Register an embeddings handler |

**Example:**
```python
app = DIALApp()
app.add_chat_completion("echo", EchoApplication())
app.add_chat_completion("essay-assistant", EssayAssistant())

# Run with uvicorn
import uvicorn
uvicorn.run(app, port=5000, host="0.0.0.0")
```

#### ChatCompletion

Abstract base class for chat completion handlers.

```python
from aidial_sdk.chat_completion import ChatCompletion, Request, Response

class MyApplication(ChatCompletion):
    async def chat_completion(
        self, 
        request: Request, 
        response: Response
    ) -> None:
        # Implementation
        pass
```

#### Request

Input data structure.

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `messages` | `List[Message]` | Conversation history |
| `temperature` | `float \| None` | Sampling temperature |
| `max_tokens` | `int \| None` | Maximum tokens to generate |
| `top_p` | `float \| None` | Nucleus sampling parameter |
| `n` | `int \| None` | Number of completions |
| `stream` | `bool \| None` | Whether to stream response |

**Message Structure:**
```python
class Message:
    role: str  # "system" | "user" | "assistant"
    content: str | None
    name: str | None
```

**Example:**
```python
async def chat_completion(self, request: Request, response: Response):
    # Get last user message
    last_message = request.messages[-1]
    user_text = last_message.content
    
    # Get temperature parameter
    temperature = request.temperature or 0.7
```

#### Response

Output data structure for building responses.

**Methods:**

| Method | Returns | Description |
|--------|---------|-------------|
| `create_single_choice()` | `Choice` | Context manager for single response |
| `create_choice(index: int)` | `Choice` | Context manager for multi-choice response |

**Example:**
```python
async def chat_completion(self, request: Request, response: Response):
    with response.create_single_choice() as choice:
        choice.append_content("Hello, world!")
```

#### Choice

Individual completion choice builder.

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `append_content` | `(content: str)` | Add text to response |
| `set_finish_reason` | `(reason: str)` | Set completion reason |

**Finish Reasons:**
- `"stop"` - Natural completion
- `"length"` - Max tokens reached
- `"content_filter"` - Content policy violation

**Example (Streaming):**
```python
with response.create_single_choice() as choice:
    for chunk in generate_text():
        choice.append_content(chunk)
    choice.set_finish_reason("stop")
```

### Complete Example: Echo Application

```python
import uvicorn
from aidial_sdk import DIALApp
from aidial_sdk.chat_completion import ChatCompletion, Request, Response


class EchoApplication(ChatCompletion):
    async def chat_completion(
        self, request: Request, response: Response
    ) -> None:
        # Extract last user message
        last_user_message = request.messages[-1]
        
        # Create single-choice response
        with response.create_single_choice() as choice:
            # Echo the content back
            choice.append_content(last_user_message.content or "")
            choice.set_finish_reason("stop")


# Create and register application
app = DIALApp()
app.add_chat_completion("echo", EchoApplication())

# Run server
if __name__ == "__main__":
    uvicorn.run(app, port=5022, host="0.0.0.0")
```

**Configuration in core/config.json:**
```json
{
  "applications": {
    "echo": {
      "displayName": "Echo App",
      "endpoint": "http://host.docker.internal:5022/openai/deployments/echo/chat/completions"
    }
  }
}
```

### Advanced Example: Essay Assistant

```python
import uvicorn
from aidial_client import AsyncDial
from aidial_sdk import DIALApp
from aidial_sdk.chat_completion import ChatCompletion, Request, Response

SYSTEM_PROMPT = """You are an essay-focused assistant. 
Respond to every request by writing a short essay."""


class EssayAssistant(ChatCompletion):
    async def chat_completion(
        self, request: Request, response: Response
    ) -> None:
        # Create DIAL client to call upstream model
        client = AsyncDial(
            base_url="http://localhost:8080",
            api_key="dial_api_key",
            api_version="2025-01-01-preview"
        )
        
        # Get user's last message
        user_message = request.messages[-1].content
        
        # Call GPT-4 with system prompt
        chunks = await client.chat.completions.create(
            deployment_name="gpt-4o",
            stream=True,
            messages=[
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": user_message}
            ]
        )
        
        # Stream response back to user
        with response.create_single_choice() as choice:
            async for chunk in chunks:
                if chunk.choices:
                    delta = chunk.choices[0].delta
                    if delta and delta.content:
                        choice.append_content(delta.content)


app = DIALApp()
app.add_chat_completion("essay-assistant", EssayAssistant())

if __name__ == "__main__":
    uvicorn.run(app, port=5025, host="0.0.0.0")
```

## AsyncDial Client (aidial-client)

Python client library for calling DIAL Core APIs.

### Installation

```bash
pip install aidial-client==0.3.0
```

### Client Initialization

```python
from aidial_client import AsyncDial

client = AsyncDial(
    base_url="http://localhost:8080",
    api_key="dial_api_key",
    api_version="2025-01-01-preview"
)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `base_url` | `str` | Yes | DIAL Core URL |
| `api_key` | `str` | Yes | API key from config.json |
| `api_version` | `str` | Yes | API version string |

### Chat Completions

#### Method: create()

```python
await client.chat.completions.create(
    deployment_name: str,
    messages: List[Dict],
    stream: bool = False,
    temperature: float | None = None,
    max_tokens: int | None = None,
    top_p: float | None = None
)
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `deployment_name` | `str` | Model or application name from config |
| `messages` | `List[Dict]` | Conversation messages |
| `stream` | `bool` | Enable streaming (default: False) |
| `temperature` | `float` | Sampling temperature (0.0-2.0) |
| `max_tokens` | `int` | Maximum tokens to generate |
| `top_p` | `float` | Nucleus sampling (0.0-1.0) |

#### Example: Non-Streaming

```python
from aidial_client import AsyncDial

async def call_model():
    client = AsyncDial(
        base_url="http://localhost:8080",
        api_key="dial_api_key",
        api_version="2025-01-01-preview"
    )
    
    response = await client.chat.completions.create(
        deployment_name="gpt-4o",
        messages=[
            {"role": "user", "content": "What is AI DIAL?"}
        ],
        stream=False
    )
    
    print(response.choices[0].message.content)
```

#### Example: Streaming

```python
async def stream_model():
    client = AsyncDial(
        base_url="http://localhost:8080",
        api_key="dial_api_key",
        api_version="2025-01-01-preview"
    )
    
    chunks = await client.chat.completions.create(
        deployment_name="gpt-4o",
        messages=[
            {"role": "user", "content": "Count to 10"}
        ],
        stream=True
    )
    
    async for chunk in chunks:
        if chunk.choices:
            delta = chunk.choices[0].delta
            if delta and delta.content:
                print(delta.content, end="", flush=True)
```

#### Example: Multiple Models

```python
async def call_multiple_models():
    client = AsyncDial(
        base_url="http://localhost:8080",
        api_key="dial_api_key",
        api_version="2025-01-01-preview"
    )
    
    models = ["gpt-4o", "claude-3-7-sonnet@20250219", "gemini-2.5-pro"]
    
    for model in models:
        response = await client.chat.completions.create(
            deployment_name=model,
            messages=[{"role": "user", "content": "Say hi"}],
            stream=False
        )
        print(f"{model}: {response.choices[0].message.content}")
```

## Configuration API

Configuration is defined in JSON files, not via programmatic API.

### Model Configuration

**File:** `core/config.json`

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
          "endpoint": "https://api.openai.com/v1/chat/completions",
          "key": "${OPENAI_API_KEY}"
        }
      ]
    }
  }
}
```

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `displayName` | `string` | Yes | Human-readable name |
| `endpoint` | `string` | Yes | Adapter endpoint URL |
| `iconUrl` | `string` | No | Icon URL for UI |
| `type` | `"chat"` | Yes | Model type |
| `upstreams` | `array` | Yes | Upstream provider configs |

### Application Configuration

```json
{
  "applications": {
    "echo": {
      "displayName": "Echo App",
      "description": "Repeats your message",
      "endpoint": "http://host.docker.internal:5022/openai/deployments/echo/chat/completions"
    }
  }
}
```

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `displayName` | `string` | Yes | Human-readable name |
| `description` | `string` | No | Description for UI |
| `endpoint` | `string` | Yes | Application endpoint URL |

**Endpoint Patterns:**

- **Containerized app:** `http://<service-name>:5000/...`
- **Local IDE app:** `http://host.docker.internal:<PORT>/...`

### API Key Configuration

```json
{
  "keys": {
    "dial_api_key": {
      "project": "TEST-PROJECT",
      "role": "default"
    },
    "another_key": {
      "project": "ANOTHER-PROJECT",
      "role": "premium"
    }
  }
}
```

### Role & Limits Configuration

```json
{
  "roles": {
    "default": {
      "limits": {
        "minute": "256000",
        "day": "1000000",
        "week": "1256000",
        "month": "11256000"
      }
    },
    "premium": {
      "limits": {
        "minute": "1000000",
        "day": "10000000"
      }
    }
  }
}
```

**Limit Units:** Token counts (not requests)

## Code Examples

### Example 1: Simple Text Transform Application

```python
import uvicorn
from aidial_sdk import DIALApp
from aidial_sdk.chat_completion import ChatCompletion, Request, Response


class UpperCaseApp(ChatCompletion):
    async def chat_completion(
        self, request: Request, response: Response
    ) -> None:
        last_message = request.messages[-1]
        text = last_message.content or ""
        
        with response.create_single_choice() as choice:
            choice.append_content(text.upper())


app = DIALApp()
app.add_chat_completion("uppercase", UpperCaseApp())

if __name__ == "__main__":
    uvicorn.run(app, port=5030, host="0.0.0.0")
```

### Example 2: Multi-Turn Conversation Application

```python
class ConversationApp(ChatCompletion):
    async def chat_completion(
        self, request: Request, response: Response
    ) -> None:
        # Access full conversation history
        messages = request.messages
        
        # Build context from history
        context = "\n".join([
            f"{msg.role}: {msg.content}"
            for msg in messages
        ])
        
        with response.create_single_choice() as choice:
            choice.append_content(f"I see you said {len(messages)} things.")
```

### Example 3: Calling External APIs

```python
import httpx
from aidial_sdk.chat_completion import ChatCompletion, Request, Response


class WeatherApp(ChatCompletion):
    async def chat_completion(
        self, request: Request, response: Response
    ) -> None:
        user_message = request.messages[-1].content
        
        # Extract location (simplified)
        location = user_message.replace("weather in ", "")
        
        # Call external API
        async with httpx.AsyncClient() as client:
            weather_data = await client.get(
                f"https://api.weather.com/{location}"
            )
        
        with response.create_single_choice() as choice:
            choice.append_content(f"Weather: {weather_data.json()}")
```

### Example 4: Streaming with AsyncDial

```python
from aidial_client import AsyncDial
from aidial_sdk import DIALApp
from aidial_sdk.chat_completion import ChatCompletion, Request, Response


class StreamingApp(ChatCompletion):
    async def chat_completion(
        self, request: Request, response: Response
    ) -> None:
        client = AsyncDial(
            base_url="http://localhost:8080",
            api_key="dial_api_key",
            api_version="2025-01-01-preview"
        )
        
        # Stream from upstream model
        chunks = await client.chat.completions.create(
            deployment_name="gpt-4o",
            stream=True,
            messages=request.messages
        )
        
        # Forward stream to user
        with response.create_single_choice() as choice:
            async for chunk in chunks:
                if chunk.choices and chunk.choices[0].delta:
                    delta = chunk.choices[0].delta
                    if delta.content:
                        choice.append_content(delta.content)
```

## Error Handling

### Common HTTP Status Codes

| Code | Meaning | Cause |
|------|---------|-------|
| `200` | OK | Request successful |
| `400` | Bad Request | Invalid JSON or parameters |
| `401` | Unauthorized | Invalid or missing API key |
| `404` | Not Found | Deployment name not in config |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Core or adapter error |
| `502` | Bad Gateway | Upstream provider error |

### Error Response Schema

```json
{
  "error": {
    "message": "Human-readable error description",
    "type": "error_type",
    "code": "error_code",
    "param": "parameter_name"
  }
}
```

### Exception Handling in Applications

```python
from aidial_sdk.chat_completion import ChatCompletion, Request, Response
import logging

logger = logging.getLogger(__name__)


class RobustApp(ChatCompletion):
    async def chat_completion(
        self, request: Request, response: Response
    ) -> None:
        try:
            # Your logic
            result = await process_message(request.messages[-1])
            
            with response.create_single_choice() as choice:
                choice.append_content(result)
                
        except ValueError as e:
            logger.error(f"Invalid input: {e}")
            with response.create_single_choice() as choice:
                choice.append_content("Sorry, I couldn't process that.")
                
        except Exception as e:
            logger.exception("Unexpected error")
            with response.create_single_choice() as choice:
                choice.append_content("An error occurred.")
```

### Handling AsyncDial Errors

```python
from aidial_client import AsyncDial
from aidial_client.exceptions import DIALException


async def safe_call():
    client = AsyncDial(...)
    
    try:
        response = await client.chat.completions.create(
            deployment_name="gpt-4o",
            messages=[{"role": "user", "content": "Hello"}]
        )
        return response.choices[0].message.content
        
    except DIALException as e:
        print(f"DIAL error: {e}")
        return "Sorry, model unavailable"
        
    except Exception as e:
        print(f"Unexpected error: {e}")
        return "System error"
```

## API Versioning

**Current Version:** `2025-01-01-preview`

**Version String Format:** `YYYY-MM-DD-preview` or `YYYY-MM-DD`

**Usage:**
```python
client = AsyncDial(
    base_url="http://localhost:8080",
    api_key="dial_api_key",
    api_version="2025-01-01-preview"  # <-- Required
)
```

**Note:** API version must match DIAL Core expectations. Check Core logs if version mismatch errors occur.

## Reference Implementation Matrix

| Feature | Echo App (T2/T3) | Essay Assistant (T5) |
|---------|------------------|----------------------|
| **DIAL SDK** | ✅ | ✅ |
| **AsyncDial Client** | ❌ | ✅ |
| **Streaming** | ✅ | ✅ |
| **System Prompt** | ❌ | ✅ |
| **Upstream Model Call** | ❌ | ✅ |
| **Local Development** | ✅ (T3) | ✅ |
| **Container Deployment** | ✅ (T2) | ❌ |

**File Locations:**
- Echo (Container): [tasks/t2/echo/app.py](../tasks/t2/echo/app.py)
- Echo (Local): [tasks/t3/echo/app.py](../tasks/t3/echo/app.py)
- Essay Assistant: [tasks/t5/essay_assistant/app.py](../tasks/t5/essay_assistant/app.py)

## Related Documentation

- [Architecture Guide](./architecture.md) - System design and data flows
- [Setup Guide](./setup.md) - Installation and configuration
- [Roadmap](./roadmap.md) - Task-based learning path
- [Glossary](./glossary.md) - Domain terminology

---

**Note:** This API reference is specific to the local development environment. Production DIAL deployments may have additional features or different endpoint structures.
