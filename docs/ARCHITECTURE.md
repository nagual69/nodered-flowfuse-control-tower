# RedForge AI Control Tower - Architecture

**Status**: 🚧 Planning Phase - Architecture Design In Progress
**Last Updated**: 2026-02-15

---

## Overview

This document will describe the architecture of the RedForge AI Control Tower for managing distributed multi-agent Node-RED workflows. The Control Tower provides enterprise-grade fleet management, governance, and observability for the primitives developed in the main [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI) project.

---

## Architecture Source

The detailed Control Tower architecture is currently documented in:
- **Source**: `docs/Node-RED AI Control Tower Architecture.rtf` (from main project)
- **Status**: To be converted to Markdown and expanded

**Placeholder Note**: Full architecture documentation will be created during Phase 5.1 (FlowFuse Platform Foundation). The architecture will include:

1. **FlowFuse Platform Layer**
   - Teams, Applications, Instances hierarchy
   - RBAC and governance model
   - Stack definitions (Standard, Agent, Edge)

2. **Control Tower Instance**
   - Dashboard architecture (5 panels)
   - Coordination flows (ingest, query, health monitor, resource manager)
   - Inter-instance messaging (MQTT, REST, WebSocket)

3. **Distributed Primitives**
   - 8 primitive instances (File Converter, Text Chunker, Embeddings, Storage, Query, Response, Error Handler, Hindsight)
   - Instance type mapping (Standard vs Agent Stack)
   - Communication patterns

4. **Observability Stack**
   - Prometheus + Grafana integration
   - OpenTelemetry distributed tracing
   - Alerting and monitoring

5. **Edge Architecture** (Optional)
   - Edge instance patterns
   - Cloud-edge synchronization
   - Batch upload strategies

---

## High-Level Architecture (Placeholder)

```
┌──────────────────────────────────────────────────────────────┐
│                    FlowFuse Platform Layer                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Teams → Applications → Instances                      │  │
│  │    ├─ File Converter (Standard Stack)                  │  │
│  │    ├─ Embeddings Gen (Agent Stack - 8GB)               │  │
│  │    ├─ Vector Storage (Agent Stack - 8GB)               │  │
│  │    └─ ... (all 8 primitives)                           │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Central Control Tower (Node-RED Dashboard 2.0)        │  │
│  │    ├─ Fleet Overview (health, resources, status)       │  │
│  │    ├─ Agent Activity Monitor (votes, coordination)     │  │
│  │    ├─ Workflow Orchestration (trigger, monitor)        │  │
│  │    ├─ Performance Analytics (latency, cost)            │  │
│  │    └─ Error Dashboard (failures, retry queue)          │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                        ↓ ↑ (MQTT/REST/WebSocket)
        External Services: Ollama, Milvus, pgvector, Hindsight
```

---

## Key Architectural Principles

### 1. Primitive Compatibility
- All primitives from the main project are fully compatible with FlowFuse deployment
- Minimal code changes required (mainly inter-instance communication)
- Message contracts (JSON schemas) work across distributed instances

### 2. Resource Isolation
- Each primitive runs on a dedicated FlowFuse instance
- Instance types sized for workload:
  - **Standard Stack** (2 vCPU, 4GB RAM): File Converter, Text Chunker, Error Handler
  - **Agent Stack** (4 vCPU, 8GB RAM): Embeddings, Storage, Query, Response, Hindsight

### 3. Centralized Orchestration
- Control Tower instance coordinates all workflows
- Distributes tasks to appropriate primitive instances
- Monitors health and performance across fleet

### 4. Enterprise Governance
- RBAC for Teams and Applications
- Audit logging for all workflow executions
- Resource quotas and policy enforcement
- Secrets management integration

### 5. High Availability
- Multiple instances per critical primitive
- Automatic failover on instance failure
- Workflow re-queue mechanisms

### 6. Observability
- Real-time dashboards (Node-RED Dashboard 2.0)
- Distributed tracing (OpenTelemetry)
- Fleet-wide metrics (Prometheus + Grafana)

---

## Design Decisions (To Be Documented)

The following architectural decisions will be documented during Phase 5:

1. **FlowFuse Deployment Model**: Cloud vs Self-Hosted
2. **Inter-Instance Communication**: HTTP vs MQTT vs WebSocket
3. **State Management**: Centralized vs Distributed
4. **Load Balancing Strategy**: Control Tower routing logic
5. **Caching Architecture**: Where to cache (Control Tower vs Primitives)
6. **Edge Deployment**: When to use edge instances
7. **Security Model**: Authentication, authorization, encryption

---

## Related Documentation

**Main Project**:
- [RedForge-Agentic-AI Architecture](https://github.com/nagual69/RedForge-Agentic-AI/blob/main/docs/ARCHITECTURE.md)
- Main project describes Gen 1 (RAG) + Gen 2 (Multi-Agent) architecture

**This Project**:
- [README.md](../README.md) - Project overview and status
- [project_plan.md](project_plan.md) - Phase 5-6 roadmap
- [MIGRATION_STRATEGY.md](MIGRATION_STRATEGY.md) - Migration assessment from standalone to FlowFuse
- [CLUSTERING_AND_REDEPLOYMENT.md](CLUSTERING_AND_REDEPLOYMENT.md) - Zero-downtime deployment and clustering architecture

**Architecture Source Materials**:
- `docs/Node-RED AI Control Tower Architecture.rtf` (main project) - To be converted
- `docs/FLOWFUSE_MIGRATION_ASSESSMENT.md` (main project) - Moved to MIGRATION_STRATEGY.md

---

## Next Steps

This architecture document will be fully developed during:
- **Phase 5.1** (Weeks 1-2): FlowFuse Platform Foundation
- **Phase 5.3** (Weeks 5-7): Control Tower Implementation

**Development Tasks**:
1. Convert RTF architecture document to Markdown
2. Add detailed diagrams (FlowFuse hierarchy, Control Tower flows, communication patterns)
3. Document design decisions and rationale
4. Create deployment architecture (cloud, self-hosted, edge)
5. Define security architecture (authentication, RBAC, secrets)

---

**Note**: This is a placeholder document. Full architecture details will be added as Phase 5 development progresses.
