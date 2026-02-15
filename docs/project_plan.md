# RedForge AI Control Tower - Implementation Roadmap

**Project**: RedForge AI Control Tower for Multi-Agent Node-RED Workflows
**Repository**: [nodered-flowfuse-control-tower](https://github.com/nagual69/nodered-flowfuse-control-tower)
**Status**: 🚧 Planning Phase - NOT STARTED
**Last Updated**: 2026-02-15

---

## Overview

This project implements **Generation 3** of the RedForge-Agentic-AI evolution: a **TypeScript-based enterprise management platform** for orchestrating fleets of Node-RED instances running distributed multi-agent AI workflows.

**What This Project Builds:**
- **Control Tower Application:** TypeScript/Node.js management app with web UI
- **Fleet Management:** Deploy, monitor, scale Node-RED instances
- **Zero-Downtime Deployment:** Blue-green deployment for agent flows
- **Enterprise Governance:** RBAC, audit logging, resource quotas
- **Fleet Observability:** Real-time dashboards, distributed tracing, analytics

**Architecture:**
```
┌─Control Tower (TypeScript App)──────────────┐
│  • Web UI (React/Vue)                       │
│  • REST API + WebSocket Server              │
│  • Orchestrates deployments                 │
│  • Manages governance & observability       │
└─────────────────────────────────────────────┘
             │ Manages ↓
┌─Managed Node-RED Instances──────────────────┐
│  Instance 1    │ Instance 2    │ Instance 3 │
│  (File Conv,   │ (Embeddings,  │ (Query,    │
│   Text Chunk,  │  Storage)     │  Response) │
│   Error Handle)│               │            │
│  Node-RED 4.1.2│ Node-RED 4.1.2│ Node-RED   │
└─────────────────────────────────────────────┘
```

---

## Prerequisites

**From Main Project** ([RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI)):
- ✅ Gen 1: Production RAG pipeline (complete)
- ✅ Gen 2: Multi-agent architecture (8 primitives, 24 flows, 96/100 standards alignment)
- ✅ Performance benchmarks validated (multi-agent benefits proven)
- ✅ Production stability demonstrated (4+ weeks uptime on standalone)

**Technical Prerequisites:**
- Docker & Docker Compose installed
- Node.js 18+ and npm/yarn
- PostgreSQL 16+ (or Docker image)
- Redis 7+ (or Docker image)
- MQTT broker (Mosquitto recommended)
- Git for version control

**Optional:**
- Kubernetes cluster (for production deployment)
- Prometheus + Grafana (for observability)
- HashiCorp Vault (for secrets management)

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-3)

#### Week 1: Control Tower Backend Setup
**Goal:** Bootstrap TypeScript/Node.js application with database

**Tasks:**
- [ ] Initialize TypeScript project
  - [ ] Setup package.json with dependencies (Express, Socket.io, Prisma/TypeORM)
  - [ ] Configure tsconfig.json (strict mode, ES2022)
  - [ ] Setup ESLint + Prettier for code quality
  - [ ] Configure Jest for testing
- [ ] Database schema design
  - [ ] Design PostgreSQL schema (instances, deployments, audit_logs, ai_decision_logs, users, roles)
  - [ ] Create Prisma schema or TypeORM entities
  - [ ] Write migration scripts
  - [ ] Setup database connection pool
- [ ] Docker infrastructure
  - [ ] Create docker-compose.yml for development stack
    - [ ] PostgreSQL container
    - [ ] Redis container
    - [ ] Mosquitto MQTT container
    - [ ] Prometheus container (optional)
    - [ ] Grafana container (optional)
  - [ ] Network configuration (bridge network for inter-container communication)
  - [ ] Volume mounts for persistence
- [ ] Basic Express server
  - [ ] Create src/index.ts with Express setup
  - [ ] Health check endpoint: GET /api/health
  - [ ] CORS configuration
  - [ ] Error handling middleware
  - [ ] Logging setup (Winston or Pino)

**Deliverables:**
- Control Tower backend running on http://localhost:3000
- PostgreSQL database initialized with schema
- Docker Compose stack operational

**Success Criteria:**
- `GET /api/health` returns 200 OK
- Database migrations run successfully
- All containers healthy in Docker Compose

---

#### Week 2: Node-RED Instance Management
**Goal:** Implement core API for deploying and managing Node-RED instances

**Tasks:**
- [ ] Instance Manager Service
  - [ ] Implement `InstanceManagerService` class
  - [ ] Docker integration via Dockerode library
  - [ ] Deploy Node-RED container with environment variables
  - [ ] Start, stop, restart instance operations
  - [ ] Health check polling (Node-RED Admin API)
  - [ ] Resource allocation (CPU, memory limits)
- [ ] Node-RED Admin API client
  - [ ] Create `NodeREDClient` class (Axios-based)
  - [ ] Deploy flows endpoint: `POST /flows`
  - [ ] Get flows endpoint: `GET /flows`
  - [ ] Get settings endpoint: `GET /settings`
  - [ ] Health check: `GET /`
  - [ ] Authentication support (adminAuth credentials)
- [ ] REST API endpoints
  - [ ] `POST /api/instances` - Deploy new Node-RED instance
  - [ ] `GET /api/instances` - List all instances
  - [ ] `GET /api/instances/:id` - Get instance details
  - [ ] `PATCH /api/instances/:id` - Update instance (scale, config)
  - [ ] `DELETE /api/instances/:id` - Destroy instance
  - [ ] `POST /api/instances/:id/start` - Start instance
  - [ ] `POST /api/instances/:id/stop` - Stop instance
