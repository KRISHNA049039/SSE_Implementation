# SSE Implementation — Nirdesh Backend

Real-time forum messaging via Server-Sent Events (SSE), following the existing `accessor → component → builder → handler` architecture.

---

## Architecture Overview

```
Frontend
  │
  ├── GET  /api/v1/events/stream          → SseHandler → SseService.connect()
  ├── POST /api/v1/events/subscribe       → SseHandler → SseService.subscribe()
  ├── POST /api/v1/events/unsubscribe     → SseHandler → SseService.unsubscribe()
  │
  ├── GET  /api/v1/forums/:id/messages    → MessageHandler → MessageBuilder → MessageAccessor (db-service)
  ├── POST /api/v1/channels/:id/messages  → MessageHandler → MessageBuilder → MessageAccessor + SseService.broadcast()
  ├── PUT  /api/v1/messages/:id           → MessageHandler → MessageBuilder → MessageAccessor + SseService.broadcast()
  ├── DELETE /api/v1/messages/:id         → MessageHandler → MessageBuilder → MessageAccessor + SseService.broadcast()
  └── POST /api/v1/channels/:id/typing    → MessageHandler → MessageBuilder → SseService.broadcast()
```

---

## Files Added / Modified

### New Files

| File | Purpose |
|------|---------|
| `model/SseEvent.java` | Event payload wrapper with static factory methods |
| `model/ForumMessage.java` | Forum message model with API-friendly field names |
| `model/MessageRequest.java` | Request body for sending/editing messages |
| `component/SseService.java` | In-memory SSE connection manager and broadcaster |
| `component/MessageComponent.java` | Validation for message requests |
| `accessor/MessageAccessor.java` | WebClient calls to db-service for message CRUD |
| `builder/MessageBuilder.java` | Orchestrates accessor + SSE broadcast |
| `handler/SseHandler.java` | SSE connection/subscribe/unsubscribe endpoints |
| `handler/MessageHandler.java` | Message CRUD endpoints |
| `exception/ChannelNotFoundException.java` | Thrown when a channel is not found |

### Modified Files

| File | Change |
|------|--------|
| `config/SecurityConfig.java` | Added SSE-required exposed headers (Content-Type, Cache-Control, Connection, X-Accel-Buffering) |
| `exception/GlobalExceptionHandler.java` | Added handler for `ChannelNotFoundException` |
| `resources/application.yaml` | Added `spring.mvc.async.request-timeout: -1` to prevent SSE connection timeouts |

---

## Endpoints

### SSE Connection

#### `GET /api/v1/events/stream`
Opens the SSE stream. Frontend keeps this connection alive.

- Auth: Bearer JWT (Keycloak)
- Response: `text/event-stream`
- On connect: sends `event: connected\ndata: ok`

#### `POST /api/v1/events/subscribe`
Subscribe to events for a channel (call when user opens a forum).

```json
// Request
{ "channelId": "uuid" }

// Response 200
{ "subscribed": true, "channelId": "uuid", "subscribers": 3 }
```

#### `POST /api/v1/events/unsubscribe`
Unsubscribe from a channel (call when user leaves a forum).

```json
// Request
{ "channelId": "uuid" }

// Response 200
{ "unsubscribed": true }
```

---

### Messages

#### `GET /api/v1/forums/:channelId/messages`
Fetch paginated messages for a forum.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `limit` | int | 50 | Max messages to return |
| `before` | string | null | Cursor — message ID to paginate before |

```json
// Response 200
{
  "messages": [ { "id": "...", "content": "...", ... } ],
  "hasMore": false
}
```

#### `POST /api/v1/channels/:channelId/messages`
Send a message. Saves to DB then broadcasts `new_message` SSE event.

```json
// Request
{ "content": "Hello!", "parentMessageId": null }

// Response 201
{
  "id": "uuid",
  "forumId": "channel-uuid",
  "userId": "user@example.com",
  "username": "alice",
  "content": "Hello!",
  "isEdited": false,
  "createdAt": "...",
  "updatedAt": "..."
}
```

#### `PUT /api/v1/messages/:messageId`
Edit a message (author only). Broadcasts `message_updated` SSE event.

```json
// Request
{ "content": "Updated content" }

// Response 200 — updated ForumMessage
```

#### `DELETE /api/v1/messages/:messageId`
Soft-delete a message (author, `forum_mod`, or `forum_admin`). Broadcasts `message_deleted` SSE event.

- Response: `204 No Content`

#### `POST /api/v1/channels/:channelId/typing`
Typing indicator. Broadcasts `user_typing` SSE event — no DB write.

```json
// Request
{ "isTyping": true }

// Response 200
```

---

## SSE Event Types

All events are pushed to subscribers of the relevant channel.

| Event name | Triggered by | Data shape |
|------------|-------------|------------|
| `connected` | On stream open | `"ok"` |
| `new_message` | POST message | `ForumMessage` object |
| `message_updated` | PUT message | `ForumMessage` object |
| `message_deleted` | DELETE message | `{ "messageId": "uuid" }` |
| `user_typing` | POST typing | `{ "userId": "...", "isTyping": true }` |

---

## How It Flows

```
1. User opens the app
   → GET /api/v1/events/stream
   → SseService stores SseEmitter keyed by userId (email from JWT)

2. User opens a forum
   → POST /api/v1/events/subscribe { channelId }
   → SseService adds userId to channelSubscriptions[channelId]

3. User sends a message
   → POST /api/v1/channels/:id/messages
   → MessageBuilder validates → MessageAccessor.save() → db-service
   → SseService.broadcastNewMessage() pushes to all channel subscribers

4. User closes the forum
   → POST /api/v1/events/unsubscribe { channelId }
   → SseService removes userId from channelSubscriptions[channelId]

5. Browser tab closes / network drops
   → SseEmitter.onCompletion / onError fires
   → SseService removes the emitter automatically
```

---

## Key Design Decisions

- `SseService` is a `@Component` (not `@Service`) to match the existing component layer convention.
- `MessageAccessor` uses `WebClient` to call the db-service, consistent with all other accessors.
- `MessageBuilder` orchestrates validation, DB write, and SSE broadcast — same builder pattern used throughout.
- `userId` is derived from `jwt.getClaim("email")` via `JwtUtils.getEmailFromToken()`, consistent with the rest of the codebase.
- No extra dependencies needed — SSE is built into `spring-boot-starter-web`.

---

## Testing Manually

```bash
# 1. Get a Keycloak token
TOKEN=$(curl -s -X POST $KEYCLOAK_TOKEN_URI \
  -d "grant_type=password&client_id=nirdesh-host&username=alice&password=password123" \
  | jq -r '.access_token')

# 2. Open SSE stream (keep this terminal open)
curl -N -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/events/stream

# 3. Subscribe to a channel
curl -X POST http://localhost:8080/api/v1/events/subscribe \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"channelId": "forum-1"}'

# 4. Send a message (in another terminal)
curl -X POST http://localhost:8080/api/v1/channels/forum-1/messages \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "Hello from curl!"}'

# Terminal 2 should immediately show:
# event: new_message
# data: {"id":"...","forumId":"forum-1","content":"Hello from curl!",...}
```

---

## Nginx Config (if behind a proxy)

```nginx
location /api/v1/events/stream {
    proxy_pass http://backend:8080;
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 86400s;
    proxy_set_header Connection '';
    proxy_http_version 1.1;
    proxy_set_header X-Accel-Buffering no;
}
```
