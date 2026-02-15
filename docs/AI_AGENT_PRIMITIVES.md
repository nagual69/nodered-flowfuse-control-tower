# AI Agent Primitives (Node-RED Flows)

**Status**: Documentation
**Version**: 1.0
**Date**: 2026-02-15

---

## Table of Contents

1. [Overview](#1-overview)
2. [The 8 Core Primitives](#2-the-8-core-primitives)
3. [Composability Examples](#3-composability-examples)
4. [Development in Node-RED](#4-development-in-node-red)
5. [Message Contracts](#5-message-contracts)
6. [Best Practices](#6-best-practices)

---

## 1. Overview

RedForge AI Control Tower manages **8 composable agent primitives**. Each primitive is a **Node-RED flow JSON file** that performs one specific step in an AI workflow. These primitives are the building blocks of multi-agent RAG systems and can be composed into complex workflows.

### What is an Agent Primitive?

An **agent primitive** is:
- A self-contained Node-RED flow exported as JSON
- Performs one well-defined AI task (e.g., embedding generation, vector search)
- Accepts standardized input messages (JSON schema)
- Produces standardized output messages (JSON schema)
- Stateless or uses Redis for shared state (multi-agent coordination)
- Deployable as an independent Node-RED instance (via Control Tower)

### Why Node-RED?

Node-RED is ideal for agent primitives because:
- **Visual Programming:** Drag-and-drop development, no code scaffolding
- **Message-Based:** Asynchronous, event-driven workflows
- **Composable:** Link primitives together via MQTT, HTTP, or internal nodes
- **Production-Ready:** Mature platform (10+ years), large ecosystem (4,000+ nodes)
- **Standards-Aligned:** JSON-based message contracts from main [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI) project

### Primitive Categories

The 8 primitives fall into 4 categories:

| Category | Primitives | Purpose |
|----------|-----------|---------|
| **Ingestion** | File Converter, Text Chunker | Process documents into vectorizable text |
| **Vectorization** | Embeddings Generator, Vector Storage | Convert text to vectors, store in database |
| **Retrieval** | Query Processor, Response Generator | Search vectors, generate AI responses |
| **Support** | Error Handler, Hindsight Memory | Handle failures, persist conversation history |

---

## 2. The 8 Core Primitives

### 2.1 File Converter Agent

**Flow File:** `examples/flows/primitives/file-converter/convert-file-to-text-tika.json`

**Purpose:** Extract text from any document format (PDF, DOCX, XLSX, images, HTML).

**Technology:** Apache Tika (universal document parser)

**Inputs:**
- **msg.payload:** Binary file buffer or file path
- **msg.metadata.contentType:** MIME type (e.g., `application/pdf`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`)

**Outputs:**
- **msg.payload:** Extracted plain text (string)
- **msg.metadata.extractedText:** Word count, character count
- **msg.metadata.status:** `"success"` or `"error"`

**Node-RED Implementation:**
```
[MQTT In: redforge/ingest/file]
    ↓
[Function: Validate Input] → Check file exists, MIME type supported
    ↓
[Tika Node: Extract Text] → Calls Apache Tika REST API
    ↓
[Function: Format Output] → Standardize msg structure
    ↓
[MQTT Out: redforge/chunking/input]
```

**Sample Flow Snippet:**
```javascript
// Function: Validate Input
if (!msg.payload) {
  msg.error = { code: 'MISSING_FILE', message: 'No file provided' };
  msg.metadata.status = 'error';
  return [null, msg]; // Route to error output
}

const supportedTypes = ['application/pdf', 'text/plain', 'application/msword'];
if (!supportedTypes.includes(msg.metadata.contentType)) {
  msg.error = { code: 'UNSUPPORTED_TYPE', message: `Type ${msg.metadata.contentType} not supported` };
  msg.metadata.status = 'error';
  return [null, msg];
}

return [msg, null]; // Route to success output

// Tika Node configuration:
// URL: http://tika:9998/tika
// Method: PUT
// Body: msg.payload (binary)
// Headers: { 'Accept': 'text/plain' }

// Function: Format Output
msg.payload = msg.payload.toString('utf-8'); // Tika returns text
msg.metadata.extractedText = {
  wordCount: msg.payload.split(/\s+/).length,
  charCount: msg.payload.length
};
msg.metadata.status = 'success';
return msg;
```

**Use Cases:**
- PDF documents (legal contracts, research papers)
- Office documents (DOCX, XLSX, PPTX)
- HTML pages (web scraping)
- Images with OCR (using Tesseract integration)

**Resource Requirements:**
- CPU: 1-2 vCPU (I/O-bound, not CPU-intensive)
- Memory: 2-4GB (Tika can use up to 2GB for large files)
- Network: Low (local Tika server)

---

### 2.2 Text Chunker Agent

**Flow File:** `examples/flows/primitives/text-chunking/text-chunking.json`

**Purpose:** Break large documents into semantic chunks suitable for embedding generation.

**Chunking Strategies:**
1. **Fixed-Size:** Split every N characters (e.g., 500 chars with 50 char overlap)
2. **Sentence-Boundary:** Split at sentence breaks (preserves semantic meaning)
3. **Semantic:** Use NLP to identify topic boundaries (most accurate, slower)

**Inputs:**
- **msg.payload:** Plain text (string)
- **msg.config.chunkSize:** Max characters per chunk (default: 500)
- **msg.config.chunkOverlap:** Overlap between chunks (default: 50)
- **msg.config.strategy:** `"fixed"` | `"sentence"` | `"semantic"`

**Outputs:**
- **msg.payload:** Array of chunk objects:
  ```javascript
  [
    {
      id: "chunk-1",
      text: "First chunk text...",
      startOffset: 0,
      endOffset: 500
    },
    {
      id: "chunk-2",
      text: "Second chunk text...",
      startOffset: 450,
      endOffset: 950
    }
  ]
  ```
- **msg.metadata.totalChunks:** Number of chunks created

**Node-RED Implementation:**
```
[MQTT In: redforge/chunking/input]
    ↓
[Function: Load Strategy] → Select chunking algorithm
    ↓
[Switch: strategy]
    ├─ [fixed] → [Function: Fixed Chunking]
    ├─ [sentence] → [Function: Sentence Chunking]
    └─ [semantic] → [NLP Node: Semantic Chunking]
    ↓
[Function: Format Chunks] → Standardize chunk objects
    ↓
[MQTT Out: redforge/embeddings/input]
```

**Sample Flow Snippet (Fixed Chunking):**
```javascript
// Function: Fixed Chunking
const text = msg.payload;
const chunkSize = msg.config.chunkSize || 500;
const overlap = msg.config.chunkOverlap || 50;

const chunks = [];
for (let i = 0; i < text.length; i += chunkSize - overlap) {
  const chunk = {
    id: `chunk-${chunks.length + 1}`,
    text: text.substring(i, i + chunkSize),
    startOffset: i,
    endOffset: Math.min(i + chunkSize, text.length)
  };
  chunks.push(chunk);
}

msg.payload = chunks;
msg.metadata.totalChunks = chunks.length;
msg.metadata.strategy =  'fixed';
return msg;
```

**Use Cases:**
- Large documents (> 10 pages)
- Context window limitations (LLMs have max token limits)
- Parallel embedding generation (chunks processed independently)

**Resource Requirements:**
- CPU: 1 vCPU (lightweight text operations)
- Memory: 2GB (holds document in memory during chunking)

---

### 2.3 Embeddings Generator Agent

**Flow File:** `examples/flows/primitives/embeddings/generate-embeddings-ollama.json`

**Purpose:** Generate vector embeddings for text chunks using an embedding model.

**Technology:** Ollama (local embedding models like nomic-embed-text)

**Inputs:**
- **msg.payload:** Array of chunk objects (from Text Chunker)
- **msg.config.model:** Embedding model name (e.g., `"nomic-embed-text:latest"`, `"all-minilm"`)
- **msg.config.ollamaHost:** Ollama server URL (e.g., `"http://ollama:11434"`)

**Outputs:**
- **msg.payload:** Array of embedding objects:
  ```javascript
  [
    {
      chunkId: "chunk-1",
      text: "First chunk text...",
      embedding: [0.023, -0.145, 0.789, ...], // 768-dim vector
      model: "nomic-embed-text",
      dimensions: 768
    }
  ]
  ```
- **msg.metadata.totalEmbeddings:** Number of embeddings generated
- **msg.metadata.tokensUsed:** Total tokens processed

**Node-RED Implementation:**
```
[MQTT In: redforge/embeddings/input]
    ↓
[Split: chunks] → Process chunks in parallel
    ↓
[Function: Prepare Ollama Request]
    ↓
[HTTP Request: Ollama /api/embeddings] → POST { model: "nomic-embed-text", prompt: chunk.text }
    ↓
[Function: Extract Embedding] → Parse response, extract vector
    ↓
[Join: chunks] → Aggregate all embeddings
    ↓
[MQTT Out: redforge/storage/input]
```

**Sample Flow Snippet:**
```javascript
// Function: Prepare Ollama Request
const ollamaUrl = msg.config.ollamaHost || 'http://ollama:11434';
const model = msg.config.model || 'nomic-embed-text:latest';

msg.url = `${ollamaUrl}/api/embeddings`;
msg.method = 'POST';
msg.headers = { 'Content-Type': 'application/json' };
msg.payload = {
  model: model,
  prompt: msg.payload.text
};
return msg;

// Function: Extract Embedding (after HTTP Request)
const response = JSON.parse(msg.payload);
msg.payload = {
  chunkId: msg.metadata.currentChunk.id,
  text: msg.metadata.currentChunk.text,
  embedding: response.embedding,
  model: msg.config.model,
  dimensions: response.embedding.length
};
return msg;
```

**Supported Models:**
- **nomic-embed-text (Recommended):** 768-dim, fast, high quality
- **all-minilm:** 384-dim, very fast, good quality
- **instructor-xl:** 768-dim, instruction-based, best quality (slower)

**Use Cases:**
- RAG ingestion pipelines (convert documents to searchable vectors)
- Semantic search (convert queries to vectors for similarity search)
- Document clustering (group similar documents)

**Resource Requirements:**
- CPU: 2-4 vCPU (model inference is CPU-intensive without GPU)
- Memory: 8GB (model loading + embeddings in memory)
- GPU: Optional (10x speedup with NVIDIA T4 or A10)

**Performance:**
- Throughput: ~50-100 chunks/second (CPU), ~500-1000 chunks/second (GPU)
- Latency: ~20-50ms per chunk (CPU), ~2-5ms per chunk (GPU)

---

### 2.4 Vector Storage Agent

 **Flow File:** `examples/flows/primitives/storage/milvus-storage-advanced.json`

**Purpose:** Store vector embeddings in a vector database for semantic search.

**Technology:** Milvus (high-performance vector database) or pgvector (PostgreSQL extension)

**Inputs:**
- **msg.payload:** Array of embedding objects (from Embeddings Generator)
- **msg.config.collectionName:** Milvus collection name (e.g., `"rag_documents"`)
- **msg.config.milvusHost:** Milvus server URL (e.g., `"milvus-standalone:19530"`)

**Outputs:**
- **msg.payload:** Insertion result:
  ```javascript
  {
    insertedIds: ["id-1", "id-2", "id-3"],
    insertedCount: 3,
    status: "success"
  }
  ```
- **msg.metadata.insertionTime:** Time taken to insert (milliseconds)

**Node-RED Implementation:**
```
[MQTT In: redforge/storage/input]
    ↓
[Function: Prepare Milvus Insert]
    ↓
[Milvus Node: Insert Vectors] → Batch insert with metadata
    ↓
[Function: Log Insertion] → Update metrics, audit log
    ↓
[MQTT Out: redforge/storage/complete]
```

**Sample Flow Snippet:**
```javascript
// Function: Prepare Milvus Insert
const milvusHost = msg.config.milvusHost || 'milvus-standalone:19530';
const collectionName = msg.config.collectionName || 'rag_documents';

// Milvus expects specific format
const entities = msg.payload.map(emb => ({
  vector: emb.embedding,
  text: emb.text,
  chunk_id: emb.chunkId,
  metadata: {
    model: emb.model,
    dimensions: emb.dimensions,
    timestamp: new Date().toISOString()
  }
}));

msg.mill = {
  operation: 'insert',
  collection: collectionName,
  data: entities
};

return msg;

// (Milvus Node handles actual insertion)

// Function: Log Insertion (after Milvus Node)
const insertResult = msg.payload;
msg.payload = {
  insertedIds: insertResult.IDs,
  insertedCount: insertResult.IDs.length,
  status: 'success'
};
msg.metadata.insertionTime = Date.now() - msg.metadata.startTime;

// Log to PostgreSQL for audit
const auditLog = {
  timestamp: new Date(),
  operation: 'VECTOR_INSERT',
  collection: msg.config.collectionName,
  count: msg.payload.insertedCount,
  duration_ms: msg.metadata.insertionTime
};
flow.set('lastInsert', auditLog, 'shared'); // Redis

return msg;
```

**Milvus vs pgvector:**
| Feature | Milvus | pgvector |
|---------|--------|----------|
| Performance | Very fast (100k+ QPS) | Good (10k+ QPS) |
| Scalability | Distributed, sharded | Single-node limit |
| Complexity | Higher (separate service) | Lower (PostgreSQL extension) |
| Best For | Large-scale (millions of vectors) | Small-scale (thousands) |

**Use Cases:**
- RAG document storage (millions of document vectors)
- Semantic search (fast similarity lookup)
- Recommendation systems (find similar items)

**Resource Requirements:**
- CPU: 2 vCPU (network-bound, not CPU-intensive)
- Memory: 4GB (batch buffering)
- Disk: Depends on collection size (1M vectors ≈ 3-6GB)

---

### 2.5 Query Processing Agent

**Flow File:** `examples/flows/primitives/query-processing/query-processing.json`

**Purpose:** Convert user queries to vector embeddings and search vector database for relevant chunks.

**Inputs:**
- **msg.payload:** User query (string, e.g., `"What is photosynthesis?"`)
- **msg.config.topK:** Number of results to return (default: 5)
- **msg.config.collectionName:** Milvus collection to search

**Outputs:**
- **msg.payload:** Array of search results:
  ```javascript
  [
    {
      id: "chunk-42",
      text: "Photosynthesis is the process by which...",
      score: 0.92, // Similarity score (0-1, higher is better)
      metadata: { source: "biology-textbook.pdf", page: 15 }
    }
  ]
  ```
- **msg.metadata.searchLatency:** Search time (milliseconds)

**Node-RED Implementation:**
```
[MQTT In: redforge/query/input]
    ↓
[Function: Embed Query] → Call Ollama to convert query to vector
    ↓
[Milvus Node: Search] → Similarity search (top K results)
    ↓
[Function: Rerank Results] → Optional: Use reranker model to improve order
    ↓
[MQTT Out: redforge/response/input]
```

**Sample Flow Snippet:**
```javascript
// Function: Embed Query
const ollamaUrl = msg.config.ollamaHost || 'http://ollama:11434';
const model = msg.config.model || 'nomic-embed-text:latest';

// Call Ollama to embed query
msg.url = `${ollamaUrl}/api/embeddings`;
msg.method = 'POST';
msg.payload = {
  model: model,
  prompt: msg.payload // User query
};
return msg;

// (After HTTP Request, extract query vector)
const queryEmbedding = JSON.parse(msg.payload).embedding;

// Milvus Search
msg.milvus = {
  operation: 'search',
  collection: msg.config.collectionName,
  vectors: [queryEmbedding],
  topK: msg.config.topK || 5,
  metricType: 'L2' // Euclidean distance (or 'IP' for inner product)
};

return msg;

// Function: Rerank Results (optional, after Milvus search)
const rawResults = msg.payload;

// Optional: Use reranker model (e.g., cross-encoder) for better results
// For now, just return raw Milvus results
msg.payload = rawResults.map(result => ({
  id: result.id,
  text: result.text,
  score: 1 - result.distance, // Convert distance to similarity score
  metadata: result.metadata
}));

msg.metadata.searchLatency = Date.now() - msg.metadata.startTime;
return msg;
```

**Advanced Features:**
- **Hybrid Search:** Combine vector similarity with keyword search (BM25)
- **Reranking:** Use cross-encoder model to reorder top results
- **Query Expansion:** Generate multiple query variations for better recall

**Use Cases:**
- RAG question answering (find relevant document chunks)
- Semantic search (find similar documents)
- Customer support (search knowledge base)

**Resource Requirements:**
- CPU: 2 vCPU (embeddings + search)
- Memory: 4GB (query embeddings, search results)
- Latency: ~50-200ms (depends on collection size and topK)

---

### 2.6 Response Generation Agent

**Flow File:** `examples/flows/primitives/response-generation/response-generation.json`

**Purpose:** Generate AI responses using LLM + retrieved context from vector search.

**Technology:** Ollama (llama3.2, mistral, phi3) or OpenAI API (GPT-4)

**Inputs:**
- **msg.payload:** Query + search results:
  ```javascript
  {
    query: "What is photosynthesis?",
    context: [
      { text: "Photosynthesis is...", score: 0.92 },
      { text: "Plants use chlorophyll...", score: 0.88 }
    ]
  }
  ```
- **msg.config.model:** LLM model name (e.g., `"llama3.2"`, `"gpt-4"`)
- **msg.config.temperature:** Sampling temperature (0-1, default: 0.7)

**Outputs:**
- **msg.payload:** Generated response:
  ```javascript
  {
    response: "Photosynthesis is the process by which plants convert sunlight into chemical energy...",
    model: "llama3.2",
    tokensUsed: 245,
    confidence: 0.89,
    sources: ["chunk-42", "chunk-58"]
  }
  ```
- **msg.metadata.generationLatency:** Response generation time (milliseconds)

**Node-RED Implementation:**
```
[MQTT In: redforge/response/input]
    ↓
[Function: Build Prompt] → Combine query + context into LLM prompt
    ↓
[HTTP Request: Ollama /api/generate] → POST { model: "llama3.2", prompt: "..." }
    ↓
[Function: Extract Response] → Parse LLM output, extract answer
    ↓
[MQTT Out: redforge/response/output]
```

**Sample Flow Snippet:**
```javascript
// Function: Build Prompt
const query = msg.payload.query;
const context = msg.payload.context.map(c => c.text).join('\n\n');

const systemPrompt = `You are a helpful AI assistant. Answer the user's question based on the provided context. If the context doesn't contain the answer, say so.

Context:
${context}

User Question: ${query}

Answer (be concise and accurate):`;

msg.payload = {
  model: msg.config.model || 'llama3.2',
  prompt: systemPrompt,
  stream: false,
  options: {
    temperature: msg.config.temperature || 0.7,
    top_p: 0.9,
    max_tokens: 500
  }
};

msg.url = `${msg.config.ollamaHost || 'http://ollama:11434'}/api/generate`;
msg.method = 'POST';

return msg;

// Function: Extract Response (after HTTP Request)
const llmResponse = JSON.parse(msg.payload);

msg.payload = {
  response: llmResponse.response,
  model: msg.config.model,
  tokensUsed: llmResponse.total_duration ? Math.floor(llmResponse.total_duration / 1000000) : null,
  confidence: null, // Ollama doesn't provide confidence, could add heuristic
  sources: msg.metadata.contextSources // From Query Processing
};

msg.metadata.generationLatency = Date.now() - msg.metadata.startTime;

// Log AI decision to Control Tower audit API
const auditLog = {
  timestamp: new Date(),
  flow_name: 'response-generation',
  model: msg.payload.model,
  prompt: systemPrompt,
  response: msg.payload.response,
  tokens_used: msg.payload.tokensUsed,
  latency_ms: msg.metadata.generationLatency
};

// Send to Control Tower
msg.auditPayload = auditLog;
return msg;
```

**Multi-Agent Voting (Advanced):**
```javascript
// Instead of one model, query 3 models and vote
const models = ['llama3.2', 'mistral', 'phi3'];
const responses = [];

for (const model of models) {
  const response = await callOllama(model, systemPrompt);
  responses.push({
    model: model,
    response: response.response,
    confidence: calculateConfidence(response) // Heuristic
  });
}

// Vote: Select response with highest confidence
const winner = responses.reduce((best, current) =>
  current.confidence > best.confidence ? current : best
);

msg.payload = {
  response: winner.response,
  model: winner.model,
  allResponses: responses, // For transparency
  votingStrategy: 'confidence-based'
};
```

**Use Cases:**
- RAG question answering (primary use case)
- Chatbots with knowledge base access
- Document summarization with context

**Resource Requirements:**
- CPU: 2-4 vCPU (LLM inference, CPU-only)
- Memory: 8GB (model loading)
- GPU: Strongly recommended (10x speedup, NVIDIA T4 or better)
- Latency: ~2-10 seconds (CPU), ~200-500ms (GPU)

---

### 2.7 Error Handler Agent

**Flow File:** `examples/flows/primitives/error-handling/error-handler.json`

**Purpose:** Centralized error handling, retry logic, dead-letter queue, circuit breaker.

**Inputs:**
- **msg.error:** Error object from any primitive
- **msg.metadata.retryCount:** Number of retries attempted
- **msg.metadata.source:** Which primitive encountered the error

**Outputs:**
- **msg.payload:** Retry decision:
  ```javascript
  {
    action: "retry" | "dlq" | "discard",
    retryDelay: 5000, // ms to wait before retry
    maxRetries: 3
  }
  ```

**Node-RED Implementation:**
```
[MQTT In: redforge/errors/input]
    ↓
[Function: Classify Error] → Transient (network) vs Permanent (bad data)
    ↓
[Switch: Error Type]
    ├─ [Transient] → [Function: Retry Logic] → Exponential backoff
    └─ [Permanent] → [Function: Dead-Letter Queue] → Store for manual review
    ↓
[MQTT Out: redforge/retry/queue OR redforge/dlq]
```

**Sample Flow Snippet:**
```javascript
// Function: Classify Error
const error = msg.error;
const transientErrors = ['ECONNREFUSED', 'ETIMEDOUT', 'RATE_LIMIT_EXCEEDED'];
const permanentErrors = ['INVALID_INPUT', 'UNSUPPORTED_TYPE', 'UNAUTHORIZED'];

let errorType = 'unknown';
if (transientErrors.includes(error.code)) {
  errorType = 'transient';
} else if (permanentErrors.includes(error.code)) {
  errorType = 'permanent';
}

msg.errorType = errorType;
return msg;

// Function: Retry Logic (for transient errors)
const retryCount = msg.metadata.retryCount || 0;
const maxRetries = 3;

if (retryCount < maxRetries) {
  const retryDelay = Math.min(1000 * Math.pow(2, retryCount), 30000); // Exponential backoff, max 30s
  msg.payload = {
    action: 'retry',
    retryDelay: retryDelay,
    retryCount: retryCount + 1
  };
  return [msg, null]; // Output 1 = retry queue
} else {
  msg.payload = {
    action: 'dlq',
    reason: 'MAX_RETRIES_EXCEEDED'
  };
  return [null, msg]; // Output 2 = dead-letter queue
}

// Function: Dead-Letter Queue
const dlqEntry = {
  timestamp: new Date(),
  source: msg.metadata.source,
  error: msg.error,
  originalMessage: msg.originalPayload,
  retryCount: msg.metadata.retryCount
};

// Store in PostgreSQL for manual review
flow.set('dlq_' + Date.now(), dlqEntry, 'shared'); // Redis

// Alert operations team
msg.alert = {
  severity: 'error',
  message: `Message from ${msg.metadata.source} sent to DLQ: ${msg.error.message}`
};

return msg;
```

**Circuit Breaker Pattern:**
```javascript
// Track failure rate per service
const serviceName = msg.metadata.source;
const circuitState = flow.get(`circuit:${serviceName}`, 'shared') || { failures: 0, state: 'closed' };

if (circuitState.state === 'open') {
  // Circuit open: Reject requests immediately
  msg.error = { code: 'CIRCUIT_OPEN', message: `Service ${serviceName} is unavailable (circuit breaker open)` };
  return [null, msg]; // Route to error output
}

// Process request...

if (error) {
  circuitState.failures++;
  if (circuit State.failures >= 5) {
    circuitState.state = 'open';
    circuitState.openedAt = Date.now();
    // Circuit will auto-close after 60 seconds
    setTimeout(() => {
      circuitState.state = 'closed';
      circuitState.failures = 0;
      flow.set(`circuit:${serviceName}`, circuitState, 'shared');
    }, 60000);
  }
  flow.set(`circuit:${serviceName}`, circuitState, 'shared');
}
```

**Use Cases:**
- Network failures (Ollama unavailable, Milvus timeout)
- Rate limiting (OpenAI API quota exceeded)
- Bad data (malformed JSON, unsupported file type)
- System overload (circuit breaker prevents cascading failures)

**Resource Requirements:**
- CPU: 1 vCPU (lightweight)
- Memory: 2GB (retry queue, DLQ in memory)

---

### 2.8 Hindsight Memory Agent

**Flow File:** `examples/flows/primitives/memory/hindsight-memory.json`

**Purpose:** Persist conversation history, enable context-aware conversations.

**Technology:** PostgreSQL (relational database for structured memory)

**Inputs:**
- **msg.payload:** Conversation turn:
  ```javascript
  {
    sessionId: "user-123",
    role: "user" | "assistant",
    message: "What is photosynthesis?",
    timestamp: "2026-02-15T10:30:00Z"
  }
  ```

**Outputs:**
- **msg.payload:** Full conversation history for session:
  ```javascript
  {
    sessionId: "user-123",
    history: [
      { role: "user", message: "What is photosynthesis?", timestamp: "..." },
      { role: "assistant", message: "Photosynthesis is...", timestamp: "..." },
      { role: "user", message: "How does it work in plants?", timestamp: "..." }
    ],
    totalTurns: 3
  }
  ```

**Node-RED Implementation:**
```
[MQTT In: redforge/memory/store]
    ↓
[Function: Format SQL Insert]
    ↓
[PostgreSQL Node: Insert Turn] → INSERT INTO hindsight (session_id, role, message, timestamp)
    ↓
[PostgreSQL Node: Fetch History] → SELECT * FROM hindsight WHERE session_id = ? ORDER BY timestamp
    ↓
[Function: Format History]
    ↓
[MQTT Out: redforge/memory/history]
```

**Sample Flow Snippet:**
```javascript
// Function: Format SQL Insert
const sessionId = msg.payload.sessionId;
const role = msg.payload.role;
const message = msg.payload.message;
const timestamp = msg.payload.timestamp || new Date().toISOString();

msg.topic = `
  INSERT INTO hindsight (session_id, role, message, timestamp)
  VALUES ($1, $2, $3, $4)
`;
msg.payload = [sessionId, role, message, timestamp];

return msg;

// (PostgreSQL Node handles insertion)

// PostgreSQL Node: Fetch History
msg.topic = `
  SELECT role, message, timestamp
  FROM hindsight
  WHERE session_id = $1
  ORDER BY timestamp ASC
  LIMIT 50
`;
msg.payload = [sessionId];

return msg;

// Function: Format History (after SELECT)
const turns = msg.payload; // Array of {role, message, timestamp}

msg.payload = {
  sessionId: msg.metadata.sessionId,
  history: turns,
  totalTurns: turns.length
};

return msg;
```

**Use Cases:**
- Multi-turn conversations (chatbots)
- Context-aware RAG (use conversation history to improve relevance)
- User feedback tracking (store thumbs up/down reactions)

**Resource Requirements:**
- CPU: 1 vCPU (database I/O)
- Memory: 2-4GB (query results buffering)
- Database: PostgreSQL with 10-100GB storage

---

## 3. Composability Examples

### 3.1 Simple RAG Pipeline

**Flow:** Ingest document → Generate embeddings → Store → Query → Respond

```
[File Upload]
    ↓
┌────────────────────┐
│ File Converter     │ → Plain text
└────────────────────┘
    ↓
┌────────────────────┐
│ Text Chunker       │ → 50 chunks
└────────────────────┘
    ↓
┌────────────────────┐
│ Embeddings Gen     │ → 50 vectors (768-dim)
└────────────────────┘
    ↓
┌────────────────────┐
│ Vector Storage     │ → Milvus insert (instant search)
└────────────────────┘

[User Query: "What is photosynthesis?"]
    ↓
┌────────────────────┐
│ Query Processing   │ → Top 5 relevant chunks
└────────────────────┘
    ↓
┌────────────────────┐
│ Response Gen       │ → "Photosynthesis is..."
└────────────────────┘
```

**MQTT Topics (Inter-Instance Communication):**
- `redforge/ingest/file` → File Converter
- `redforge/chunking/input` → Text Chunker
- `redforge/embeddings/input` → Embeddings Generator
- `redforge/storage/input` → Vector Storage
- `redforge/query/input` → Query Processing
- `redforge/response/input` → Response Generator

---

### 3.2 Multi-Agent RAG (Voting)

**Flow:** 3 response agents vote on best answer

```
[User Query]
    ↓
┌────────────────────┐
│ Query Coordinator  │ → Broadcast query to 3 agents
└────────────────────┘
    ↓ (MQTT broadcast)
    ├──→ ┌─────────────────────┐
    │    │ Response Agent A    │ (llama3.2)
    │    │  → "Answer A..."    │
    │    │  → confidence: 0.85 │
    │    └─────────────────────┘
    │
    ├──→ ┌─────────────────────┐
    │    │ Response Agent B    │ (mistral)
    │    │  → "Answer B..."    │
    │    │  → confidence: 0.92 │ ← WINNER
    │    └─────────────────────┘
    │
    └──→ ┌─────────────────────┐
         │ Response Agent C    │ (phi3)
         │  → "Answer C..."    │
         │  → confidence: 0.78 │
         └─────────────────────┘
    ↓ (All 3 responses collected)
┌────────────────────┐
│ Voting Coordinator │ → Select best response (highest confidence)
└────────────────────┘
    ↓
[Return "Answer B..." to user]
```

**Benefits:**
- **Accuracy:** Higher than single model (voting reduces hallucinations)
- **Cost:** Lower than GPT-4 (use 3 local models instead)
- **Transparency:** See all 3 responses, understand voting decision

---

## 4. Development in Node-RED

### 4.1 Import Primitive Flow

1. **Start Node-RED instance:**
   ```bash
   docker run -p 1880:1880 nodered/node-red:4.1.2
   ```

2. **Open UI:** http://localhost:1880

3. **Import flow:**
   - Menu → Import → Clipboard
   - Paste flow JSON (e.g., `convert-file-to-text-tika.json`)
   - Click "Import"

### 4.2 Configure Nodes

**Example: Configure Ollama node in Embeddings Generator**

1. **Double-click HTTP Request node** (Ollama /api/embeddings)
2. **Set URL:** `http://ollama:11434/api/embeddings`
3. **Set Method:** POST
4. **Set Headers:** `Content-Type: application/json`
5. **Deploy**

### 4.3 Test Primitive

1. **Inject test message:**
   - Add "Inject" node
   - Set payload:
     ```json
     {
       "text": "Test document text",
       "config": {
         "model": "nomic-embed-text",
         "ollamaHost": "http://localhost:11434"
       }
     }
     ```
   - Wire to primitive input
   - Click "Inject" button

2. **View output:**
   - Add "Debug" node to primitive output
   - Check Debug sidebar for results

### 4.4 Deploy to Control Tower

1. **Export updated flow:**
   - Select all nodes in primitive
   - Menu → Export → Clipboard
   - Save as `my-custom-primitive.json`

2. **Deploy via Control Tower API:**
   ```bash
   curl -X POST http://localhost:3000/api/instances \
     -H "Content-Type: application/json" \
     -d '{
       "name": "embeddings-custom",
       "flows": ["/path/to/my-custom-primitive.json"],
       "resources": { "cpu": "2", "memory": "8Gi" }
     }'
   ```

---

## 5. Message Contracts

All primitives use standardized message schemas defined in the main [RedForge-Agentic-AI](https://github.com/nagual69/RedForge-Agentic-AI) project:

- **Input Schema:** `schemas/msg_input.json`
- **Output Schema:** `schemas/msg_output.json`

**Standard Message Structure:**
```javascript
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

---

## 6. Best Practices

### 6.1 Design

1. **Keep Primitives Focused:** One primitive = one well-defined task
2. **Use Standard Schemas:** Follow msg_input.json and msg_output.json
3. **Minimize State:** Prefer stateless primitives; use Redis for shared state only when necessary
4. **Handle Errors:** Always add error output wires, route to Error Handler

### 6.2 Performance

1. **Batch When Possible:** Process chunks in parallel (use Split/Join nodes)
2. **Cache Expensive Operations:** Cache embedding models in memory (Ollama)
3. **Optimize Message Size:** Don't send large binary files via MQTT; use object storage + references

### 6.3 Monitoring

1. **Add Metrics Nodes:** Track latency, throughput, error rate
2. **Log to Audit API:** Send AI decisions to Control Tower audit endpoint
3. **Use Debug Nodes:** Liberally during development, remove in production

---

## Related Documentation

- **[ARCHITECTURE.md](ARCHITECTURE.md)** - Overall Control Tower architecture
- **[MULTI_AGENT_PATTERNS.md](MULTI_AGENT_PATTERNS.md)** - Coordination patterns (to be created)
- **[DEPLOYMENT_STRATEGIES.md](DEPLOYMENT_STRATEGIES.md)** - Deployment options
- **[Main Project](https://github.com/nagual69/RedForge-Agentic-AI)** - Full agent primitive flows + schemas

---

**Version History:**

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-15 | Initial comprehensive documentation of 8 agent primitives |
