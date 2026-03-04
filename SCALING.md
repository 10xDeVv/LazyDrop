# LazyDrop Scaling Guide

This document maps out concrete scaling strategies for LazyDrop — from the current single-node architecture to handling 10K+ concurrent sessions. Each section identifies the **current bottleneck**, the **breaking point**, and the **specific change** required.

---

## Current Architecture (Single-Node Baseline)

```
Vercel CDN ──► Next.js Frontend (Edge/Serverless)
                    │
                    ▼
             DigitalOcean Droplet
             ┌──────────────────────────┐
             │  Spring Boot (single JVM)│
             │  ├── REST API            │
             │  ├── STOMP WebSockets    │
             │  ├── Scheduled Jobs      │
             │  └── File Proxy Upload   │
             └──────────┬───────────────┘
                        │
               ┌────────┴────────┐
               ▼                 ▼
          PostgreSQL 16     DO Spaces (S3)
          (single node)     (CDN-backed)
```

**Capacity estimate (single 2vCPU/4GB droplet):**
- ~200–500 concurrent REST connections
- ~500–1,000 concurrent WebSocket connections (SockJS/STOMP, in-memory SimpleBroker)
- ~50–100 concurrent file proxy uploads (the JVM holds the full stream in memory)
- Hikari pool: 5 connections → ~50–100 concurrent DB-hitting requests before queueing

**This is fine for:** early users, portfolio demos, and ~100 concurrent sessions.

---

## Phase 1: Vertical Scaling + Quick Wins (100 → 1,000 users)

### 1.1 Increase Droplet Size
Cheapest first move. Go from 2vCPU/4GB → 4vCPU/8GB.

**Changes:**
- `application.yml` → increase Hikari pool:
  ```yaml
  hikari:
    maximum-pool-size: 20
    minimum-idle: 5
  ```
- `Dockerfile` → relax JVM memory:
  ```
  -XX:MaxRAMPercentage=70.0
  ```

### 1.2 ~~Stop Proxying File Uploads Through the Backend~~ ✅ DONE
The proxy upload endpoint has been removed. All file uploads now use the **two-phase signed URL flow**:
1. Frontend requests a signed upload URL from the backend (`POST /sessions/{id}/files/upload-url`)
2. Frontend PUTs the file directly to DO Spaces using the signed URL (zero backend bytes)
3. Frontend confirms the upload with the backend (`POST /sessions/{id}/files/confirm`)

The backend never touches file bytes — only tiny JSON metadata requests. The multipart limit has been reduced from 2048MB to 10MB accordingly.

### 1.3 Add Database Indexes
Ensure these exist (check Flyway migrations):
```sql
CREATE INDEX IF NOT EXISTS idx_drop_session_status_expires 
  ON drop_session(status, expires_at) WHERE status IN ('OPEN', 'CONNECTED');

CREATE INDEX IF NOT EXISTS idx_drop_session_owner 
  ON drop_session(owner_id);

CREATE INDEX IF NOT EXISTS idx_drop_file_session 
  ON drop_file(drop_session_id);

CREATE INDEX IF NOT EXISTS idx_participant_session 
  ON drop_session_participants(drop_session_id);
```

### 1.4 Add Connection Pooling with PgBouncer
Put PgBouncer in front of PostgreSQL. Each backend instance can open 20 Hikari connections, but PostgreSQL handles max ~100 connections well. PgBouncer multiplexes.

```yaml
# docker-compose.yml addition
pgbouncer:
  image: edoburu/pgbouncer
  environment:
    DATABASE_URL: postgresql://lazydrop:lazydrop@db:5432/lazydrop
    POOL_MODE: transaction
    MAX_CLIENT_CONN: 200
    DEFAULT_POOL_SIZE: 20
```

---

## Phase 2: Horizontal API Scaling (1,000 → 5,000 users)

### 2.1 The Problem: WebSockets + Multiple Instances
The REST API is already stateless — you can run 3 instances behind a load balancer today. **But WebSockets break.**

Current: `SimpleBroker` is in-memory. If User A connects to Instance 1 and User B connects to Instance 2, they can't see each other's events even though they're in the same session.

### 2.2 The Fix: External Message Broker (Redis or RabbitMQ)

Replace the in-memory `SimpleBroker` with a shared broker.

**Option A: Redis Pub/Sub (simplest)**
Spring doesn't natively support Redis as a STOMP broker, but you can:
1. Keep the `SimpleBroker` for local delivery.
2. Intercept `WebSocketNotifier.sendEventAfterCommit()` to also publish to a Redis channel.
3. Each instance subscribes to the Redis channel and forwards to its local WebSocket clients.

