# SSE Implementation Guide — Spring Boot (Nirdesh Backend)

Complete guide to add Server-Sent Events to `com.icloudlogic.nirdesh_backend`.

## Where Each File Goes

```
src/main/java/com/icloudlogic/nirdesh_backend/
├── model/
│   ├── SseEvent.java              ← Event payload wrapper
│   ├── Message.java               ← Your existing message entity (add fields if needed)
│   └── MessageRequest.java        ← Request body for sending a message
├── component/
│   └── SseService.java            ← In-memory connection manager + broadcaster
├── handler/
│   ├── SseController.java         ← /api/events/* endpoints
│   └── MessageController.java     ← /api/channels/:id/messages endpoints
├── accessor/
│   └── MessageAccessor.java       ← DB access for messages (your existing pattern)
├── config/
│   └── CorsConfig.java            ← CORS config (update existing)
└── exception/
    └── ChannelNotFoundException.java  ← (add if not exists)
```

---

## Step 1: Dependencies (build.gradle.kts)

You likely already have these. Confirm they're present:

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    // No extra dependency needed for SSE — it's built into Spring Web
}
```

---

## Step 2: model/SseEvent.java

A simple wrapper for all SSE payloads. Keeps the event structure consistent.

```java
package com.icloudlogic.nirdesh_backend.model;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class SseEvent {

    private String type;      // "new_message", "message_updated", "message_deleted", "user_typing"
    private Object data;      // the actual payload
    private String channelId; // which channel this event belongs to

    public SseEvent() {}

    public SseEvent(String type, Object data, String channelId) {
        this.type = type;
        this.data = data;
        this.channelId = channelId;
    }

    // Static factory methods for each event type
    public static SseEvent newMessage(Object message, String channelId) {
        return new SseEvent("new_message", message, channelId);
    }

    public static SseEvent messageUpdated(Object message, String channelId) {
        return new SseEvent("message_updated", message, channelId);
    }

    public static SseEvent messageDeleted(String messageId, String channelId) {
        return new SseEvent("message_deleted", java.util.Map.of("messageId", messageId), channelId);
    }

    public static SseEvent userTyping(String userId, boolean isTyping, String channelId) {
        return new SseEvent("user_typing",
            java.util.Map.of("userId", userId, "isTyping", isTyping), channelId);
    }

    // Getters and setters
    public String getType() { return type; }
    public void setType(String type) { this.type = type; }
    public Object getData() { return data; }
    public void setData(Object data) { this.data = data; }
    public String getChannelId() { return channelId; }
    public void setChannelId(String channelId) { this.channelId = channelId; }
}
```

---

## Step 3: model/MessageRequest.java

Request body when the frontend sends a message.

```java
package com.icloudlogic.nirdesh_backend.model;

public class MessageRequest {

    private String content;
    private String parentMessageId; // null for top-level, set for thread replies

    public MessageRequest() {}

    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }
    public String getParentMessageId() { return parentMessageId; }
    public void setParentMessageId(String parentMessageId) { this.parentMessageId = parentMessageId; }
}
```

---

## Step 4: component/SseService.java

The core of SSE. Manages all open connections in memory and broadcasts events.

```java
package com.icloudlogic.nirdesh_backend.component;