- [ ] Request validation
  - [ ] Implement Zod or Joi schemas
  - [ ] Validate instance creation payloads
  - [ ] Validate resource limits
- [ ] Frontend scaffolding
  - [ ] Initialize React or Vue.js project
  - [ ] Setup routing (React Router or Vue Router)
  - [ ] Create basic layout component
  - [ ] API client setup (Axios)

**Deliverables:**
- REST API functional for instance lifecycle management
- First Node-RED instance deployable via API
- Frontend project initialized

**Success Criteria:**
- Can deploy Node-RED instance via `POST /api/instances`
- Instance appears in Docker containers list
- Can query instance status via API
- Node-RED UI accessible on assigned port

---

#### Week 3: Fleet Dashboard MVP
**Goal:** Build web UI for fleet overview and instance monitoring

**Tasks:**
- [ ] Health Monitor Service
  - [ ] Implement `HealthMonitorService` class
  - [ ] Background polling loop (every 30s)
  - [ ] Store health status in database
  - [ ] Detect status changes (running → failed)
  - [ ] Emit WebSocket events on status change
- [ ] Metrics Collector Service
  - [ ] Implement `MetricsCollectorService` class
  - [ ] Collect Docker stats (CPU, memory, network)
  - [ ] Aggregate metrics per instance
  - [ ] Store metrics in PostgreSQL or time-series DB
- [ ] WebSocket server
  - [ ] Setup Socket.io on Express server
  - [ ] Implement authentication for WebSocket connections
  - [ ] Real-time instance status updates
  - [ ] Real-time metrics streaming
- [ ] Fleet Dashboard UI
  - [ ] **Instance List Page**
    - [ ] Grid view of all instances
    - [ ] Health status badges (green/yellow/red)
    - [ ] Resource utilization bars (CPU, memory)
    - [ ] Quick actions (start, stop, deploy)
  - [ ] **Instance Detail Page**
    - [ ] Instance metadata (name, flows, resources, env vars)
    - [ ] Health history chart
    - [ ] Recent logs (if available)
    - [ ] Action buttons (restart, redeploy, destroy)
  - [ ] WebSocket integration for live updates
  - [ ] Error handling and loading states

**Deliverables:**
- Fleet Dashboard accessible at http://localhost:3000
- Real-time instance status updates
- Health monitoring operational

**Success Criteria:**
- Dashboard displays all deployed instances
- Status changes reflect in real-time (<5s latency)
- Can start/stop instances from UI
- Metrics visible (CPU, memory)

---

### Phase 2: Zero-Downtime Deployment (Weeks 4-6)

#### Week 4: Blue-Green Orchestration
**Goal:** Implement deployment state machine for zero-downtime updates

**Tasks:**
- [ ] Deployment Orchestrator Service
  - [ ] Implement `DeploymentOrchestratorService` class
  - [ ] Deployment state machine:
    ```typescript
    enum DeploymentPhase {
      DRAIN = 'DRAIN',
      DEPLOY = 'DEPLOY',
      VERIFY = 'VERIFY',
      RESUME = 'RESUME',
      AUDIT = 'AUDIT',
      CLEANUP = 'CLEANUP'
    }
    ```
  - [ ] `deployBlueGreen(instanceId, newFlowsPath)` method
  - [ ] Phase tracking in database (deployments table)
  - [ ] Progress calculation (% complete)
- [ ] MQTT integration
  - [ ] Setup MQTT client (mqtt.js)
  - [ ] Publish drain/resume signals
  - [ ] Subscribe to acknowledgment topics
  - [ ] Message buffering coordination
- [ ] Instance cloning logic
  - [ ] Clone instance configuration (env vars, resources)
  - [ ] Deploy new Docker container (green instance)
  - [ ] Assign unique name and ports
  - [ ] Wait for healthy state
- [ ] Deployment API endpoints
  - [ ] `POST /api/deployments` - Initiate blue-green deployment
  - [ ] `GET /api/deployments` - List deployment history
  - [ ] `GET /api/deployments/:id` - Get deployment status
  - [ ] `POST /api/deployments/:id/rollback` - Rollback deployment
- [ ] DRAIN phase implementation
  - [ ] Publish `{instance_id}/control` with `{ mode: 'drain' }`
  - [ ] Wait for in-flight messages to complete
  - [ ] Poll MQTT queue depth
  - [ ] Timeout handling (default 5 minutes)

**Deliverables:**
- Deployment orchestration API functional
- DRAIN phase working (messages queued)
- Green instance deployment automated

**Success Criteria:**
- Can initiate deployment via API
- Old instance enters DRAIN mode
- New green instance starts healthy
- Deployment tracked in database

---

#### Week 5: State Preservation & Verification
**Goal:** Implement Redis state coordination and smoke testing

**Tasks:**
- [ ] Redis coordination
  - [ ] Configure Redis client (ioredis)
  - [ ] Verify shared context stores across blue/green
  - [ ] Test voting state preservation
  - [ ] Test conversation history persistence
- [ ] Checkpoint/replay mechanism
  - [ ] Design checkpoint schema (PostgreSQL table: workflow_checkpoints)
  - [ ] Implement checkpoint creation in Node-RED flows (via function nodes)
  - [ ] Implement checkpoint replay on green instance startup
  - [ ] Test with long-running workflow (100+ pages document)