```java
// WebSocketNotifier.java — add Redis fan-out
@Component
@RequiredArgsConstructor
public class WebSocketNotifier {
    private final SimpMessagingTemplate messagingTemplate;
    private final RedisTemplate<String, String> redisTemplate;

    public <T> void sendEventAfterCommit(String sessionId, MessageType type, T payload) {
        WebSocketMessage<T> msg = new WebSocketMessage<>(type, payload);
        // Publish to Redis — all instances receive it
        redisTemplate.convertAndSend("ws:session:" + sessionId, serialize(msg));
    }
}

// RedisWebSocketRelay.java — subscribes and delivers locally
@Component
public class RedisWebSocketRelay implements MessageListener {
    private final SimpMessagingTemplate messagingTemplate;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        // Deserialize and forward to local STOMP subscribers
        String channel = new String(message.getChannel()); // "ws:session:{id}"
        String sessionId = channel.substring("ws:session:".length());
        messagingTemplate.convertAndSend("/topic/session/" + sessionId, deserialize(message));
    }
}
```

**Option B: RabbitMQ STOMP Broker (production-grade)**
Spring WebSocket has native support:
```java
@Override
public void configureMessageBroker(MessageBrokerRegistry registry) {
    registry.enableStompBrokerRelay("/topic", "/queue")
            .setRelayHost("rabbitmq")
            .setRelayPort(61613);
}
```
All instances relay through RabbitMQ. No custom code needed.

### 2.3 Load Balancer with Sticky Sessions
WebSocket upgrades must hit the same backend instance for the life of the connection.

```nginx
# nginx.conf
upstream backend {
    ip_hash;  # sticky sessions
    server backend-1:8080;
    server backend-2:8080;
    server backend-3:8080;
}

server {
    location /ws {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    location / {
        proxy_pass http://backend;
    }
}
```

If using DigitalOcean App Platform or a managed LB, enable session affinity (cookie-based or IP-hash).

### 2.4 Scaled Architecture Diagram

```
                    Vercel CDN
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
       Instance 1   Instance 2   Instance 3
       (Spring Boot) (Spring Boot) (Spring Boot)
           │            │            │
           └─────┬──────┴──────┬─────┘
                 ▼             ▼
           Redis/RabbitMQ   PgBouncer
           (WS fan-out)        │
                            PostgreSQL
                         DO Spaces (S3)
```

### 2.5 Separate Scheduled Jobs
With multiple instances, `@Scheduled` jobs (session expiry, upload cleanup) would run on **every** instance simultaneously.

**Fix options:**
1. **ShedLock** (simplest): distributed lock using a DB table. Only one instance executes the job.
   ```java
   @SchedulerLock(name = "cleanUpExpiredSessions", lockAtLeastFor = "PT1M", lockAtMostFor = "PT5M")
   @Scheduled(fixedRate = 60000)
   public void cleanUpExpiredSession() { ... }
   ```
2. **Dedicated worker instance**: one backend instance runs with `scheduling.enabled=true`, others with `false`.
3. **External job scheduler**: use a cron service (e.g., DigitalOcean Functions, GitHub Actions scheduled workflow).

---

## Phase 3: Database Scaling (5,000 → 50,000 users)

### 3.1 Read Replicas
Most LazyDrop queries are reads (get session, list files, list participants). PostgreSQL streaming replication gives you read replicas.

**Implementation:**
- Use two DataSources in Spring: a primary (writes) and a replica (reads).
- Route `@Transactional(readOnly = true)` queries to the replica.

```java
@Configuration
public class DataSourceConfig {
    @Bean @Primary
    public DataSource writeDataSource() { /* primary */ }

    @Bean
    public DataSource readDataSource() { /* replica */ }
}
```

Many methods already have `@Transactional(readOnly = true)` — those automatically route to the replica.

### 3.2 Partition/Archive Expired Sessions
The `drop_session` and `drop_file` tables grow indefinitely. Expired sessions are never queried again.

**Strategy:**
- Add a scheduled job that moves sessions with `status = EXPIRED` or `ENDED` (older than 7 days) to archive tables.
- Or use PostgreSQL table partitioning by `created_at` month.

### 3.3 Connection Pooling at Scale
With 5+ backend instances × 20 Hikari connections = 100+ DB connections. PostgreSQL starts to struggle at ~200.

PgBouncer in `transaction` mode is essential at this point. Each backend connects to PgBouncer, which multiplexes to a smaller PostgreSQL pool.

---

## Phase 4: CDN + Edge Optimization

### 4.1 ~~DO Spaces CDN for Downloads~~ ✅ DONE
Download signed URLs now go through the DO Spaces CDN edge network. A dedicated `cdnPresigner` bean generates
download URLs pointing to the CDN endpoint, while uploads and deletes still hit the origin.

```
Upload:   Browser → https://tor1.digitaloceanspaces.com/lazydrop-bucket/...  (origin — writes)
Download: Browser → https://lazydrop-bucket.tor1.cdn.digitaloceanspaces.com/...  (CDN — reads, edge-cached)
```