import com.icloudlogic.nirdesh_backend.model.SseEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class SseService {

    private static final Logger log = LoggerFactory.getLogger(SseService.class);

    // userId → their open SSE connection
    private final Map<String, SseEmitter> emitters = new ConcurrentHashMap<>();

    // channelId → set of userIds subscribed to that channel
    private final Map<String, Set<String>> channelSubscriptions = new ConcurrentHashMap<>();

    /**
     * Called when frontend hits GET /api/events/stream
     * Creates an SseEmitter and stores it for this user.
     */
    public SseEmitter connect(String userId) {
        // Remove any existing connection for this user
        SseEmitter existing = emitters.get(userId);
        if (existing != null) {
            existing.complete();
        }

        SseEmitter emitter = new SseEmitter(Long.MAX_VALUE); // no timeout
        emitters.put(userId, emitter);

        log.info("SSE connected: userId={}, total connections={}", userId, emitters.size());

        // Clean up when connection closes (browser tab closed, network drop, etc.)
        emitter.onCompletion(() -> {
            emitters.remove(userId);
            log.info("SSE disconnected (completion): userId={}", userId);
        });
        emitter.onTimeout(() -> {
            emitters.remove(userId);
            log.info("SSE disconnected (timeout): userId={}", userId);
        });
        emitter.onError(e -> {
            emitters.remove(userId);
            log.warn("SSE disconnected (error): userId={}, error={}", userId, e.getMessage());
        });

        // Send a heartbeat immediately to confirm connection
        try {
            emitter.send(SseEmitter.event().name("connected").data("ok"));
        } catch (IOException e) {
            emitters.remove(userId);
        }

        return emitter;
    }

    /**
     * Called when frontend hits POST /api/events/subscribe
     * Registers user to receive events for a specific channel.
     */
    public void subscribe(String userId, String channelId) {
        channelSubscriptions
            .computeIfAbsent(channelId, k -> ConcurrentHashMap.newKeySet())
            .add(userId);
        log.debug("SSE subscribe: userId={}, channelId={}", userId, channelId);
    }

    /**
     * Called when frontend hits POST /api/events/unsubscribe
     */
    public void unsubscribe(String userId, String channelId) {
        Set<String> subs = channelSubscriptions.get(channelId);
        if (subs != null) {
            subs.remove(userId);
        }
        log.debug("SSE unsubscribe: userId={}, channelId={}", userId, channelId);
    }

    /**
     * Broadcast an event to all users subscribed to a channel.
     * Called after saving a message, edit, or delete to the DB.
     */
    public void broadcast(String channelId, SseEvent event) {
        Set<String> subscribers = channelSubscriptions.getOrDefault(channelId, Set.of());

        if (subscribers.isEmpty()) {
            return; // no one to notify
        }

        for (String userId : subscribers) {
            SseEmitter emitter = emitters.get(userId);
            if (emitter != null) {
                try {
                    emitter.send(SseEmitter.event()
                        .name(event.getType())  // "new_message", "message_updated", etc.
                        .data(event.getData())); // the message object as JSON
                } catch (IOException e) {
                    // Connection dropped — clean up
                    emitters.remove(userId);
                    log.warn("SSE send failed, removing emitter: userId={}", userId);
                }
            }
        }
    }

    /**
     * Convenience methods for each event type
     */
    public void broadcastNewMessage(String channelId, Object messageDto) {
        broadcast(channelId, SseEvent.newMessage(messageDto, channelId));
    }

    public void broadcastMessageUpdated(String channelId, Object messageDto) {
        broadcast(channelId, SseEvent.messageUpdated(messageDto, channelId));
    }

    public void broadcastMessageDeleted(String channelId, String messageId) {
        broadcast(channelId, SseEvent.messageDeleted(messageId, channelId));
    }

    public void broadcastTyping(String channelId, String userId, boolean isTyping) {
        broadcast(channelId, SseEvent.userTyping(userId, isTyping, channelId));
    }

    // Diagnostics
    public int getConnectionCount() { return emitters.size(); }
    public int getSubscriberCount(String channelId) {
        return channelSubscriptions.getOrDefault(channelId, Set.of()).size();
    }
}
```

---

## Step 5: handler/SseController.java

Exposes the SSE endpoints the frontend calls.

```java
package com.icloudlogic.nirdesh_backend.handler;

import com.icloudlogic.nirdesh_backend.component.SseService;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.util.Map;

@RestController
@RequestMapping("/api/events")
public class SseController {

    private final SseService sseService;

    public SseController(SseService sseService) {
        this.sseService = sseService;
    }