- [ ] Smoke testing framework
  - [ ] Implement `runSmokeTest(instanceId)` method
  - [ ] Test suite:
    - [ ] Health check (GET /)
    - [ ] Flows deployed check (GET /flows)
    - [ ] Sample inference test (send test message through flow)
    - [ ] Redis connection test (read/write test key)
    - [ ] MQTT connection test (pub/sub test)
    - [ ] External service checks (Ollama, Milvus)
  - [ ] Test result aggregation
  - [ ] Pass/fail determination
- [ ] Rollback automation
  - [ ] Implement `rollback(deploymentId)` method
  - [ ] Stop green instance
  - [ ] Resume blue instance from DRAIN
  - [ ] Replay queued messages to blue
  - [ ] Destroy failed green instance
  - [ ] Update deployment status to 'rolled_back'
- [ ] VERIFY and RESUME phases
  - [ ] VERIFY: Execute smoke tests, check results
  - [ ] RESUME: Publish `{green_id}/control` with `{ mode: 'active' }`
  - [ ] Update load balancer/routing
  - [ ] Replay held messages from MQTT queue

**Deliverables:**
- State preserved across blue/green swap
- Smoke tests automated
- Rollback functional

**Success Criteria:**
- Voting state survives deployment
- Smoke tests catch deployment failures
- Failed deployment triggers automatic rollback
- Queued messages replayed on RESUME

---

#### Week 6: Deployment UI
**Goal:** Build web UI for deployment management

**Tasks:**
- [ ] Deployment Manager Page
  - [ ] **Initiate Deployment Form**
    - [ ] Select instance dropdown
    - [ ] Upload flow JSON file or select from Git
    - [ ] Deployment strategy selector (blue-green, canary - future)
    - [ ] Confirmation dialog with impact summary
  - [ ] **Deployment Progress Visualization**
    - [ ] Phase timeline (DRAIN → DEPLOY → VERIFY → RESUME → AUDIT → CLEANUP)
    - [ ] Current phase highlight
    - [ ] Progress bar (0-100%)
    - [ ] Real-time metrics display:
      - Messages held in queue
      - In-flight message count
      - Green instance health status
      - Smoke test results (live)
  - [ ] **Deployment History Table**
    - [ ] List all past deployments
    - [ ] Status badges (completed, failed, rolled_back)
    - [ ] Duration, timestamp, user
    - [ ] Drill-down to deployment details
  - [ ] **Rollback Controls**
    - [ ] Manual rollback button (confirmation required)
    - [ ] Rollback confirmation dialog with warnings
- [ ] WebSocket integration for live updates
  - [ ] Subscribe to `deployment:phase` events
  - [ ] Subscribe to `deployment:progress` events
  - [ ] Update UI in real-time (<2s latency)
- [ ] Notifications
  - [ ] Success toast on deployment completion
  - [ ] Error toast on deployment failure
  - [ ] Browser notifications (optional)

**Deliverables:**
- Deployment Manager UI functional
- Real-time deployment progress visible
- Rollback UI operational

**Success Criteria:**
- Can initiate deployment from UI
- See real-time phase updates
- Smoke test results displayed live
- Can rollback from UI
- Deployment history browsable

---

### Phase 3: Governance & Observability (Weeks 7-9)

#### Week 7: RBAC Implementation
**Goal:** Implement role-based access control

**Tasks:**
- [ ] Authentication layer
  - [ ] Choose strategy: JWT or OAuth2
  - [ ] Implement login endpoint: `POST /api/auth/login`
  - [ ] Implement token generation (JWT) or OAuth2 flow
  - [ ] Implement token validation middleware
  - [ ] Implement logout endpoint: `POST /api/auth/logout`
  - [ ] Password hashing (bcrypt)
- [ ] User management
  - [ ] Database schema: users table (id, username, email, password_hash, role)
  - [ ] API endpoints:
    - [ ] `GET /api/users` - List users (admin only)
    - [ ] `POST /api/users` - Create user (admin only)
    - [ ] `GET /api/users/:id` - Get user details
    - [ ] `PATCH /api/users/:id` - Update user
    - [ ] `DELETE /api/users/:id` - Delete user (admin only)
- [ ] Role definitions
  - [ ] Define roles enum: ADMIN, OPERATOR, VIEWER
  - [ ] Define permissions per role:
    - ADMIN: Full access (create/delete instances, users)
    - OPERATOR: Deploy, scale, monitor instances (no user management)
    - VIEWER: Read-only access (no modifications)
  - [ ] Permission model:
    ```typescript
    interface Permission {
      resource: 'instances' | 'deployments' | 'users' | 'audit';
      action: 'create' | 'read' | 'update' | 'delete';
    }
    ```
- [ ] Authorization middleware
  - [ ] Implement `requirePermission(resource, action)` middleware
  - [ ] Check user role against permission model
  - [ ] Return 403 Forbidden if unauthorized
  - [ ] Apply to all protected routes
- [ ] Frontend authentication
  - [ ] Login page
  - [ ] Token storage (localStorage or httpOnly cookies)
  - [ ] Protected routes (redirect to login if not authenticated)
  - [ ] Role-based UI rendering (hide admin-only buttons for viewers)

**Deliverables:**
- RBAC fully implemented
- Users can login with role-based permissions
- Unauthorized actions blocked

