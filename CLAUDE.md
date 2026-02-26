# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Agent Dispatch** is a messaging protocol implementation for autonomous AI agents. The project defines the **Agent Dispatch Messaging Protocol (ADMP)** - a universal inbox standard that enables reliable, secure, structured message exchange among autonomous agents using both HTTP and SMTP transports.

### Core Concept

ADMP is built on two fundamental principles:
1. Every agent has an **inbox**
2. Every message follows the same **contract** - regardless of transport

Think of it as "SMTP for AI agents" - a universal, minimal, and extensible framework for inter-agent communication that provides deterministic delivery, security, and auditability independent of vendor or environment.

## Repository Structure

```
agentdispatch/
├── src/                # Server source code
│   ├── server.js       # Main HTTP server
│   ├── middleware/     # Auth, validation, rate limiting
│   ├── routes/         # API route handlers
│   ├── services/       # Business logic (messages, agents, DID)
│   ├── storage/        # Pluggable storage backends (memory, postgres)
│   └── utils/          # Shared utilities
├── cli/                # @agentdispatch/cli npm package
│   ├── src/            # CLI source (TypeScript)
│   ├── bin/            # Executable entry points
│   └── test/           # CLI tests
├── whitepaper/
│   └── v1.md           # Complete ADMP v1.0 specification (RFC-style)
├── docs/               # Generated documentation
│   ├── AGENT-GUIDE.md      # Integration guide for AI agents
│   ├── API-REFERENCE.md    # HTTP API reference
│   ├── CLI-REFERENCE.md    # CLI command reference
│   ├── ARCHITECTURE.md     # System architecture
│   └── ERROR-CODES.md      # Error code reference
├── examples/           # Usage examples
├── scripts/            # Deployment scripts
├── openapi.yaml        # OpenAPI 3.1 specification
└── fly.toml            # Fly.io deployment config
```

## Development Workflow

This repository uses a **structured development methodology** centered around PRD-driven task execution with confirmation gates.

### Custom Skills

The project provides four skills (use via `/skill` command):

1. **dev-workflow-orchestrator**: Full pipeline - PRD → tasks → processing with user confirmations
2. **prd-writer**: Creates detailed Product Requirements Documents with clarifying questions
3. **tasklist-generator**: Generates high-level tasks and gated sub-tasks from PRDs
4. **task-processor**: Processes task lists one sub-task at a time with test/commit protocol

### Slash Commands

- `/dev-pipeline` - Runs the full orchestrated workflow
- `/prd-writer` - Creates PRD documents in `/tasks/`
- `/generate-tasks` - Generates task lists from existing PRDs
- `/process-tasks` - Executes tasks with gated confirmations

### Workflow Phases

**Phase 1: PRD Creation**
- Ask clarifying questions to understand requirements
- Generate structured PRD following template in `ai-dev-tasks/create-prd.md`
- Save as `/tasks/[n]-prd-[feature-name].md` (zero-padded 4-digit sequence)
- Target audience: junior developers (explicit, unambiguous requirements)

**Phase 2: Task Generation**
- Read the PRD
- Generate parent tasks and present for approval ("Go")
- After approval, generate sub-tasks with relevant files and testing notes
- Save as `/tasks/tasks-[prd-file-name].md`

**Phase 3: Task Processing**
- Work one sub-task at a time
- Pause between sub-tasks for user confirmation ("yes"/"y")
- When parent task completes:
  1. Run full test suite
  2. Stage changes (`git add .`)
  3. Clean up temporary files
  4. Commit with conventional commit style and multi-`-m` messages
  5. Mark parent task complete
- Keep "Relevant Files" section accurate throughout

## Architecture Concepts

### Message Model

All ADMP messages use a canonical JSON envelope:

```json
{
  "version": "1.0",
  "id": "uuid",
  "type": "task.request",
  "from": "agent://auth.backend",
  "to": "agent://storage.pg",
  "subject": "create_user",
  "correlation_id": "c-12345",
  "headers": {"priority":"high"},
  "body": {"email":"user@example.com"},
  "ttl_sec": 86400,
  "timestamp": "2025-10-22T17:30:00Z",
  "signature": {
    "alg": "ed25519",
    "kid": "domain.com/key-2025-10-01",
    "sig": "base64..."
  }
}
```

### Message Lifecycle

```
queued → delivered → leased → acked
                    ↘
                     nack → queued
```

### Core Operations

- **SEND**: Submit message to agent's inbox
- **PULL**: Retrieve message for processing (with lease)
- **ACK**: Confirm successful processing (deletes from inbox)
- **NACK**: Reject/defer processing (requeues)
- **REPLY**: Return correlated response

