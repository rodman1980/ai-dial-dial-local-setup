# ADR-002: Local Development via host.docker.internal

**Status:** Accepted

**Date:** 2025-12-30

**Deciders:** Project Architects

## Context

Applications can run in two modes:
1. **Containerized:** Application built as Docker image, runs in container
2. **Local:** Application runs directly on host machine (in IDE)

We need to decide how DIAL Core (running in Docker) connects to applications running locally on the host machine for rapid development.

## Decision

Use **`host.docker.internal`** as the hostname for applications running on the host machine.

**Configuration Pattern:**
```json
{
  "applications": {
    "echo": {
      "endpoint": "http://host.docker.internal:5022/openai/deployments/echo/chat/completions"
    }
  }
}
```

## Rationale

### Why Local Development Matters

**Containerized Development Problems:**
1. **Slow Feedback Loop:**
   - Edit code → Build image → Restart container → Test
   - Takes 30-60 seconds per change

2. **Difficult Debugging:**
   - Can't use IDE debugger directly
   - Log visibility limited
   - Breakpoints don't work

3. **Resource Intensive:**
   - Each container uses memory
   - Multiple rebuilds consume disk space

**Local Development Benefits:**
1. **Fast Iteration:**
   - Edit code → Restart Python process → Test
   - Takes 2-5 seconds

2. **Better Debugging:**
   - IDE debugger works natively
   - Breakpoints, variable inspection
   - Live reload possible

3. **Lower Resource Usage:**
   - No extra containers
   - No image builds

### Why host.docker.internal?

**macOS & Windows:**
- `host.docker.internal` is a Docker Desktop feature
- Automatically resolves to host machine's IP
- Works out-of-box, no configuration needed

**Linux:**
- Requires manual configuration (see below)
- Alternative: use `172.17.0.1` (Docker bridge IP)
- Can add via `extra_hosts` in docker-compose.yml

### Alternatives Evaluated

**Alternative 1: localhost**
```json
"endpoint": "http://localhost:5022/..."
```
❌ **Problem:** Inside Core container, `localhost` refers to the container itself, not the host machine.

**Alternative 2: Host Machine IP**
```json
"endpoint": "http://192.168.1.100:5022/..."
```
❌ **Problem:** IP changes with network (WiFi, VPN), not portable.

**Alternative 3: Docker Bridge IP**
```json
"endpoint": "http://172.17.0.1:5022/..."
```
✅ **Works on Linux** but less readable than `host.docker.internal`.

## Consequences

### Positive

- **Fast Development:** Edit Python → Reload → Test in seconds
- **IDE Integration:** Full debugging capabilities
- **Learner-Friendly:** Clear separation between containerized services and local apps
- **Resource Efficient:** No need to rebuild Docker images
- **Hot Reload:** Can add live reload to Python apps

### Negative

- **Platform-Specific:**
  - macOS/Windows: Works automatically
  - Linux: Requires extra configuration

- **Port Management:** Must ensure ports don't conflict with other services

- **Network Confusion:** Learners may initially confuse `localhost` with `host.docker.internal`

- **Production Gap:** Local pattern doesn't translate to production (all services would be containerized)

## Implementation

### macOS / Windows Setup

No configuration needed! Just use:
```json
{
  "endpoint": "http://host.docker.internal:5022/..."
}
```

### Linux Setup

**Option 1: Add extra_hosts**

Edit `docker-compose.yml`:
```yaml
services:
  core:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

**Option 2: Use Bridge IP**

```json
{
  "endpoint": "http://172.17.0.1:5022/..."
}
```

Find bridge IP:
```bash
docker network inspect bridge | grep Gateway
```

### Application Port Conventions

| Application | Port | Task |
|-------------|------|------|
| Echo (containerized) | 5000 | T2 |
| Echo (local) | 5022 | T3 |
| Essay Assistant | 5025 | T5 |

**Why different ports?**
- Avoids conflicts if both container and local versions run simultaneously
- Clear separation for learning

### Verification

**Check host access from container:**
```bash
docker compose exec core curl http://host.docker.internal:5022/health
```

**Expected:** Connection succeeds (or specific app response).

## Alternatives Considered

### Alternative 1: Always Containerize

**All applications run in Docker containers.**

**Pros:**
- Consistent with production
- Easier networking (service names)
- No platform differences

**Cons:**
- ❌ Slow feedback loop (rebuild required)
- ❌ Difficult debugging
- ❌ Poor learning experience
- ❌ Discourages experimentation

**Verdict:** Rejected for learning environment.

### Alternative 2: Docker Compose Override

Use `docker-compose.override.yml` to add local app services.

```yaml
services:
  echo-local:
    image: localhost:5022
    network_mode: "host"
```

**Cons:**
- Confusing for learners
- Doesn't actually run app (just proxy)
- More complex setup

**Verdict:** Over-engineered for learning use case.

### Alternative 3: ngrok / Tunnel

Expose local app via public URL, configure Core to use tunnel.

**Cons:**
- ❌ Requires external service
- ❌ Latency overhead
- ❌ Security concerns
- ❌ Complexity

**Verdict:** Not suitable for local development.

## Migration Path

### From Containerized (T2) to Local (T3)

**Before (T2):**
```json
{
  "endpoint": "http://echo:5000/..."
}
```

**After (T3):**
```json
{
  "endpoint": "http://host.docker.internal:5022/..."
}
```

**Steps:**
1. Stop containerized app: `docker compose down echo`
2. Update config with `host.docker.internal` endpoint
3. Run app locally: `python app.py`
4. Restart Core: `docker compose restart core`

## Related ADRs

- [ADR-005: Task-Based Learning](./ADR-005-task-based-learning.md) - Why T2 uses containers, T3+ uses local
- [ADR-004: Docker Compose Orchestration](./ADR-004-docker-compose-orchestration.md) - Overall Docker setup

## References

- [Docker Desktop Documentation - host.docker.internal](https://docs.docker.com/desktop/networking/#i-want-to-connect-from-a-container-to-a-service-on-the-host)
- [Architecture Guide - Development Patterns](../architecture.md#development-patterns)
- [Setup Guide - Configuration](../setup.md#configuration)
- [Roadmap - Task 3](../roadmap.md#task-3-local-development-workflow)