    /**
     * Frontend connects here on app load.
     * GET /api/events/stream
     *
     * The browser keeps this connection open and listens for events.
     * Auth token is passed as query param because EventSource doesn't support headers.
     */
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter stream(@AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getSubject(); // Keycloak user UUID
        return sseService.connect(userId);
    }

    /**
     * Frontend calls this when user opens a forum.
     * POST /api/events/subscribe
     * Body: { "channelId": "uuid" }
     */
    @PostMapping("/subscribe")
    public ResponseEntity<Map<String, Object>> subscribe(
        @RequestBody Map<String, String> body,
        @AuthenticationPrincipal Jwt jwt
    ) {
        String userId = jwt.getSubject();
        String channelId = body.get("channelId");

        if (channelId == null || channelId.isBlank()) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", "INVALID_INPUT", "message", "channelId is required"));
        }

        sseService.subscribe(userId, channelId);
        return ResponseEntity.ok(Map.of(
            "subscribed", true,
            "channelId", channelId,
            "subscribers", sseService.getSubscriberCount(channelId)
        ));
    }

    /**
     * Frontend calls this when user leaves a forum.
     * POST /api/events/unsubscribe
     * Body: { "channelId": "uuid" }
     */
    @PostMapping("/unsubscribe")
    public ResponseEntity<Map<String, Boolean>> unsubscribe(
        @RequestBody Map<String, String> body,
        @AuthenticationPrincipal Jwt jwt
    ) {
        sseService.unsubscribe(jwt.getSubject(), body.get("channelId"));
        return ResponseEntity.ok(Map.of("unsubscribed", true));
    }
}
```

---

## Step 6: handler/MessageController.java

Handles message CRUD. Calls `SseService` after every DB write to broadcast the event.

```java
package com.icloudlogic.nirdesh_backend.handler;

import com.icloudlogic.nirdesh_backend.accessor.MessageAccessor;
import com.icloudlogic.nirdesh_backend.component.SseService;
import com.icloudlogic.nirdesh_backend.model.Message;
import com.icloudlogic.nirdesh_backend.model.MessageRequest;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api")
public class MessageController {

    private final MessageAccessor messageAccessor;
    private final SseService sseService;

    public MessageController(MessageAccessor messageAccessor, SseService sseService) {
        this.messageAccessor = messageAccessor;
        this.sseService = sseService;
    }

    /**
     * GET /api/forums/:channelId/messages
     * Load messages for a forum (called on forum open and pagination)
     */
    @GetMapping("/forums/{channelId}/messages")
    public ResponseEntity<Map<String, Object>> getMessages(
        @PathVariable String channelId,
        @RequestParam(defaultValue = "50") int limit,
        @RequestParam(required = false) String before,
        @AuthenticationPrincipal Jwt jwt
    ) {
        List<Message> messages = messageAccessor.findByChannelId(channelId, limit, before);
        boolean hasMore = messages.size() == limit;

        return ResponseEntity.ok(Map.of(
            "messages", messages,
            "hasMore", hasMore
        ));
    }

    /**
     * POST /api/channels/:channelId/messages
     * Send a message — saves to DB then broadcasts via SSE
     *
     * Note: frontend SSE service calls /api/channels/:id/messages (not /api/forums/)
     */
    @PostMapping("/channels/{channelId}/messages")
    public ResponseEntity<Message> sendMessage(
        @PathVariable String channelId,
        @RequestBody MessageRequest request,
        @AuthenticationPrincipal Jwt jwt
    ) {
        // Validate
        if (request.getContent() == null || request.getContent().isBlank()) {
            return ResponseEntity.badRequest().build();
        }
        if (request.getContent().length() > 10000) {
            return ResponseEntity.badRequest().build();
        }

        String userId = jwt.getSubject();
        String username = jwt.getClaimAsString("preferred_username");

        // 1. Save to DB
        Message saved = messageAccessor.save(channelId, request, userId, username);

        // 2. Broadcast via SSE — no DB involved
        sseService.broadcastNewMessage(channelId, saved);

        return ResponseEntity.status(201).body(saved);
    }

