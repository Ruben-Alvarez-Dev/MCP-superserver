# MCP-SUPERSERVER API Documentation

## Overview

MCP-SUPERSERVER provides a comprehensive REST API and Model Context Protocol (MCP) interface for AI memory and reasoning operations.

## Table of Contents

- [REST API Endpoints](#rest-api-endpoints)
- [MCP Tools Reference](#mcp-tools-reference)
  - [Neo4j Memory Server](#neo4j-memory-server)
  - [Obsidian Memory Server](#obsidian-memory-server)
  - [Ollama Server](#ollama-server)
  - [Sequential Thinking Server](#sequential-thinking-server)
  - [Task Master Server](#task-master-server)
- [Error Codes](#error-codes)
- [Authentication](#authentication)

---

## REST API Endpoints

### Base URL
```
http://localhost:3000
```

### Health Check

**GET** `/health`

Check the health status of the MCP Hub and all connected services.

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00Z",
  "services": {
    "neo4j": "healthy",
    "ollama": "healthy",
    "obsidian": "healthy"
  },
  "uptime": 3600
}
```

### Metrics

**GET** `/metrics`

Prometheus-compatible metrics endpoint.

**Response Format:** Text/plain with Prometheus metrics

### Tool Execution

**POST** `/tools/call`

Execute an MCP tool through the REST API.

**Request Body:**
```json
{
  "server": "neo4j-memory",
  "tool": "create_entity",
  "arguments": {
    "label": "Person",
    "id": "user-123",
    "properties": {
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
}
```

**Response:**
```json
{
  "success": true,
  "result": {
    "entity": {
      "label": "Person",
      "id": "user-123",
      "name": "John Doe"
    },
    "internalId": 123
  }
}
```

---

## MCP Tools Reference

### Neo4j Memory Server

**Server Name:** `neo4j-memory`

#### create_entity

Create a new entity (node) in the Neo4j graph.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| label | string | Yes | Entity label (e.g., Person, Concept, Event) |
| id | string | Yes | Unique identifier for the entity |
| properties | object | No | Additional properties for the entity |

**Example Request:**
```json
{
  "label": "Project",
  "id": "project-alpha",
  "properties": {
    "name": "Project Alpha",
    "status": "active",
    "startDate": "2024-01-01"
  }
}
```

**Example Response:**
```json
{
  "success": true,
  "entity": {
    "label": "Project",
    "id": "project-alpha",
    "name": "Project Alpha",
    "status": "active",
    "startDate": "2024-01-01"
  },
  "internalId": 456
}
```

#### get_entity

Get an entity by ID from the Neo4j graph.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| label | string | Yes | Entity label |
| id | string | Yes | Entity identifier |

**Example Response:**
```json
{
  "success": true,
  "entity": {
    "id": "project-alpha",
    "name": "Project Alpha",
    "status": "active"
  }
}
```

#### find_entities

Find entities by label and properties.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| label | string | Yes | Entity label |
| properties | object | No | Properties to match |
| limit | number | No | Maximum number of results (default: 100) |

**Example Request:**
```json
{
  "label": "Project",
  "properties": {
    "status": "active"
  },
  "limit": 50
}
```

#### update_entity

Update an existing entity in the Neo4j graph.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| label | string | Yes | Entity label |
| id | string | Yes | Entity identifier |
| properties | object | Yes | Properties to update |

#### delete_entity

Delete an entity from the Neo4j graph.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| label | string | Yes | Entity label |
| id | string | Yes | Entity identifier |

#### count_entities

Count entities by label.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| label | string | Yes | Entity label |

#### create_relationship

Create a relationship between two entities.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| fromLabel | string | Yes | Source entity label |
| fromId | string | Yes | Source entity identifier |
| relationshipType | string | Yes | Type of relationship (e.g., KNOWS, RELATED_TO) |
| toLabel | string | Yes | Target entity label |
| toId | string | Yes | Target entity identifier |
| properties | object | No | Additional properties for the relationship |

**Example Request:**
```json
{
  "fromLabel": "Person",
  "fromId": "user-123",
  "relationshipType": "WORKS_ON",
  "toLabel": "Project",
  "toId": "project-alpha",
  "properties": {
    "role": "developer",
    "since": "2024-01-01"
  }
}
```

#### get_relationships

Get relationships for an entity.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| label | string | Yes | Entity label |
| id | string | Yes | Entity identifier |
| direction | string | No | Relationship direction: incoming, outgoing, or both (default: both) |
| type | string | No | Filter by relationship type |

#### query_graph

Query the graph for connected entities and patterns.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| label | string | Yes | Starting entity label |
| id | string | Yes | Starting entity identifier |
| pattern | string | No | Pattern type: connected, path, or stats (default: connected) |
| maxDepth | number | No | Maximum depth for graph traversal (default: 2) |

#### find_shortest_path

Find the shortest path between two entities.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| fromLabel | string | Yes | Source entity label |
| fromId | string | Yes | Source entity identifier |
| toLabel | string | Yes | Target entity label |
| toId | string | Yes | Target entity identifier |
| maxDepth | number | No | Maximum path length (default: 5) |

---

### Obsidian Memory Server

**Server Name:** `obsidian-memory`

#### read_note

Read a markdown note from the Obsidian vault.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| filename | string | Yes | Name of the markdown file (e.g., 2024-01-15.md) |

**Example Response:**
```json
{
  "success": true,
  "filename": "2024-01-15.md",
  "frontmatter": {
    "date": "2024-01-15",
    "tags": ["development", "api"]
  },
  "body": "# Daily Log\n\nWorked on API documentation...",
  "content": "---\ndate: 2024-01-15\ntags:\n  - development\n  - api\n---\n\n# Daily Log\n\nWorked on API documentation..."
}
```

#### write_note

Write or overwrite a markdown note in the Obsidian vault.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| filename | string | Yes | Name of the markdown file |
| content | string | Yes | Markdown content |
| frontmatter | object | No | YAML frontmatter properties |

#### append_note

Append content to an existing markdown note.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| filename | string | Yes | Name of the markdown file |
| content | string | Yes | Content to append |

#### list_notes

List all markdown notes in the vault.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| limit | number | No | Maximum number of notes to return (default: 100) |
| sort | string | No | Sort order: newest or oldest (default: newest) |

#### search_notes

Search for notes by filename or content.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| query | string | Yes | Search query |
| searchContent | boolean | No | Whether to search within file contents (default: false) |

#### create_note

Create a new markdown note with timestamp and optional frontmatter.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| title | string | Yes | Note title |
| content | string | Yes | Markdown content |
| filename | string | No | Custom filename |
| frontmatter | object | No | YAML frontmatter properties |
| tags | array | No | Tags to add |

#### get_tags

Extract all tags from notes in the vault.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| limit | number | No | Maximum number of notes to scan (default: 50) |

---

### Ollama Server

**Server Name:** `ollama`

#### chat

Send a chat completion request to an Ollama model.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| messages | array | Yes | Array of chat messages |
| model | string | No | Model name (optional, uses default if not specified) |
| stream | boolean | No | Whether to stream the response (default: false) |
| taskType | string | No | Task type for model routing (default: chat) |

**Example Request:**
```json
{
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "Explain Neo4j graph databases." }
  ],
  "taskType": "chat"
}
```

**Example Response:**
```json
{
  "success": true,
  "model": "llama3.3",
  "message": {
    "role": "assistant",
    "content": "Neo4j is a graph database management system..."
  },
  "done": true,
  "duration_ms": 1234
}
```

#### complete

Generate a completion from a prompt.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| prompt | string | Yes | Text prompt for completion |
| model | string | No | Model name |
| taskType | string | No | Task type for model routing |

#### embed

Generate embeddings for text.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| text | string | Yes | Text to embed |
| model | string | No | Embedding model name |

**Example Response:**
```json
{
  "success": true,
  "model": "nomic-embed-text",
  "embedding": [0.123, -0.456, 0.789, ...],
  "dimension": 768,
  "duration_ms": 234
}
```

#### vision

Analyze an image with a vision model.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| image | string | Yes | Base64 encoded image data or image URL |
| prompt | string | Yes | Prompt for image analysis |
| model | string | No | Vision model name |

#### list_models

List all available Ollama models.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| refresh | boolean | No | Force refresh from Ollama API (default: false) |

**Example Response:**
```json
{
  "success": true,
  "count": 6,
  "models": [
    {
      "name": "llama3.3",
      "size": 4700000000,
      "modified": "2024-01-15T10:00:00Z",
      "digest": "abc123..."
    },
    {
      "name": "qwq",
      "size": 12000000000,
      "modified": "2024-01-14T15:30:00Z",
      "digest": "def456..."
    }
  ]
}
```

#### get_model_info

Get detailed information about a specific model.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| model | string | Yes | Model name |

#### pull_model

Download a model from Ollama library.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| model | string | Yes | Model name to pull (e.g., llama3.2, mistral, codellama) |

#### set_default_model

Set the default model for future requests.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| model | string | Yes | Model name to use as default |
| taskType | string | No | Task type to set model for (default: chat) |

#### reasoning

Perform reasoning task with specialized model.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| prompt | string | Yes | Reasoning prompt |
| model | string | No | Reasoning model name |

#### coding

Generate code with specialized model.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| prompt | string | Yes | Code generation prompt |
| model | string | No | Coding model name |
| language | string | No | Programming language hint |

---

### Sequential Thinking Server

**Server Name:** `sequential-thinking`

#### start_thinking

Start a new reasoning chain.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| prompt | string | Yes | The initial prompt or problem to reason about |
| context | object | No | Additional context for the reasoning |
| goal | string | No | The goal or expected outcome |
| tags | array | No | Tags for categorizing the chain |
| branchFrom | string | No | Chain ID to branch from |

**Example Response:**
```json
{
  "success": true,
  "chainId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "started",
  "message": "Reasoning chain started",
  "chain": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "prompt": "How should we architecture this feature?",
    "status": "in_progress",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

#### add_step

Add a reasoning step to an existing chain.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| chainId | string | Yes | The chain ID |
| thought | string | Yes | The thought or reasoning for this step |
| data | object | No | Additional data for this step |
| stepType | string | No | Type of reasoning step (default: analysis) |
| confidence | number | No | Confidence level (0-1) |

**Step Types:**
- `observation` - Direct observation
- `analysis` - Analytical reasoning
- `inference` - Logical inference
- `conclusion` - Concluding thought
- `question` - Question to explore
- `hypothesis` - Hypothetical reasoning

**Example Request:**
```json
{
  "chainId": "550e8400-e29b-41d4-a716-446655440000",
  "thought": "The feature requires a microservice architecture for scalability",
  "stepType": "analysis",
  "confidence": 0.8,
  "data": {
    "consideredOptions": ["monolith", "microservices", "serverless"]
  }
}
```

#### get_chain

Get details of a reasoning chain.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| chainId | string | Yes | The chain ID |
| includeSteps | boolean | No | Whether to include steps in response (default: true) |

#### conclude

Conclude a reasoning chain with a final result.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| chainId | string | Yes | The chain ID |
| conclusion | string | Yes | The final conclusion or result |
| success | boolean | No | Whether the reasoning was successful (default: true) |
| confidence | number | No | Overall confidence in the conclusion |

#### list_chains

List all reasoning chains.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| status | string | No | Filter by status: in_progress, completed, failed, all (default: all) |
| limit | number | No | Maximum chains to return (default: 50) |

#### branch_chain

Create a branch from an existing reasoning chain.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| chainId | string | Yes | The chain ID to branch from |
| atStep | number | No | Step number to branch from |

---

### Task Master Server

**Server Name:** `task-master`

#### create_task

Create a new task.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| title | string | Yes | Task title |
| description | string | No | Detailed task description |
| priority | string | No | Task priority (default: medium) |
| status | string | No | Initial task status (default: pending) |
| assignee | string | No | Task assignee |
| tags | array | No | Task tags |
| dueDate | string | No | Due date (ISO 8601 format) |
| parentTaskId | string | No | Parent task ID if this is a subtask |
| metadata | object | No | Additional metadata |

**Priority Levels:**
- `critical` - Highest priority, blocks other work
- `high` - High priority, should be done soon
- `medium` - Normal priority (default)
- `low` - Low priority, can be deferred

**Status Values:**
- `pending` - Not started
- `in_progress` - Currently being worked on
- `blocked` - Blocked by dependencies
- `deferred` - Deferred to later
- `completed` - Finished
- `cancelled` - Cancelled

**Example Request:**
```json
{
  "title": "Implement user authentication",
  "description": "Add JWT-based authentication to the API",
  "priority": "high",
  "status": "pending",
  "assignee": "john-doe",
  "tags": ["security", "api"],
  "dueDate": "2024-02-01T00:00:00Z"
}
```

**Example Response:**
```json
{
  "success": true,
  "taskId": "660e8400-e29b-41d4-a716-446655440000",
  "status": "created",
  "task": {
    "id": "660e8400-e29b-41d4-a716-446655440000",
    "title": "Implement user authentication",
    "description": "Add JWT-based authentication to the API",
    "priority": "high",
    "status": "pending",
    "assignee": "john-doe",
    "tags": ["security", "api"],
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

#### get_task

Get details of a task.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| taskId | string | Yes | Task ID |
| includeSubtasks | boolean | No | Include subtasks in response (default: true) |

#### update_task

Update an existing task.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| taskId | string | Yes | Task ID |
| title | string | No | New task title |
| description | string | No | New task description |
| status | string | No | New task status |
| priority | string | No | New task priority |
| assignee | string | No | New assignee |
| progress | number | No | Progress percentage (0-100) |

#### complete_task

Mark a task as completed.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| taskId | string | Yes | Task ID |
| result | string | No | Task result or completion notes |

#### delete_task

Delete a task.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| taskId | string | Yes | Task ID |
| deleteSubtasks | boolean | No | Also delete subtasks (default: false) |

#### list_tasks

List tasks with optional filtering.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| status | string | No | Filter by status |
| priority | string | No | Filter by priority |
| assignee | string | No | Filter by assignee |
| tags | array | No | Filter by tags (any match) |
| parentTaskId | string | No | Filter by parent task (get subtasks) |
| limit | number | No | Maximum results (default: 100) |

#### add_subtask

Add a subtask to an existing task.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| parentTaskId | string | Yes | Parent task ID |
| title | string | Yes | Subtask title |
| description | string | No | Subtask description |
| priority | string | No | Subtask priority (default: medium) |

#### set_dependency

Set a dependency between tasks.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| taskId | string | Yes | The task that has the dependency |
| dependsOnTaskId | string | Yes | The task this depends on |
| dependencyType | string | No | Type of dependency (default: must_complete_before) |

**Dependency Types:**
- `must_complete_before` - Task must complete before this one can start
- `should_complete_before` - Task should complete before this one (soft dependency)
- `blocks` - Task blocks this one from starting

#### get_dependencies

Get dependencies for a task.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| taskId | string | Yes | Task ID |
| direction | string | No | Dependency direction: incoming, outgoing, or both (default: both) |

---

## Error Codes

### HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 400 | Bad Request - Invalid parameters |
| 404 | Not Found - Resource not found |
| 500 | Internal Server Error |

### MCP Error Response Format

```json
{
  "success": false,
  "error": "Error message describing what went wrong",
  "code": "ERROR_CODE",
  "details": {
    "additional": "context"
  }
}
```

### Common Error Codes

| Code | Description |
|------|-------------|
| `ENTITY_NOT_FOUND` | Requested entity does not exist in Neo4j |
| `RELATIONSHIP_NOT_FOUND` | Requested relationship does not exist |
| `FILE_NOT_FOUND` | Requested file does not exist in Obsidian vault |
| `MODEL_NOT_AVAILABLE` | Requested Ollama model is not available |
| `CHAIN_NOT_FOUND` | Requested reasoning chain does not exist |
| `TASK_NOT_FOUND` | Requested task does not exist |
| `INVALID_INPUT` | Input validation failed |
| `NEO4J_CONNECTION_ERROR` | Cannot connect to Neo4j database |
| `OLLAMA_CONNECTION_ERROR` | Cannot connect to Ollama service |
| `PROTOCOL_OMEGA_VIOLATION` | Logging requirements not met |

---

## Authentication

### API Key Authentication (Optional)

If enabled, API requests must include an API key:

```bash
curl -H "X-API-Key: your-api-key" http://localhost:3000/tools/call
```

### Neo4j Authentication

Neo4j connections use username/password authentication configured via environment variables:

```bash
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_secure_password
```

### Obsidian File System Access

Obsidian server accesses files directly on the mounted volume. No authentication required for local file access.

---

## WebSocket Protocol

### Connection

```javascript
const ws = new WebSocket('ws://localhost:3000/ws');
```

### Message Format

**Request:**
```json
{
  "id": "unique-request-id",
  "type": "tool_call",
  "server": "neo4j-memory",
  "tool": "create_entity",
  "arguments": { ... }
}
```

**Response:**
```json
{
  "id": "unique-request-id",
  "type": "result",
  "success": true,
  "result": { ... }
}
```

### Streaming Responses

For tools that support streaming (e.g., `chat` with `stream: true`):

```json
{
  "id": "unique-request-id",
  "type": "stream_chunk",
  "chunk": "Partial response text...",
  "done": false
}
```

---

## Rate Limiting

### Default Limits

| Endpoint | Requests | Window |
|----------|----------|--------|
| `/health` | 1000 | 1 minute |
| `/tools/call` | 100 | 1 minute |
| `/metrics` | 100 | 1 minute |

### Rate Limit Response

When rate limited, the response includes:

```json
{
  "success": false,
  "error": "Rate limit exceeded",
  "retryAfter": 30
}
```

Headers:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1642287600
Retry-After: 30
```

---

## Examples

### Complete Workflow Example

```javascript
// 1. Start a reasoning chain
const chain = await callTool('sequential-thinking', 'start_thinking', {
  prompt: 'Design user authentication system'
});
// Returns: chainId = "550e8400..."

// 2. Add reasoning steps
await callTool('sequential-thinking', 'add_step', {
  chainId: chain.chainId,
  thought: 'Need JWT-based stateless authentication',
  stepType: 'analysis',
  confidence: 0.9
});

await callTool('sequential-thinking', 'add_step', {
  chainId: chain.chainId,
  thought: 'Use bcrypt for password hashing',
  stepType: 'analysis'
});

// 3. Create tasks based on reasoning
const task = await callTool('task-master', 'create_task', {
  title: 'Implement JWT authentication',
  priority: 'high',
  status: 'pending'
});

// 4. Create entity in Neo4j
await callTool('neo4j-memory', 'create_entity', {
  label: 'Feature',
  id: 'jwt-auth',
  properties: {
    name: 'JWT Authentication',
    status: 'planned'
  }
});

// 5. Log to Obsidian
await callTool('obsidian-memory', 'create_note', {
  title: 'Authentication Design',
  content: '# JWT Authentication\n\nDesign complete...',
  tags: ['authentication', 'security']
});

// 6. Conclude reasoning chain
await callTool('sequential-thinking', 'conclude', {
  chainId: chain.chainId,
  conclusion: 'Authentication system designed with JWT and bcrypt',
  success: true,
  confidence: 0.95
});
```

---

## SDK Integration

### JavaScript/TypeScript

```typescript
import { MCPSuperserverClient } from '@mcp-superserver/sdk';

const client = new MCPSuperserverClient({
  baseUrl: 'http://localhost:3000',
  apiKey: process.env.MCP_API_KEY
});

// Create entity
const entity = await client.neo4j.createEntity({
  label: 'Person',
  id: 'user-123',
  properties: { name: 'John Doe' }
});

// Chat with Ollama
const response = await client.ollama.chat({
  messages: [{ role: 'user', content: 'Hello!' }]
});
```

### Python

```python
from mcp_superserver import MCPSuperserverClient

client = MCPSuperserverClient(
    base_url='http://localhost:3000',
    api_key='your-api-key'
)

# Create entity
entity = client.neo4j.create_entity(
    label='Person',
    id='user-123',
    properties={'name': 'John Doe'}
)

# Chat with Ollama
response = client.ollama.chat([
    {'role': 'user', 'content': 'Hello!'}
])
```

---

## Versioning

API version: `v1.0.0`

The API follows semantic versioning. Major version changes may include breaking changes.

---

## Support

- **Documentation**: https://github.com/rubenalvarezdev/mcp-superserver
- **Issues**: https://github.com/rubenalvarezdev/mcp-superserver/issues
- **Health Check**: http://localhost:3000/health
