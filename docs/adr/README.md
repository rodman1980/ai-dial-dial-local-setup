# Architecture Decision Records (ADR)

## Overview

This directory contains Architecture Decision Records (ADRs) documenting significant design decisions made for the DIAL Local Setup project.

## ADR Format

Each ADR follows this structure:

```markdown
# ADR-XXX: [Decision Title]

**Status:** Accepted | Rejected | Superseded | Deprecated

**Date:** YYYY-MM-DD

**Deciders:** [Names/Roles]

## Context

What is the issue we're facing?

## Decision

What did we decide to do?

## Consequences

What are the positive and negative outcomes?

## Alternatives Considered

What other options did we evaluate?
```

## Index of ADRs

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [ADR-001](./ADR-001-configuration-driven-architecture.md) | Configuration-Driven Architecture | Accepted | 2025-12-30 |
| [ADR-002](./ADR-002-local-development-host-docker-internal.md) | Local Development via host.docker.internal | Accepted | 2025-12-30 |
| [ADR-003](./ADR-003-separated-secrets-keys-json.md) | Separated Secrets (keys.json) | Accepted | 2025-12-30 |
| [ADR-004](./ADR-004-docker-compose-orchestration.md) | Docker Compose for Orchestration | Accepted | 2025-12-30 |
| [ADR-005](./ADR-005-task-based-learning.md) | Task-Based Learning Progression | Accepted | 2025-12-30 |

## Contributing

When creating a new ADR:

1. Use the next available number: `ADR-XXX`
2. Create file: `ADR-XXX-short-title.md`
3. Update this README index
4. Link from relevant documentation

## ADR Lifecycle

- **Proposed:** Under discussion
- **Accepted:** Decision made, implemented
- **Rejected:** Evaluated but not adopted
- **Superseded:** Replaced by newer ADR
- **Deprecated:** No longer relevant