**Success Criteria:**
- ADMIN can create/delete instances and users
- OPERATOR can deploy but not delete instances
- VIEWER can only view (all modification buttons disabled)
- Unauthorized API calls return 403

---

#### Week 8: Audit Logging
**Goal:** Implement comprehensive audit logging for compliance

**Tasks:**
- [ ] Audit Logger Service
  - [ ] Implement `AuditLoggerService` class
  - [ ] Log all Control Tower actions:
    - INSTANCE_CREATED, INSTANCE_STARTED, INSTANCE_STOPPED, INSTANCE_DESTROYED
    - DEPLOYMENT_STARTED, DEPLOYMENT_COMPLETED, DEPLOYMENT_ROLLED_BACK
    - USER_CREATED, USER_DELETED, ROLE_ASSIGNED
    - CONFIG_UPDATED
  - [ ] Capture metadata: user_id, action, resource_type, resource_id, details (JSONB), ip_address, user_agent
  - [ ] Store in audit_logs table (PostgreSQL)
- [ ] AI decision logging integration
  - [ ] API endpoint for Node-RED instances: `POST /api/audit/ai_decision`
  - [ ] Accept log entries:
    ```typescript
    {
      flow_name: string;
      model: string;
      prompt: string;
      response: string;
      confidence?: number;
      tokens_used?: number;
      latency_ms: number;
    }
    ```
  - [ ] Store in ai_decision_logs table
  - [ ] Index by timestamp, instance_id, model
- [ ] Audit log query API
  - [ ] `GET /api/audit` - Query audit logs
    - Query params: user_id, action, resource_type, start_date, end_date, limit, offset
  - [ ] `GET /api/audit/ai_decisions` - Query AI decision logs
    - Query params: instance_id, model, start_date, end_date, limit, offset
  - [ ] Pagination support (limit/offset)
  - [ ] Export endpoint: `GET /api/audit/export` (CSV or JSON)
- [ ] Compliance reporting
  - [ ] Implement report generation functions:
    - [ ] HIPAA Audit Report (access logs, AI decisions, security incidents)
    - [ ] SOC 2 Audit Report (system access, configuration changes)
    - [ ] GDPR Data Access Report (user data queries)
  - [ ] API endpoint: `POST /api/reports/generate`
  - [ ] Report formats: PDF, CSV, JSON
- [ ] Audit UI
  - [ ] Governance Console page
  - [ ] Audit log table with filters (user, action, date range)
  - [ ] AI decision log table with filters (instance, model, date range)
  - [ ] Export button
  - [ ] Report generation UI

**Deliverables:**
- Dual-level audit logging operational
- AI decisions logged from Node-RED instances
- Compliance reports generated

**Success Criteria:**
- Every Control Tower action logged
- AI decisions visible in audit logs
- Can export full audit trail
- Compliance report generation functional

---

#### Week 9: Fleet Monitoring Dashboards
**Goal:** Build real-time fleet monitoring UI with embedded Grafana

**Tasks:**
- [ ] Metrics Collector Service (enhanced)
  - [ ] Scrape Prometheus metrics from Node-RED instances
  - [ ] Aggregate fleet-wide statistics:
    - Total instances, healthy/unhealthy counts
    - Total active workflows
    - Average latency across fleet
    - Total LLM token usage
    - Total cost (estimated)
  - [ ] Expose metrics API: `GET /api/metrics/fleet`
  - [ ] Per-instance metrics: `GET /api/metrics/instances/:id`
- [ ] Prometheus integration
  - [ ] Configure Prometheus to scrape Control Tower metrics
  - [ ] Configure Prometheus to scrape Node-RED instance metrics (if node-red-contrib-prometheus installed)
  - [ ] Define custom metrics:
    - `nodered_instance_up{instance_id}`
    - `nodered_instance_cpu_usage{instance_id}`
    - `nodered_instance_memory_bytes{instance_id}`
    - `controltower_deployment_duration_seconds{strategy}`
    - `controltower_deployment_success_total`
    - `controltower_deployment_failure_total`
    - `agent_workflow_duration_seconds{flow}`
    - `agent_llm_tokens_used_total{model}`
- [ ] Grafana dashboards
  - [ ] **Fleet Overview Dashboard**
    - Instance count gauge (total, healthy, failed)
    - Resource utilization chart (CPU, memory across fleet)
    - Workflow throughput chart (workflows/minute)
    - Error rate chart (errors/minute)
  - [ ] **Deployment Dashboard**
    - Deployment success rate gauge
    - Deployment duration histogram
    - Recent deployments table
  - [ ] **Performance Dashboard**
    - Workflow latency percentiles (p50, p95, p99) per primitive
    - Throughput by primitive (queries/sec, embeddings/sec)
    - Error rate by primitive
  - [ ] **Cost Dashboard**
    - LLM token usage by model (bar chart)
    - Estimated cost by model (based on token pricing)
    - Cost per workflow (calculated)
  - [ ] Export dashboard JSON files
- [ ] Grafana embedding in Control Tower UI
  - [ ] Performance Analytics page
  - [ ] Embed Grafana dashboards via iframe
  - [ ] Single sign-on (SSO) setup (optional)
  - [ ] Dashboard selector (Fleet, Deployment, Performance, Cost)
- [ ] Alerting configuration UI
  - [ ] Alerting rules page
  - [ ] Create alert rule form (condition, threshold, notification channel)
  - [ ] List active alerts
  - [ ] Alert history

