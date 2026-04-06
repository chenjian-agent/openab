# RFC 001: Session Management

| Field | Value |
|-------|-------|
| **RFC** | 001 |
| **Title** | Session Management |
| **Author** | @thepagent |
| **Status** | Draft |
| **Tracking** | [#75](https://github.com/thepagent/agent-broker/issues/75), [#78](https://github.com/thepagent/agent-broker/issues/78) |
| **Created** | 2026-04-06 |

---

## Summary

This RFC proposes a comprehensive session management system for agent-broker, covering lifecycle control, isolation, observability, security, and multi-agent support. The design is split into four incremental phases to allow iterative delivery.

## Motivation

Today agent-broker's session handling is minimal:

- `SessionPool` stores connections in a `HashMap<String, AcpConnection>` keyed by Discord thread ID.
- A single `cleanup_idle` loop runs every 60 seconds using a coarse `session_ttl_hours` setting.
- There is no user-facing session control (`/close`), no management API, no metrics, and no per-user limits.
- The pool write lock is held for the entire duration of prompt streaming, causing cross-session deadlocks (#58).

As adoption grows, we need proper lifecycle management, observability, and guardrails.

## Related Issues

| Issue | Description |
|-------|-------------|
| [#38](https://github.com/thepagent/agent-broker/issues/38) | Per-thread isolated working directories |
| [#39](https://github.com/thepagent/agent-broker/issues/39) | Lightweight management API for session observability and lifecycle control |
| [#40](https://github.com/thepagent/agent-broker/issues/40) | `/close` command to manually terminate a thread session |
| [#58](https://github.com/thepagent/agent-broker/issues/58) | Pool write lock held during streaming causes cross-session deadlock |

---

## Design

### 1. Session Lifecycle

#### 1.1 Manual termination — `/close` (#40)

The Discord handler intercepts messages matching `/close`. On match:

1. Call `pool.remove(thread_id)` — drops `AcpConnection`, `kill_on_drop` terminates the child process.
2. Reply "✅ Session closed." in the thread.
3. Archive the Discord thread.

No new structs required; this is a thin command layer on top of the existing pool.

#### 1.2 Idle timeout vs. hard TTL

Split the current single `session_ttl_hours` into two independent timers:

| Config key | Default | Purpose |
|------------|---------|---------|
| `idle_timeout_minutes` | 30 | Reclaim sessions with no activity |
| `session_ttl_hours` | 24 | Hard maximum session lifetime |

When idle timeout fires:
1. Post "⏰ Session expired due to inactivity." in the thread.
2. Remove the session from the pool.

The existing `cleanup_idle` loop (60-second interval) evaluates both timers.

#### 1.3 Per-user session limits

Track active sessions per user:

```rust
user_sessions: RwLock<HashMap<UserId, HashSet<ThreadId>>>
```

New config:

```toml
[pool]
max_sessions_per_user = 3
```

When a user exceeds the limit, reply:
> "You have too many active sessions. Use `/close` in an existing thread to free one."

#### 1.4 Graceful shutdown

On SIGINT/SIGTERM, before clearing the pool:

1. Post "🔄 Broker restarting, this session will end." in each active thread.
2. Allow a short drain period (configurable `shutdown_grace_seconds`, default 5).
3. Drop all connections.

Phase 2 extension: persist `SessionMetadata` to disk or S3 so sessions can be logically resumed after restart.

---

### 2. Session Isolation & Stability

#### 2.1 Per-thread working directories (#38)

Change the working directory passed to `AcpConnection::spawn`:

```
{base_working_dir}/{thread_id}/
```

The directory is created on session start and removed on session cleanup. This gives each session its own filesystem namespace, preventing file collisions between concurrent agents.

#### 2.2 Cross-session deadlock fix (#58)

**Problem:** `with_connection` acquires a pool-level write lock that is held for the entire prompt streaming duration. This blocks all other sessions from calling `get_or_create`.

**Solution:** Two-level locking.

```rust
pub struct SessionPool {
    connections: RwLock<HashMap<String, Arc<Mutex<AcpConnection>>>>,
    metadata: RwLock<HashMap<String, SessionMetadata>>,
    config: AgentConfig,
    max_sessions: usize,
}
```

- Outer `RwLock` protects map insert/remove — acquired briefly, then released.
- Inner `Mutex<AcpConnection>` protects per-session streaming — only blocks the same session.

```rust
pub async fn with_connection<F, R>(&self, thread_id: &str, f: F) -> Result<R>
where
    F: FnOnce(&mut AcpConnection) -> Pin<Box<dyn Future<Output = Result<R>> + Send + '_>>,
{
    let conn = {
        let conns = self.connections.read().await;
        conns.get(thread_id).cloned()
            .ok_or_else(|| anyhow!("no connection for thread {thread_id}"))?
    }; // read lock released here
    let mut guard = conn.lock().await; // per-session lock
    f(&mut guard).await
}
```

---

### 3. Session State

#### 3.1 Session metadata

```rust
pub enum SessionStatus {
    Active,
    Idle,
    Expired,
    Closed,
}

pub struct SessionMetadata {
    pub thread_id: String,
    pub user_id: String,
    pub agent_name: String,
    pub created_at: Instant,
    pub last_active: Instant,
    pub message_count: u64,
    pub status: SessionStatus,
}
```

Stored in `SessionPool::metadata` (see §2.2). Updated on every prompt and lifecycle event.

- **Phase 1:** In-memory only, used for observability and lifecycle decisions.
- **Phase 2:** Serializable to disk/S3 for restart recovery.

#### 3.2 Context window management

Context window limits are primarily the downstream agent's responsibility (kiro-cli, claude, etc.). At the broker level:

- Track `message_count` per session.
- When `message_count` exceeds a configurable threshold, post a warning:
  > "⚠️ This session has a long conversation history. Consider starting a new thread for best results."

Conversation summarization (requiring an extra LLM call) is deferred to Phase 2.

---

### 4. Session Observability (#39)

#### 4.1 Management API

A lightweight HTTP server on a separate port, sharing `Arc<SessionPool>`:

```toml
[management]
enabled = true
port = 9090
```

Endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/sessions` | List all active sessions (metadata array) |
| `GET` | `/sessions/:thread_id` | Single session detail |
| `DELETE` | `/sessions/:thread_id` | Force terminate a session |
| `GET` | `/health` | Broker health + pool stats |
| `GET` | `/metrics` | Prometheus-compatible metrics |

Implementation: `axum` with shared `Arc<SessionPool>` state. Runs in a separate `tokio::spawn` alongside the Discord client.

#### 4.2 Metrics

Exposed via `/metrics` in Prometheus exposition format:

| Metric | Type | Description |
|--------|------|-------------|
| `broker_active_sessions` | Gauge | Current active session count |
| `broker_sessions_created_total` | Counter | Total sessions created |
| `broker_session_duration_seconds` | Histogram | Session lifetime distribution |
| `broker_messages_per_session` | Histogram | Messages per session distribution |
| `broker_pool_exhaustion_total` | Counter | Times pool limit was hit |

#### 4.3 Audit trail

Leverage the existing `tracing` infrastructure. Add structured fields to all session events:

```rust
info!(
    thread_id = %thread_id,
    user_id = %user_id,
    event = "session_created",
    agent = %agent_name,
    "new session"
);
```

Events: `session_created`, `session_prompt`, `session_closed`, `session_expired`, `session_error`.

No additional dependencies required.

---

### 5. Session Security & Access Control

#### 5.1 Session ownership

`SessionMetadata` records `owner_user_id` (the user who triggered session creation).

- `/close` is restricted to the session owner or users with a configurable admin role.
- Other users can still send messages in the thread (Discord threads are inherently public).

#### 5.2 Rate limiting per session

A per-session sliding window rate limiter:

```toml
[pool]
max_messages_per_minute = 10
```

When exceeded, reply:
> "⏳ Rate limited, please wait."

This prevents a single session from monopolizing an agent process.

---

### 6. Multi-agent Support

#### 6.1 Session routing

Extend config from a single `[agent]` to a named `[agents.<name>]` table:

```toml
[agents.kiro]
command = "kiro-cli"
args = ["acp", "--trust-all-tools"]
working_dir = "/home/agent"

[agents.claude]
command = "claude"
args = ["--acp"]
working_dir = "/home/agent"
```

Routing options (can be combined):
- **Per-channel:** map Discord channels to agents in config.
- **Per-command:** user sends `/agent claude` to select.
- **Default:** fallback agent when no explicit selection.

Pool key becomes `(thread_id, agent_name)`.

#### 6.2 Session handoff (Phase 2)

`/handoff <agent>` command:
1. Close current agent connection.
2. Spawn new agent with selected config.
3. Optionally inject a conversation summary into the new agent's first prompt.

---

## Implementation Phases

| Phase | Scope | Issues | Complexity |
|-------|-------|--------|------------|
| **1** | Cross-session deadlock fix, `/close` command, idle timeout notification, session metadata struct | #58, #40 | Low–Medium |
| **2** | Management API, Prometheus metrics, per-user session limits, rate limiting | #39 | Medium |
| **3** | Per-thread working directories, session ownership enforcement, audit trail | #38 | Medium |
| **4** | Multi-agent routing, session persistence/recovery, session handoff | — | High |

Each phase is independently shippable. Later phases build on earlier ones but do not block them.

---

## Open Questions

1. **Management API auth:** Should `/sessions` endpoints require authentication (API key, mTLS, or network-level restriction)?
2. **Session resume:** Should we support resuming a session after agent crash? This requires agent-side support for conversation replay.
3. **Multi-agent routing:** Per-channel config vs. user command vs. both?
4. **Rate limiting scope:** Per-session, per-user, or both?

---

## References

- [ACP Protocol Specification](https://github.com/anthropics/acp)
- Tracking issue: [#75](https://github.com/thepagent/agent-broker/issues/75)
- Discussion issue: [#78](https://github.com/thepagent/agent-broker/issues/78)
