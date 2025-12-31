# ADR-001: Configuration-Driven Architecture

**Status:** Accepted

**Date:** 2025-12-30

**Deciders:** Project Architects

## Context

We need to decide how to configure models, applications, API keys, and routing rules in the DIAL Local Setup. Options include:
- Hard-coded configuration in source code
- Environment variables
- JSON/YAML configuration files
- Database-backed configuration

## Decision

Use **JSON configuration files** (`core/config.json`) as the primary configuration mechanism for:
- Model definitions and upstreams
- Application endpoints
- API keys and roles
- Rate limits and routing rules

Configuration is loaded at Core startup and can be updated by restarting the Core container.

## Rationale

### Advantages of JSON Configuration

1. **Learning-Friendly:**
   - Easy to read and understand structure
   - Visual editors (VS Code) provide syntax highlighting
   - Can add comments via documentation

2. **Version Control:**
   - Configuration changes tracked in Git
   - Easy to diff and review changes
   - Can branch/merge configurations

3. **Multi-Environment Support:**
   - Can maintain separate configs for dev/test/demo
   - Easy to share configurations with learners
   - Can template and generate programmatically

4. **Validation:**
   - JSON schema validation possible
   - Clear error messages for syntax issues
   - Tools like `jq` for validation

5. **No Code Changes:**
   - Add new models without rebuilding containers
   - Non-developers can configure system
   - Rapid experimentation

### Why Not Other Options?

**Environment Variables:**
- ❌ Verbose for complex nested structures
- ❌ Difficult to version control
- ❌ Limited data types (strings only)
- ❌ No hierarchy/nesting support

**Hard-Coded:**
- ❌ Requires code changes for every model
- ❌ Not accessible to non-developers
- ❌ Slows learning iteration

**Database:**
- ❌ Additional infrastructure complexity
- ❌ Overkill for learning environment
- ❌ Harder to version control

## Consequences

### Positive

- **Fast Onboarding:** Learners can see all configuration in one file
- **Easy Experimentation:** Add/remove models by editing JSON
- **Portable:** Share configurations as files
- **Transparent:** All routing visible in config
- **Tooling:** Can build config generators, validators

### Negative

- **Restart Required:** Config changes need Core restart (~10 seconds)
- **No Hot Reload:** Can't update config while Core running
- **Merge Conflicts:** Multiple people editing same file
- **No Validation at Edit Time:** Errors only detected at startup
- **Secret Management:** Need separate solution for API keys (see ADR-003)

## Implementation

### Configuration Structure

```json
{
  "routes": {},
  "applications": {
    "<app-id>": {
      "displayName": "...",
      "endpoint": "..."
    }
  },
  "models": {
    "<model-id>": {
      "displayName": "...",
      "endpoint": "...",
      "upstreams": [...]
    }
  },
  "keys": {
    "<api-key>": {
      "project": "...",
      "role": "..."
    }
  },
  "roles": {
    "<role-name>": {
      "limits": {...}
    }
  }
}
```

### Loading Mechanism

1. Core reads `/opt/config/config.json` on startup
2. Optionally merges `/opt/config/keys.json` (see ADR-003)
3. Validates structure and required fields
4. Builds internal routing table
5. Logs configuration summary

### Validation Tools

- JSON syntax: `jq empty core/config.json`
- Structure: Custom Python validator (see testing.md)
- Pre-restart script: Validates before Core restart

## Alternatives Considered

### Alternative 1: Environment Variables

```yaml
DIAL_MODELS: '{"gpt-4o": {...}}'
DIAL_APPLICATIONS: '{"echo": {...}}'
```

**Rejected because:**
- Too verbose for complex structures
- Hard to read and maintain
- Limited tooling support

### Alternative 2: YAML Configuration

```yaml
models:
  gpt-4o:
    displayName: GPT-4o
    endpoint: http://...
```

**Rejected because:**
- DIAL Core expects JSON
- Indentation-sensitive (error-prone for learners)
- JSON more widely known

### Alternative 3: Split Files per Model/App

```
models/gpt-4o.json
models/claude.json
applications/echo.json
```

**Rejected because:**
- Harder to see full system configuration
- More complex file management
- Overkill for learning environment

## Related ADRs

- [ADR-003: Separated Secrets](./ADR-003-separated-secrets-keys-json.md) - How to handle sensitive keys
- [ADR-004: Docker Compose Orchestration](./ADR-004-docker-compose-orchestration.md) - How config is mounted

## References

- [Architecture Guide - Configuration Architecture](../architecture.md#configuration-architecture)
- [Setup Guide - Configuration](../setup.md#configuration)
- [DIAL Core Documentation](https://github.com/epam/ai-dial-core)