Configuration: set `SPACES_CDN_ENDPOINT` in `.env`. Falls back to origin if not set.

### 4.2 Frontend: Vercel Edge Middleware
Vercel already handles this, but ensure:
- Static assets are cached at the edge.
- API routes that proxy to the backend use Vercel's `rewrites` in `next.config.mjs` to keep the backend URL private.

---

## Phase 5: Advanced (50,000+ concurrent users)

### 5.1 Managed Database
Move from self-hosted PostgreSQL to DigitalOcean Managed Database. Benefits:
- Automatic failover
- Built-in connection pooling
- Automated backups
- Read replicas with a toggle

### 5.2 Container Orchestration (Kubernetes)
Move from individual droplets to DigitalOcean Kubernetes (DOKS):
- Auto-scaling based on CPU/WebSocket connection count
- Rolling deployments with zero downtime
- Health check + automatic restart

### 5.3 Dedicated WebSocket Service
Split the monolith:
```
┌─────────────┐   ┌──────────────────┐
│ REST API    │   │ WebSocket Service │
│ (stateless) │   │ (sticky sessions) │
└──────┬──────┘   └────────┬─────────┘
       │                   │
       └─────┬─────────────┘
             ▼
         Redis Pub/Sub
             │
         PostgreSQL
```

The REST API scales horizontally without sticky sessions. The WebSocket service scales separately with its own autoscaler tuned for connection count rather than CPU.

### 5.4 Rate Limiting
Add API rate limiting (currently absent):
```java
// Per-IP: 100 requests/minute for unauthenticated
// Per-user: 300 requests/minute for authenticated
// Per-session: 10 file uploads/minute
```

Use Spring's `Bucket4j` + Redis for distributed rate limiting.

### 5.5 Observability
Before scaling further, you need visibility:
- **Metrics**: Micrometer + Prometheus + Grafana (request latency, WS connection count, DB pool utilization)
- **Tracing**: OpenTelemetry for distributed traces
- **Alerting**: PagerDuty/Slack on error rate spikes, pool exhaustion, or WS connection drops

---

## Scaling Decision Matrix

| Metric | Current Limit | Bottleneck | Fix | Phase |
|--------|--------------|------------|-----|-------|
| Concurrent REST requests | ~200 | Hikari pool (5 conns) | Increase pool + vertical scale | 1 |
| Concurrent uploads | ∞ (client-side) | ~~JVM proxies file bytes~~ | ✅ Signed-URL-only (done) | ✅ |
| Concurrent WebSockets | ~1,000 | JVM memory (single node) | Vertical scale, then horizontal + Redis | 1→2 |
| Multi-instance WS | ❌ | In-memory SimpleBroker | Redis Pub/Sub or RabbitMQ relay | 2 |
| DB connections | ~100 | PostgreSQL max_connections | PgBouncer | 1→2 |
| Scheduled job conflicts | ❌ | Multiple instances run same job | ShedLock | 2 |
| Read-heavy queries | ~5K QPS | Single DB instance | Read replicas | 3 |
| Table growth | Unbounded | No archival | Partition + archive expired sessions | 3 |
| Download bandwidth | Edge-cached | ~~S3 origin~~ | ✅ DO Spaces CDN (done) | ✅ |
| Auto-scaling | Manual | Fixed droplet count | Kubernetes + HPA | 5 |

---

## Recommended Order of Operations

```
Now (single node, <500 users):
  ✅ You are here

Next week:
  1. ✅ Remove/deprecate the proxy upload endpoint (DONE)
  2. Add missing DB indexes
  3. Increase Hikari pool to 15-20

When you hit ~1,000 concurrent users:
  4. Add Redis (for WS fan-out + rate limiting + ShedLock)
  5. Run 2-3 backend instances behind a load balancer
  6. Add PgBouncer

When you hit ~5,000 concurrent users:
  7. Move to managed PostgreSQL with read replica
  8. Archive expired sessions
  9. Add observability stack

When you hit ~50,000 concurrent users:
  10. Move to Kubernetes
  11. Split WebSocket into its own service
  12. Dedicated worker nodes for scheduled jobs
```

---

## Cost Estimates (DigitalOcean)

| Phase | Infrastructure | Monthly Cost |
|-------|---------------|-------------|
| Current | 1 droplet (2vCPU/4GB) + Spaces + Supabase free | ~$24 |
| Phase 1 | 1 droplet (4vCPU/8GB) + Spaces | ~$48 |
| Phase 2 | 3 droplets + Redis + PgBouncer + LB | ~$120–180 |
| Phase 3 | Managed DB + read replica + 3 app nodes | ~$250–400 |
| Phase 5 | DOKS cluster + managed DB + Redis | ~$500+ |

The key insight: **Phase 1 changes (signed-URL-only uploads + bigger pool) cost $0 in infrastructure and probably 5x your capacity.**