    /**
     * PUT /api/messages/:messageId
     * Edit a message — only the author can edit
     */
    @PutMapping("/messages/{messageId}")
    public ResponseEntity<Message> editMessage(
        @PathVariable String messageId,
        @RequestBody Map<String, String> body,
        @AuthenticationPrincipal Jwt jwt
    ) {
        String userId = jwt.getSubject();
        String newContent = body.get("content");

        Message updated = messageAccessor.update(messageId, newContent, userId);
        // updated.channelId is needed to broadcast to the right channel
        sseService.broadcastMessageUpdated(updated.getChannelId(), updated);

        return ResponseEntity.ok(updated);
    }

    /**
     * DELETE /api/messages/:messageId
     * Delete a message — author or moderator/admin
     */
    @DeleteMapping("/messages/{messageId}")
    public ResponseEntity<Void> deleteMessage(
        @PathVariable String messageId,
        @AuthenticationPrincipal Jwt jwt
    ) {
        String userId = jwt.getSubject();
        List<String> roles = jwt.getClaimAsStringList("realm_access.roles");
        boolean isMod = roles != null && (roles.contains("forum_mod") || roles.contains("forum_admin"));

        String channelId = messageAccessor.delete(messageId, userId, isMod);
        sseService.broadcastMessageDeleted(channelId, messageId);

        return ResponseEntity.noContent().build();
    }

    /**
     * POST /api/channels/:channelId/typing
     * Typing indicator — broadcast to channel subscribers, no DB write
     */
    @PostMapping("/channels/{channelId}/typing")
    public ResponseEntity<Void> typing(
        @PathVariable String channelId,
        @RequestBody Map<String, Boolean> body,
        @AuthenticationPrincipal Jwt jwt
    ) {
        boolean isTyping = Boolean.TRUE.equals(body.get("isTyping"));
        sseService.broadcastTyping(channelId, jwt.getSubject(), isTyping);
        return ResponseEntity.ok().build();
    }
}
```

---

## Step 7: accessor/MessageAccessor.java

DB access layer following your existing `accessor` pattern.

```java
package com.icloudlogic.nirdesh_backend.accessor;

import com.icloudlogic.nirdesh_backend.model.Message;
import com.icloudlogic.nirdesh_backend.model.MessageRequest;
import org.springframework.stereotype.Repository;

import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.transaction.Transactional;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Repository
public class MessageAccessor {

    @PersistenceContext
    private EntityManager em;

    /**
     * Fetch messages for a channel, newest first, with cursor pagination.
     * Maps DB columns to API field names.
     */
    @SuppressWarnings("unchecked")
    public List<Message> findByChannelId(String channelId, int limit, String beforeMessageId) {
        String sql = """
            SELECT
                m.message_id        AS id,
                m.channel_id        AS forumId,
                m.created_by        AS userId,
                m.comment           AS content,
                m.parent_message_id AS parentMessageId,
                m.created_date      AS createdAt,
                m.modified_date     AS updatedAt,
                (m.modified_date != m.created_date) AS isEdited
            FROM messages m
            WHERE m.channel_id = :channelId
              AND m.is_active = true
              AND (:beforeId IS NULL OR m.created_date < (
                    SELECT created_date FROM messages WHERE message_id = :beforeId
                  ))
            ORDER BY m.created_date ASC
            LIMIT :limit
            """;

        return em.createNativeQuery(sql, Message.class)
            .setParameter("channelId", channelId)
            .setParameter("beforeId", beforeMessageId)
            .setParameter("limit", limit)
            .getResultList();
    }