**Deliverables:**
- Fleet monitoring dashboards operational
- Grafana embedded in Control Tower UI
- Alerting configured

**Success Criteria:**
- Fleet metrics visible in real-time
- Grafana dashboards display meaningful data
- Alerts trigger on instance failures
- Performance bottlenecks identifiable

---

### Phase 4: Production Hardening (Weeks 10-12)

#### Week 10: High Availability
**Goal:** Ensure Control Tower and Node-RED instances are highly available

**Tasks:**
- [ ] Control Tower HA
  - [ ] Refactor for stateless operation (all state in PostgreSQL/Redis)
  - [ ] Test multi-instance deployment (3 replicas)
  - [ ] Setup Nginx or HAProxy load balancer:
    - Round-robin for HTTP requests
    - Session affinity for WebSocket connections
  - [ ] Test failover (kill one instance, verify traffic routes to others)
  - [ ] Kubernetes manifests (if deploying to K8s):
    - Deployment with 3 replicas
    - Service (ClusterIP)
    - HorizontalPodAutoscaler (scale 3-10 based on CPU)
- [ ] Node-RED instance redundancy
  - [ ] Implement instance redundancy configuration (2-3 replicas per agent type)
  - [ ] Instance Manager: Support deploying multiple replicas
  - [ ] Load balancing across replicas:
    - MQTT topic-based routing (round-robin)
    - HTTP endpoint load balancing (via Nginx or K8s Service)
  - [ ] Shared Redis context (already implemented in Week 5)
  - [ ] Test failover: Kill one replica, verify traffic routes to survivors
- [ ] Automatic failover detection
  - [ ] Health Monitor: Detect instance failures (3 consecutive failed health checks)
  - [ ] Automatic instance restart (via Docker restart policy or K8s liveness probe)
  - [ ] Alert on repeated failures (instance crashlooping)
- [ ] Database HA (PostgreSQL)
  - [ ] Setup PostgreSQL replication (primary + 1-2 replicas)
  - [ ] Automatic failover (via Patroni or managed service like RDS Multi-AZ)
  - [ ] Connection pooling (PgBouncer)
- [ ] Redis HA
  - [ ] Setup Redis Sentinel (3+ nodes) or Redis Cluster
  - [ ] Automatic failover on primary failure
  - [ ] Client configuration for Sentinel/Cluster

**Deliverables:**
- Control Tower runs with 3+ replicas
- Node-RED instances have 2+ replicas per agent type
- Automatic failover tested and functional

**Success Criteria:**
- Kill 1 Control Tower replica → No downtime
- Kill 1 Node-RED replica → Traffic routes to others
- Database failover completes in <30s
- Redis failover completes in <10s

---

#### Week 11: Disaster Recovery
**Goal:** Implement backup/restore procedures

**Tasks:**
- [ ] Backup strategy
  - [ ] **PostgreSQL Backups**
    - [ ] Automated daily backups (pg_dump or WAL archiving)
    - [ ] Backup retention policy (30 days daily, 12 months monthly)
    - [ ] Backup storage (S3, GCS, or local NFS)
    - [ ] Backup validation (weekly restore test)
  - [ ] **Redis Backups**
    - [ ] RDB snapshots (daily)
    - [ ] AOF persistence enabled
    - [ ] Backup to S3/GCS
  - [ ] **Node-RED Flow Backups**
    - [ ] Flow JSON version control (Git repository)
    - [ ] Automated Git push on flow deployment
    - [ ] Tag releases (v1.0, v1.1, etc.)
  - [ ] **Control Tower Configuration Backups**
    - [ ] Export users, roles, policies (JSON)
    - [ ] Store in Git repository
- [ ] Restore procedures
  - [ ] **Database Restore**
    - [ ] Restore from latest pg_dump
    - [ ] Point-in-time recovery (PITR) from WAL archives
    - [ ] Verify data integrity post-restore
  - [ ] **Redis Restore**
    - [ ] Restore from RDB snapshot
    - [ ] Replay AOF if available
  - [ ] **Flow Restore**
    - [ ] Checkout specific Git tag
    - [ ] Redeploy flows to Node-RED instances
  - [ ] **Configuration Restore**
    - [ ] Import users, roles, policies from JSON
- [ ] DR runbook
  - [ ] **Scenario 1: Database Corruption**
    - Step-by-step restore from backup
    - Estimated RTO: 30 minutes
    - Estimated RPO: 24 hours (daily backup)
  - [ ] **Scenario 2: Node-RED Instance Failure (Complete Loss)**
    - Redeploy instance via Control Tower
    - Restore flows from Git
    - Estimated RTO: 10 minutes
  - [ ] **Scenario 3: Control Tower Failure (Complete Loss)**
    - Redeploy Control Tower (Docker/K8s)
    - Restore database from backup
    - Restore configuration
    - Estimated RTO: 1 hour
  - [ ] **Scenario 4: Region Failure (Multi-Region Setup)**
    - Failover to secondary region
    - Database replication catchup
    - Estimated RTO: 15 minutes
- [ ] DR testing
  - [ ] Schedule quarterly DR drills
  - [ ] Test full database restore
  - [ ] Test flow redeployment from Git
  - [ ] Document test results

**Deliverables:**
- Automated backup system operational
- Restore procedures documented and tested
- DR runbook validated

**Success Criteria:**
- Backups run daily without failures
- Database restore completes in <30 minutes
- Flow restore completes in <10 minutes
- DR drill succeeds

