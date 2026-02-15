# RedForge AI Control Tower - Architecture

**Status**: 🚧 In Development
**Last Updated**: 2026-02-15

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Control Tower Architecture (TypeScript)](#2-control-tower-architecture-typescript)
3. [Node-RED Agent Runtime Architecture](#3-node-red-agent-runtime-architecture)
4. [Zero-Downtime Deployment Architecture](#4-zero-downtime-deployment-architecture)
5. [Enterprise Features](#5-enterprise-features)
6. [Deployment Strategies](#6-deployment-strategies)
7. [Integration Points](#7-integration-points)
8. [Scalability & Performance](#8-scalability--performance)

---

## 1. System Overview

### 1.1 What is the Control Tower?

The **RedForge AI Control Tower** is a **TypeScript/Node.js management application** that provides enterprise-grade orchestration for fleets of Node-RED instances running Agentic AI workflows. It acts as the central control plane for deploying, monitoring, and managing distributed AI agent systems.

**Architecture Pattern:**
- **Control Plane:** TypeScript application (web UI + REST API)
- **Data Plane:** Node-RED instances running AI agent flows
- **Clear Separation:** Management logic vs. workload execution

```
┌─────────────────────────────────────────────────────┐
│     Control Tower (TypeScript Management App)       │
│  ┌───────────────────────────────────────────────┐  │
│  │  Web UI (React/Vue)                           │  │
│  │    • Fleet Dashboard                          │  │
│  │    • Deployment Manager                       │  │
│  │    • Performance Analytics                    │  │
│  │    • Governance Console                       │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │  Backend (Node.js/TypeScript + Express)       │  │
│  │    • REST API + WebSocket Server              │  │
│  │    • Instance Manager                         │  │
│  │    • Deployment Orchestrator                  │  │
│  │    • Health Monitor                           │  │
│  │    • Governance Engine                        │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                    │ Manages ↓
┌─────────────────────────────────────────────────────┐
│      Managed Node-RED Instances (Agent Runtimes)    │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │Instance 1│  │Instance 2│  │Instance 3│         │
│  │File Conv │  │Embeddings│  │  Query   │         │
│  │Text Chunk│  │ Storage  │  │ Response │         │
│  │Error Hdlr│  │          │  │          │         │
│  └──────────┘  └──────────┘  └──────────┘         │
│  Node-RED       Node-RED      Node-RED             │
│  4.1.2          4.1.2         4.1.2                │
│                                                     │
│  Shared: Redis, MQTT, PostgreSQL, Prometheus       │
└─────────────────────────────────────────────────────┘
```

### 1.2 Why Node-RED for Agent Runtimes?

Node-RED provides the ideal runtime for AI agent workflows:

- **Visual Flow Programming:** Drag-and-drop development in browser UI
- **Message-Based Architecture:** Asynchronous, event-driven workflows
- **Mature Ecosystem:** 4,000+ community nodes
- **Standards-Aligned:** JSON-based message contracts from main project
- **Low-Code Platform:** Accessible to both developers and domain experts
- **Production-Ready:** Mature, stable, widely deployed

Each of the 8 agent primitives from the main [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI) project is implemented as a Node-RED flow JSON file.

### 1.3 Key Design Principles

**1. Separation of Concerns**
- Control Tower = Management/orchestration (TypeScript)
- Node-RED Instances = Workload execution (AI agent flows)
- Services = Shared infrastructure (Redis, MQTT, PostgreSQL)

**2. Declarative Configuration**
- Desired state specified via API
- Control Tower reconciles actual state to match
- Kubernetes-inspired resource model

**3. Zero-Downtime Operations**
- Blue-green deployment for agent updates
- State preservation during transitions
- Message buffering to prevent data loss

**4. Enterprise-Grade Governance**
- Role-based access control (RBAC)
- Complete audit logging
- Resource quotas and policy enforcement

**5. Observable by Default**
- Metrics collection (Prometheus)
- Distributed tracing (OpenTelemetry future)
- Real-time dashboards

---

## 2. Control Tower Architecture (TypeScript)

### 2.1 Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Frontend** | React or Vue.js | Web-based management UI |
| **Backend** | TypeScript 5+ | Type-safe application logic |
| **Runtime** | Node.js 18+ | JavaScript runtime |
| **Web Framework** | Express.js | REST API server |
| **WebSocket** | Socket.io | Real-time bidirectional communication |
| **Database ORM** | Prisma or TypeORM | PostgreSQL database access |
| **HTTP Client** | Axios | Node-RED Admin API calls |
| **Container Runtime** | Dockerode | Docker API for instance management |
| **Validation** | Zod or Joi | Request/response validation |
| **Testing** | Jest | Unit and integration tests |

### 2.2 Backend Architecture

#### 2.2.1 Core Services

```typescript
// Service Layer Architecture
interface ControlTowerServices {
  instanceManager: InstanceManagerService;
  deploymentOrchestrator: DeploymentOrchestratorService;
  healthMonitor: HealthMonitorService;
  governanceEngine: GovernanceEngineService;
  metricsCollector: MetricsCollectorService;
  auditLogger: AuditLoggerService;
}
```

**Instance Manager Service**
- Deploy new Node-RED instances (Docker containers or K8s pods)
- Start, stop, restart instances
- Scale instances (horizontal scaling)
- Destroy instances
- Health checks via Node-RED Admin API

**Deployment Orchestrator Service**
- Blue-green deployment state machine
- Instance cloning and flow deployment
- Smoke testing and verification
- Rollback automation
- MQTT coordination (drain/resume signals)

**Health Monitor Service**
- Poll Node-RED instances (health endpoint)
- Collect instance metrics (CPU, memory, flows deployed)
- Detect failures and trigger alerts
- Update instance status in database

**Governance Engine Service**
- RBAC enforcement (check permissions)
- Resource quota validation
- Policy evaluation
- Compliance reporting

**Metrics Collector Service**
- Scrape Prometheus metrics
- Aggregate fleet-wide statistics
- Calculate cost metrics (LLM token usage)
- Generate performance reports

**Audit Logger Service**
- Log all Control Tower actions
- Collect AI decision logs from Node-RED instances
- Store logs in PostgreSQL
- Export compliance reports

#### 2.2.2 REST API Endpoints

```typescript
// Instance Management
POST   /api/instances            // Deploy new Node-RED instance
GET    /api/instances            // List all instances
GET    /api/instances/:id        // Get instance details
PATCH  /api/instances/:id        // Update instance (scale, config)
DELETE /api/instances/:id        // Destroy instance
POST   /api/instances/:id/start  // Start stopped instance
POST   /api/instances/:id/stop   // Stop running instance

// Deployment Management
POST   /api/deployments          // Initiate blue-green deployment
GET    /api/deployments          // List deployment history
GET    /api/deployments/:id      // Get deployment status
POST   /api/deployments/:id/rollback  // Rollback deployment

// Fleet Monitoring
GET    /api/metrics/fleet        // Fleet-wide metrics
GET    /api/metrics/instances/:id // Instance-specific metrics
GET    /api/health               // Control Tower health check

// Governance
GET    /api/audit                // Query audit logs
GET    /api/users                // List users
POST   /api/users                // Create user
GET    /api/roles                // List roles
POST   /api/policies             // Create policy

// Configuration
GET    /api/config               // Get Control Tower config
PATCH  /api/config               // Update config
```

#### 2.2.3 WebSocket API

Real-time updates for:
- Deployment progress (phases, metrics)
- Instance health status changes
- Fleet-wide alerts
- Audit log stream

```typescript
// WebSocket Event Types
enum WSEvent {
  DEPLOYMENT_PHASE_CHANGE = 'deployment:phase',
  DEPLOYMENT_PROGRESS = 'deployment:progress',
  INSTANCE_STATUS_CHANGE = 'instance:status',
  FLEET_ALERT = 'fleet:alert',
  AUDIT_LOG_ENTRY = 'audit:entry'
}

// Example: Deployment Status Update
ws://localhost:3000/ws/deployments/:deploymentId
{
  "event": "deployment:phase",
  "data": {
    "phase": "DRAIN",
    "messagesHeld": 42,
    "inFlightCount": 3,
    "progress": 0.4,
    "timestamp": "2026-02-15T10:30:00Z"
  }
}
```

#### 2.2.4 Database Schema (PostgreSQL)

```sql
-- Node-RED Instances
CREATE TABLE instances (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL UNIQUE,
  status VARCHAR(50) NOT NULL, -- 'running', 'stopped', 'deploying', 'failed'
  flows JSONB NOT NULL,        -- Array of flow JSON file paths
  resources JSONB NOT NULL,    -- { cpu: "2", memory: "8Gi" }
  env JSONB NOT NULL,          -- Environment variables
  health_endpoint VARCHAR(500),
  container_id VARCHAR(255),   -- Docker container ID or K8s pod name
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Deployments
CREATE TABLE deployments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  instance_id UUID NOT NULL REFERENCES instances(id),
  strategy VARCHAR(50) NOT NULL, -- 'blue-green', 'canary', 'rolling'
  status VARCHAR(50) NOT NULL,   -- 'pending', 'in_progress', 'completed', 'failed', 'rolled_back'
  current_phase VARCHAR(50),     -- 'DRAIN', 'DEPLOY', 'VERIFY', 'RESUME', 'AUDIT', 'CLEANUP'
  old_flows JSONB,
  new_flows JSONB,
  metrics JSONB,                 -- Deployment metrics
  error TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);

-- Audit Logs (Control Tower Actions)
CREATE TABLE audit_logs (
  id BIGSERIAL PRIMARY KEY,
  timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  user_id VARCHAR(255),
  action VARCHAR(100) NOT NULL, -- 'INSTANCE_CREATED', 'DEPLOYMENT_STARTED', etc.
  resource_type VARCHAR(50),    -- 'instance', 'deployment', 'user', 'role'
  resource_id VARCHAR(255),
  details JSONB,
  ip_address INET,
  INDEX idx_audit_timestamp (timestamp DESC),
  INDEX idx_audit_user (user_id),
  INDEX idx_audit_resource (resource_type, resource_id)
);

-- AI Decision Logs (from Node-RED Instances)
CREATE TABLE ai_decision_logs (
  id BIGSERIAL PRIMARY KEY,
  timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  instance_id UUID NOT NULL REFERENCES instances(id),
  flow_name VARCHAR(255) NOT NULL,
  model VARCHAR(100),
  prompt TEXT,
  response TEXT,
  confidence FLOAT,
  tokens_used INTEGER,
  latency_ms INTEGER,
  metadata JSONB,
  INDEX idx_ai_logs_timestamp (timestamp DESC),
  INDEX idx_ai_logs_instance (instance_id),
  INDEX idx_ai_logs_model (model)
);

-- Users (RBAC)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username VARCHAR(255) NOT NULL UNIQUE,
  email VARCHAR(255) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(50) NOT NULL, -- 'admin', 'operator', 'viewer'
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_login TIMESTAMPTZ
);

-- Roles & Permissions
CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL UNIQUE,
  permissions JSONB NOT NULL, -- Array of permission objects
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 2.3 Frontend Architecture

#### 2.3.1 Application Structure

```
control-tower-ui/
├── src/
│   ├── pages/
│   │   ├── FleetDashboard.tsx        # Instance list, status overview
│   │   ├── DeploymentManager.tsx     # Deployment UI
│   │   ├── AgentCoordination.tsx     # Multi-agent workflow viz
│   │   ├── PerformanceAnalytics.tsx  # Metrics and charts
│   │   ├── GovernanceConsole.tsx     # RBAC, audit logs
│   │   └── Settings.tsx              # Control Tower configuration
│   ├── components/
│   │   ├── InstanceCard.tsx          # Instance status card
│   │   ├── DeploymentTimeline.tsx    # Deployment phase visualization
│   │   ├── MetricsChart.tsx          # Chart components
│   │   └── AuditLogTable.tsx         # Audit log viewer
│   ├── services/
│   │   ├── api.ts                    # REST API client
│   │   ├── websocket.ts              # WebSocket client
│   │   └── auth.ts                   # Authentication
│   ├── stores/
│   │   ├── instanceStore.ts          # Instance state management
│   │   ├── deploymentStore.ts        # Deployment state
│   │   └── authStore.ts              # Auth state
│   └── types/
│       └── index.ts                  # TypeScript types
```

#### 2.3.2 Key UI Pages

**1. Fleet Dashboard**
- Grid view of all Node-RED instances
- Health status indicators (green/yellow/red)
- Resource utilization (CPU, memory)
- Quick actions (start, stop, deploy)

**2. Deployment Manager**
- Initiate deployment (select instance, upload flow JSON)
- Real-time deployment progress (DRAIN → DEPLOY → VERIFY → RESUME → AUDIT)
- Deployment history with rollback capability
- Deployment metrics (duration, messages held, errors)

**3. Agent Coordination View**
- Visualize multi-agent workflows across instances
- Show voting patterns, refinement loops
- Track agent team status
- Display coordination state (from Redis)

**4. Performance Analytics**
- Fleet-wide metrics (throughput, latency, errors)
- Cost analytics (LLM token usage, compute costs)
- Quality metrics (accuracy, confidence scores)
- Embedded Grafana dashboards

**5. Governance Console**
- User management (create, edit, delete users)
- Role assignment
- Audit log search and export
- Compliance reports

---

## 3. Node-RED Agent Runtime Architecture

### 3.1 Agent Primitive Flows

Each of the 8 agent primitives from the main [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI) project is a Node-RED flow JSON file:

```
examples/flows/primitives/
├── file-converter/
│   └── convert-file-to-text-tika.json
├── text-chunking/
│   └── text-chunking.json
├── embeddings/
│   └── generate-embeddings-ollama.json
├── storage/
│   └── milvus-storage-advanced.json
├── query-processing/
│   └── query-processing.json
├── response-generation/
│   └── response-generation.json
├── error-handling/
│   └── error-handler.json
└── memory/
    └── hindsight-memory.json
```

**Flow Characteristics:**
- **JSON-based:** Portable, version-controllable
- **Message-driven:** Processes `msg` objects with standardized schema
- **Composable:** Can be linked together into pipelines
- **Configurable:** Environment variables for Ollama URL, Milvus credentials, etc.

### 3.2 Message Contracts

All flows communicate via standardized `msg` objects defined in the main project:

```javascript
// Standard message format
msg = {
  payload: <data>,              // Actual data (text, embeddings, etc.)
  metadata: {
    flowId: "unique-id",        // Unique identifier for this workflow
    timestamp: "ISO-8601",      // When message was created
    source: "previous-flow",    // Which flow sent this message
    contentType: "text|binary|json",
    status: "success|error|retrying"
  },
  config: {},                   // Flow-specific configuration overrides
  error: null                   // Error object if status === 'error'
}
```

See main project schemas:
- `schemas/msg_input.json`
- `schemas/msg_output.json`

### 3.3 Node-RED Instance Types

**Standard Instance (2-4GB RAM):**
- File Converter Agent
- Text Chunker Agent
- Error Handler Agent

**Agent Instance (8GB RAM):**
- Embeddings Agent (requires model loading)
- Storage Agent (batch vector operations)
- Query Agent (vector similarity search)
- Response Agent (LLM inference)
- Memory Agent (database operations)

### 3.4 Multi-Agent Coordination (Across Instances)

**Voting Pattern:**
```
User Query
    ↓
┌───────────────────┐
│ Query Coordinator │ (Node-RED instance)
└───────────────────┘
    ↓ (broadcast via MQTT)
    ├──→ [Instance 1: llama3.2] ──┐
    ├──→ [Instance 2: mistral]   ─┤ → [Voting Coordinator] → Best Answer
    └──→ [Instance 3: phi3]      ─┘
```

**State Management:**
- Voting state stored in Redis: `voting_session:{id}:state`
- Voting coordinator (Node-RED flow) aggregates results
- Control Tower monitors coordination health
- Audit logs capture all votes and final decision

**Refinement Pattern:**
```
User Query
    ↓
┌──────────────────┐
│ Generator Agent  │ (Instance 1)
└──────────────────┘
    ↓ (draft response)
┌──────────────────┐
│ Critic Agent     │ (Instance 2)
└──────────────────┘
    ↓ (critique + improvements)
┌──────────────────┐
│ Generator Agent  │ (Instance 1, iteration 2)
└──────────────────┘
    ↓ (refined response)
```

---

## 4. Zero-Downtime Deployment Architecture

### 4.1 Blue-Green Deployment for Node-RED Flows

**Goal:** Update agent flows (prompts, models, logic) without dropping active inference requests.

**Challenge:** Node-RED reloads flows on deploy, disconnecting all active messages.

**Solution:** Control Tower orchestrates blue-green deployment with state preservation.

#### 4.1.1 Deployment Phases

```typescript
enum DeploymentPhase {
  DRAIN = 'DRAIN',       // Stop new tasks, wait for in-flight to complete
  DEPLOY = 'DEPLOY',     // Deploy flows to green instance
  VERIFY = 'VERIFY',     // Run smoke tests
  RESUME = 'RESUME',     // Route traffic to green, replay queued tasks
  AUDIT = 'AUDIT',       // Log deployment metrics
  CLEANUP = 'CLEANUP'    // Destroy blue instance
}
```

#### 4.1.2 Control Tower Orchestration

```typescript
class DeploymentOrchestrator {
  async deployBlueGreen(instanceId: string, newFlowsPath: string): Promise<DeploymentResult> {
    const deployment = await this.createDeployment(instanceId, 'blue-green', newFlowsPath);

    try {
      // Phase 1: DRAIN (stop accepting new tasks)
      await this.updatePhase(deployment.id, 'DRAIN');
      await this.mqttPublish(`${instanceId}/control`, { mode: 'drain' });
      await this.waitForDrain(instanceId, { timeout: 300_000, checkInterval: 5_000 });

      // Phase 2: DEPLOY (create green instance with new flows)
      await this.updatePhase(deployment.id, 'DEPLOY');
      const greenInstance = await this.cloneInstance(instanceId);
      await this.deployFlows(greenInstance.id, newFlowsPath);
      await this.waitForHealthy(greenInstance.id);

      // Phase 3: VERIFY (smoke tests)
      await this.updatePhase(deployment.id, 'VERIFY');
      const smokeTest = await this.runSmokeTest(greenInstance.id);
      if (!smokeTest.passed) {
        throw new Error(`Smoke test failed: ${smokeTest.error}`);
      }

      // Phase 4: RESUME (route traffic to green, replay queued messages)
      await this.updatePhase(deployment.id, 'RESUME');
      await this.mqttPublish(`${greenInstance.id}/control`, { mode: 'active' });
      await this.updateLoadBalancer(instanceId, greenInstance.id);
      await this.replayQueuedMessages(greenInstance.id);

      // Phase 5: AUDIT (log metrics)
      await this.updatePhase(deployment.id, 'AUDIT');
      await this.logDeploymentMetrics(deployment.id, {
        drainDuration: deployment.metrics.drainDuration,
        messagesHeld: deployment.metrics.messagesHeld,
        smokeTestResults: smokeTest,
        totalDuration: Date.now() - deployment.created_at
      });

      // Phase 6: CLEANUP (destroy old instance)
      await this.updatePhase(deployment.id, 'CLEANUP');
      await this.destroyInstance(instanceId);

      return { status: 'completed', deployment };
    } catch (error) {
      // Rollback on failure
      await this.rollback(deployment.id);
      throw error;
    }
  }
}
```

### 4.2 State Preservation (Redis)

**Purpose:** Maintain agent coordination state during deployment.

**State Types:**

| State | Redis Key | Purpose |
|-------|-----------|---------|
| Voting Results | `team:votes:{session_id}` | Multi-agent voting state |
| Conversation History | `agent:memory:{session_id}` | User conversation context |
| Model Caches | `agent:embeddings:cache` | Loaded embedding models |
| Circuit Breaker | `circuit_breaker:{agent}` | Error handling state |
| Team Registry | `teams:registry` | Active agent teams |

**Redis Configuration in Node-RED:**

```javascript
// settings.js (Node-RED)
contextStorage: {
  default: { module: "memory" },
  shared: {
    module: "redis",
    config: {
      host: process.env.REDIS_HOST || "redis",
      port: 6379,
      db: 0,
      prefix: "redforge:"
    }
  }
}
```

**Usage in Flows:**

```javascript
// Store voting state (persist across blue-green swap)
flow.set('team:votes', votes, 'shared');

// Retrieve in green instance
const votes = flow.get('team:votes', 'shared');
```

**Control Tower Responsibility:**
- Ensure Redis connection shared across blue/green instances
- Monitor Redis health during deployment
- Backup critical state before deployment (optional)

### 4.3 Message Buffering (MQTT)

**Purpose:** Queue incoming tasks during deployment to prevent data loss.

**MQTT Topic Structure:**

```
redforge/
├── ingest/input              # New tasks arrive here
├── hold/{instance-id}        # Tasks held during DRAIN phase
├── active/{instance-id}      # Tasks to active instance
└── dlq/{instance-id}         # Dead-letter queue (failed tasks)
```

**Deploy Gate Flow (in Node-RED):**

```javascript
// Check deployment mode
const mode = flow.get('deployment_mode', 'shared'); // 'active' or 'drain'

if (mode === 'drain') {
  // Hold message in MQTT queue
  node.send([null, msg]); // Output 2 = hold queue
} else {
  // Process normally
  node.send([msg, null]); // Output 1 = continue
}
```

**Control Tower Coordination:**

```typescript
// Signal DRAIN
await mqttClient.publish(`${instanceId}/control`, JSON.stringify({ mode: 'drain' }));

// Wait for in-flight messages to complete
await this.waitForDrain(instanceId);

// Signal RESUME (to green instance)
await mqttClient.publish(`${greenInstance.id}/control`, JSON.stringify({ mode: 'active' }));

// Replay held messages
const heldMessages = await mqttClient.subscribe(`redforge/hold/${instanceId}`);
for (const msg of heldMessages) {
  await mqttClient.publish(`redforge/active/${greenInstance.id}`, msg);
}
```

### 4.4 Checkpoint/Replay for Long-Running Workflows

**Problem:** Some AI workflows take minutes (e.g., processing 1,000-page document).

**Solution:** Checkpoint progress at strategic points, resume from checkpoint after deployment.

**Checkpoint Format:**

```json
{
  "checkpointId": "ckpt-abc123",
  "instanceId": "embeddings-agent-1",
  "workflowId": "workflow-456",
  "stage": "embeddings_generation",
  "state": {
    "pagesProcessed": 250,
    "totalPages": 1000,
    "embeddingsGenerated": 1500,
    "currentBatch": 6
  },
  "timestamp": "2026-02-15T10:30:00Z"
}
```

**Checkpoint Storage:** PostgreSQL table `workflow_checkpoints`

**Replay Logic (in Node-RED):**

```javascript
// Check for existing checkpoint
const checkpoint = flow.get(`checkpoint:${msg.metadata.workflowId}`, 'shared');

if (checkpoint) {
  // Resume from checkpoint
  msg.state = checkpoint.state;
  msg.metadata.resumed = true;
} else {
  // Start fresh
  msg.state = { pagesProcessed: 0, ... };
}

// Process...

// Create checkpoint every 100 pages
if (msg.state.pagesProcessed % 100 === 0) {
  flow.set(`checkpoint:${msg.metadata.workflowId}`, {
    stage: 'embeddings_generation',
    state: msg.state,
    timestamp: new Date().toISOString()
  }, 'shared');
}
```

### 4.5 Smoke Testing

**Purpose:** Verify green instance is functioning before routing production traffic.

**Tests Performed:**

1. **Health Check:** GET `http://green-instance:1880/` returns 200
2. **Flow Deployment Check:** Flows are deployed and active
3. **Sample Inference:** Send test message through agent flow, verify response
4. **Redis Connection:** Verify can read/write to Redis
5. **MQTT Connection:** Verify can publish/subscribe to MQTT
6. **External Services:** Verify can connect to Ollama, Milvus

**Smoke Test Implementation:**

```typescript
async runSmokeTest(instanceId: string): Promise<SmokeTestResult> {
  const tests: SmokeTest[] = [
    { name: 'health_check', test: () => this.checkHealth(instanceId) },
    { name: 'flows_deployed', test: () => this.checkFlowsDeployed(instanceId) },
    { name: 'sample_inference', test: () => this.runSampleInference(instanceId) },
    { name: 'redis_connection', test: () => this.checkRedis(instanceId) },
    { name: 'mqtt_connection', test: () => this.checkMQTT(instanceId) }
  ];

  const results = await Promise.all(tests.map(t => this.runTest(t)));
  const passed = results.every(r => r.passed);

  return { passed, results, timestamp: new Date() };
}
```

### 4.6 Rollback on Failure

**Trigger Conditions:**
- Smoke test failure
- Green instance crashes during deployment
- Manual rollback initiated by operator

**Rollback Procedure:**

```typescript
async rollback(deploymentId: string): Promise<void> {
  const deployment = await this.getDeployment(deploymentId);
  const { instance_id: blueInstanceId, metrics } = deployment;

  // 1. Stop routing to green instance
  if (metrics.greenInstanceId) {
    await this.mqttPublish(`${metrics.greenInstanceId}/control`, { mode: 'drain' });
  }

  // 2. Resume traffic to original blue instance
  await this.mqttPublish(`${blueInstanceId}/control`, { mode: 'active' });
  await this.updateLoadBalancer(metrics.greenInstanceId, blueInstanceId);

  // 3. Replay any messages that went to green
  await this.replayQueuedMessages(blueInstanceId);

  // 4. Destroy failed green instance
  if (metrics.greenInstanceId) {
    await this.destroyInstance(metrics.greenInstanceId);
  }

  // 5. Update deployment status
  await this.updateDeployment(deploymentId, { status: 'rolled_back' });
}
```

---

## 5. Enterprise Features

### 5.1 Role-Based Access Control (RBAC)

**Implemented in Control Tower (TypeScript):**

```typescript
// Role Definitions
enum Role {
  ADMIN = 'admin',       // Full access (create/delete instances, users)
  OPERATOR = 'operator', // Deploy, scale, monitor instances
  VIEWER = 'viewer'      // Read-only access
}

// Permission Model
interface Permission {
  resource: 'instances' | 'deployments' | 'users' | 'audit';
  action: 'create' | 'read' | 'update' | 'delete';
}

// Role → Permissions Mapping
const rolePermissions: Record<Role, Permission[]> = {
  [Role.ADMIN]: [
    { resource: '*', action: '*' } // Full access
  ],
  [Role.OPERATOR]: [
    { resource: 'instances', action: 'read' },
    { resource: 'instances', action: 'update' },
    { resource: 'deployments', action: '*' },
    { resource: 'audit', action: 'read' }
  ],
  [Role.VIEWER]: [
    { resource: 'instances', action: 'read' },
    { resource: 'deployments', action: 'read' },
    { resource: 'audit', action: 'read' }
  ]
};

// Authorization Middleware
function requirePermission(resource: string, action: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const user = req.user; // From auth middleware
    const hasPermission = await checkPermission(user, resource, action);

    if (!hasPermission) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    next();
  };
}

// Usage in Routes
router.post('/api/instances',
  authenticate,
  requirePermission('instances', 'create'),
  createInstance
);
```

### 5.2 Audit Logging

**Two Levels of Audit Logs:**

**1. Control Tower Actions (Management Plane)**

```sql
CREATE TABLE audit_logs (
  id BIGSERIAL PRIMARY KEY,
  timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  user_id VARCHAR(255),
  action VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50),
  resource_id VARCHAR(255),
  details JSONB,
  ip_address INET,
  user_agent TEXT
);
```

**Logged Actions:**
- `INSTANCE_CREATED`, `INSTANCE_STARTED`, `INSTANCE_STOPPED`, `INSTANCE_DESTROYED`
- `DEPLOYMENT_STARTED`, `DEPLOYMENT_COMPLETED`, `DEPLOYMENT_ROLLED_BACK`
- `USER_CREATED`, `USER_DELETED`, `ROLE_ASSIGNED`
- `CONFIG_UPDATED`

**2. AI Decision Logs (Data Plane - from Node-RED)**

```sql
CREATE TABLE ai_decision_logs (
  id BIGSERIAL PRIMARY KEY,
  timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  instance_id UUID NOT NULL,
  flow_name VARCHAR(255) NOT NULL,
  model VARCHAR(100),
  prompt TEXT,
  response TEXT,
  confidence FLOAT,
  tokens_used INTEGER,
  latency_ms INTEGER,
  metadata JSONB
);
```

**Logged Decisions:**
- Which model was used (llama3.2, mistral, GPT-4)
- Prompt sent to model
- Response generated
- Confidence score (for voting patterns)
- Token usage (for cost tracking)
- Latency (for performance monitoring)

**Audit Log Collection from Node-RED:**

```javascript
// In Node-RED flow (function node)
const logEntry = {
  flow_name: 'response-generation',
  model: msg.config.model || 'llama3.2',
  prompt: msg.payload.prompt,
  response: msg.payload.response,
  confidence: msg.payload.confidence || null,
  tokens_used: msg.payload.tokens_used || null,
  latency_ms: Date.now() - msg.metadata.startTime
};

// Send to Control Tower audit API
node.send({
  topic: 'audit/ai_decision',
  payload: logEntry
});
```

### 5.3 Resource Quotas

**Purpose:** Prevent runaway resource consumption per team/project.

**Quota Types:**

```typescript
interface ResourceQuota {
  teamId: string;
  maxInstances: number;           // Max concurrent Node-RED instances
  maxCPU: string;                 // e.g., "16 cores"
  maxMemory: string;              // e.g., "64Gi"
  maxDeploymentsPerDay: number;   // Rate limit deployments
  maxLLMTokensPerDay: number;     // Cost control
}
```

**Enforcement:**

```typescript
async validateQuota(teamId: string, requestedResources: Resources): Promise<boolean> {
  const quota = await this.getQuota(teamId);
  const currentUsage = await this.getCurrentUsage(teamId);

  // Check instance count
  if (currentUsage.instances >= quota.maxInstances) {
    throw new QuotaExceededError('Max instances exceeded');
  }

  // Check CPU
  const totalCPU = currentUsage.cpu + parseCPU(requestedResources.cpu);
  if (totalCPU > parseCPU(quota.maxCPU)) {
    throw new QuotaExceededError('CPU quota exceeded');
  }

  // Check memory
  const totalMemory = currentUsage.memory + parseMemory(requestedResources.memory);
  if (totalMemory > parseMemory(quota.maxMemory)) {
    throw new QuotaExceededError('Memory quota exceeded');
  }

  return true;
}
```

### 5.4 Compliance Reporting

**Purpose:** Generate reports for HIPAA, SOC 2, GDPR audits.

**Report Types:**

1. **Audit Trail Export** (all actions in date range)
2. **AI Decision Report** (all AI-generated responses with prompts)
3. **Access Report** (who accessed what resources)
4. **Data Retention Report** (ensure old data is purged per policy)

**Example: HIPAA Audit Report**

```typescript
async generateHIPAAReport(startDate: Date, endDate: Date): Promise<Report> {
  // 1. All access to AI decision logs (who queried patient data)
  const accessLogs = await this.db.query(`
    SELECT user_id, timestamp, action, resource_id
    FROM audit_logs
    WHERE timestamp BETWEEN $1 AND $2
    AND resource_type = 'ai_decision_logs'
    ORDER BY timestamp DESC
  `, [startDate, endDate]);

  // 2. All AI decisions made (what prompts, responses)
  const aiDecisions = await this.db.query(`
    SELECT flow_name, model, prompt, response, timestamp
    FROM ai_decision_logs
    WHERE timestamp BETWEEN $1 AND $2
    ORDER BY timestamp DESC
  `, [startDate, endDate]);

  // 3. Any security incidents (failed auth, quota violations)
  const incidents = await this.db.query(`
    SELECT timestamp, action, details
    FROM audit_logs
    WHERE timestamp BETWEEN $1 AND $2
    AND action IN ('AUTH_FAILED', 'QUOTA_EXCEEDED', 'UNAUTHORIZED_ACCESS')
    ORDER BY timestamp DESC
  `, [startDate, endDate]);

  return {
    reportType: 'HIPAA_AUDIT',
    dateRange: { start: startDate, end: endDate },
    sections: {
      accessLogs,
      aiDecisions,
      securityIncidents: incidents
    },
    generatedAt: new Date()
  };
}
```

### 5.5 Secrets Management

**Sensitive Data:**
- Database credentials (PostgreSQL, Redis)
- API keys (Ollama, OpenAI, Anthropic)
- MQTT credentials
- LLM model endpoints

**Storage Options:**

**1. Environment Variables (Development)**
```bash
DATABASE_URL=postgresql://user:pass@localhost:5432/controltower
REDIS_URL=redis://:password@localhost:6379
OLLAMA_API_KEY=sk-...
```

**2. Kubernetes Secrets (Production)**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: controltower-secrets
type: Opaque
data:
  database-url: <base64-encoded>
  redis-url: <base64-encoded>
  ollama-api-key: <base64-encoded>
```

**3. HashiCorp Vault (Enterprise)**
- Centralized secret storage
- Dynamic secrets (time-limited)
- Audit logging
- Secret rotation

**Control Tower Integration:**

```typescript
// Fetch secrets from Vault
const secrets = await vaultClient.read('secret/data/controltower');
process.env.DATABASE_URL = secrets.data.databaseUrl;
process.env.OLLAMA_API_KEY = secrets.data.ollamaApiKey;
```

---

## 6. Deployment Strategies

### 6.1 Docker Compose (Development/Small-Scale)

**Architecture:**
- Control Tower: Single container
- Node-RED Instances: Multiple containers (1 per agent type)
- Services: Redis, MQTT, PostgreSQL, Prometheus

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  # Control Tower Application
  control-tower:
    build: ./control-tower
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/controltower
      - REDIS_URL=redis://redis:6379
      - MQTT_URL=mqtt://mosquitto:1883
    depends_on:
      - postgres
      - redis
      - mosquitto
    volumes:
      - ./control-tower/logs:/app/logs

  # Node-RED Instance: Embeddings
  nodered-embeddings:
    image: nodered/node-red:4.1.2
    ports:
      - "1881:1880"
    environment:
      - OLLAMA_HOST=host.docker.internal
      - OLLAMA_PORT=11434
      - OLLAMA_MODEL=nomic-embed-text
      - REDIS_HOST=redis
      - MQTT_BROKER=mosquitto
    volumes:
      - ./flows/embeddings:/data
      - nodered-embeddings-data:/data
    depends_on:
      - redis
      - mosquitto

  # Node-RED Instance: Storage
  nodered-storage:
    image: nodered/node-red:4.1.2
    ports:
      - "1882:1880"
    environment:
      - MILVUS_HOST=milvus-standalone
      - MILVUS_PORT=19530
      - REDIS_HOST=redis
    volumes:
      - ./flows/storage:/data
      - nodered-storage-data:/data

  # Node-RED Instance: Query
  nodered-query:
    image: nodered/node-red:4.1.2
    ports:
      - "1883:1880"
    environment:
      - MILVUS_HOST=milvus-standalone
      - REDIS_HOST=redis
    volumes:
      - ./flows/query:/data
      - nodered-query-data:/data

  # Shared Services
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: controltower
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  mosquitto:
    image: eclipse-mosquitto:2
    ports:
      - "1883:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - mosquitto-data:/mosquitto/data

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/config:/etc/prometheus
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  postgres-data:
  redis-data:
  mosquitto-data:
  prometheus-data:
  grafana-data:
  nodered-embeddings-data:
  nodered-storage-data:
  nodered-query-data:
```

### 6.2 Kubernetes (Production)

**Architecture:**
- Control Tower: Deployment (3 replicas) + Service + Ingress
- Node-RED Instances: StatefulSet (persistent volumes)
- Services: Redis Cluster (or managed), MQTT, PostgreSQL (RDS/Cloud SQL)

**Kubernetes Manifests:**

```yaml
# Control Tower Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: control-tower
  namespace: redforge
spec:
  replicas: 3
  selector:
    matchLabels:
      app: control-tower
  template:
    metadata:
      labels:
        app: control-tower
    spec:
      containers:
      - name: control-tower
        image: redforge/control-tower:latest
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: control-tower-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: control-tower-secrets
              key: redis-url
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5

---
# Control Tower Service
apiVersion: v1
kind: Service
metadata:
  name: control-tower
  namespace: redforge
spec:
  selector:
    app: control-tower
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP

---
# Node-RED StatefulSet: Embeddings
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nodered-embeddings
  namespace: redforge
spec:
  serviceName: "nodered-embeddings"
  replicas: 3
  selector:
    matchLabels:
      app: nodered-embeddings
  template:
    metadata:
      labels:
        app: nodered-embeddings
    spec:
      containers:
      - name: nodered
        image: nodered/node-red:4.1.2
        ports:
        - containerPort: 1880
        env:
        - name: OLLAMA_HOST
          value: "ollama.redforge.svc.cluster.local"
        - name: OLLAMA_MODEL
          value: "nomic-embed-text"
        - name: REDIS_HOST
          value: "redis.redforge.svc.cluster.local"
        resources:
          requests:
            cpu: "2"
            memory: "8Gi"
          limits:
            cpu: "4"
            memory: "16Gi"
        volumeMounts:
        - name: flows
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: flows
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi

---
# Ingress (with TLS)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: control-tower-ingress
  namespace: redforge
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - control-tower.redforge.ai
    secretName: control-tower-tls
  rules:
  - host: control-tower.redforge.ai
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: control-tower
            port:
              number: 80
```

### 6.3 Edge Deployment

**Hybrid Cloud-Edge Architecture:**
- **Cloud:** Control Tower + shared services (Redis persistence, PostgreSQL)
- **Edge:** Node-RED instances + local Ollama (low-latency, privacy)
- **Synchronization:** Periodic vector uploads to cloud Milvus

**Use Cases:**
- Factory floor AI (low latency for production line decisions)
- Hospital AI (PHI stays on-premise, only aggregated metrics to cloud)
- Retail stores (local customer service, batch sync to central)

**Edge Node-RED Instance:**
```yaml
# Deployed at edge location (e.g., Raspberry Pi, industrial PC)
services:
  nodered-edge:
    image: nodered/node-red:4.1.2-minimal
    environment:
      - OLLAMA_HOST=localhost # Local Ollama
      - MILVUS_HOST=cloud.redforge.ai # Cloud Milvus
      - SYNC_INTERVAL=3600 # Sync every hour
    volumes:
      - ./flows/edge:/data
```

---

## 7. Integration Points

### 7.1 Node-RED Admin API

Control Tower uses Node-RED's built-in Admin API for instance management:

**Key Endpoints:**

```typescript
class NodeREDClient {
  constructor(private baseUrl: string) {}

  // Deploy flows to instance
  async deployFlows(flows: any[]): Promise<DeployResponse> {
    const response = await axios.post(
      `${this.baseUrl}/flows`,
      { flows, rev: this.getCurrentRev() },
      { headers: { 'Content-Type': 'application/json' } }
    );
    return response.data;
  }

  // Get currently deployed flows
  async getFlows(): Promise<any[]> {
    const response = await axios.get(`${this.baseUrl}/flows`);
    return response.data;
  }

  // Get instance settings
  async getSettings(): Promise<Settings> {
    const response = await axios.get(`${this.baseUrl}/settings`);
    return response.data;
  }

  // Health check
  async checkHealth(): Promise<boolean> {
    try {
      const response = await axios.get(`${this.baseUrl}/`, { timeout: 5000 });
      return response.status === 200;
    } catch {
      return false;
    }
  }

  // Get runtime metrics (requires contrib-stats node)
  async getMetrics(): Promise<RuntimeMetrics> {
    const response = await axios.get(`${this.baseUrl}/metrics`);
    return response.data;
  }
}
```

**Authentication:**
- Node-RED Admin API can be secured with `adminAuth` setting
- Control Tower stores credentials securely (Kubernetes secrets, Vault)

```javascript
// Node-RED settings.js
adminAuth: {
  type: "credentials",
  users: [{
    username: "controltower",
    password: "$2b$08$...", // bcrypt hash
    permissions: "*"
  }]
}
```

### 7.2 LLM Provider Integration (via Node-RED)

Node-RED instances connect directly to LLM providers. Control Tower tracks usage via audit logs.

**Supported Providers:**

**1. Ollama (Local Inference)**
```javascript
// In Node-RED flow
msg.url = `${process.env.OLLAMA_HOST}/api/generate`;
msg.payload = {
  model: "llama3.2",
  prompt: msg.payload.prompt,
  stream: false
};
// Send via HTTP Request node
```

**2. OpenAI API**
```javascript
msg.url = "https://api.openai.com/v1/chat/completions";
msg.headers = {
  "Authorization": `Bearer ${process.env.OPENAI_API_KEY}`
};
msg.payload = {
  model: "gpt-4",
  messages: [{ role: "user", content: msg.payload.prompt }]
};
```

**3. Anthropic Claude API**
```javascript
msg.url = "https://api.anthropic.com/v1/messages";
msg.headers = {
  "x-api-key": process.env.ANTHROPIC_API_KEY,
  "anthropic-version": "2023-06-01"
};
msg.payload = {
  model: "claude-opus-4",
  messages: [{ role: "user", content: msg.payload.prompt }],
  max_tokens: 1024
};
```

**Cost Tracking:**
- Node-RED logs token usage to Control Tower audit API
- Control Tower aggregates costs per instance/team
- Dashboard shows: tokens used, cost (at current rates), cost per workflow

### 7.3 Vector Database Integration

**Milvus (High-Scale Deployments)**
```javascript
// In Node-RED flow (Milvus node)
msg.collection = "rag_documents";
msg.payload = {
  data: [
    { vector: msg.payload.embedding, id: msg.payload.id, metadata: {...} }
  ]
};
// Insert via Milvus node
```

**pgvector (Simpler Deployments)**
```javascript
// In Node-RED flow (PostgreSQL node)
msg.topic = `
  INSERT INTO rag.embeddings (id, vector, metadata)
  VALUES ($1, $2::vector, $3)
`;
msg.payload = [msg.payload.id, msg.payload.embedding, JSON.stringify(msg.metadata)];
```

**Control Tower Role:**
- Monitor vector DB health (via Prometheus metrics)
- Track storage usage per instance
- Alert on storage capacity issues

### 7.4 Observability Integration

**Prometheus Metrics:**

Control Tower exposes metrics:
```
# Instance metrics
nodered_instance_up{instance_id="embeddings-1"} 1
nodered_instance_memory_bytes{instance_id="embeddings-1"} 8589934592
nodered_instance_cpu_usage{instance_id="embeddings-1"} 0.45

# Deployment metrics
controltower_deployment_duration_seconds{strategy="blue-green"} 120
controltower_deployment_success_total 45
controltower_deployment_failure_total 2

# Agent metrics (collected from Node-RED)
agent_workflow_duration_seconds{flow="embeddings"} 2.5
agent_llm_tokens_used_total{model="llama3.2"} 125000
agent_errors_total{flow="storage"} 12
```

**Grafana Dashboards:**

Control Tower includes pre-built dashboards:
1. **Fleet Overview:** Instance count, health, resource usage
2. **Deployment Dashboard:** Deployment history, success rate, duration
3. **Performance Dashboard:** Workflow latency, throughput, error rate
4. **Cost Dashboard:** LLM token usage, compute costs, cost per workflow
5. **Agent Coordination:** Voting patterns, team activity, decision quality

**OpenTelemetry (Future):**
- Distributed tracing across Control Tower → Node-RED → Ollama → Milvus
- Trace visualization in Jaeger
- Root cause analysis for slow workflows

---

## 8. Scalability & Performance

### 8.1 Control Tower Scalability

**Horizontal Scaling:**
- Run 3+ Control Tower replicas behind load balancer
- Session affinity for WebSocket connections
- Shared state in PostgreSQL + Redis

**Load Balancing:**
```nginx
upstream control-tower {
  server control-tower-1:3000;
  server control-tower-2:3000;
  server control-tower-3:3000;

  # WebSocket support
  keepalive 32;
}

server {
  listen 80;
  server_name control-tower.redforge.ai;

  location / {
    proxy_pass http://control-tower;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
  }
}
```

**Performance Targets:**
- API response time: <100ms (p95), <500ms (p99)
- WebSocket latency: <50ms
- Concurrent connections: 10,000+
- Deployments: 100+ per hour

### 8.2 Node-RED Instance Scalability

**Horizontal Scaling:**
- Deploy 2-4 replicas per agent type
- Load balance via Nginx or Kubernetes Service
- Share state via Redis

**Scaling Strategies:**

**1. Static Scaling (Pre-provisioned)**
```yaml
# 3 replicas of embeddings agent (for HA)
replicas: 3
```

**2. Horizontal Pod Autoscaler (Kubernetes)**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nodered-embeddings-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: nodered-embeddings
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**3. KEDA (Event-Driven Autoscaling)**
- Scale based on MQTT queue depth
- Scale based on inference request rate
- Scale to zero when idle (cost savings)

### 8.3 Performance Optimization

**Database Optimization:**
- Connection pooling (PostgreSQL)
- Query optimization (indexes, EXPLAIN ANALYZE)
- Read replicas for audit log queries

**Caching:**
- Redis cache for frequent queries (instance status, metrics)
- CDN for static assets (UI)
- Memoization for expensive computations

**Network Optimization:**
- HTTP/2 for API (multiplexing)
- WebSocket compression
- gRPC for Control Tower ↔ Node-RED (future)

**Monitoring:**
- Prometheus metrics for bottleneck identification
- Distributed tracing for end-to-end latency analysis
- Profiling (Node.js --prof, flame graphs)

---

## 9. Security Architecture

### 9.1 Authentication & Authorization

**Authentication Methods:**
- **JWT:** Stateless tokens for API access
- **OAuth2:** Integration with identity providers (Okta, Auth0, Azure AD)
- **API Keys:** For programmatic access (CI/CD)

**Authorization:**
- RBAC (Admin, Operator, Viewer roles)
- Permission checks on every API endpoint
- Audit logging for all access attempts

### 9.2 Network Security

**TLS Encryption:**
- HTTPS for all Control Tower API traffic
- TLS for PostgreSQL connections
- TLS for Redis connections (if sensitive data)
- MQTT over TLS (MQTTS)

**Network Segmentation:**
- Control Tower in management network
- Node-RED instances in workload network
- Services in private network
- Ingress controller as single entry point

**Firewall Rules:**
```
# Allow Control Tower → Node-RED Admin API
ALLOW 3000:3000 → 1880:1880 (HTTP)

# Allow Node-RED → Services
ALLOW 1880:1880 → 5432 (PostgreSQL)
ALLOW 1880:1880 → 6379 (Redis)
ALLOW 1880:1880 → 1883 (MQTT)
ALLOW 1880:1880 → 11434 (Ollama)
ALLOW 1880:1880 → 19530 (Milvus)

# Deny all other traffic
DENY *
```

### 9.3 Data Security

**Encryption at Rest:**
- PostgreSQL: Transparent Data Encryption (TDE)
- Redis: AOF/RDB encryption
- Milvus: Encrypted storage backend
- Secrets: Kubernetes Secrets or HashiCorp Vault

**PII Detection:**
- Node-RED flows scan for PII (SSN, credit card, email)
- Auto-redaction or alerting
- Compliance with GDPR, CCPA

**Data Retention:**
- Configurable retention policies
- Automated purging of old audit logs
- Backup and disaster recovery procedures

---

## Related Documentation

- **[project_plan.md](project_plan.md)** - Implementation roadmap
- **[DEPLOYMENT_STRATEGIES.md](DEPLOYMENT_STRATEGIES.md)** - Deployment patterns
- **[CLUSTERING_AND_REDEPLOYMENT.md](CLUSTERING_AND_REDEPLOYMENT.md)** - Zero-downtime deployment details
- **[Main Project Architecture](https://github.com/nagual69/RedForge-Agentic-AI/blob/main/docs/ARCHITECTURE.md)** - AI agent primitives

---

**Built with ❤️ on TypeScript + Node.js + Node-RED**
