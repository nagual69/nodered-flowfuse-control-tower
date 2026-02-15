# Node-RED Clustering & Zero-Downtime Redeployment Architecture

> **Status**: Assessment Complete | **Version**: 1.0 | **Date**: 2026-02-14
>
> This document provides an architectural assessment for clustering Node-RED instances and achieving zero-downtime flow redeployment with full state preservation. It targets the Gen 3 / Control Tower phase of RedForge Agentic AI.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Current State Analysis](#2-current-state-analysis)
3. [Node-RED Runtime Internals](#3-node-red-runtime-internals)
4. [Clustering Architecture](#4-clustering-architecture)
5. [Zero-Downtime Redeployment Strategy](#5-zero-downtime-redeployment-strategy)
6. [State Management Architecture](#6-state-management-architecture)
7. [Outbound Message Tracking & Timeline](#7-outbound-message-tracking--timeline)
8. [Infrastructure Components](#8-infrastructure-components)
9. [Implementation Roadmap](#9-implementation-roadmap)
10. [Key Architectural Decisions](#10-key-architectural-decisions)
11. [Verification Plan](#11-verification-plan)
12. [Appendix: Reference Configurations](#appendix-reference-configurations)

---

## 1. Problem Statement

A developer has updated an existing Node-RED flow, passed all tests, and is ready to deploy to production. The current flow is **live and actively processing data**. We need to answer:

1. **How do we replace the flow without losing messages currently in the pipeline?**
2. **How do we capture and hold incoming data during the deployment window?**
3. **How do we re-inject held and checkpointed data after the new flow is live?**
4. **How do we track recent outbound messages for state timeline reconstruction?**
5. **How do we scale this to multiple Node-RED instances (clustering)?**

The solution must be architecturally sound, production-grade, and compatible with the existing RedForge message contract (schemas in `schemas/msg_input.json` and `schemas/msg_output.json`).

---

## 2. Current State Analysis

### What Exists Today

| Component | Current State | Gap |
|-----------|--------------|-----|
| **Node-RED instances** | Single container (`docker/docker-compose.yml:26-84`) | No redundancy; single point of failure |
| **Flow state** | In-memory context (`flow.get()`/`flow.set()`) | Lost on restart; not shared across instances |
| **Message transport** | Link In/Out nodes (in-process IPC) | No buffering; messages dropped on redeploy |
| **Context persistence** | Default `memory` store | No persistence to disk or external store |
| **Load balancer** | None | No traffic routing or draining capability |
| **Message broker** | None | No decoupling of producers from consumers |
| **Audit trail** | Debug nodes only | No persistent record of processed messages |
| **ACP checkpoints** | Schema defined (`schemas/msg_input.json:75-97`) | Not yet operationalized in flows |
| **Dead-letter queue** | In-memory (`error-handler.json`) | Lost on restart |
| **Deploy mechanism** | Manual UI button or API call | Full restart; no drain/verify cycle |

### Key Risks During Current Redeployment

```
[User clicks Deploy]
        │
        ▼
┌──────────────────────────────────────────────┐
│ Node-RED Runtime                              │
│                                               │
│  msg-A ──► [Chunker] ──► [Embedder] ──►  ✗  │ ← msg-A LOST (mid-pipeline)
│  msg-B ──► [Chunker] ──►  ✗                  │ ← msg-B LOST (mid-pipeline)
│  msg-C ──►  ✗                                 │ ← msg-C LOST (just entered)
│                                               │
│  flow.get('teams') = {...}  →  undefined      │ ← State LOST
│  flow.get('retry_queue') = [...]  →  []       │ ← Retry queue LOST
│                                               │
│  [New flow version starts with clean slate]   │
└──────────────────────────────────────────────┘
```

**Impact**: Every message in the pipeline at deploy time is silently dropped. Flow context data (team registrations, retry queues, circuit breaker state, cached strategies) is wiped. No record exists of what was lost.

---

## 3. Node-RED Runtime Internals

Understanding how Node-RED handles deployment internally is critical to designing the solution.

### 3.1 Deploy Types

Node-RED's Admin API (`POST /flows`) accepts a `deploymentType` header:

| Type | Behavior | Impact |
|------|----------|--------|
| `full` | Stops ALL flows, reloads everything, starts ALL flows | Most disruptive; every message in every flow is dropped |
| `flows` | Stops only *modified* flow tabs, starts them with new config | Other tabs keep running; messages in modified tabs are dropped |
| `nodes` | Stops only *changed* nodes, restarts them in place | Least disruptive; wiring changes may still require tab restart |

**The `flows` type is the key enabler**. It means we can update a single flow tab (e.g., Ingest-Simple) without disrupting other tabs (e.g., Query-Simple, Error-Handler). This reduces the blast radius from "entire system" to "single pipeline."

### 3.2 Flow Lifecycle During Deploy

When a flow tab is restarted:

1. **`close` event** fires on every node in the tab (with `done` callback)
2. Nodes have a configurable timeout to finish cleanup (default: 15 seconds)
3. After all nodes close (or timeout), the tab is removed from the runtime
4. New tab configuration is loaded and nodes are instantiated
5. **`input` handlers** are registered on new nodes
6. Flow is ready to receive messages

**Critical gap**: Between steps 2 and 5, any message arriving at a Link In or MQTT In node for that tab is **silently dropped** because no input handler exists.

### 3.3 Context Store Architecture

Node-RED supports pluggable context stores since v0.19:

```javascript
// settings.js
contextStorage: {
    default: { module: "memory" },           // Current: volatile
    persistent: { module: "localfilesystem" } // Option: file-based
    shared: { module: "redis-context" }       // Option: Redis-backed
}
```

Nodes select stores explicitly:
```javascript
flow.set('key', value);              // Uses 'default' store
flow.set('key', value, 'persistent'); // Uses named 'persistent' store
flow.set('key', value, 'shared');     // Uses named 'shared' store
```

**Key insight**: By changing the `default` store to Redis, ALL existing `flow.get()`/`flow.set()` calls in the codebase automatically become persistent and shared -- **zero code changes required**.

### 3.4 Graceful Shutdown

Node-RED supports `gracefulShutdown` in `settings.js`:

```javascript
gracefulShutdown: {
    timeout: 30000,  // ms to wait for nodes to close
    signals: ['SIGINT', 'SIGTERM']
}
```

This gives in-flight messages up to 30 seconds to complete before forced termination. However, it only applies to full process shutdown, not to flow-level redeploy. We need a separate drain mechanism for flow-level operations.

---

## 4. Clustering Architecture

### 4.1 Recommended Topology: Blue-Green with Shared State

```
                         ┌─────────────────────┐
                         │   Nginx (Layer 7)    │
                         │   - Sticky sessions  │
                         │   - WebSocket proxy  │
                         │   - Health checks    │
                         │   - Traffic drain    │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
              ┌───────────┐  ┌───────────┐  ┌───────────┐
              │  NR-Blue   │  │  NR-Green  │  │ NR-Canary │
              │  (Active)  │  │  (Standby) │  │ (Optional)│
              │  Port 1880 │  │  Port 1881 │  │ Port 1882 │
              └─────┬──────┘  └─────┬──────┘  └─────┬─────┘
                    │               │               │
              ┌─────┴───────────────┴───────────────┴─────┐
              │           Shared Infrastructure            │
              │                                            │
              │  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
              │  │  Redis    │  │ PostgreSQL│  │ Mosquitto│ │
              │  │  6379     │  │  5432     │  │  1883    │ │
              │  │           │  │           │  │          │ │
              │  │ - Context │  │ - Vectors │  │ - Hold Q │ │
              │  │ - Locks   │  │ - Checks  │  │ - Events │ │
              │  │ - PubSub  │  │ - Audit   │  │ - DLQ    │ │
              │  └──────────┘  └──────────┘  └──────────┘ │
              └────────────────────────────────────────────┘
```

### 4.2 Why Blue-Green

| Consideration | Blue-Green | Active-Active | Active-Passive |
|---------------|-----------|---------------|----------------|
| Zero-downtime deploy | Yes | Yes (rolling) | No (failover gap) |
| Full rollback | Instant (swap back) | Complex (roll back each) | N/A |
| State consistency | Simple (one active) | Complex (distributed) | Simple |
| Resource cost | 2x Node-RED instances | 2x+ instances | 2x instances |
| Complexity | Low | High | Low |
| Fits Node-RED model | Yes | Problematic (Link nodes are local) | Partially |

**Blue-Green is the clear winner** for Node-RED because:
- Node-RED's Link In/Out nodes are process-local. Active-Active would require replacing all Link nodes with MQTT or HTTP calls.
- Blue-Green gives instant rollback by simply routing traffic back to the previous instance.
- Only one instance processes messages at a time, avoiding distributed state coordination.

### 4.3 Instance Roles

**Blue (Active)**:
- Receives all production traffic via load balancer
- Runs the current known-good flow configuration
- Writes context to shared Redis store
- Publishes outbound tracking to PostgreSQL audit log

**Green (Standby/Deploy Target)**:
- Runs the same flow configuration as Blue (kept in sync)
- Does NOT receive production traffic (LB marks it as `down`)
- Available as the deploy target for new flow versions
- After deploy + verify, LB swaps Green to `active` and Blue to `standby`

**Canary (Optional, Phase 2)**:
- Receives a configurable % of traffic (e.g., 5%)
- Runs the new flow version for validation before full rollout
- If error rate exceeds threshold, traffic shifts back to Blue
- After validation period, Green is updated and full swap occurs

### 4.4 Shared State via Redis

All Node-RED instances connect to the same Redis instance for context storage. This means:

- Team registrations (`flow.get('teams')`) are visible to whichever instance is active
- Retry queues survive instance switches
- Circuit breaker state is shared
- No "cold start" when traffic moves from Blue to Green

**Redis also provides**:
- **Distributed locks** (`SETNX`): Prevent two instances from processing the same message during switchover
- **Pub/Sub channels**: Signal deploy-drain events to all instances
- **Expiring keys**: TTL-based cleanup for ephemeral state (mirrors the existing cleanup-hooks.json pattern)

### 4.5 What About FlowFuse?

FlowFuse (formerly FlowForge) provides multi-instance Node-RED management and could complement this architecture:

- **Fleet management**: Deploy flow updates to multiple instances from a central dashboard
- **Snapshots**: Version control for flow configurations
- **Team management**: RBAC for who can deploy what

However, FlowFuse does NOT provide:
- Message buffering during deploy
- In-flight checkpoint/replay
- Blue-green traffic routing
- Outbound audit logging

**Recommendation**: The architecture described here is compatible with FlowFuse. FlowFuse could replace the manual deploy script (Phase C) as the deployment orchestrator, while the state management layer (Redis, MQTT, PostgreSQL) handles the data safety aspects.

---

## 5. Zero-Downtime Redeployment Strategy

### 5.1 The Five-Phase Deploy Cycle

```
Time ──────────────────────────────────────────────────────────────────►

Phase 1          Phase 2          Phase 3         Phase 4       Phase 5
DRAIN            DEPLOY           VERIFY          RESUME        AUDIT
(0-30s)          (30-45s)         (45-60s)        (60s)         (60s+)

┌──────────┐    ┌──────────┐    ┌──────────┐    ┌─────────┐   ┌────────┐
│ Stop new │    │ Deploy   │    │ Smoke    │    │ Route   │   │ Compare│
│ messages │    │ new flow │    │ test new │    │ traffic │   │ before/│
│ to Green │    │ to Green │    │ flow on  │    │ to      │   │ after  │
│          │    │ via API  │    │ Green    │    │ Green   │   │ output │
│ Buffer   │    │          │    │          │    │         │   │        │
│ in MQTT  │    │ "flows"  │    │ Validate │    │ Replay  │   │ Record │
│ hold Q   │    │ type     │    │ schema   │    │ MQTT    │   │ deploy │
│          │    │ deploy   │    │ output   │    │ hold Q  │   │ event  │
│ Capture  │    │          │    │          │    │         │   │        │
│ in-flight│    │          │    │ Check    │    │ Replay  │   │ Alert  │
│ to       │    │          │    │ error    │    │ check-  │   │ if err │
│ Postgres │    │          │    │ rates    │    │ points  │   │ rate   │
│ checks   │    │          │    │          │    │         │   │ high   │
└──────────┘    └──────────┘    └──────────┘    └─────────┘   └────────┘
     │               │               │               │             │
     │               │               │               │             │
     ▼               ▼               ▼               ▼             ▼
 [LB: drain]    [API: POST]     [inject test]   [LB: active]  [pg query]
 [MQTT: hold]   [/flows]        [schema check]  [MQTT: flush]  [compare]
 [PG: ckpt]     [wait ready]    [rollback?]     [PG: replay]   [log]
```

### 5.2 Phase 1: DRAIN (T+0 to T+30s)

**Goal**: Stop new messages from entering the target flow while allowing in-flight messages to complete.

**Steps**:

1. **Signal drain start**: Publish to Redis channel `redforge:deploy:drain`
   ```
   PUBLISH redforge:deploy:drain '{"flowTab":"ingest-simple-tab","timestamp":"..."}'
   ```

2. **MQTT gate closes**: The Deploy Gate node at the flow's entry point detects the drain signal and redirects new messages to the MQTT hold queue (`redforge/hold/ingest-simple-tab`) instead of passing them into the flow.

3. **In-flight timeout**: Wait up to 30 seconds for messages currently in the pipeline to complete. The orchestrator polls the flow's "active message count" (tracked via a flow context counter incremented at entry, decremented at exit).

4. **Checkpoint stragglers**: If any messages are still in-flight after 30 seconds (e.g., waiting on a slow Ollama embedding call), the Checkpoint Gate nodes snapshot those messages to PostgreSQL:
   ```sql
   INSERT INTO rag.message_checkpoints
     (flow_id, flow_tab, node_id, stage, msg_snapshot, checkpoint_id, status)
   VALUES
     ('ingest-1707900000', 'ingest-simple-tab', 'ingest-ollama-embed',
      'execute', '{"payload":"...","metadata":{...},"_chunks":[...]}',
      'ckpt-abc123', 'pending');
   ```

5. **Capture outbound snapshot**: Query the last N outbound messages from the audit log for the target flow tab:
   ```sql
   SELECT * FROM rag.message_audit_log
   WHERE flow_tab = 'ingest-simple-tab' AND direction = 'outbound'
   ORDER BY created_at DESC LIMIT 50;
   ```
   This becomes the "before" snapshot for Phase 5 comparison.

### 5.3 Phase 2: DEPLOY (T+30s to T+45s)

**Goal**: Deploy the new flow version to the Green instance.

**Steps**:

1. **Push new flow via Admin API**:
   ```bash
   # Read existing flows from Green
   FLOWS=$(curl -s http://green:1880/flows)

   # Replace the target tab in the flow array
   UPDATED=$(echo "$FLOWS" | jq --argjson new "$(cat new-ingest-simple.json)" \
     '[.[] | select(.z != "ingest-simple-tab" and .id != "ingest-simple-tab")] + $new')

   # Deploy with "flows" type (tab-level restart only)
   curl -X POST http://green:1880/flows \
     -H "Content-Type: application/json" \
     -H "Node-RED-Deployment-Type: flows" \
     -d "$UPDATED"
   ```

2. **Wait for ready**: Poll the Green instance health endpoint until the new flow tab reports ready:
   ```bash
   until curl -sf http://green:1880/flows | jq -e '.[] | select(.id == "ingest-simple-tab")'; do
     sleep 1
   done
   ```

3. **Record deploy event**:
   ```sql
   INSERT INTO rag.deploy_events
     (flow_tab, version, instance, status, deployed_at)
   VALUES
     ('ingest-simple-tab', 'v2.1.0', 'green', 'deployed', NOW());
   ```

### 5.4 Phase 3: VERIFY (T+45s to T+60s)

**Goal**: Validate the new flow version works correctly before routing production traffic.

**Steps**:

1. **Inject synthetic test message**: Send a known-good test payload directly to the Green instance's flow:
   ```bash
   curl -X POST http://green:1880/inject/ingest-simple-tab \
     -H "Content-Type: application/json" \
     -d '{"payload":"test document for deploy verification","metadata":{"flowId":"deploy-verify-'$(date +%s)'","source":"deploy-verify","timestamp":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}}'
   ```

2. **Validate output**: Check that the flow produces a valid output matching `schemas/msg_output.json`:
   - `metadata.status === "success"`
   - `payload.inserted > 0` (for ingest flows)
   - `error === null`

3. **Check error rate**: Query recent errors on Green:
   ```sql
   SELECT COUNT(*) as errors FROM rag.message_audit_log
   WHERE flow_tab = 'ingest-simple-tab'
     AND status = 'error'
     AND created_at > NOW() - INTERVAL '5 minutes'
     AND node_id LIKE 'green-%';
   ```

4. **Rollback decision**: If verification fails:
   - Redeploy the previous flow version to Green
   - Flush MQTT hold queue back to Blue (which still has the known-good version)
   - Record rollback event
   - Alert operators

### 5.5 Phase 4: RESUME (T+60s)

**Goal**: Route traffic to the updated Green instance and replay buffered/checkpointed messages.

**Steps**:

1. **Swap load balancer**: Update Nginx upstream to route traffic to Green:
   ```bash
   # Via Nginx API or config reload
   # Mark Blue as backup, Green as primary
   ```

2. **Replay MQTT hold queue**: The Deploy Gate node on Green subscribes to the hold topic and processes buffered messages in FIFO order:
   ```
   Topic: redforge/hold/ingest-simple-tab
   QoS: 1 (at-least-once delivery)
   ```
   Messages are replayed at a configurable rate (e.g., 10/second) to avoid overwhelming the newly deployed flow.

3. **Replay checkpointed messages**: The checkpoint replay script reads pending checkpoints from PostgreSQL and re-injects them at their checkpoint stage:
   ```sql
   SELECT * FROM rag.message_checkpoints
   WHERE flow_tab = 'ingest-simple-tab'
     AND status = 'pending'
   ORDER BY created_at ASC;
   ```

   For each checkpoint:
   - Deserialize `msg_snapshot`
   - Inject into the flow at the node identified by `node_id` (using the Node-RED Admin API inject endpoint or a Link In node)
   - Mark checkpoint as `replayed`

4. **Signal drain end**: Publish to Redis:
   ```
   PUBLISH redforge:deploy:resume '{"flowTab":"ingest-simple-tab","timestamp":"..."}'
   ```
   The Deploy Gate node resumes normal message routing.

### 5.6 Phase 5: AUDIT (T+60s+)

**Goal**: Verify deployment success and create an audit trail.

**Steps**:

1. **Compare outbound logs**: Query the "after" outbound messages and compare with the Phase 1 "before" snapshot:
   - Same number of messages processed?
   - Same success rate?
   - Any unexpected error patterns?

2. **Verify checkpoint replay**: Confirm all checkpointed messages have `status = 'replayed'`:
   ```sql
   SELECT COUNT(*) as pending FROM rag.message_checkpoints
   WHERE flow_tab = 'ingest-simple-tab'
     AND status = 'pending';
   -- Expected: 0
   ```

3. **Record deployment success**:
   ```sql
   UPDATE rag.deploy_events
   SET status = 'verified', verified_at = NOW()
   WHERE flow_tab = 'ingest-simple-tab'
     AND instance = 'green'
     AND status = 'deployed';
   ```

4. **Sync Blue**: Deploy the same new flow version to Blue (now standby) so both instances are identical. This ensures Blue is ready as the rollback target for the next deployment.

---

## 6. State Management Architecture

### 6.1 Persistent Context Store (Redis)

**Purpose**: Replace volatile in-memory context with persistent, shared state.

**Node-RED `settings.js` configuration**:
```javascript
contextStorage: {
    default: {
        module: require("node-red-contrib-context-redis"),
        config: {
            host: process.env.REDIS_HOST || "redis",
            port: parseInt(process.env.REDIS_PORT) || 6379,
            db: 0,
            prefix: "nr:",
            password: process.env.REDIS_PASSWORD || undefined
        }
    },
    memory: {
        module: "memory"
    }
}
```

**Impact on existing flows** (zero code changes required):

| Flow | Context Key | Current Behavior | With Redis |
|------|------------|-----------------|------------|
| team-manager.json | `flow.get('teams')` | Lost on restart | Persisted & shared |
| team-manager.json | `flow.get('auditLog')` | Lost on restart | Persisted & shared |
| cleanup-hooks.json | `flow.get('error_log')` | Lost on restart | Persisted & shared |
| cleanup-hooks.json | `flow.get('circuit_breakers')` | Lost on restart | Persisted & shared |
| cleanup-hooks.json | `flow.get('error_retry_queue')` | Lost on restart | Persisted & shared |
| cleanup-hooks.json | `flow.get('text_chunks_temp')` | Lost on restart | Persisted (use `memory` store for ephemeral) |

**For high-frequency ephemeral data** that should NOT be persisted (e.g., temporary chunk caches), explicitly specify the memory store:
```javascript
flow.set('text_chunks_temp', data, 'memory');
```

### 6.2 In-Flight Message Checkpointing

**Purpose**: Capture the state of messages that are mid-pipeline when a drain signal fires.

**Database schema** (extends existing PostgreSQL):
```sql
-- In docker/init-clustering.sql

CREATE TABLE IF NOT EXISTS rag.message_checkpoints (
    id              SERIAL PRIMARY KEY,
    flow_id         VARCHAR(255) NOT NULL,
    flow_tab        VARCHAR(255) NOT NULL,
    node_id         VARCHAR(255) NOT NULL,
    stage           VARCHAR(50) NOT NULL,
    msg_snapshot    JSONB NOT NULL,
    checkpoint_id   VARCHAR(255) UNIQUE NOT NULL,
    status          VARCHAR(20) DEFAULT 'pending'
                    CHECK (status IN ('pending', 'replayed', 'expired', 'failed')),
    instance        VARCHAR(50),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    replayed_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ DEFAULT NOW() + INTERVAL '24 hours'
);

CREATE INDEX IF NOT EXISTS idx_ckpt_flow_status
    ON rag.message_checkpoints(flow_tab, status);
CREATE INDEX IF NOT EXISTS idx_ckpt_expires
    ON rag.message_checkpoints(expires_at)
    WHERE status = 'pending';
```

**Checkpoint Gate subflow design**:

```
┌────────────────────────────────────────────────────────┐
│  Checkpoint-Gate Subflow                               │
│                                                        │
│  Input ──► [Check Drain Signal] ──┬──► Output (normal) │
│                                   │                    │
│                                   └──► [Snapshot to PG] │
│                                        (drain active)  │
└────────────────────────────────────────────────────────┘
```

The gate checks a Redis key (`redforge:drain:{flowTab}`) on every message:
- If key is absent: pass message through (normal operation, ~0.1ms overhead)
- If key is present: serialize `msg` to JSONB, write to `message_checkpoints`, drop message from pipeline

**This leverages the existing ACP checkpoint schema** already defined in `schemas/msg_input.json`:
```json
{
    "checkpoint": {
        "checkpointId": "ckpt-a1b2c3d4-e5f6-7890-a1b2-c3d4e5f6a7b8",
        "stage": "execute",
        "state": {"extractionProgress": 0.5, "pagesProcessed": 10},
        "timestamp": "2025-08-14T12:00:00Z"
    }
}
```

### 6.3 Message Hold Queue (MQTT)

**Purpose**: Buffer incoming messages during the deploy window so they are not lost.

**Architecture**:
```
                                    ┌───────────────────┐
                                    │    Mosquitto       │
                                    │    MQTT Broker     │
  External ──► [MQTT In] ──►       │                    │
  Sources      (always on)         │  redforge/ingest/  │ ──► [Deploy Gate] ──► Flow
                                    │  input             │
                                    │                    │
               During deploy:       │  redforge/hold/    │ ◄── [Deploy Gate]
                                    │  ingest-simple-tab │      (drain active)
                                    │                    │
               After deploy:        │  redforge/hold/    │ ──► [Deploy Gate]
                                    │  ingest-simple-tab │      (replay mode)
                                    └───────────────────┘
```

**MQTT Topic Structure**:
```
redforge/
├── ingest/
│   └── input                    # Production ingest entry point
├── query/
│   └── input                    # Production query entry point
├── hold/
│   ├── ingest-simple-tab        # Held messages during ingest deploy
│   └── query-simple-tab         # Held messages during query deploy
├── dlq/
│   ├── ingest-simple-tab        # Dead letters per flow
│   └── query-simple-tab
└── deploy/
    ├── drain                    # Deploy drain signal (retained msg)
    └── resume                   # Deploy resume signal
```

**QoS and Persistence**:
- QoS 1 (at-least-once): Ensures no message is lost during broker restart
- Persistent sessions: Client subscriptions survive reconnection
- Retained messages: Deploy signals are retained so late-connecting clients see them
- Message expiry: Hold queue messages expire after 24 hours (configurable)

**Why MQTT over alternatives**:

| Criteria | MQTT (Mosquitto) | RabbitMQ | Kafka |
|----------|-----------------|----------|-------|
| Node-RED support | Built-in core nodes | Plugin required | Plugin required |
| Memory footprint | ~5MB | ~100MB+ | ~500MB+ |
| Operational complexity | Minimal | Moderate | High |
| QoS guarantees | QoS 0/1/2 | Confirms/Transactions | At-least-once/Exactly-once |
| Sufficient for this use case | Yes | Overkill | Overkill |
| Upgrade path | EMQX for clustering | N/A | Swap when needed |

### 6.4 Deploy Gate Subflow

A reusable Node-RED subflow that sits at the entry point of every production flow:

```
┌─────────────────────────────────────────────────────────────────┐
│  Deploy-Gate Subflow                                            │
│                                                                 │
│  [MQTT In] ──► [Check Mode] ──┬──► [Increment Counter] ──► Out │
│                               │     (normal mode)               │
│                               │                                 │
│                               ├──► [Publish to Hold Q] ──► ∅   │
│                               │     (drain mode)                │
│                               │                                 │
│                               └──► [Replay from Hold Q] ──► Out │
│                                     (resume mode)               │
│                                                                 │
│  Redis key: redforge:drain:{flowTab}                            │
│  Modes: normal | draining | replaying                           │
└─────────────────────────────────────────────────────────────────┘
```

**Active message counter** (for drain completion detection):
```javascript
// At flow entry (Deploy Gate):
const count = flow.get('_active_messages') || 0;
flow.set('_active_messages', count + 1);

// At flow exit (final node before Link Out):
const count = flow.get('_active_messages') || 1;
flow.set('_active_messages', count - 1);
```

The deploy script polls this counter during Phase 1 (DRAIN). When it reaches 0, all in-flight messages have completed and it's safe to deploy.

---

## 7. Outbound Message Tracking & Timeline

### 7.1 Audit Log Schema

```sql
-- In docker/init-clustering.sql

CREATE TABLE IF NOT EXISTS rag.message_audit_log (
    id              SERIAL PRIMARY KEY,
    flow_id         VARCHAR(255) NOT NULL,
    flow_tab        VARCHAR(255) NOT NULL,
    direction       VARCHAR(10) NOT NULL
                    CHECK (direction IN ('inbound', 'outbound')),
    node_id         VARCHAR(255),
    payload_hash    VARCHAR(64),
    payload_size    INTEGER,
    metadata        JSONB NOT NULL,
    msg_summary     JSONB,
    status          VARCHAR(20),
    processing_time INTEGER,
    instance        VARCHAR(50),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_audit_flow_tab_time
    ON rag.message_audit_log(flow_tab, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_audit_flow_id
    ON rag.message_audit_log(flow_id);
CREATE INDEX IF NOT EXISTS idx_audit_direction
    ON rag.message_audit_log(flow_tab, direction, created_at DESC);
```

### 7.2 Audit Logger Subflow

A lightweight, non-blocking subflow placed at flow entry and exit points:

```
┌──────────────────────────────────────────────────┐
│  Audit-Logger Subflow                            │
│                                                  │
│  Input ──┬──► [Log to PG] (async, non-blocking)  │
│          │                                        │
│          └──► Output (msg passes through)         │
│                                                  │
│  Env: DIRECTION = "inbound" | "outbound"         │
│       FLOW_TAB = "ingest-simple-tab"             │
└──────────────────────────────────────────────────┘
```

**Implementation** (function node inside subflow):
```javascript
// Audit-Logger function node
const crypto = require('crypto');

const direction = env.get('DIRECTION') || 'outbound';
const flowTab = env.get('FLOW_TAB') || 'unknown';
const instance = env.get('INSTANCE_ID') || 'default';

// Hash payload (don't store full payload for space/privacy)
const payloadStr = JSON.stringify(msg.payload || '');
const payloadHash = crypto.createHash('sha256').update(payloadStr).digest('hex');

// Build summary (key fields only)
const summary = {
    payloadType: typeof msg.payload,
    payloadKeys: msg.payload && typeof msg.payload === 'object'
        ? Object.keys(msg.payload) : undefined,
    chunkCount: msg.metadata?.chunkCount,
    embeddingsCount: msg.metadata?.embeddingsCount,
    inserted: msg.payload?.inserted,
    backend: msg.payload?.backend,
    errorCode: msg.error?.code
};

// Non-blocking write (fire and forget)
const auditMsg = {
    payload: {
        flow_id: msg.metadata?.flowId || 'unknown',
        flow_tab: flowTab,
        direction: direction,
        node_id: node.id,
        payload_hash: payloadHash,
        payload_size: payloadStr.length,
        metadata: msg.metadata || {},
        msg_summary: summary,
        status: msg.metadata?.status || 'unknown',
        processing_time: msg.metadata?.processingTime,
        instance: instance
    }
};

// Send to PostgreSQL via a separate output
// Original message continues on Output 1; audit on Output 2
node.send([msg, auditMsg]);
```

### 7.3 Pre-Deploy Timeline Snapshot

Before any deployment, the deploy script captures the current state:

```sql
-- Capture last 50 outbound messages for the target flow
SELECT
    flow_id,
    created_at,
    status,
    processing_time,
    msg_summary,
    payload_hash
FROM rag.message_audit_log
WHERE flow_tab = 'ingest-simple-tab'
  AND direction = 'outbound'
ORDER BY created_at DESC
LIMIT 50;
```

This provides:
- **Throughput baseline**: Messages per minute before deploy
- **Error rate baseline**: Percentage of errors before deploy
- **Processing time baseline**: Average latency before deploy
- **Payload continuity**: Hash comparison to detect data corruption

### 7.4 Post-Deploy Comparison

After the deploy + replay cycle completes, the script runs the same query and compares:

```
┌─────────────────────────────────────────────────────────┐
│  Deploy Report: ingest-simple-tab                        │
│  Deployed: 2026-02-14T10:30:00Z                         │
│                                                          │
│  Metric              Before    After     Delta           │
│  ─────────────────── ──────── ──────── ──────           │
│  Throughput (msg/min)  12.3     11.8     -4.1%          │
│  Error rate            0.0%     0.0%     0.0%           │
│  Avg latency (ms)      1,234    1,189    -3.6%          │
│  Messages in hold Q     0        0       (all replayed)  │
│  Checkpoints pending    0        0       (all replayed)  │
│                                                          │
│  Status: SUCCESS - All metrics within normal range       │
└─────────────────────────────────────────────────────────┘
```

---

## 8. Infrastructure Components

### 8.1 New Services Summary

| Service | Image | Port | Purpose | Profile |
|---------|-------|------|---------|---------|
| Redis | `redis:7-alpine` | 6379 | Context store, pub/sub, locks | Core |
| Mosquitto | `eclipse-mosquitto:2` | 1883 | Message hold queue, DLQ | Core |
| Nginx | `nginx:alpine` | 80/443 | Load balancer, traffic routing | Core |
| NR-Green | Same as NR-Blue | 1881 | Blue-Green standby instance | Core |

### 8.2 Docker Compose Additions

The following services would be added to `docker/docker-compose.yml`:

**Redis**:
```yaml
redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "${REDIS_PORT:-6379}:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    networks:
      - redforge-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
```

**Mosquitto**:
```yaml
mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "${MQTT_PORT:-1883}:1883"
    volumes:
      - mosquitto_data:/mosquitto/data
      - ./mosquitto.conf:/mosquitto/config/mosquitto.conf:ro
    networks:
      - redforge-network
    healthcheck:
      test: ["CMD", "mosquitto_sub", "-t", "$$SYS/broker/uptime", "-C", "1", "-W", "3"]
      interval: 30s
      timeout: 10s
      retries: 3
```

**Nginx**:
```yaml
nginx:
    image: nginx:alpine
    container_name: nginx-lb
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - nodered
      - nodered-green
    networks:
      - redforge-network
```

**Node-RED Green Instance**:
```yaml
nodered-green:
    build:
      context: .
      dockerfile: Dockerfile.nodered
    container_name: nodered-green
    restart: unless-stopped
    ports:
      - "${NODERED_GREEN_PORT:-1881}:1880"
    environment:
      # Same as Blue, plus:
      - INSTANCE_ID=green
      - REDIS_HOST=redis
      - MQTT_HOST=mosquitto
    volumes:
      - nodered_green_data:/data
      - ../examples/flows:/data/lib/flows:ro
    networks:
      - redforge-network
```

### 8.3 Dockerfile Addition

Add to `docker/Dockerfile.nodered`:
```dockerfile
# Clustering support
RUN npm install --no-optional \
    node-red-contrib-context-redis@latest
```

Note: MQTT nodes are already built into Node-RED core. No additional package needed.

### 8.4 Nginx Configuration

```nginx
# docker/nginx.conf
upstream nodered_active {
    # Blue-Green routing: only one is active at a time
    server nodered:1880;
    server nodered-green:1880 backup;
}

server {
    listen 80;

    # Node-RED Editor UI (sticky sessions for WebSocket)
    location / {
        proxy_pass http://nodered_active;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400;  # WebSocket keepalive
    }

    # Health check endpoint
    location /health {
        proxy_pass http://nodered_active;
    }

    # Admin API (used by deploy script)
    location /flows {
        proxy_pass http://nodered_active;
        proxy_set_header Content-Type application/json;
    }
}
```

### 8.5 Mosquitto Configuration

```
# docker/mosquitto.conf
listener 1883
protocol mqtt

# Persistence (survive broker restart)
persistence true
persistence_location /mosquitto/data/

# Retained messages for deploy signals
retain_available true

# Message size limit (16MB for large msg snapshots)
message_size_limit 16777216

# Logging
log_dest stdout
log_type warning
log_type error
log_type notice

# Authentication (configure for production)
allow_anonymous true
# password_file /mosquitto/config/passwd
```

---

## 9. Implementation Roadmap

### Phase A: Foundation Infrastructure (Week 1-2)

| # | Task | Files | Dependencies |
|---|------|-------|-------------|
| A1 | Add Redis service | `docker/docker-compose.yml` | None |
| A2 | Add Mosquitto service | `docker/docker-compose.yml`, `docker/mosquitto.conf` | None |
| A3 | Add clustering SQL schema | `docker/init-clustering.sql` | None |
| A4 | Add `node-red-contrib-context-redis` | `docker/Dockerfile.nodered` | A1 |
| A5 | Create Node-RED settings.js override | `docker/settings.js` | A4 |
| A6 | Update environment variables | `docker/.env.example` | A1, A2 |
| A7 | Verify existing flows work with Redis context | Manual test | A5 |

### Phase B: State Management Primitives (Week 3-4)

| # | Task | Files | Dependencies |
|---|------|-------|-------------|
| B1 | Create Checkpoint-Gate subflow | `examples/flows/primitives/checkpoint/checkpoint-gate.json` | A3 |
| B2 | Create Audit-Logger subflow | `examples/flows/primitives/audit/audit-logger.json` | A3 |
| B3 | Create Deploy-Gate subflow | `examples/flows/primitives/deploy-gate/deploy-gate.json` | A2 |
| B4 | Create checkpoint replay script | `scripts/replay-checkpoints.js` | A3, B1 |
| B5 | Add checkpoint + audit schemas | `schemas/checkpoint.json`, `schemas/audit.json` | None |
| B6 | Integration tests | `tests/clustering.test.js` | B1-B4 |

### Phase C: Deploy Orchestration (Week 5-6)

| # | Task | Files | Dependencies |
|---|------|-------|-------------|
| C1 | Create deploy orchestration script | `scripts/deploy-flow.sh` | B1-B3 |
| C2 | Create per-flow deploy config | `config/deploy-config.json` | None |
| C3 | Add Nginx load balancer | `docker/nginx.conf`, `docker/docker-compose.yml` | A1-A2 |
| C4 | Add Green Node-RED instance | `docker/docker-compose.yml` | C3 |
| C5 | Modify orchestrators for MQTT entry | `examples/flows/orchestrators/ingest/ingest-simple.json` | B3 |
| C6 | Deploy report generator | `scripts/deploy-report.js` | B2 |

### Phase D: Integration & Hardening (Week 7-8)

| # | Task | Files | Dependencies |
|---|------|-------|-------------|
| D1 | E2E test: zero-loss deploy | `tests/e2e/deploy-zero-loss.test.js` | C1-C5 |
| D2 | E2E test: checkpoint replay | `tests/e2e/checkpoint-replay.test.js` | B4 |
| D3 | E2E test: audit log accuracy | `tests/e2e/audit-accuracy.test.js` | B2 |
| D4 | Deployment runbook | `docs/guides/DEPLOYMENT_RUNBOOK.md` | C1 |
| D5 | Update architecture docs | `docs/architecture/ARCHITECTURE.md` | All |
| D6 | Performance benchmarks | `scripts/benchmark-deploy.js` | C1-C5 |

---

## 10. Key Architectural Decisions

| # | Decision | Choice | Alternatives Considered | Rationale |
|---|----------|--------|------------------------|-----------|
| 1 | Clustering pattern | Blue-Green | Active-Active, Active-Passive | Node-RED Link nodes are process-local; Blue-Green avoids distributed state coordination |
| 2 | Context persistence | Redis | LocalFilesystem, PostgreSQL | Native Node-RED support via plugin; zero code changes; sub-ms latency; shared across instances |
| 3 | Message buffering | MQTT (Mosquitto) | RabbitMQ, Kafka, Redis Streams | Built-in Node-RED nodes; 5MB footprint; QoS 1 sufficient; persistent sessions |
| 4 | Checkpoint storage | PostgreSQL (existing) | Redis, S3, File | Already in stack; JSONB for flexible msg serialization; ACID for reliability |
| 5 | Deploy mechanism | Admin API + `flows` type | Full restart, Node-RED Projects | Tab-level restart minimizes blast radius; other flows unaffected |
| 6 | Load balancer | Nginx | HAProxy, Traefik | Proven WebSocket support; simple config; handles sticky sessions for editor UI |
| 7 | Outbound tracking | PostgreSQL audit table | Elasticsearch, File logs | Queryable; joinable with checkpoints; standard SQL for deploy reports |
| 8 | Drain signaling | Redis Pub/Sub | MQTT retained, HTTP polling | Sub-ms latency; all instances receive signal simultaneously; no polling overhead |
| 9 | ACP checkpoint format | Use existing schema | New custom format | Schema already defines checkpointId, stage, state, timestamp -- just operationalize it |

---

## 11. Verification Plan

### Test 1: Redis Context Persistence
- Store team state via `flow.set('teams', {...})`
- Restart Node-RED container
- Verify `flow.get('teams')` returns the stored data
- **Pass criteria**: State survives container restart

### Test 2: MQTT Hold Queue (Zero Message Loss)
- Send 100 messages to `redforge/ingest/input`
- After 50 messages, trigger deploy-drain
- Verify remaining 50 are held in `redforge/hold/ingest-simple-tab`
- Complete deploy, trigger resume
- Verify all 100 messages are processed
- **Pass criteria**: 100 inbound = 100 outbound in audit log

### Test 3: In-Flight Checkpointing
- Start processing a message with slow Ollama embedding (simulated 60s delay)
- Trigger deploy-drain at T+10s (message is mid-pipeline)
- Verify message is checkpointed to PostgreSQL with correct `node_id` and `stage`
- Deploy new flow, replay checkpoint
- Verify message completes processing from checkpoint point (not from start)
- **Pass criteria**: Checkpoint row transitions pending → replayed; output is correct

### Test 4: Blue-Green Swap
- Blue is active, processing messages normally
- Deploy new flow to Green via Admin API
- Swap LB to Green
- Verify messages flow through Green with new flow logic
- **Pass criteria**: Zero errors during swap; new flow behavior observed

### Test 5: Rollback on Verify Failure
- Deploy intentionally broken flow to Green
- Verify phase detects error
- System automatically redeploys previous version
- MQTT hold queue replays to Blue (still running known-good)
- **Pass criteria**: No messages lost; system self-heals

### Test 6: Audit Log Accuracy
- Process 50 messages through ingest pipeline
- Query audit log for flow tab
- Verify 50 inbound entries + 50 outbound entries
- Verify payload hashes are unique and consistent
- Verify processing_time values are reasonable
- **Pass criteria**: Complete 1:1 audit trail

### Test 7: Deploy Report Generation
- Capture pre-deploy snapshot (last 50 outbound messages)
- Execute full deploy cycle
- Generate deploy report
- Verify throughput, error rate, and latency deltas are calculated correctly
- **Pass criteria**: Report accurately reflects before/after state

---

## Appendix: Reference Configurations

### A. Environment Variables (New)

```bash
# Redis (Context Store)
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=

# MQTT (Message Broker)
MQTT_HOST=mosquitto
MQTT_PORT=1883

# Clustering
INSTANCE_ID=blue                    # blue | green | canary
NODERED_GREEN_PORT=1881
DRAIN_TIMEOUT_MS=30000              # Max wait for in-flight messages
CHECKPOINT_EXPIRY_HOURS=24          # Auto-expire old checkpoints
AUDIT_RETENTION_DAYS=7              # Auto-cleanup old audit entries
AUDIT_MAX_PER_FLOW=1000             # Max audit entries per flow tab
DEPLOY_VERIFY_TIMEOUT_MS=15000      # Max wait for smoke test
REPLAY_RATE_LIMIT=10                # Messages per second during replay
```

### B. Flow Modification Pattern

To make an existing flow deploy-safe, add these nodes:

```
Before (current):
  [Inject/Link In] ──► [Init-Metadata] ──► ... ──► [Complete] ──► [Link Out]

After (deploy-safe):
  [MQTT In] ──► [Deploy-Gate] ──► [Audit-In] ──► [Init-Metadata]
      ──► [Checkpoint-Gate] ──► [Ollama-Embed]
      ──► ... ──► [Complete] ──► [Audit-Out] ──► [MQTT Out / Link Out]
```

The existing Inject and Link In nodes remain for development/testing. The MQTT In + Deploy Gate path is the production entry point.

### C. Deploy Script Usage

```bash
# Deploy a new version of ingest-simple to the Green instance
./scripts/deploy-flow.sh \
    --flow-tab ingest-simple-tab \
    --flow-file examples/flows/orchestrators/ingest/ingest-simple.json \
    --target green \
    --drain-timeout 30 \
    --verify

# Rollback (swap traffic back to Blue)
./scripts/deploy-flow.sh --rollback --flow-tab ingest-simple-tab

# Dry run (show what would happen without executing)
./scripts/deploy-flow.sh \
    --flow-tab ingest-simple-tab \
    --flow-file examples/flows/orchestrators/ingest/ingest-simple.json \
    --dry-run
```

### D. Monitoring Queries

```sql
-- Active deployments
SELECT * FROM rag.deploy_events
WHERE status = 'deployed' AND verified_at IS NULL
ORDER BY deployed_at DESC;

-- Pending checkpoints (should be 0 in steady state)
SELECT flow_tab, COUNT(*) as pending
FROM rag.message_checkpoints
WHERE status = 'pending'
GROUP BY flow_tab;

-- Message throughput per flow (last hour)
SELECT flow_tab, direction,
    COUNT(*) as messages,
    AVG(processing_time) as avg_latency_ms,
    COUNT(*) FILTER (WHERE status = 'error') as errors
FROM rag.message_audit_log
WHERE created_at > NOW() - INTERVAL '1 hour'
GROUP BY flow_tab, direction
ORDER BY flow_tab, direction;

-- Audit trail for a specific flow execution
SELECT direction, node_id, status, processing_time, created_at, msg_summary
FROM rag.message_audit_log
WHERE flow_id = 'ingest-1707900000'
ORDER BY created_at ASC;
```

---

## Related Documents

- [Architecture Overview](ARCHITECTURE.md) -- System design
- [Multi-Agent Architecture](MULTI_AGENT_ARCHITECTURE.md) -- Phase 4 agent patterns
- [Error Codes Reference](../reference/ERROR_CODES.md) -- Error taxonomy
- [E2E Testing Guide](../guides/E2E_TESTING.md) -- Manual test procedures
- [Agent Team Guide](../guides/AGENT_TEAM_GUIDE.md) -- Multi-agent patterns
