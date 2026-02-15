# RedForge AI Control Tower - Deployment Strategies

**Version**: 2.0.0
**Date**: February 2026
**Status**: Planning Phase

---

## Table of Contents

1. [Architecture Options](#1-architecture-options)
2. [Deployment Platforms](#2-deployment-platforms)
3. [Migration Paths](#3-migration-paths)
4. [Sizing Guidelines](#4-sizing-guidelines)
5. [Best Practices](#5-best-practices)

---

## 1. Architecture Options

### 1.1 Single-Instance Development

**Overview:**
All agent flows run in one Node-RED instance, managed by a single Control Tower replica.

**Architecture:**
```
┌─Control Tower────────┐
│  • API (1 replica)   │
│  • UI (embedded)     │
└──────────────────────┘
         │ Manages
         ↓
┌─Node-RED Instance────┐
│  • All 8 primitives   │
│  • 2-4 vCPU           │
│  • 16GB RAM           │
└──────────────────────┘
         │ Connects to
         ↓
┌─Shared Services──────┐
│  • PostgreSQL        │
│  • Redis             │
│  • MQTT (Mosquitto)  │
│  • Ollama            │
│  • Milvus/pgvector   │
└──────────────────────┘
```

**Suitable For:**
- Development and prototyping
- Small-scale demos
- Single-developer environments
- Testing new agent flows

**Resource Requirements:**
- Control Tower: 2 vCPU, 4GB RAM
- Node-RED: 2-4 vCPU, 16GB RAM
- Services: 4 vCPU, 16GB RAM
- **Total: 8-10 vCPU, 36GB RAM**

**Pros:**
- Simple deployment (single Docker Compose file)
- Low cost (~$50-100/month on cloud)
- Easy debugging (everything in one place)
- Fast local development cycle

**Cons:**
- No high availability
- Limited scalability (single instance bottleneck)
- No resource isolation (one agent can starve others)
- Single point of failure

**Setup Time:** < 1 hour

---

### 1.2 Multi-Instance Production

**Overview:**
Each agent type runs on dedicated Node-RED instances, with Control Tower managing the fleet.

**Architecture:**
```
┌─Control Tower (HA)───────────────┐
│  • API (3 replicas)              │
│  • Load Balancer                 │
│  • UI (embedded)                 │
└──────────────────────────────────┘
         │ Manages fleet ↓
┌────────────────────────────────────────────────┐
│  Node-RED Fleet (Isolated Instances)           │
├────────────────────────────────────────────────┤
│  Instance 1: File Converter (2 replicas)       │
│  Instance 2: Text Chunker (2 replicas)         │
│  Instance 3: Embeddings (3 replicas, 8GB)     │
│  Instance 4: Storage (3 replicas, 8GB)        │
│  Instance 5: Query (3 replicas, 4GB)          │
│  Instance 6: Response (3 replicas, 8GB)       │
│  Instance 7: Error Handler (2 replicas)        │
│  Instance 8: Memory (2 replicas, 4GB)         │
└────────────────────────────────────────────────┘
         │ Connects to
         ↓
┌─Shared Services (HA)──────────────┐
│  • PostgreSQL (primary + replica) │
│  • Redis Cluster (3+ nodes)       │
│  • MQTT (3 Mosquitto nodes)       │
│  • Ollama (2+ nodes)               │
│  • Milvus Cluster                  │
└────────────────────────────────────┘
```

**Suitable For:**
- Production deployments
- Enterprise environments
- Multi-tenant systems
- High-availability requirements

**Resource Requirements:**
- Control Tower: 6 vCPU, 12GB RAM (3 replicas × 2 vCPU, 4GB)
- Node-RED Fleet: 40-50 vCPU, 100-120GB RAM (20 instances)
- Services: 8-12 vCPU, 32-48GB RAM
- **Total: 54-68 vCPU, 144-180GB RAM**

**Pros:**
- High availability (2-3 replicas per agent type)
- Horizontal scalability (add more replicas as needed)
- Resource isolation (each agent type sized appropriately)
- Automatic failover (Control Tower routes to healthy instances)
- Independent scaling (scale embeddings without scaling query)

**Cons:**
- Higher complexity (20+ containers/pods)
- Higher cost (~$800-1500/month on cloud)
- Network latency (inter-instance communication)
- Requires orchestration platform (Kubernetes recommended)

**Setup Time:** 1-2 days (with Kubernetes)

---

### 1.3 Hybrid Cloud-Edge

**Overview:**
Edge Node-RED instances process data locally (low-latency, privacy), cloud Control Tower coordinates overall workflow.

**Architecture:**
```
┌─Cloud Control Tower─────────────────────────┐
│  • Central orchestration                    │
│  • Fleet monitoring                         │
│  • Configuration management                 │
│  • Aggregated analytics                     │
└─────────────────────────────────────────────┘
         │ Manages (via Internet/VPN)
         ↓
┌─Edge Location 1 (Factory)────────────────────┐
│  └─ Node-RED Edge Instance                   │
│     ├─ File Converter (local documents)     │
│     ├─ Embeddings (local Ollama)            │
│     ├─ Query (local inference)              │
│     └─ Sync Agent (batch upload vectors)    │
└──────────────────────────────────────────────┘

┌─Edge Location 2 (Retail Store)───────────────┐
│  └─ Node-RED Edge Instance                   │
│     ├─ Query (customer service queries)     │
│     ├─ Response (cached responses)           │
│     └─ Sync Agent (batch upload logs)        │
└──────────────────────────────────────────────┘

┌─Edge Location 3 (Hospital)────────────────────┐
│  └─ Node-RED Edge Instance                   │
│     ├─ File Converter (PHI documents)        │
│     ├─ Embeddings (local, HIPAA-compliant)  │
│     └─ Sync Agent (anonymized vectors only)  │
└──────────────────────────────────────────────┘
```

**Suitable For:**
- Distributed operations (factories, retail stores, hospitals)
- Low-latency requirements (edge inference < 500ms)
- Data privacy regulations (PHI, PII must stay local)
- Intermittent connectivity scenarios

**Resource Requirements:**
- Cloud Control Tower: 6 vCPU, 12GB RAM
- Cloud Services: 8 vCPU, 32GB RAM
- **Per Edge Location:**
  - Edge Node-RED: 2-4 vCPU, 8-16GB RAM
  - Edge Ollama: 4-8 vCPU, 16-32GB RAM (with GPU preferred)
  - Edge Redis: 1 vCPU, 2GB RAM

**Pros:**
- Low latency (edge inference typically < 500ms vs 2000ms cloud)
- Data sovereignty (sensitive data stays local)
- Resilient to network outages (edge functions independently)
- Cost savings (reduced cloud data transfer)

**Cons:**
- Complex deployment (multiple sites)
- Edge hardware management (Raspberry Pi, industrial PCs, etc.)
- Synchronization challenges (eventual consistency)
- Limited edge compute (may need model quantization)

**Setup Time:** 1-2 weeks (per edge location)

---

## 2. Deployment Platforms

### 2.1 Docker Compose (Development & Small-Scale)

**Best For:** Development, small-scale production (<100 concurrent workflows)

**Setup Time:** < 1 hour

**Pros:**
- Simple configuration (single `docker-compose.yml`)
- Fast local development
- Easy to troubleshoot (all logs in one place)
- No Kubernetes expertise required

**Cons:**
- Manual scaling (edit `docker-compose.yml`, restart)
- No automatic failover
- Limited to single host (no multi-node)
- No built-in service mesh

**Sample `docker-compose.yml`:**

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

  # Node-RED Instance: Embeddings Agent
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
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 8G

  # Node-RED Instance: Storage Agent
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
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 8G

  # Node-RED Instance: Query Agent
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
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G

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

**Scaling:**
```bash
# Scale embeddings instances to 3 replicas
docker-compose up -d --scale nodered-embeddings=3

# View logs
docker-compose logs -f control-tower
docker-compose logs -f nodered-embeddings
```

---

### 2.2 Kubernetes (Production & Auto-Scaling)

**Best For:** Production deployments, auto-scaling, enterprise

**Setup Time:** 1-2 days

**Pros:**
- Automatic scaling (HorizontalPodAutoscaler)
- Self-healing (automatic pod restarts)
- Rolling updates (zero-downtime deployments)
- Multi-node support (distribute across cluster)
- Service mesh integration (Istio for advanced networking)
- Resource quotas and limits
- Secrets management (Kubernetes Secrets or Vault)

**Cons:**
- Complex configuration (YAML manifests)
- Steep learning curve
- Requires Kubernetes expertise
- Higher operational overhead

**Sample Kubernetes Manifests:**

**Control Tower Deployment:**
```yaml
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

**Node-RED StatefulSet (Embeddings):**
```yaml
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
apiVersion: v1
kind: Service
metadata:
  name: nodered-embeddings
  namespace: redforge
spec:
  selector:
    app: nodered-embeddings
  ports:
  - port: 1880
    targetPort: 1880
  type: ClusterIP
```

**HorizontalPodAutoscaler (Auto-Scaling):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nodered-embeddings-hpa
  namespace: redforge
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

**Scaling Strategies:**

1. **Static Scaling (Manual):**
   ```bash
   kubectl scale statefulset nodered-embeddings --replicas=5 -n redforge
   ```

2. **HorizontalPodAutoscaler (CPU/Memory-based):**
   - Automatically scales based on CPU/memory usage
   - Target: 70% CPU, 80% memory utilization

3. **KEDA (Event-Driven Autoscaling):**
   ```yaml
   apiVersion: keda.sh/v1alpha1
   kind: ScaledObject
   metadata:
     name: nodered-embeddings-scaled
     namespace: redforge
   spec:
     scaleTargetRef:
       name: nodered-embeddings
     minReplicaCount: 2
     maxReplicaCount: 10
     triggers:
     - type: prometheus
       metadata:
         serverAddress: http://prometheus.redforge.svc:9090
         metricName: agent_workflow_queue_depth
         query: sum(agent_workflow_queue_depth{flow="embeddings"})
         threshold: '50'
   ```

---

### 2.3 Bare Metal (Air-Gapped & High-Security)

**Best For:** Air-gapped environments, high-security deployments, regulatory compliance

**Setup Time:** 1 week

**Pros:**
- Full control over infrastructure
- No cloud vendor lock-in
- Suitable for air-gapped networks
- Compliance with strict security policies (e.g., government, military)

**Cons:**
- Manual server provisioning
- Complex networking setup
- Requires dedicated DevOps team
- Scaling is manual (add physical servers)
- High upfront cost (hardware purchase)

**Setup Overview:**

1. **Server Provisioning:**
   - Control Tower: 3 servers (16 vCPU, 32GB RAM each)
   - Node-RED Instances: 20 servers (4-8 vCPU, 8-16GB RAM each)
   - Services: 5 servers (8 vCPU, 32GB RAM each)
   - **Total: 28 physical servers**

2. **Network Configuration:**
   - Private VLAN for Control Tower ↔ Node-RED communication
   - Separate VLAN for services (PostgreSQL, Redis, MQTT)
   - Firewall rules (restrict access by IP + port)

3. **Software Installation:**
   - Install Docker on all servers
   - Deploy services with systemd units:
     ```ini
     # /etc/systemd/system/control-tower.service
     [Unit]
     Description=RedForge Control Tower
     After=docker.service
     Requires=docker.service

     [Service]
     ExecStart=/usr/bin/docker run --rm \
       --name control-tower \
       -p 3000:3000 \
       -e DATABASE_URL=... \
       redforge/control-tower:latest
     Restart=always

     [Install]
     WantedBy=multi-user.target
     ```

4. **Scaling Process:**
   - Add physical server
   - Install Docker + systemd service
   - Register server with Control Tower (manual API call)

---

## 3. Migration Paths

### 3.1 From Standalone Node-RED to Control Tower

**Scenario:** You have a single Node-RED instance with all 8 agent flows, want to migrate to Control Tower-managed fleet.

**Duration:** 2-4 weeks

**Steps:**

**Phase 1: Deploy Control Tower Infrastructure (Week 1)**

1. **Deploy Control Tower Stack:**
   ```bash
   # Clone repository
   git clone https://github.com/nagual69/nodered-flowfuse-control-tower.git
   cd nodered-flowfuse-control-tower

   # Deploy infrastructure
   cd docker
   docker-compose up -d postgres redis mosquitto prometheus grafana

   # Start Control Tower
   cd ../control-tower
   npm install
   npm run build
   npm start
   ```

2. **Access Control Tower UI:** http://localhost:3000
   - Create first user (admin)
   - Configure PostgreSQL connection
   - Verify health check

**Phase 2: Export Flows from Standalone (Week 1)**

1. **Export all flows from standalone Node-RED:**
   - Open standalone Node-RED UI: http://localhost:1880
   - Menu → Export → All Flows → Download as JSON

2. **Separate primitives into individual files:**
   ```bash
   # Split monolithic flows.json into primitives
   examples/flows/primitives/
   ├── file-converter/convert-file-to-text-tika.json
   ├── text-chunking/text-chunking.json
   ├── embeddings/generate-embeddings-ollama.json
   ├── storage/milvus-storage-advanced.json
   ├── query-processing/query-processing.json
   ├── response-generation/response-generation.json
   ├── error-handling/error-handler.json
   └── memory/hindsight-memory.json
   ```

**Phase 3: Deploy Agent Instances via Control Tower (Week 2)**

1. **Deploy non-critical primitives first (low risk):**

   **File Converter:**
   ```bash
   curl -X POST http://localhost:3000/api/instances \
     -H "Content-Type: application/json" \
     -d '{
       "name": "file-converter-1",
       "flows": ["examples/flows/primitives/file-converter/convert-file-to-text-tika.json"],
       "resources": { "cpu": "1", "memory": "2Gi" },
       "env": {
         "TIKA_HOST": "tika",
         "REDIS_HOST": "redis",
         "MQTT_BROKER": "mosquitto"
       }
     }'
   ```

   **Text Chunker:**
   ```bash
   curl -X POST http://localhost:3000/api/instances \
     -H "Content-Type: application/json" \
     -d '{
       "name": "text-chunker-1",
       "flows": ["examples/flows/primitives/text-chunking/text-chunking.json"],
       "resources": { "cpu": "1", "memory": "2Gi" },
       "env": {
         "REDIS_HOST": "redis",
         "MQTT_BROKER": "mosquitto"
       }
     }'
   ```

2. **Deploy critical primitives (Week 2):**

   **Embeddings:**
   ```bash
   curl -X POST http://localhost:3000/api/instances \
     -H "Content-Type: application/json" \
     -d '{
       "name": "embeddings-1",
       "flows": ["examples/flows/primitives/embeddings/generate-embeddings-ollama.json"],
       "resources": { "cpu": "2", "memory": "8Gi" },
       "env": {
         "OLLAMA_HOST": "host.docker.internal",
         "OLLAMA_MODEL": "nomic-embed-text",
         "REDIS_HOST": "redis",
         "MQTT_BROKER": "mosquitto"
       }
     }'
   ```

   Repeat for: Storage, Query, Response, Error Handler, Memory

**Phase 4: Update Inter-Instance Communication (Week 2-3)**

1. **Refactor flows to use MQTT instead of direct sub-flow calls:**

   **OLD (Standalone - Direct Sub-Flow):**
   ```javascript
   // In orchestrator flow:
   msg.payload = { text: "..." };
   return msg; // Goes to Embeddings sub-flow in same instance
   ```

   **NEW (Control Tower - MQTT):**
   ```javascript
   // In File Converter flow:
   const mqttClient = global.get('mqttClient');
   mqttClient.publish('redforge/embeddings/input', JSON.stringify({
     text: msg.payload.text,
     metadata: {
       flowId: msg.metadata.flowId,
       timestamp: new Date().toISOString(),
       source: 'file-converter-1'
     }
   }));
   ```

   **In Embeddings flow (subscribe to MQTT):**
   ```javascript
   // MQTT In node subscribed to 'redforge/embeddings/input'
   const payload = JSON.parse(msg.payload);
   msg.payload = payload.text;
   msg.metadata = payload.metadata;
   return msg;
   ```

**Phase 5: Cutover Traffic (Week 3-4)**

1. **Enable MQTT routing in Control Tower:**
   - Configure MQTT topic routing rules (Control Tower → Agent Instances)

2. **Run parallel testing:**
   - Send 10% of traffic to new Control Tower architecture
   - Compare results with standalone (accuracy, latency)
   - Fix any discrepancies

3. **Full cutover:**
   - Route 100% of traffic to Control Tower
   - Monitor for 48 hours
   - Shut down standalone Node-RED instance

**Validation:**
- [ ] All 8 primitives running on separate Node-RED instances
- [ ] MQTT communication working (messages routed correctly)
- [ ] Control Tower dashboard shows all instances healthy
- [ ] End-to-end workflow completes successfully
- [ ] Latency within acceptable range (< 2x standalone)

---

### 3.2 From Single-Model LLM to Multi-Agent

**Scenario:** You're using a single large model (GPT-4) directly, want multi-agent voting for better accuracy at lower cost.

**Duration:** 1-2 weeks

**Steps:**

**Phase 1: Deploy Control Tower with 3 Response Agent Instances (Week 1)**

1. **Deploy Model A - llama3.2 (fast, local):**
   ```bash
   curl -X POST http://localhost:3000/api/instances \
     -H "Content-Type: application/json" \
     -d '{
       "name": "response-llama32",
       "flows": ["examples/flows/primitives/response-generation/response-generation.json"],
       "resources": { "cpu": "2", "memory": "8Gi" },
       "env": {
         "OLLAMA_HOST": "host.docker.internal",
         "OLLAMA_MODEL": "llama3.2",
         "REDIS_HOST": "redis"
       }
     }'
   ```

2. **Deploy Model B - mistral (balanced):**
   ```bash
   curl -X POST http://localhost:3000/api/instances \
     -H "Content-Type: application/json" \
     -d '{
       "name": "response-mistral",
       "flows": ["examples/flows/primitives/response-generation/response-generation.json"],
       "resources": { "cpu": "2", "memory": "8Gi" },
       "env": {
         "OLLAMA_HOST": "host.docker.internal",
         "OLLAMA_MODEL": "mistral",
         "REDIS_HOST": "redis"
       }
     }'
   ```

3. **Deploy Model C - phi3 (efficient):**
   ```bash
   curl -X POST http://localhost:3000/api/instances \
     -H "Content-Type: application/json" \
     -d '{
       "name": "response-phi3",
       "flows": ["examples/flows/primitives/response-generation/response-generation.json"],
       "resources": { "cpu": "2", "memory": "8Gi" },
       "env": {
         "OLLAMA_HOST": "host.docker.internal",
         "OLLAMA_MODEL": "phi3",
         "REDIS_HOST": "redis"
       }
     }'
   ```

**Phase 2: Implement Voting Coordinator (Week 1)**

1. **Create voting coordinator flow** (`voting-coordinator.json`):
   ```javascript
   // Voting Coordinator Flow
   // Step 1: Receive query
   msg.payload = { query: "What is photosynthesis?" };

   // Step 2: Broadcast to 3 response agents
   const agents = ['response-llama32', 'response-mistral', 'response-phi3'];
   for (const agent of agents) {
     mqttClient.publish(`redforge/${agent}/input`, JSON.stringify({
       query: msg.payload.query,
       sessionId: msg.metadata.sessionId
     }));
   }

   // Step 3: Collect responses (wait for all 3)
   flow.set('voting:' + msg.metadata.sessionId, {
     responses: [],
     expectedCount: 3,
     startTime: Date.now()
   });

   // (In separate MQTT subscriber node)
   // Step 4: Aggregate responses
   const session = flow.get('voting:' + msg.metadata.sessionId);
   session.responses.push({
     agent: msg.metadata.source,
     response: msg.payload.response,
     confidence: msg.payload.confidence
   });

   if (session.responses.length === session.expectedCount) {
     // Step 5: Vote (select best response)
     const bestResponse = session.responses.reduce((best, current) => {
       return current.confidence > best.confidence ? current : best;
     });

     msg.payload = {
       response: bestResponse.response,
       winner: bestResponse.agent,
       allResponses: session.responses
     };

     return msg; // Send to user
   }
   ```

**Phase 3: A/B Test Multi-Agent vs Single-Model (Week 2)**

1. **Run A/B test:**
   - Route 50% of queries to multi-agent voting
   - Route 50% to GPT-4 (baseline)
   - Collect metrics: accuracy, latency, cost

2. **Compare Results:**
   | Metric | GPT-4 (Single) | Multi-Agent (Voting) | Improvement |
   |--------|----------------|---------------------|-------------|
   | Accuracy | 85% | 92% | +7% |
   | Latency | 2.1s | 3.5s | -1.4s (slower) |
   | Cost/Query | $0.03 | $0.005 | -83% |

**Phase 4: Cutover to Multi-Agent (Week 2)**

1. **If multi-agent wins (better accuracy at lower cost):**
   - Route 100% of traffic to voting coordinator
   - Monitor for 1 week
   - Deprecate GPT-4 API calls

---

## 4. Sizing Guidelines

### 4.1 Agent Type Resource Requirements

| Agent Type | CPU (per replica) | Memory (per replica) | Rationale |
|------------|-------------------|---------------------|-----------|
| **File Converter** | 1-2 vCPU | 2-4GB | Document processing (Apache Tika), I/O-bound |
| **Text Chunker** | 1 vCPU | 2GB | Lightweight text operations, CPU-bound |
| **Embeddings** | 2-4 vCPU | 8GB | Model inference (Ollama), memory-intensive |
| **Storage** | 2 vCPU | 4GB | Vector DB writes (Milvus), network-bound |
| **Query** | 2 vCPU | 4GB | Vector search (Milvus), latency-sensitive |
| **Response** | 2-4 vCPU | 8GB | LLM inference (Ollama), memory-intensive |
| **Error Handler** | 1 vCPU | 2GB | Lightweight, retry logic |
| **Memory** | 1-2 vCPU | 4GB | Database operations (Hindsight), I/O-bound |

**Replica Recommendations:**
- **Development:** 1 replica per agent type
- **Production (HA):** 2-3 replicas per agent type
- **High-Load:** 3-10 replicas (auto-scaling based on metrics)

---

### 4.2 Total Fleet Sizing

**Small Deployment (100-500 concurrent workflows):**
- Control Tower: 2 vCPU, 4GB RAM (1 replica)
- Node-RED Fleet: 20-30 vCPU, 50-70GB RAM (8 instance types × 1-2 replicas)
- Services: 4 vCPU, 16GB RAM
- **Total: 26-36 vCPU, 70-90GB RAM**
- **Cost: ~$200-400/month** (AWS/GCP/Azure)

**Medium Deployment (500-2000 concurrent workflows):**
- Control Tower: 6 vCPU, 12GB RAM (3 replicas)
- Node-RED Fleet: 40-60 vCPU, 100-150GB RAM (8 instance types × 2-3 replicas)
- Services: 8 vCPU, 32GB RAM
- **Total: 54-74 vCPU, 144-194GB RAM**
- **Cost: ~$800-1500/month**

**Large Deployment (2000-10,000 concurrent workflows):**
- Control Tower: 12 vCPU, 24GB RAM (6 replicas)
- Node-RED Fleet: 100-200 vCPU, 300-600GB RAM (8 × 5-10 replicas with auto-scaling)
- Services: 16 vCPU, 64GB RAM (HA PostgreSQL, Redis Cluster, MQTT cluster)
- **Total: 128-236 vCPU, 388-688GB RAM**
- **Cost: ~$3000-8000/month**

---

### 4.3 Service Sizing

**PostgreSQL (Control Tower Metadata + Audit Logs):**
- Development: 2 vCPU, 4GB RAM, 50GB SSD
- Production: 4 vCPU, 16GB RAM, 200GB SSD (primary + 1 read replica)
- High-Load: 8 vCPU, 32GB RAM, 500GB SSD (with connection pooling via PgBouncer)

**Redis (Multi-Agent Coordination State):**
- Development: 1 vCPU, 2GB RAM
- Production: 2 vCPU, 8GB RAM (Redis Sentinel 3 nodes)
- High-Load: 4 vCPU, 16GB RAM (Redis Cluster with sharding)

**MQTT (Message Buffering):**
- Development: 1 vCPU, 1GB RAM (single Mosquitto instance)
- Production: 3 × (1 vCPU, 2GB RAM) (3-node Mosquitto cluster)
- High-Load: 6 × (2 vCPU, 4GB RAM) (clustered MQTT with load balancer)

**Ollama (LLM Inference):**
- Per Model: 4-8 vCPU, 16-32GB RAM (CPU-only)
- With GPU: 8 vCPU, 16GB RAM + NVIDIA T4/A10 GPU (10x faster inference)
- Multi-Model Setup: Deploy separate Ollama instances per model (better isolation)

**Milvus (Vector Database):**
- Development: 4 vCPU, 8GB RAM, 100GB SSD (standalone mode)
- Production: 8 vCPU, 32GB RAM, 500GB SSD (cluster mode: 1 coordinator, 3 data nodes)
- High-Load: 16+ vCPU, 64GB RAM, 1TB SSD (scaled cluster with read replicas)

---

## 5. Best Practices

### 5.1 Deployment

1. **Start Small, Scale Gradually:**
   - Begin with Docker Compose (1 replica per agent)
   - Measure baseline performance
   - Add replicas as load increases
   - Migrate to Kubernetes when >20 instances

2. **Use Infrastructure as Code:**
   - Docker Compose: Version-control `docker-compose.yml`
   - Kubernetes: Version-control all manifests in Git
   - Terraform/Pulumi: Provision infrastructure reproducibly

3. **Separate Environments:**
   - **Development:** 1 replica per agent, shared services
   - **Staging:** 2 replicas per agent, production-like setup
   - **Production:** 3+ replicas, HA services

4. **Automate Deployments:**
   - CI/CD pipeline: Test → Build → Deploy
   - Automated smoke tests post-deployment
   - Rollback automation on failure

---

### 5.2 Monitoring & Observability

1. **Metrics to Track:**
   - Instance health (up/down)
   - Resource utilization (CPU, memory, disk)
   - Workflow latency (p50, p95, p99)
   - Error rate (per agent type)
   - Cost (LLM tokens used, compute costs)

2. **Alerting Rules:**
   - Instance down for >5 minutes → Page on-call
   - CPU >90% for >10 minutes → Auto-scale
   - Error rate >5% → Alert team
   - Deployment failed → Rollback automatically

3. **Dashboards:**
   - Fleet Overview (instance count, health)
   - Performance (latency, throughput)
   - Cost Analytics (LLM token usage by model)
   - Deployment History (success rate, duration)

---

### 5.3 Security

1. **Network Segmentation:**
   - Control Tower in management network
   - Node-RED instances in workload network
   - Services in private network
   - Only expose Control Tower UI publicly (HTTPS)

2. **Secrets Management:**
   - Use Kubernetes Secrets or HashiCorp Vault
   - Never hardcode credentials in flows
   - Rotate secrets quarterly

3. **TLS Everywhere:**
   - HTTPS for Control Tower API
   - TLS for PostgreSQL connections
   - MQTTS (MQTT over TLS)

4. **RBAC:**
   - Admin: Full access
   - Operator: Deploy, scale, monitor (no user management)
   - Viewer: Read-only

---

### 5.4 Cost Optimization

1. **Right-Size Instances:**
   - Profile each agent type (CPU, memory usage)
   - Use smallest instance type that meets SLA
   - Example: Text Chunker doesn't need 8GB RAM (use 2GB)

2. **Auto-Scaling:**
   - Scale down during low traffic (nights, weekends)
   - Use KEDA for event-driven scaling (queue depth)
   - Set aggressive scale-down policies

3. **LLM Cost Optimization:**
   - Use local models (Ollama) when possible (70% cost savings vs GPT-4)
   - Implement caching (avoid redundant API calls)
   - Cost-based routing (use cheapest model that meets SLA)

4. **Storage Optimization:**
   - Archive stale vectors (>90 days old)
   - Compress audit logs (gzip)
   - Use cheaper storage tiers (S3 Glacier for old backups)

---

## Related Documentation

- **[ARCHITECTURE.md](ARCHITECTURE.md)** - Comprehensive technical architecture
- **[project_plan.md](project_plan.md)** - Implementation roadmap
- **[CLUSTERING_AND_REDEPLOYMENT.md](CLUSTERING_AND_REDEPLOYMENT.md)** - Zero-downtime deployment details
- **[AI_AGENT_PRIMITIVES.md](AI_AGENT_PRIMITIVES.md)** - 8 primitive flows (to be created)
- **[MULTI_AGENT_PATTERNS.md](MULTI_AGENT_PATTERNS.md)** - Coordination patterns (to be created)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2026-02-15 | Complete rewrite: Removed FlowFuse migration, added deployment strategies for TypeScript Control Tower managing Node-RED instances |
| 1.1 | 2026-02-15 | FlowFuse migration assessment (deprecated) |

---

**Note**: This document describes deployment strategies for the RedForge AI Control Tower, a TypeScript-based management platform for Node-RED Agentic AI deployments.
