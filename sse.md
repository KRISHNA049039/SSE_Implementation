# SSE Debugging Session — April 2026

## What We Built

Real-time forum messaging using Server-Sent Events (SSE). The backend holds open HTTP connections to connected users and pushes events (new messages, edits, deletes, typing indicators) without the frontend needing to poll.

---

## How It Works Conceptually

```
Browser                     Backend (Spring Boot)            DB Service (port 8081)
  |                               |                                  |
  |-- GET /api/v1/events/stream ->|                                  |
  |<-- SSE connection open -------|                                  |
  |   (stays open forever)        |                                  |
  |                               |                                  |
  |-- POST /events/subscribe ---->|                                  |
  |   { channelId: "abc" }        | registers user → channel         |
  |<-- { subscribed: true } ------|                                  |
  |                               |                                  |
  |-- POST /channels/abc/messages>|                                  |
  |   { content: "hello" }        |-- save message --------------->  |
  |                               |<-- saved message --------------- |
  |                               | broadcast to all subscribers     |
  |<-- event: new_message --------|                                  |
  |   data: { content: "hello" }  |                                  |
```

### Key components

- `SseService` — holds a `Map<userId, SseEmitter>` in memory. When a user connects, their emitter is stored. When an event fires, it loops through subscribers and pushes to each emitter.
- `SseHandler` — exposes `/stream`, `/subscribe`, `/unsubscribe`, `/test-broadcast` endpoints.
- `MessageBuilder` — after saving a message to the db-service, calls `sseService.broadcastNewMessage()`.
- `JwtUtils` — extracts user identity from JWT. Uses `email` claim for real users, falls back to `sub` for M2M tokens.

---

## What Was Fixed During This Session

### 1. Port 8080 already in use
Backend wouldn't start. Fix: kill the old process first.
```powershell
netstat -ano | findstr :8080   # find PID
taskkill /PID <PID> /F
```

### 2. 401 Unauthorized — audience mismatch
The `nirdesh_auth` client issues tokens with `"aud": "account"`. Spring Security was rejecting them because it expected a different audience.

Fixed in `SecurityConfig.java` — added a custom `JwtDecoder` that accepts `"account"`, `"nirdesh-backend"`, and `"nirdesh_auth"` as valid audiences:

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuerUri);
    List<String> validAudiences = List.of("account", "nirdesh-backend", "nirdesh_auth");
    OAuth2TokenValidator<Jwt> audienceValidator = token -> {
        if (token.getAudience().stream().anyMatch(validAudiences::contains)) {
            return OAuth2TokenValidatorResult.success();
        }
        return OAuth2TokenValidatorResult.failure(new OAuth2Error("invalid_token", "Invalid audience", null));
    };
    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(new JwtTimestampValidator(), audienceValidator));
    return decoder;
}
```

### 3. 404 on /api/v1/channels
The channels endpoint didn't exist. Created:
- `model/Channel.java`
- `accessor/ChannelAccessor.java` — calls db-service at `/api/channels`
- `builder/ChannelBuilder.java`
- `handler/ChannelHandler.java` — exposes `GET /api/v1/channels`, `GET /api/v1/channels/{id}`, `POST /api/v1/channels`

### 4. 500 on /api/v1/events/stream
M2M tokens (`client_credentials` grant) don't have an `email` claim. `JwtUtils.getEmailFromToken()` was throwing `IllegalStateException`.

Fixed in `JwtUtils.java` — falls back to `sub` when email is missing:
```java
public static String getEmailFromToken(Jwt jwt) {
    String email = jwt.getClaim("email");
    if (email != null && !email.trim().isEmpty()) return email;
    String subject = jwt.getSubject();
    if (subject != null && !subject.trim().isEmpty()) return subject;
    throw new IllegalStateException("User identity not found in token");
}
```

### 5. SSE broadcast not firing
`broadcastNewMessage` is called in `doOnSuccess` — it only fires if the db-service call succeeds. Since the db-service isn't built yet, the save fails and no broadcast happens.

Added a `POST /api/v1/events/test-broadcast` endpoint to trigger broadcasts directly without needing the db-service. **Remove this before production.**

---

## Current Status

| Endpoint | Status | Notes |
|----------|--------|-------|
| `GET /api/v1/events/stream` | ✅ Working | Returns open SSE connection |
| `POST /api/v1/events/subscribe` | ✅ Working | Registers user to channel |
| `POST /api/v1/events/unsubscribe` | ✅ Working | |
| `POST /api/v1/events/test-broadcast` | ✅ Working | Test only — remove before prod |
| `GET /api/v1/channels` | ⚠️ 400 | Backend endpoint exists, db-service endpoint not built yet |
| `POST /api/v1/channels` | ⚠️ 400 | Same — waiting on db-service |
| Auth (JWT validation) | ✅ Fixed | Audience mismatch resolved |

---

## How to Test SSE Manually

### Step 1 — Get a token
```powershell
$body = "grant_type=client_credentials&client_id=nirdesh_m2m&client_secret=GUMWboRT9rCPjyrnQeCM4QsAgLBWY7lj"
$token = (Invoke-RestMethod -Method Post -Uri "http://localhost:8180/realms/nirdesh/protocol/openid-connect/token" -Body $body).access_token
```

### Step 2 — Open SSE stream (keep this terminal open)
```powershell
curl -N -H "Authorization: Bearer $token" http://localhost:8080/api/v1/events/stream
```
Expected output:
```
event:connected
data:ok
```

### Step 3 — Subscribe to a channel (new terminal, get fresh token first)
```powershell
Invoke-RestMethod -Method Post -Uri "http://localhost:8080/api/v1/events/subscribe" `
  -Headers @{"Authorization"="Bearer $token"; "Content-Type"="application/json"} `
  -Body '{"channelId": "test-channel-1"}'
```

### Step 4 — Trigger a broadcast
```powershell
Invoke-RestMethod -Method Post -Uri "http://localhost:8080/api/v1/events/test-broadcast" `
  -Headers @{"Authorization"="Bearer $token"; "Content-Type"="application/json"} `
  -Body '{"channelId": "test-channel-1", "message": "hello SSE"}'
```

Watch terminal from Step 2 — you should see:
```
event:new_message
data:{"id":"test-id-123","content":"hello SSE","forumId":"test-channel-1","userId":"..."}
```

### Note on tokens
Tokens expire after ~5 minutes. If you get a 401, just re-run Step 1 to get a fresh token and re-open the stream.

---

## What the Backend Developer Needs to Do Next

1. Build `GET /api/channels` on the db-service — returns list of channels from the `channels` table.
2. Build `POST /api/channels` on the db-service — creates a new channel.
3. Once those exist, the frontend 400 error on `/api/v1/channels` will resolve automatically.
4. Remove `POST /api/v1/events/test-broadcast` from `SseHandler.java` before deploying to production.