### Transport Bindings

**HTTP (Internal Agents)**
- REST API at `/v1/agents/{id}/messages`
- Bearer token or HMAC authentication
- Immediate delivery (status: `delivered`)

**SMTP (Federated/External Agents)**
- Standard email with JSON body + custom `X-Agent-*` headers
- DKIM + Ed25519 signature verification
- DSN-based delivery confirmation (status: `sent` → `delivered`)
- Both transports share same PostgreSQL datastore

## Key Documentation

### Essential Reading

1. **whitepaper/v1.md** - Complete ADMP specification
   - RFC-style technical standard
   - Message format, delivery semantics, security architecture
   - Federation model using DNS + PKI
   - Interoperability with MCP, A2A, SMTP

2. **discovery/openapi-spec.md** - Implementation details
   - Full OpenAPI 3.1 spec for HTTP API
   - Node.js (Express) SMTP bridge implementation
   - Python (FastAPI) SMTP bridge implementation
   - DKIM verification + signature validation code

3. **discovery/problem.md** - Original motivation
   - Real-world issue report showing agent communication breakdown
   - Two agents (backend service + client) unable to coordinate
   - Demonstrates need for universal messaging protocol

4. **discovery/ideation.md** - Research on SMTP integration
   - Why SMTP for agent communication
   - Authentication patterns (per-agent credentials, signed tokens)
   - Comparison with MCP and Code Mode approaches
   - Hybrid architecture recommendations

## Implementation Notes

### Security Model

- **Authentication**: Ed25519/HMAC signatures on every message
- **Authorization**: Policy-based (from→to, subject/type regex, size limits)
- **Replay Protection**: Timestamp validation (±5 minutes)
- **Key Discovery**: DNS TXT (`_agentkeys.<domain>`) or HTTPS JWKS (`/.well-known/agent-keys.json`)
- **Transport Security**: TLS required for all channels

### Delivery Guarantees

- **At-least-once delivery** with idempotency keys
- **Lease-based processing** prevents message loss during agent crashes
- **TTL expiration** for graceful timeout handling
- **Dead-letter queues** for failed messages

### Design Principles

1. **Transport Independence**: Same semantics over HTTP, SMTP, or future bindings
2. **Federation-Ready**: Uses existing DNS and PKI infrastructure
3. **Minimal Core**: Essential verbs only, extensible via message types
4. **Deterministic**: Explicit ack/nack, no silent failures
5. **Auditable**: Every state transition logged and traceable

## Development Targets

When implementing ADMP components, reference:

- **HTTP API**: See OpenAPI spec in `discovery/openapi-spec.md`
- **SMTP Bridge**: Node/Python examples in `discovery/openapi-spec.md` (lines 286-676)
- **Message Schema**: Whitepaper section 3.1 (lines 100-140)
- **Security Primitives**: Whitepaper section 6 (lines 193-236)

## Common Patterns

### Agent Registration

Agents register with central hub providing:
- Agent ID (e.g., `auth.backend`)
- SMTP credentials (for federated messaging)
- Capabilities list
- Public key (for signature verification)

### Message Sending (HTTP)

```
POST /v1/agents/{to_agent_id}/messages
Authorization: Bearer <token>
Content-Type: application/json

{...envelope...}
```

### Message Sending (SMTP)

```
From: sender@agents.domain.com
To: receiver@agents.otherdomain.com
Subject: [ADMP] task.request create_user
X-Agent-ID: auth.backend
X-Agent-Signature: <sig>
Content-Type: application/json

{...envelope...}
```

### Message Processing

1. Agent calls `POST /inbox/pull` (leases message)
2. Processes the task
3. Calls `POST /messages/{id}/ack` (removes from inbox)
4. Optionally calls `/reply` to send correlated response

## Dependencies

- **Runtime**: Node.js 20+ or Bun 1.1+
- **Storage**: In-memory (default) or PostgreSQL (set `STORAGE_BACKEND=postgres`)
- **Optional**: Mailgun for SMTP outbox, Redis for rate limiting

## Notes for Contributors

- See `CONTRIBUTING.md` for full setup guide
- Use **conventional commits** with clear commit messages
- Run `bun test` before submitting PRs
- Storage backends implement the interface in `src/storage/memory.js`

## Related Standards

- **MCP (Model Context Protocol)**: Tool integration layer - ADMP provides messaging transport for MCP endpoints
- **A2A (Agent-to-Agent)**: RPC-style tasks - ADMP offers queued messaging abstraction
- **SMTP**: Email standard - ADMP uses SMTP for federated agent communication
- **HTTP REST**: Intra-domain transport - ADMP verbs as HTTP endpoints

