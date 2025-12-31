# ADR-003: Separated Secrets (keys.json)

**Status:** Accepted

**Date:** 2025-12-30

**Deciders:** Project Architects, Security Team

## Context

DIAL applications require API keys for upstream model providers (OpenAI, Anthropic, Google). These keys must be:
- **Kept Secret:** Not exposed in version control
- **Easy to Use:** Simple for learners to configure
- **Flexible:** Support multiple environments (dev, test)

We need to decide where and how to store these sensitive credentials.

## Decision

**Split configuration into two files:**

1. **`core/config.json`** (public, tracked in Git)
   - Contains model structure, displayName, endpoint
   - NO sensitive upstreams

2. **`core/keys.json`** (secret, gitignored)
   - Contains only `upstreams` with API keys
   - Merged with config.json at Core startup

**DIAL Core automatically merges both files at runtime.**

## Configuration Pattern

### Public Config (config.json)

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

### Secret Config (keys.json)

```json
{
  "models": {
    "gpt-4o": {
      "upstreams": [
        {
          "endpoint": "https://ai-proxy.lab.epam.com/openai/deployments/gpt-4o/chat/completions",
          "key": "sk-proj-abc123..."
        }
      ]
    }
  }
}
```

### Merged Result (Runtime)

Core combines both:
```json
{
  "models": {
    "gpt-4o": {
      "displayName": "GPT-4o",
      "endpoint": "http://adapter-dial:5000/openai/deployments/gpt-4o/chat/completions",
      "type": "chat",
      "upstreams": [
        {
          "endpoint": "https://ai-proxy.lab.epam.com/...",
          "key": "sk-proj-abc123..."
        }
      ]
    }
  }
}
```

## Rationale

### Why Separate Files?

**Problem with Single File:**
```json
{
  "models": {
    "gpt-4o": {
      "upstreams": [{
        "key": "sk-proj-abc123..."  // ❌ Secret in Git!
      }]
    }
  }
}
```

If config.json is tracked in Git:
- ❌ API keys visible in repository history
- ❌ Keys exposed if repo becomes public
- ❌ Keys in pull requests/code reviews
- ❌ Hard to rotate keys (need to rewrite Git history)

**Solution with Separate Files:**
- ✅ config.json tracked in Git (no secrets)
- ✅ keys.json in .gitignore (never committed)
- ✅ Can share config.json safely
- ✅ Easy to have per-developer keys
- ✅ Simple key rotation

### Why Not Other Solutions?

**Environment Variables:**
```bash
export GPT4O_API_KEY="sk-proj-..."
```

**Cons:**
- ❌ DIAL Core doesn't support env var substitution in upstreams
- ❌ Must configure each key separately
- ❌ Harder to manage many keys
- ❌ Not visible in one place

**Encrypted Config:**
```bash
ansible-vault encrypt keys.json
```

**Cons:**
- ❌ Additional tooling complexity
- ❌ Key sharing requires secure channel
- ❌ Overkill for learning environment

**Secret Management Service:**
- AWS Secrets Manager
- HashiCorp Vault

**Cons:**
- ❌ Requires external infrastructure
- ❌ Too complex for local learning
- ❌ Network dependency

## Consequences

### Positive

- **Security:** API keys never in Git
- **Simplicity:** Still JSON-based (familiar)
- **Portability:** Can share config.json without secrets
- **Flexibility:** Each developer has own keys.json
- **Version Control:** Public config changes tracked
- **Onboarding:** Clear separation of public vs secret

### Negative

- **Two Files:** Must maintain consistency between config.json and keys.json
- **Setup Step:** Learners must create keys.json manually
- **Merge Errors:** Typo in model name → upstream not merged
- **No Validation:** Core doesn't validate keys.json structure until runtime
- **Documentation:** Must explain split config pattern

### Mitigation Strategies

**Problem: Forgetting to create keys.json**

**Solution:** Provide template:
```bash
# keys.json.template (tracked in Git)
{
  "models": {
    "gpt-4o": {
      "upstreams": [{
        "endpoint": "https://...",
        "key": "YOUR_API_KEY_HERE"
      }]
    }
  }
}
```

**Problem: Accidental commit of keys.json**

**Solution:**
1. Add to .gitignore: `core/keys.json`
2. Pre-commit hook to check for API key patterns
3. Clear README warning

## Implementation

### .gitignore Configuration

```gitignore
# Secrets - never commit
core/keys.json
```

Verify:
```bash
git status core/keys.json
# Should show: not tracked
```

### keys.json Structure

**Minimal:**
```json
{
  "models": {
    "<model-id-from-config>": {
      "upstreams": [
        {
          "endpoint": "https://...",
          "key": "actual-api-key"
        }
      ]
    }
  }
}
```