    /**
     * Save a new message to the messages table.
     */
    @Transactional
    public Message save(String channelId, MessageRequest request, String userId, String username) {
        String messageId = UUID.randomUUID().toString();
        LocalDateTime now = LocalDateTime.now();

        em.createNativeQuery("""
            INSERT INTO messages
              (message_id, channel_id, comment, parent_message_id, is_active, created_by, created_date, modified_date)
            VALUES
              (:id, :channelId, :content, :parentId, true, :userId, :now, :now)
            """)
            .setParameter("id", messageId)
            .setParameter("channelId", channelId)
            .setParameter("content", request.getContent())
            .setParameter("parentId", request.getParentMessageId())
            .setParameter("userId", userId)
            .setParameter("now", now)
            .executeUpdate();

        // Build and return the response object
        Message msg = new Message();
        msg.setId(messageId);
        msg.setForumId(channelId);
        msg.setUserId(userId);
        msg.setUsername(username);
        msg.setContent(request.getContent());
        msg.setParentMessageId(request.getParentMessageId());
        msg.setCreatedAt(now.toString());
        msg.setUpdatedAt(now.toString());
        msg.setEdited(false);
        return msg;
    }

    /**
     * Update message content. Only the author can edit.
     */
    @Transactional
    public Message update(String messageId, String newContent, String userId) {
        int updated = em.createNativeQuery("""
            UPDATE messages
            SET comment = :content, modified_date = NOW()
            WHERE message_id = :id AND created_by = :userId AND is_active = true
            """)
            .setParameter("content", newContent)
            .setParameter("id", messageId)
            .setParameter("userId", userId)
            .executeUpdate();

        if (updated == 0) {
            throw new RuntimeException("Message not found or permission denied");
        }

        // Fetch and return updated message
        return findById(messageId);
    }

    /**
     * Soft delete. Returns channelId so the controller can broadcast to the right channel.
     */
    @Transactional
    public String delete(String messageId, String userId, boolean isModerator) {
        // First get the channelId before deleting
        String channelId = (String) em.createNativeQuery(
            "SELECT channel_id FROM messages WHERE message_id = :id AND is_active = true")
            .setParameter("id", messageId)
            .getSingleResult();

        String whereClause = isModerator
            ? "WHERE message_id = :id AND is_active = true"
            : "WHERE message_id = :id AND created_by = :userId AND is_active = true";

        int deleted = em.createNativeQuery(
            "UPDATE messages SET is_active = false " + whereClause)
            .setParameter("id", messageId)
            .setParameter("userId", userId)
            .executeUpdate();

        if (deleted == 0) {
            throw new RuntimeException("Message not found or permission denied");
        }

        return channelId;
    }

    private Message findById(String messageId) {
        return (Message) em.createNativeQuery("""
            SELECT message_id AS id, channel_id AS forumId, created_by AS userId,
                   comment AS content, parent_message_id AS parentMessageId,
                   created_date AS createdAt, modified_date AS updatedAt,
                   (modified_date != created_date) AS isEdited
            FROM messages WHERE message_id = :id
            """, Message.class)
            .setParameter("id", messageId)
            .getSingleResult();
    }
}
```

---

## Step 8: model/Message.java

The message entity with API-friendly field names.

```java
package com.icloudlogic.nirdesh_backend.model;

