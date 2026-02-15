# RedForge AI Control Tower
## Enterprise Management for Multi-Agent Node-RED Workflows

**Status:** 🚧 Planning Phase - NOT STARTED
**Prerequisites:** Requires stable multi-agent RAG system from [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI)

[![Node-RED](https://img.shields.io/badge/Node--RED-4.1.2-red)](https://nodered.org/)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue)](https://www.docker.com/)
[![Standards](https://img.shields.io/badge/Standards-96%2F100-brightgreen)](https://github.com/nagual69/RedForge-Agentic-AI)

---

## Overview

**RedForge AI Control Tower** is a TypeScript-based enterprise management platform for deploying, managing, and orchestrating fleets of collaborative AI agent workflows running on Node-RED. It provides a web-based control center with centralized governance, zero-downtime deployment, and real-time observability for distributed multi-agent systems.

### What This Project Does

Extends the [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI) composable agent primitives with enterprise capabilities:

- **🎯 AI Control Tower:** Centralized dashboard for orchestrating distributed agent workflows
- **🔐 Enterprise Governance:** RBAC, audit logging, resource quotas, policy enforcement
- **🚀 Zero-Downtime Deployment:** Blue-green deployment for agent flows without interruption
- **📊 Fleet Observability:** Real-time monitoring, distributed tracing, performance analytics
- **⚡ High Availability:** Instance redundancy, automatic failover, disaster recovery
- **🌐 Edge Support:** Hybrid cloud-edge deployments with batch synchronization

---

## What Makes This Different?

| Capability | Gen 2 (Main Project) | Gen 3 (Control Tower) |
|------------|---------------------|---------------------|
| **Deployment** | Standalone Node-RED | Distributed multi-instance fleet |
| **Target** | Individual developers | Enterprise teams |
| **Scale** | Single instance | 10-100+ distributed instances |
| **Governance** | Basic | Enterprise RBAC + audit logging |
| **Agent Updates** | Manual redeploy | Zero-downtime blue-green |
| **Monitoring** | Local debug nodes | Fleet-wide dashboards + tracing |

---

## Architecture Vision

```
┌─────────────────────────────────────────────────────────────────┐
│         Control Tower (TypeScript Management Application)       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Web UI (React/Vue)                                       │  │
│  │    ├─ Fleet Dashboard (instance status, resources)       │  │
│  │    ├─ Agent Coordination View (multi-agent workflows)    │  │
│  │    ├─ Deployment Manager (blue-green, rollback)          │  │
│  │    ├─ Performance Analytics (latency, cost, quality)     │  │
│  │    └─ Governance Console (RBAC, audit logs, quotas)      │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  REST API + WebSocket Server (Node.js/TypeScript)        │  │
│  │    ├─ Instance Management (deploy, scale, health check)  │  │
│  │    ├─ Flow Deployment API (blue-green orchestration)     │  │
│  │    ├─ Monitoring & Metrics Collection                    │  │
│  │    └─ Governance Layer (RBAC, audit, quotas)             │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            │ Manages ↓
┌─────────────────────────────────────────────────────────────────┐
│           Managed Node-RED Instances (Agent Runtimes)           │
│                                                                 │
│  Instance 1           Instance 2           Instance 3          │
│  ┌──────────────┐    ┌──────────────┐    ┌────────────┐       │
│  │ File Conv    │    │ Embeddings   │    │ Query      │       │
│  │ Text Chunk   │    │ Storage      │    │ Response   │       │
│  │ Error Handle │    │              │    │            │       │
│  └──────────────┘    └──────────────┘    └────────────┘       │
│   Node-RED 4.1.2      Node-RED 4.1.2      Node-RED 4.1.2      │
│   (2-4GB RAM)         (8GB RAM)           (4GB RAM)            │
│                                                                 │
│  Shared Services: Redis, MQTT, PostgreSQL, Prometheus          │
└─────────────────────────────────────────────────────────────────┘
```

**Key Components:**
- **Control Tower App:** TypeScript/Node.js management application with web UI
- **Management API:** REST API + WebSocket for real-time instance control
- **Node-RED Instances:** Distributed Node-RED runtimes running agent flow primitives
- **State Layer:** Redis for multi-agent coordination state (voting results, team memory)
- **Message Bus:** MQTT for async task distribution and zero-downtime deployment
- **Audit Store:** PostgreSQL for AI decision logging and compliance
- **Observability:** Prometheus + Grafana for fleet-wide metrics

---

## Core Capabilities

### 1. Zero-Downtime Agent Deployment

Update agent flows (prompts, models, logic) without dropping active inference requests:

- **Blue-Green Deployment:** Run old + new agent versions simultaneously
- **State Preservation:** Redis maintains voting results, conversation history during updates
- **Message Buffering:** MQTT queues hold incoming tasks during deployment
- **Checkpoint/Replay:** Resume long-running workflows from exact failure point

See [docs/CLUSTERING_AND_REDEPLOYMENT.md](docs/CLUSTERING_AND_REDEPLOYMENT.md) for technical details.

### 2. Multi-Agent Coordination at Scale

Orchestrate complex multi-agent patterns across distributed instances:

- **Voting Pattern:** 3 agent instances vote on best answer (consensus)
- **Refinement Pattern:** Generator → Critic → Improved Output (iterative)
- **Hierarchical Pattern:** Coordinator delegates to specialized sub-agents
- **Sequential Pattern:** Pipeline processing across multiple instances

### 3. Enterprise Governance

- **RBAC:** Role-based access control for agent teams and workflows
- **Audit Logging:** Track every AI decision (model used, prompt, response, confidence)
- **Resource Quotas:** Limit CPU, memory, LLM token usage per team/project
- **Compliance:** Export audit trails for HIPAA, SOC 2, GDPR requirements

### 4. Fleet Observability

- **Real-Time Dashboard:** Agent health, active workflows, coordination status
- **Distributed Tracing:** Track inference requests across multiple agent instances
- **Cost Analytics:** LLM token usage, embedding generation costs per agent
- **Quality Metrics:** Response accuracy, latency, user feedback aggregation

### 5. High Availability Patterns

- **Instance Redundancy:** 2-3 replicas per critical agent type
- **Automatic Failover:** Route traffic to healthy instances on failure
- **Disaster Recovery:** Backup/restore procedures for all state (Redis, PostgreSQL, flows)

---

## Project Status

### Current Phase: Planning
- [ ] Gen 2 validation from main project (prerequisite)
- [ ] Control Tower architecture finalization
- [ ] Deployment strategy selection (Docker Compose, Kubernetes, etc.)

### Phase 1: Foundation (Weeks 1-3)
- [ ] **Week 1:**
  - TypeScript/Node.js backend setup (Express + WebSocket server)
  - Database schema design (PostgreSQL for Control Tower metadata)
  - Docker infrastructure setup (Redis, MQTT, PostgreSQL, Prometheus)
- [ ] **Week 2:**
  - REST API for Node-RED instance management (deploy, start, stop, health)
  - Node-RED Admin API integration
  - Basic frontend scaffolding (React/Vue)
- [ ] **Week 3:**
  - Web UI: Fleet dashboard (instance list, status, resources)
  - Instance deployment automation (Docker containers or K8s pods)
  - Health monitoring and metrics collection

### Phase 2: Zero-Downtime Deployment (Weeks 4-6)
- [ ] **Week 4:**
  - Blue-green deployment orchestration API
  - Deploy state machine (DRAIN → DEPLOY → VERIFY → RESUME → AUDIT)
  - MQTT message buffering integration
- [ ] **Week 5:**
  - State preservation coordination (Redis context stores)
  - Checkpoint/replay mechanism for long-running workflows
  - Rollback automation
- [ ] **Week 6:**
  - Web UI: Deployment manager (initiate deploy, view progress, rollback)
  - Real-time WebSocket updates for deploy phases
  - Deploy history and audit trail

### Phase 3: Governance & Observability (Weeks 7-9)
- [ ] **Week 7:**
  - RBAC implementation (users, roles, permissions)
  - Authentication layer (JWT or OAuth2)
  - Authorization middleware for API endpoints
- [ ] **Week 8:**
  - Audit logging API (track all Control Tower actions)
  - AI decision logging integration (from Node-RED instances)
  - Compliance report generation
- [ ] **Week 9:**
  - Web UI: Fleet monitoring dashboards
  - Grafana integration (embed dashboards in Control Tower UI)
  - Alerting configuration UI

### Phase 4: Production Hardening (Weeks 10-12)
- [ ] **Week 10:**
  - High availability for Control Tower itself (multi-instance, load balancing)
  - Node-RED instance redundancy management (2-3 replicas per agent type)
  - Automatic failover detection and routing
- [ ] **Week 11:**
  - Disaster recovery procedures (backup/restore)
  - Control Tower database backups (PostgreSQL)
  - Node-RED flow version control integration (Git)
- [ ] **Week 12:**
  - Performance optimization (API response times, WebSocket scaling)
  - Load testing (1,000+ concurrent workflows)
  - Security hardening (TLS, rate limiting, input validation)

### Phase 5: Advanced Features (Weeks 13-16)
- [ ] **Week 13:** Multi-region deployment support (region-aware routing)
- [ ] **Week 14:** Edge agent coordination (cloud-edge synchronization)
- [ ] **Week 15:** Advanced multi-agent patterns (canary deployments, A/B testing)
- [ ] **Week 16:** AI model marketplace integration (model versioning, cost comparison)

**Full roadmap:** [docs/project_plan.md](docs/project_plan.md)

---

## Why a Separate Project?

The Control Tower represents a **fundamentally different deployment architecture** (distributed instances vs standalone Node-RED). Separating it from the main project allows:

✅ **Focus main project** on agent primitive development and standards alignment
✅ **Validate Gen 2 benefits** before investing in Gen 3 infrastructure
✅ **Allow user choice:** Standalone Node-RED OR enterprise fleet management
✅ **Independent timelines:** Gen 3 doesn't block Gen 2 development
✅ **Different audiences:** Individual developers (Gen 2) vs Enterprise teams (Gen 3)

---

## Control Tower API (Preview)

The Control Tower exposes a REST API + WebSocket interface for programmatic management:

```typescript
// Deploy a new Node-RED instance with agent flows
POST /api/instances
{
  "name": "embeddings-agent-1",
  "flows": ["generate-embeddings-ollama.json"],
  "resources": { "cpu": "2", "memory": "8Gi" },
  "env": {
    "OLLAMA_HOST": "host.docker.internal",
    "OLLAMA_MODEL": "nomic-embed-text"
  }
}

// Start blue-green deployment
POST /api/deployments
{
  "instanceId": "embeddings-agent-1",
  "flowUpdate": "generate-embeddings-ollama-v2.json",
  "strategy": "blue-green"
}

// Get deployment status (WebSocket)
ws://localhost:3000/ws/deployments/:deploymentId
{
  "phase": "DRAIN",
  "messagesHeld": 42,
  "inFlightCount": 3,
  "progress": 0.4
}

// Query fleet metrics
GET /api/metrics/fleet
{
  "totalInstances": 12,
  "healthyInstances": 11,
  "activeWorkflows": 487,
  "avgLatency": 245,
  "totalCost": 12.47
}

// Audit log query
GET /api/audit?instanceId=query-agent-1&limit=100
[
  {
    "timestamp": "2026-02-15T10:30:00Z",
    "action": "AI_DECISION",
    "model": "llama3.2",
    "prompt": "...",
    "response": "...",
    "confidence": 0.92
  }
]
```

Full API documentation will be available at `/api/docs` (OpenAPI/Swagger).

---

## Prerequisites

### Required from Main Project
- ✅ [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI) Gen 2 complete (96/100 standards alignment)
- ✅ 8 agent primitives stable (file converter, chunker, embeddings, storage, query, response, error handler, memory)
- ✅ Multi-agent team patterns validated (voting, refinement, hierarchical)
- ✅ Production stability metrics (4+ weeks uptime on standalone)

### Required Infrastructure
- **Docker & Docker Compose** (or Kubernetes for production)
- **Redis** (agent state + coordination)
- **MQTT Broker** (Mosquitto or equivalent)
- **PostgreSQL** (audit logging + pgvector)
- **Prometheus + Grafana** (optional but recommended for observability)

### Required Performance Data
Before starting Control Tower deployment:
- Gen 2 performance benchmarks (latency, quality, cost)
- Resource utilization per primitive (CPU, memory profiles)
- Multi-agent coordination overhead measurements

---

## Quick Start (Coming Soon)

```bash
# Clone repository
git clone https://github.com/nagual69/nodered-flowfuse-control-tower.git
cd nodered-flowfuse-control-tower

# Install Control Tower dependencies
npm install

# Deploy infrastructure (Redis, MQTT, PostgreSQL, Prometheus)
cd docker
docker-compose up -d

# Start Control Tower application
cd ..
npm run dev

# Access Control Tower web UI
open http://localhost:3000

# Deploy your first Node-RED agent instance
curl -X POST http://localhost:3000/api/instances \
  -H "Content-Type: application/json" \
  -d '{"name": "embeddings-agent", "flows": "examples/flows/embeddings.json"}'
```

*(Full quick start guide will be published when Phase 1 begins)*

---

## Documentation

### Planning & Architecture
- **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** - Control Tower technical architecture
- **[docs/project_plan.md](docs/project_plan.md)** - Detailed implementation roadmap
- **[docs/DEPLOYMENT_STRATEGIES.md](docs/DEPLOYMENT_STRATEGIES.md)** - Deployment patterns (Docker, K8s, edge)

### Technical Guides
- **[docs/CLUSTERING_AND_REDEPLOYMENT.md](docs/CLUSTERING_AND_REDEPLOYMENT.md)** - Zero-downtime deployment architecture
- **[docs/MULTI_AGENT_PATTERNS.md](docs/MULTI_AGENT_PATTERNS.md)** - Coordination patterns at scale (coming soon)
- **[docs/GOVERNANCE_GUIDE.md](docs/GOVERNANCE_GUIDE.md)** - RBAC, audit logging, compliance (coming soon)
- **[docs/OBSERVABILITY_GUIDE.md](docs/OBSERVABILITY_GUIDE.md)** - Fleet monitoring setup (coming soon)

---

## Success Metrics

### Technical
- **Uptime:** 99.9% across entire agent fleet
- **Latency:** <100ms inter-instance coordination overhead
- **Scale:** Support 1,000+ concurrent multi-agent workflows
- **Deployment Speed:** <5 minutes for zero-downtime agent updates

### Business
- **Developer Productivity:** 50% reduction in deployment/troubleshooting time
- **Cost Efficiency:** 30% reduction in cost per workflow via resource optimization
- **Compliance:** 100% audit coverage for AI decisions
- **Quality:** 95%+ accuracy on multi-agent voting patterns

---

## Relationship to Main Project

**Main Project** ([RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI)):
- **Gen 1:** Production RAG pipeline ✅ Complete
- **Gen 2:** Multi-agent architecture (8 primitives, team patterns, 96/100 standards) ✅ Complete
- **Deployment:** Standalone Node-RED (single instance)
- **Target:** Individual developers, single-team deployments

**THIS Project** (Control Tower):
- **Gen 3:** Enterprise fleet management 📋 Planned
- **Deployment:** Distributed multi-instance Node-RED fleet
- **Target:** Enterprise teams, multi-tenant systems
- **Builds Upon:** Gen 2 agent primitives + multi-agent patterns

---

## Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Control Tower Backend** | TypeScript + Node.js + Express | Management API server |
| **Control Tower Frontend** | React or Vue.js | Web-based management UI |
| **Agent Runtime** | Node-RED 4.1.2 | Flow-based agent orchestration |
| **State Store** | Redis | Multi-agent coordination state |
| **Message Bus** | MQTT (Mosquitto) | Async task distribution |
| **Vector DB** | Milvus or pgvector | Embeddings storage |
| **Audit Store** | PostgreSQL | AI decision logging + Control Tower metadata |
| **LLM** | Ollama (local) or OpenAI/Anthropic APIs | Inference |
| **Observability** | Prometheus + Grafana | Fleet monitoring |
| **Orchestration** | Docker Compose or Kubernetes | Container management |
| **Real-time Updates** | WebSocket (Socket.io) | Live dashboard updates |

---

## Use Cases

### 1. Enterprise Document Intelligence
- Deploy 20+ agent instances across 3 regions
- Multi-agent voting for compliance-critical documents
- Zero-downtime updates to improve agent prompts
- Complete audit trail for regulatory compliance

### 2. Multi-Tenant AI Platforms
- Isolated agent instances per customer
- Resource quotas and cost tracking per tenant
- Centralized governance and monitoring
- High availability with automatic failover

### 3. Hybrid Cloud-Edge AI
- Edge agents process sensitive data locally
- Cloud Control Tower coordinates overall workflow
- Batch synchronization for vectors and results
- Low-latency edge inference + scalable cloud analytics

### 4. AI Research & Development
- Deploy experimental agent versions alongside production
- A/B test different models and prompts
- Gradual rollout with canary deployments
- Performance analytics for model comparison

---

## Contributing

This project follows the same contribution guidelines as the main [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI) project.

**Note:** This project is currently in the planning phase. Contributions will be welcome once implementation begins (post-Gen 2 validation).

---

## License

MIT License - See LICENSE file for details

---

## Questions & Support

- **Main Project:** [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI)
- **Issues:** [GitHub Issues](https://github.com/nagual69/nodered-flowfuse-control-tower/issues)
- **Discussions:** [GitHub Discussions](https://github.com/nagual69/nodered-flowfuse-control-tower/discussions)
- **Node-RED Docs:** [nodered.org/docs](https://nodered.org/docs/)

---

**Built with ❤️ on Node-RED** | Enterprise-grade AI orchestration without the complexity