**Multi-Upstream:**
```json
{
  "models": {
    "gpt-4o": {
      "upstreams": [
        {
          "endpoint": "https://primary-api.com/...",
          "key": "key1"
        },
        {
          "endpoint": "https://fallback-api.com/...",
          "key": "key2"
        }
      ]
    }
  }
}
```

### Setup Instructions

**Step 1: Create keys.json**
```bash
cp keys.json.template core/keys.json
```

**Step 2: Add API keys**
```bash
nano core/keys.json
# Replace YOUR_API_KEY_HERE with actual keys
```

**Step 3: Verify not tracked**
```bash
git status | grep keys.json
# Should show nothing (or "untracked" if not in .gitignore)
```

**Step 4: Restart Core**
```bash
docker compose stop core && docker compose up -d core
```

**Step 5: Verify merge**

Check Core logs:
```bash
docker compose logs core | grep -i "upstreams"
```

### Validation Script

```python
# tools/validate_keys.py
import json
import sys

def validate_keys():
    """Check keys.json matches config.json model IDs"""
    
    try:
        with open('core/config.json') as f:
            config = json.load(f)
    except Exception as e:
        print(f"❌ Can't read config.json: {e}")
        return False
    
    try:
        with open('core/keys.json') as f:
            keys = json.load(f)
    except FileNotFoundError:
        print("⚠️  No keys.json found (optional)")
        return True
    except Exception as e:
        print(f"❌ Can't read keys.json: {e}")
        return False
    
    config_models = set(config.get('models', {}).keys())
    keys_models = set(keys.get('models', {}).keys())
    
    missing = keys_models - config_models
    if missing:
        print(f"⚠️  keys.json has models not in config.json: {missing}")
    
    print(f"✅ Found keys for: {', '.join(keys_models)}")
    return True

if __name__ == "__main__":
    sys.exit(0 if validate_keys() else 1)
```

### README Warning

Add prominent section:

```markdown
## ⚠️ IMPORTANT: Never Commit API Keys

**After completing all tasks, remove API keys from config files:**

1. Move sensitive upstreams to `core/keys.json`
2. Verify `.gitignore` includes `core/keys.json`
3. Remove keys from any committed files
4. Check Git history doesn't contain keys

See [ADR-003](docs/adr/ADR-003-separated-secrets-keys-json.md) for details.
```

## Alternatives Considered

### Alternative 1: Single Config with Placeholders

```json
{
  "upstreams": [{
    "key": "${OPENAI_API_KEY}"
  }]
}
```

Then set env var or use tool to replace placeholders.

**Rejected because:**
- DIAL Core doesn't support env var substitution
- Requires additional tooling
- Less clear to learners

### Alternative 2: Encrypted Config File

```bash
# Encrypt entire config
openssl enc -aes-256-cbc -in config.json -out config.json.enc

# Decrypt at runtime
openssl enc -d -aes-256-cbc -in config.json.enc -out config.json
```

**Rejected because:**
- Complexity for learning environment
- Key management problem (where to store decryption key?)
- Not supported by DIAL Core natively

### Alternative 3: Per-Model Config Files

```
models/
  gpt-4o/
    config.json      (public)
    secrets.json     (gitignored)
```

**Rejected because:**
- Harder to see full system configuration
- More file management
- DIAL Core expects single or two files

## Security Recommendations

### For Learners

1. **Never commit keys:**
   - Always use keys.json
   - Double-check before `git push`

2. **Use restricted keys:**
   - Create separate API keys for learning
   - Set spending limits on provider dashboards

3. **Rotate after sharing:**
   - If keys accidentally exposed, rotate immediately

### For Instructors

1. **Provide template:**
   - Include keys.json.template in repo
   - Show example without real keys

2. **Demo splitting:**
   - Show config.json (safe to share)
   - Show keys.json (keep private)

3. **Check submissions:**
   - Scan for API key patterns before accepting
   - Use Git hooks to prevent commits

### Key Detection Patterns

```bash
# Check for accidental key commits
git grep -E 'sk-[a-zA-Z0-9]{32,}' core/

# Check for common key patterns
git grep -E '(api[_-]?key|secret[_-]?key|bearer[_-]?token)[\s]*[:=][\s]*["\047]?[a-zA-Z0-9_-]{20,}' core/
```

## Related ADRs

- [ADR-001: Configuration-Driven Architecture](./ADR-001-configuration-driven-architecture.md) - Overall config approach
- [ADR-004: Docker Compose Orchestration](./ADR-004-docker-compose-orchestration.md) - How configs are mounted

## References

- [Setup Guide - Securing API Keys](../setup.md#securing-api-keys)
- [Architecture Guide - Security Considerations](../architecture.md#security-considerations)
- [README - API Key Warning](../../README.md)