import com.fasterxml.jackson.annotation.JsonInclude;
import java.util.List;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class Message {

    private String id;
    private String forumId;          // maps from channel_id
    private String userId;           // maps from created_by
    private String username;         // joined from Keycloak or users table
    private String content;          // maps from comment
    private String parentMessageId;  // maps from parent_message_id
    private List<Message> replies;   // child messages (empty for MVP)
    private boolean isEdited;        // computed: modified_date != created_date
    private String createdAt;
    private String updatedAt;

    public Message() {
        this.replies = List.of(); // default empty
    }

    // Getters and setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

    public String getForumId() { return forumId; }
    public void setForumId(String forumId) { this.forumId = forumId; }

    public String getUserId() { return userId; }
    public void setUserId(String userId) { this.userId = userId; }

    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }

    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }

    public String getParentMessageId() { return parentMessageId; }
    public void setParentMessageId(String parentMessageId) { this.parentMessageId = parentMessageId; }

    public List<Message> getReplies() { return replies; }
    public void setReplies(List<Message> replies) { this.replies = replies; }

    public boolean isEdited() { return isEdited; }
    public void setEdited(boolean edited) { isEdited = edited; }

    public String getCreatedAt() { return createdAt; }
    public void setCreatedAt(String createdAt) { this.createdAt = createdAt; }

    public String getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(String updatedAt) { this.updatedAt = updatedAt; }

    // Needed for SSE broadcast — controller needs channelId to broadcast to right channel
    public String getChannelId() { return forumId; }
}
```

---

## Step 9: config/CorsConfig.java

Update your existing `CorsConfig` to add SSE-specific headers.

```java
package com.icloudlogic.nirdesh_backend.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;
import org.springframework.web.filter.CorsFilter;

import java.util.List;

@Configuration
public class CorsConfig {

    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();

        // Allow both frontend apps
        config.setAllowedOrigins(List.of(
            "http://localhost:5173",   // host app (nirdesh_frontend)
            "http://localhost:3001"    // forums micro-frontend
        ));

        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);

        // These headers are required for SSE to work through proxies
        config.setExposedHeaders(List.of(
            "Content-Type",
            "Cache-Control",
            "Connection",
            "X-Accel-Buffering"  // tells Nginx not to buffer SSE responses
        ));

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return new CorsFilter(source);
    }
}
```

---

## Step 10: application.yml / application.properties

Add these settings to prevent Spring from timing out SSE connections:

```yaml
# application.yml
spring:
  mvc:
    async:
      request-timeout: -1  # disable timeout for SSE connections

  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8180/realms/nirdesh

server:
  port: 8080
```

---

## How It All Connects

```
Frontend opens forum
    ↓
GET /api/events/stream  →  SseController.stream()
                        →  SseService.connect(userId)
                        →  stores SseEmitter in memory
                        →  returns open HTTP connection

Frontend opens a channel
    ↓
POST /api/events/subscribe { channelId }
    →  SseService.subscribe(userId, channelId)
    →  adds userId to channelSubscriptions[channelId]

User sends a message
    ↓
POST /api/channels/:id/messages { content }
    →  MessageController.sendMessage()
    →  MessageAccessor.save()  ← writes to messages table
    →  SseService.broadcastNewMessage(channelId, savedMessage)
    →  loops through channelSubscriptions[channelId]
    →  pushes "new_message" event to each SseEmitter
    →  all connected users see the message instantly
```

---

## Testing SSE Manually

You can test the SSE endpoint with curl before connecting the frontend:

```bash
# 1. Get a token from Keycloak
TOKEN=$(curl -s -X POST http://localhost:8180/realms/nirdesh/protocol/openid-connect/token \
  -d "grant_type=password&client_id=nirdesh-host&username=alice&password=password123" \
  | jq -r '.access_token')

# 2. Connect to SSE stream (keep this terminal open)
curl -N -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/events/stream

# 3. In another terminal, subscribe to a channel
curl -X POST http://localhost:8080/api/events/subscribe \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"channelId": "forum-1"}'

# 4. In another terminal, send a message
curl -X POST http://localhost:8080/api/channels/forum-1/messages \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "Hello from curl!"}'

# You should see the event appear in terminal 2 immediately:
# event: new_message
# data: {"id":"...","forumId":"forum-1","content":"Hello from curl!",...}
```

---

## Nginx Config (if behind a proxy)

If you deploy behind Nginx, add this to prevent buffering which breaks SSE:

```nginx
location /api/events/stream {
    proxy_pass http://backend:8080;
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 86400s;  # 24 hours
    proxy_set_header Connection '';
    proxy_http_version 1.1;
    proxy_set_header X-Accel-Buffering no;
}
```