---

#### Week 12: Performance & Security
**Goal:** Optimize performance and harden security

**Tasks:**
- [ ] Performance optimization
  - [ ] **API Latency**
    - Profile API endpoints (p95, p99 latency)
    - Optimize slow queries (add indexes, query optimization)
    - Implement caching (Redis) for frequent queries
    - Target: p95 < 100ms, p99 < 500ms
  - [ ] **WebSocket Scaling**
    - Test WebSocket connections under load (10,000+ clients)
    - Implement Redis adapter for Socket.io (multi-instance support)
    - Test horizontal scaling (add more Control Tower replicas)
  - [ ] **Database Performance**
    - Add indexes on frequently queried columns (timestamp, instance_id, user_id)
    - Partition large tables (audit_logs, ai_decision_logs) by date
    - Vacuum and analyze tables
  - [ ] **Node-RED Instance Performance**
    - Profile agent flows (identify slow nodes)
    - Optimize function nodes (avoid blocking operations)
    - Tune Node.js heap size if needed
- [ ] Load testing
  - [ ] Setup load testing tool (k6, Locust, or Artillery)
  - [ ] Test scenarios:
    - [ ] 1,000 concurrent ingestion workflows
    - [ ] 10,000 queries/minute across fleet
    - [ ] 100 concurrent deployments
  - [ ] Measure:
    - API latency under load
    - WebSocket message throughput
    - Database query times
    - Node-RED instance throughput
  - [ ] Identify bottlenecks and optimize
  - [ ] Target: System handles load with <5% error rate
