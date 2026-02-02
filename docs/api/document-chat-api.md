# OpenClaw Chat WebSocket API Documentation

This document describes the WebSocket API used by the OpenClaw chat UI. Use this to build custom web applications that interface with the OpenClaw gateway.

## Table of Contents

1. [Connection Setup](#connection-setup)
2. [Frame Types](#frame-types)
3. [Connection Handshake](#connection-handshake)
4. [Chat API Methods](#chat-api-methods)
5. [Chat Events](#chat-events)
6. [Tool Streaming](#tool-streaming)
7. [Session Management](#session-management)
8. [Message Content Formats](#message-content-formats)
9. [Error Handling](#error-handling)
10. [Complete Method Reference](#complete-method-reference)

---

## Connection Setup

### WebSocket URL Format

```
ws[s]://[host]:[port]/
```

| Protocol | Use Case |
|----------|----------|
| `ws://`  | Local development (HTTP) |
| `wss://` | Production (HTTPS/TLS) |

**Default local gateway:** `ws://127.0.0.1:18789`

### Connection Flow

```
1. Client opens WebSocket connection
2. Client sends "connect" request
3. Server responds with "hello-ok" containing capabilities
4. Client can now send requests and receive events
5. Server broadcasts events as they occur
```

### Authentication

The API supports two authentication modes:

**Device Identity (Secure Contexts)**
- Generate device keypair using Web Crypto API
- Sign authentication payload with device private key
- Server returns `deviceToken` for subsequent connections

**Token/Password (Fallback)**
- Pass `token` or `password` in connect params
- Used when device identity is not available

---

## Frame Types

The protocol uses three frame types (Protocol v3):

### Request Frame

```typescript
{
  type: "req",
  id: string,        // UUID for matching response
  method: string,    // RPC method name
  params?: unknown   // Method-specific parameters
}
```

**Example:**
```json
{
  "type": "req",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "method": "chat.history",
  "params": {
    "sessionKey": "main",
    "limit": 50
  }
}
```

### Response Frame

```typescript
{
  type: "res",
  id: string,           // Matches request id
  ok: boolean,
  payload?: unknown,    // Result (only if ok=true)
  error?: {
    code: string,
    message: string,
    details?: unknown,
    retryable?: boolean,
    retryAfterMs?: number
  }
}
```

**Success Example:**
```json
{
  "type": "res",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "ok": true,
  "payload": {
    "messages": [...],
    "thinkingLevel": "normal"
  }
}
```

**Error Example:**
```json
{
  "type": "res",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "ok": false,
  "error": {
    "code": "not_found",
    "message": "Session not found",
    "retryable": false
  }
}
```

### Event Frame

```typescript
{
  type: "event",
  event: string,        // Event type name
  payload?: unknown,    // Event-specific data
  seq?: number,         // Sequence number for ordering
  stateVersion?: {
    presence?: number,
    health?: number
  }
}
```

**Example:**
```json
{
  "type": "event",
  "event": "chat",
  "seq": 42,
  "payload": {
    "runId": "run_abc123",
    "sessionKey": "main",
    "state": "delta",
    "message": {...}
  }
}
```

---

## Connection Handshake

### Connect Request

Send immediately after WebSocket opens:

```json
{
  "type": "req",
  "id": "<uuid>",
  "method": "connect",
  "params": {
    "clientVersion": "1.0.0",
    "clientType": "web",
    "token": "<optional-auth-token>",
    "deviceAuth": {
      "deviceId": "<device-uuid>",
      "signature": "<base64-signature>",
      "timestamp": 1699999999999
    }
  }
}
```

### HelloOk Response

```typescript
{
  type: "hello-ok",
  protocol: 3,
  server: {
    version: string,
    commit?: string,
    host?: string,
    connId: string
  },
  features: {
    methods: string[],    // Available RPC methods
    events: string[]      // Available event types
  },
  snapshot?: {
    presence?: PresenceEntry[],
    health?: HealthSnapshot,
    sessionDefaults?: {
      defaultAgentId?: string,
      mainSessionKey?: string
    }
  },
  canvasHostUrl?: string,
  auth?: {
    deviceToken?: string,   // Store for reconnection
    role: string,
    scopes: string[],
    issuedAtMs?: number
  },
  policy: {
    maxPayload: number,        // Max frame size in bytes
    maxBufferedBytes: number,
    tickIntervalMs: number     // Server tick interval
  }
}
```

---

## Chat API Methods

### Load Chat History

Retrieve message history for a session.

**Request:**
```json
{
  "type": "req",
  "id": "<uuid>",
  "method": "chat.history",
  "params": {
    "sessionKey": "main",
    "limit": 200
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionKey` | string | Yes | Session identifier |
| `limit` | number | No | Max messages (1-1000, default 200) |

**Response:**
```json
{
  "type": "res",
  "id": "<uuid>",
  "ok": true,
  "payload": {
    "messages": [
      {
        "role": "user",
        "content": [{"type": "text", "text": "Hello"}],
        "timestamp": 1699999999999
      },
      {
        "role": "assistant",
        "content": [{"type": "text", "text": "Hi there!"}],
        "timestamp": 1700000000000
      }
    ],
    "thinkingLevel": "normal"
  }
}
```

### Send Chat Message

Send a message to a session and receive streaming responses.

**Request:**
```json
{
  "type": "req",
  "id": "<uuid>",
  "method": "chat.send",
  "params": {
    "sessionKey": "main",
    "message": "What is the weather like?",
    "thinking": "normal",
    "deliver": true,
    "idempotencyKey": "<uuid>",
    "attachments": [
      {
        "type": "image",
        "mimeType": "image/png",
        "content": "<base64-encoded-data>"
      }
    ],
    "timeoutMs": 120000
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionKey` | string | Yes | Session identifier |
| `message` | string | Yes | User message text |
| `thinking` | string | No | Thinking level: `"none"`, `"low"`, `"normal"`, `"high"` |
| `deliver` | boolean | No | Whether to deliver to channel (default true) |
| `idempotencyKey` | string | Yes | UUID for deduplication |
| `attachments` | array | No | Image attachments |
| `timeoutMs` | number | No | Request timeout |

**Response:**
```json
{
  "type": "res",
  "id": "<uuid>",
  "ok": true,
  "payload": null
}
```

After sending, listen for `chat` events with matching `sessionKey` for streaming responses.

### Abort Chat Run

Stop an in-progress chat run.

**Request:**
```json
{
  "type": "req",
  "id": "<uuid>",
  "method": "chat.abort",
  "params": {
    "sessionKey": "main",
    "runId": "run_abc123"
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionKey` | string | Yes | Session identifier |
| `runId` | string | No | Specific run to abort (aborts current if omitted) |

---

## Chat Events

### Chat Event

Emitted during message generation.

```typescript
{
  type: "event",
  event: "chat",
  payload: {
    runId: string,
    sessionKey: string,
    seq: number,
    state: "delta" | "final" | "aborted" | "error",
    message?: MessageContent,
    errorMessage?: string,
    usage?: {
      inputTokens: number,
      outputTokens: number
    },
    stopReason?: string
  }
}
```

#### State Values

| State | Description | Action |
|-------|-------------|--------|
| `delta` | Partial streaming content | Append to display, accumulate text |
| `final` | Complete message | Reload history for full context |
| `aborted` | User aborted the run | Clear streaming state |
| `error` | Error occurred | Display `errorMessage` |

**Delta Example:**
```json
{
  "type": "event",
  "event": "chat",
  "payload": {
    "runId": "run_abc123",
    "sessionKey": "main",
    "seq": 1,
    "state": "delta",
    "message": {
      "role": "assistant",
      "content": [{"type": "text", "text": "The weather"}]
    }
  }
}
```

**Final Example:**
```json
{
  "type": "event",
  "event": "chat",
  "payload": {
    "runId": "run_abc123",
    "sessionKey": "main",
    "seq": 15,
    "state": "final",
    "message": {
      "role": "assistant",
      "content": [{"type": "text", "text": "The weather today is sunny with a high of 72F."}]
    },
    "usage": {
      "inputTokens": 150,
      "outputTokens": 25
    },
    "stopReason": "end_turn"
  }
}
```

---

## Tool Streaming

### Agent Event

Emitted during tool execution.

```typescript
{
  type: "event",
  event: "agent",
  payload: {
    runId: string,
    seq: number,
    stream: "tool" | "compaction",
    ts: number,
    sessionKey?: string,
    data: Record<string, unknown>
  }
}
```

### Tool Stream Phases

#### Tool Start
```json
{
  "type": "event",
  "event": "agent",
  "payload": {
    "runId": "run_abc123",
    "seq": 5,
    "stream": "tool",
    "ts": 1699999999999,
    "data": {
      "phase": "start",
      "toolCallId": "tc_xyz789",
      "name": "web_search",
      "args": {
        "query": "weather forecast"
      }
    }
  }
}
```

#### Tool Update (Partial Result)
```json
{
  "type": "event",
  "event": "agent",
  "payload": {
    "runId": "run_abc123",
    "seq": 6,
    "stream": "tool",
    "ts": 1700000000000,
    "data": {
      "phase": "update",
      "toolCallId": "tc_xyz789",
      "name": "web_search",
      "partialResult": "Fetching results..."
    }
  }
}
```

#### Tool Result
```json
{
  "type": "event",
  "event": "agent",
  "payload": {
    "runId": "run_abc123",
    "seq": 7,
    "stream": "tool",
    "ts": 1700000001000,
    "data": {
      "phase": "result",
      "toolCallId": "tc_xyz789",
      "name": "web_search",
      "result": {
        "results": [...]
      }
    }
  }
}
```

### Compaction Events

```json
{
  "type": "event",
  "event": "agent",
  "payload": {
    "runId": "run_abc123",
    "stream": "compaction",
    "data": {
      "phase": "start"
    }
  }
}
```

---

## Session Management

### Session Key Format

```
sessionKey: string

Examples:
- "main"                           # Main/default session
- "agent:myagent:main"             # Agent-specific main session
- "main:direct:+1234567890"        # Direct message channel session
- "telegram:group:123456:@user"    # Telegram group session
```

### List Sessions

**Request:**
```json
{
  "type": "req",
  "id": "<uuid>",
  "method": "sessions.list",
  "params": {
    "limit": 50,
    "activeMinutes": 60,
    "includeGlobal": false,
    "includeUnknown": false,
    "includeDerivedTitles": true,
    "includeLastMessage": true,
    "search": "project"
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `limit` | number | No | Max sessions to return |
| `activeMinutes` | number | No | Filter by recency (minutes) |
| `includeGlobal` | boolean | No | Include global sessions |
| `includeUnknown` | boolean | No | Include unknown type sessions |
| `includeDerivedTitles` | boolean | No | Include AI-generated titles |
| `includeLastMessage` | boolean | No | Include last message preview |
| `agentId` | string | No | Filter by agent |
| `search` | string | No | Search filter |

**Response:**
```json
{
  "type": "res",
  "id": "<uuid>",
  "ok": true,
  "payload": {
    "ts": 1699999999999,
    "path": "/sessions",
    "count": 5,
    "defaults": {
      "model": "claude-3-opus",
      "contextTokens": 200000
    },
    "sessions": [
      {
        "key": "main",
        "kind": "direct",
        "label": "Main Chat",
        "displayName": "Main Chat",
        "surface": "web",
        "updatedAt": 1699999999999,
        "sessionId": "sess_abc123",
        "thinkingLevel": "normal",
        "model": "claude-3-opus",
        "inputTokens": 1500,
        "outputTokens": 500,
        "totalTokens": 2000
      }
    ]
  }
}
```

### Session Object

```typescript
interface SessionRow {
  key: string;
  kind: "direct" | "group" | "global" | "unknown";
  label?: string;
  displayName?: string;
  surface?: string;           // Channel type (web, telegram, etc.)
  subject?: string;
  room?: string;
  space?: string;
  updatedAt: number | null;
  sessionId?: string;
  thinkingLevel?: string;
  verboseLevel?: string;
  model?: string;
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;
}
```

### Delete Session

**Request:**
```json
{
  "type": "req",
  "id": "<uuid>",
  "method": "sessions.delete",
  "params": {
    "sessionKey": "main"
  }
}
```

### Reset Session History

**Request:**
```json
{
  "type": "req",
  "id": "<uuid>",
  "method": "sessions.reset",
  "params": {
    "sessionKey": "main"
  }
}
```

---

## Message Content Formats

### User Message

```typescript
{
  role: "user",
  content: [
    { type: "text", text: string },
    {
      type: "image",
      source: {
        type: "base64",
        media_type: string,  // e.g., "image/png"
        data: string         // Base64 encoded
      }
    }
  ],
  timestamp: number
}
```

### Assistant Message

```typescript
{
  role: "assistant",
  content: [
    { type: "text", text: string },
    {
      type: "tool_use",
      id: string,
      name: string,
      input: object
    }
  ],
  timestamp: number,
  runId?: string
}
```

### Tool Result Message

```typescript
{
  role: "user",
  content: [
    {
      type: "tool_result",
      tool_use_id: string,
      content: string | object
    }
  ],
  timestamp: number
}
```

---

## Error Handling

### Error Response Structure

```typescript
{
  type: "res",
  id: string,
  ok: false,
  error: {
    code: string,
    message: string,
    details?: unknown,
    retryable?: boolean,
    retryAfterMs?: number
  }
}
```

### Common Error Codes

| Code | Description |
|------|-------------|
| `invalid_params` | Request validation failed |
| `not_found` | Session or resource not found |
| `permission_denied` | Authentication/authorization failed |
| `rate_limited` | Too many requests |
| `internal_error` | Server error |
| `timeout` | Request timed out |

### Reconnection Strategy

Implement exponential backoff for connection failures:

```javascript
const backoffMs = [800, 1600, 3200, 6400, 15000];
let attempt = 0;

function reconnect() {
  const delay = backoffMs[Math.min(attempt, backoffMs.length - 1)];
  setTimeout(() => {
    attempt++;
    connect();
  }, delay);
}
```

---

## Complete Method Reference

### Chat & History
| Method | Description |
|--------|-------------|
| `chat.history` | Load message history |
| `chat.send` | Send message |
| `chat.abort` | Abort current run |
| `chat.inject` | Inject system message |

### Sessions
| Method | Description |
|--------|-------------|
| `sessions.list` | List all sessions |
| `sessions.resolve` | Resolve session details |
| `sessions.patch` | Update session settings |
| `sessions.delete` | Delete session |
| `sessions.reset` | Clear session history |
| `sessions.preview` | Get session preview |
| `sessions.compact` | Compact session storage |

### Agents & Identity
| Method | Description |
|--------|-------------|
| `agents.list` | List available agents |
| `agent.identity.get` | Get agent identity |

### Configuration
| Method | Description |
|--------|-------------|
| `config.get` | Get configuration |
| `config.set` | Set configuration |
| `config.patch` | Patch configuration |
| `config.schema` | Get config schema |

### Status & Health
| Method | Description |
|--------|-------------|
| `status` | Gateway status |
| `health` | Health check |
| `models.list` | Available models |

### Channels
| Method | Description |
|--------|-------------|
| `channels.status` | Get channel status |
| `channels.logout` | Logout from channel |

---

## Event Reference

| Event | Description |
|-------|-------------|
| `chat` | Chat message streaming |
| `agent` | Tool execution events |
| `presence` | User presence updates |
| `tick` | Server heartbeat |
| `shutdown` | Server shutdown notice |
| `device.pair.requested` | Device pairing request |
| `device.pair.resolved` | Device pairing resolved |
| `exec.approval.requested` | Execution approval needed |
| `exec.approval.resolved` | Execution approval resolved |
| `cron` | Cron job status change |

---

## Example: Minimal Chat Client

```javascript
class ChatClient {
  constructor(url) {
    this.url = url;
    this.ws = null;
    this.pending = new Map();
    this.eventHandlers = new Map();
  }

  connect() {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(this.url);

      this.ws.onopen = () => {
        this.send('connect', { clientType: 'web' })
          .then(resolve)
          .catch(reject);
      };

      this.ws.onmessage = (event) => {
        const frame = JSON.parse(event.data);

        if (frame.type === 'res') {
          const handler = this.pending.get(frame.id);
          if (handler) {
            this.pending.delete(frame.id);
            if (frame.ok) {
              handler.resolve(frame.payload);
            } else {
              handler.reject(frame.error);
            }
          }
        } else if (frame.type === 'event') {
          const handlers = this.eventHandlers.get(frame.event) || [];
          handlers.forEach(h => h(frame.payload));
        }
      };

      this.ws.onerror = reject;
    });
  }

  send(method, params) {
    return new Promise((resolve, reject) => {
      const id = crypto.randomUUID();
      this.pending.set(id, { resolve, reject });

      this.ws.send(JSON.stringify({
        type: 'req',
        id,
        method,
        params
      }));
    });
  }

  on(event, handler) {
    if (!this.eventHandlers.has(event)) {
      this.eventHandlers.set(event, []);
    }
    this.eventHandlers.get(event).push(handler);
  }

  async loadHistory(sessionKey, limit = 200) {
    return this.send('chat.history', { sessionKey, limit });
  }

  async sendMessage(sessionKey, message) {
    return this.send('chat.send', {
      sessionKey,
      message,
      idempotencyKey: crypto.randomUUID()
    });
  }

  async abort(sessionKey) {
    return this.send('chat.abort', { sessionKey });
  }
}

// Usage
const client = new ChatClient('ws://127.0.0.1:18789');

client.on('chat', (payload) => {
  if (payload.state === 'delta') {
    // Accumulate streaming text
    console.log('Streaming:', payload.message?.content);
  } else if (payload.state === 'final') {
    // Message complete
    console.log('Complete:', payload.message?.content);
  }
});

client.on('agent', (payload) => {
  if (payload.stream === 'tool') {
    console.log('Tool event:', payload.data);
  }
});

await client.connect();
const history = await client.loadHistory('main');
await client.sendMessage('main', 'Hello!');
```

---

## Protocol Version

Current protocol version: **3**

The protocol version is included in the `hello-ok` response. Clients should verify compatibility.