- [ ] Security hardening
  - [ ] **TLS Encryption**
    - [ ] Enable HTTPS for Control Tower (Let's Encrypt certificate)
    - [ ] Enable TLS for PostgreSQL connections
    - [ ] Enable TLS for Redis connections
    - [ ] Enable MQTTS (MQTT over TLS)
  - [ ] **Rate Limiting**
    - [ ] Implement rate limiting middleware (express-rate-limit)
    - [ ] Per-user rate limits (100 req/min for VIEWER, 1000 req/min for ADMIN)
    - [ ] Per-IP rate limits for unauthenticated endpoints (10 req/min for /login)
  - [ ] **Input Validation**
    - [ ] Review all API endpoints for input validation
    - [ ] Sanitize user inputs (prevent SQL injection, XSS)
    - [ ] Validate file uploads (flow JSON files)
  - [ ] **Secrets Management**
    - [ ] Move secrets out of environment variables
    - [ ] Integrate HashiCorp Vault or AWS Secrets Manager
    - [ ] Rotate secrets regularly (quarterly)
  - [ ] **Security Headers**
    - [ ] Implement Helmet.js for security headers
    - [ ] Content-Security-Policy, X-Frame-Options, etc.
  - [ ] **Dependency Scanning**
    - [ ] Run npm audit and fix vulnerabilities
    - [ ] Setup Dependabot or Snyk for automated dependency updates
- [ ] Security audit
  - [ ] Internal security review (checklist):
    - [ ] RBAC enforcement tested
    - [ ] SQL injection tests (all API endpoints)
    - [ ] XSS tests (all user inputs)
    - [ ] Authentication bypass attempts
  - [ ] External penetration testing (optional, if budget allows)

**Deliverables:**
- Performance optimizations applied
- Load testing passed
- Security hardening complete

**Success Criteria:**
- API p95 latency < 100ms under load
- System handles 1,000 concurrent workflows
- All high/critical security vulnerabilities patched
- Security audit passed

---

### Phase 5: Advanced Features (Weeks 13-16)

#### Week 13: Multi-Region Support
**Goal:** Enable deployment across multiple geographic regions

**Tasks:**
- [ ] Region-aware architecture
  - [ ] Database schema: Add `region` field to instances table
  - [ ] Instance Manager: Support region parameter on deploy
  - [ ] Region topology:
    - [ ] **Control Tower:** Single global instance or regional replicas
    - [ ] **Node-RED Instances:** Deployed in specific regions
    - [ ] **PostgreSQL:** Multi-region replication (primary in one region, read replicas in others)
    - [ ] **Redis:** Multi-region replication or per-region Redis clusters
- [ ] Region-aware routing
  - [ ] Implement routing logic: Route queries to nearest region
  - [ ] Latency-based routing (measure RTT to each region)
  - [ ] Failover routing (if one region fails, route to next nearest)
- [ ] Cross-region replication
  - [ ] PostgreSQL: Setup logical replication between regions
  - [ ] Redis: Setup Redis Cross-Region Replication or manual sync
  - [ ] Vector DB (Milvus): Cross-region collection replication (complex, may defer)
- [ ] Multi-region deployment guide
  - [ ] Document how to deploy Control Tower in multi-region setup
  - [ ] Provide Kubernetes manifests for multi-cluster deployment
  - [ ] Network configuration (VPN, VPC peering, or public internet with TLS)

**Deliverables:**
- Multi-region deployment architecture designed
- Region-aware routing implemented
- Cross-region replication functional

**Success Criteria:**
- Deploy Node-RED instances in 3 regions (US, EU, APAC)
- Queries route to nearest region
- Failover tested (disable one region, traffic routes to others)

---

#### Week 14: Edge Coordination
**Goal:** Support hybrid cloud-edge deployments

**Tasks:**
- [ ] Edge agent architecture
  - [ ] Lightweight Node-RED instances for edge (resource-constrained devices)
  - [ ] Local Ollama deployment (edge inference)
  - [ ] Local Redis for edge agent coordination
  - [ ] Minimal flow set (File Converter, Query, Response)
- [ ] Cloud-edge synchronization
  - [ ] Design sync protocol:
    - [ ] Edge → Cloud: Periodic vector uploads (batch, every 1 hour)
    - [ ] Cloud → Edge: Model updates, configuration changes
  - [ ] Implement sync agent (Node-RED flow or standalone script)
  - [ ] Sync API endpoints:
    - [ ] `POST /api/sync/vectors` - Upload vectors from edge
    - [ ] `GET /api/sync/config/:instance_id` - Download config for edge
- [ ] Edge deployment automation
  - [ ] Create edge deployment script (Docker Compose for edge devices)
  - [ ] Auto-register edge instance with Control Tower on first boot
  - [ ] Health monitoring for edge instances (intermittent connectivity handling)
- [ ] Use case implementation
  - [ ] **Factory Floor AI:** Edge instance processes sensor data locally, uploads aggregated vectors
  - [ ] **Retail Store AI:** Edge instance handles customer queries locally, syncs daily
  - [ ] Test latency: Edge query response <500ms (vs 2000ms cloud)

**Deliverables:**
- Edge agent deployment functional
- Cloud-edge sync working
- Use case tested

**Success Criteria:**
- Edge instance runs on Raspberry Pi or similar
- Vectors sync from edge to cloud successfully
- Edge queries respond with low latency (<500ms)
- Edge instance survives network disconnects

---

#### Week 15: Advanced Deployment Patterns
**Goal:** Implement canary deployments and A/B testing

**Tasks:**
- [ ] Canary deployment strategy
  - [ ] Design canary deployment flow:
    - Deploy new version to 10% of instances
    - Monitor metrics (error rate, latency)
    - If metrics acceptable, roll out to 50%, then 100%
    - If metrics degrade, rollback
  - [ ] Implement `deployCanary(instanceId, newFlowsPath, canaryPercent)` method
  - [ ] Traffic splitting logic (via MQTT routing or load balancer weights)
  - [ ] Automated canary analysis:
    - Compare error rates (canary vs baseline)
    - Compare latency (canary vs baseline)
    - Auto-promote or auto-rollback based on thresholds
- [ ] A/B testing framework
  - [ ] Design A/B test configuration:
    ```typescript
    {
      testName: "llama3.2 vs mistral",
      variantA: { instanceId: "response-1", weight: 50 },
      variantB: { instanceId: "response-2", weight: 50 },
      duration: "7 days",
      metrics: ["accuracy", "latency", "cost"]
    }
    ```
  - [ ] Traffic splitter (route 50% to variant A, 50% to variant B)
  - [ ] Metrics collection per variant
  - [ ] A/B test results dashboard (compare accuracy, latency, cost)
  - [ ] Winner selection (manual or automated based on metrics)
- [ ] Deployment UI updates
  - [ ] Add "Canary" strategy option to deployment form
  - [ ] Canary config: canaryPercent slider, monitoring window duration
  - [ ] A/B test creation form
  - [ ] A/B test results page (charts comparing variants)

**Deliverables:**
- Canary deployment strategy functional
- A/B testing framework operational
- Deployment UI supports canary and A/B

**Success Criteria:**
- Can deploy canary (10% traffic) and monitor metrics
- Canary auto-promotes if metrics good
- Can run A/B test comparing two LLM models
- A/B test results show clear winner

---

#### Week 16: Model Marketplace Integration
**Goal:** Integrate external LLM providers and model versioning

**Tasks:**
- [ ] LLM provider abstraction
  - [ ] Design provider interface:
    ```typescript
    interface LLMProvider {
      name: string; // "ollama", "openai", "anthropic"
      generateEmbedding(text: string, model: string): Promise<number[]>;
      generateResponse(prompt: string, model: string): Promise<string>;
      estimateCost(tokens: number, model: string): number;
    }
    ```
  - [ ] Implement providers:
    - [ ] Ollama provider (existing)
    - [ ] OpenAI provider (text-embedding-3, GPT-4)
    - [ ] Anthropic provider (Claude Opus 4)
    - [ ] Google Vertex AI provider (optional)
- [ ] Model registry
  - [ ] Database schema: models table (name, provider, version, cost_per_token)
  - [ ] API endpoints:
    - [ ] `GET /api/models` - List available models
    - [ ] `POST /api/models` - Register new model
    - [ ] `GET /api/models/:id` - Get model details (cost, performance benchmarks)
  - [ ] Model metadata: cost per token, latency benchmarks, quality scores
- [ ] Model routing logic
  - [ ] Cost-based routing: Select cheapest model that meets SLA
    - Example: Use llama3.2 (local, free) if latency <2s acceptable, else GPT-4
  - [ ] Fallback chains: Try Ollama → OpenAI if local fails
  - [ ] SLA enforcement: If latency exceeds threshold, switch to faster model
- [ ] Model versioning
  - [ ] Track model versions (llama3.2:v1, llama3.2:v2)
  - [ ] Support running multiple versions simultaneously (A/B test old vs new)
  - [ ] Rollback to previous model version if new version underperforms
- [ ] Model benchmarking
  - [ ] Benchmark framework:
    - [ ] Test dataset (100 sample queries)
    - [ ] Run queries through each model
    - [ ] Measure accuracy (human eval or automated metrics)
    - [ ] Measure latency (p50, p95, p99)
    - [ ] Calculate cost (total tokens * cost per token)
  - [ ] Benchmarking UI:
    - [ ] Model comparison table (accuracy, latency, cost)
    - [ ] Recommendation: "Best for quality: GPT-4, Best for cost: llama3.2"

**Deliverables:**
- Multi-provider support functional
- Model registry operational
- Cost-based routing implemented

**Success Criteria:**
- Can switch between Ollama, OpenAI, Anthropic via API
- Cost per workflow tracked by model
- Cheapest model auto-selected for non-critical queries
- Model benchmarking produces accurate comparison

---

## Timeline Summary

| Phase | Duration | Status |
|-------|----------|--------|
| **Prerequisites** | - | Waiting for Gen 2 validation from main project |
| Phase 1: Foundation | 3 weeks | Not started |
| Phase 2: Zero-Downtime Deployment | 3 weeks | Not started |
| Phase 3: Governance & Observability | 3 weeks | Not started |
| Phase 4: Production Hardening | 3 weeks | Not started |
| Phase 5: Advanced Features | 4 weeks | Not started |
| **Total** | **16 weeks (~4 months)** | - |

---

## Success Metrics

### Technical Metrics
- **Uptime:** 99.9% across entire fleet
- **API Latency:** p95 < 100ms, p99 < 500ms
- **WebSocket Latency:** < 50ms
- **Deployment Speed:** < 10 minutes for zero-downtime agent updates
- **Scalability:** Support 1,000+ concurrent workflows
- **HA:** Single instance failure causes 0 downtime

### Business Metrics
- **Developer Productivity:** 50% reduction in deployment time (vs manual)
- **Cost Efficiency:** 30% reduction in cost per workflow (via resource optimization)
- **Compliance:** 100% audit coverage for AI decisions
- **Quality:** 95%+ accuracy on multi-agent voting patterns

---

## When to Start This Project

**Prerequisites Checklist:**
- [ ] Main project Phase 4 complete (multi-agent primitives stable)
- [ ] Performance benchmarks show clear improvement:
  - [ ] Multi-agent quality ≥ 1.2x single-agent baseline
  - [ ] Multi-agent cost ≤ 0.2x single large model (GPT-4)
  - [ ] Multi-agent latency ≤ 1.5x single-agent (or faster with parallelism)
- [ ] Production stability metrics (4+ weeks uptime on standalone Node-RED)
- [ ] Resource utilization data collected (CPU, memory per primitive)
- [ ] Enterprise demand validated (users requesting fleet management features)
- [ ] Team capacity available (doesn't compete with Gen 2 maintenance)
- [ ] Budget allocated (infrastructure + development time)

**Do NOT Start If:**
- Gen 2 multi-agent teams are unstable or underperforming
- No enterprise use cases identified
- Budget constraints (unclear infrastructure costs)

---

## Risk Management

### High Risks
1. **Blue-Green Deployment Complexity**
   - **Risk:** State preservation across blue/green swap fails, causing data loss
   - **Mitigation:** Extensive testing in Phase 2, rollback automation
2. **Performance Under Load**
   - **Risk:** System can't handle 1,000+ concurrent workflows
   - **Mitigation:** Load testing in Phase 4, horizontal scaling, caching
3. **Multi-Region Latency**
   - **Risk:** Cross-region replication causes unacceptable query latency
   - **Mitigation:** Region-aware routing, read replicas, edge deployment

### Medium Risks
1. **Third-Party Dependencies**
   - **Risk:** Docker, Kubernetes, Redis changes break Control Tower
   - **Mitigation:** Pin dependency versions, automated testing
2. **Security Vulnerabilities**
   - **Risk:** SQL injection, XSS, or authentication bypass
   - **Mitigation:** Security audit in Phase 4, dependency scanning, input validation

### Low Risks
1. **Team Learning Curve**
   - **Risk:** Team unfamiliar with TypeScript, Node-RED Admin API
   - **Mitigation:** Training during Phase 1, pair programming

---

## Related Documentation

**Architecture & Design:**
- [ARCHITECTURE.md](ARCHITECTURE.md) - Comprehensive technical architecture
- [DEPLOYMENT_STRATEGIES.md](DEPLOYMENT_STRATEGIES.md) - Deployment patterns (to be created)
- [CLUSTERING_AND_REDEPLOYMENT.md](CLUSTERING_AND_REDEPLOYMENT.md) - Zero-downtime deployment details

**Technical Guides:**
- [AI_AGENT_PRIMITIVES.md](AI_AGENT_PRIMITIVES.md) - 8 primitive flows (to be created)
- [MULTI_AGENT_PATTERNS.md](MULTI_AGENT_PATTERNS.md) - Coordination patterns (to be created)
- [USE_CASES.md](USE_CASES.md) - Industry use cases (to be created)

**Main Project:**
- [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI) - Gen 2 multi-agent architecture

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-02-15 | Complete rewrite with TypeScript Control Tower architecture (removed FlowFuse migration, added new phase structure) |
| 0.1.1 | 2026-02-15 | Updated repository references to RedForge-Agentic-AI |
| 0.1.0 | 2026-01-17 | Initial project plan (FlowFuse-based) |

---

**Note**: This project is currently in the planning phase. Development will begin once the prerequisites from the main [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI) project are satisfied.
