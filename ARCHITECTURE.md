# LazyDrop Architecture

This document describes the internal backend modules and the core data model.

---

## Backend Module Structure

The backend follows a domain-modular structure:

```
com.lazydrop.modules/
├── session/
│   ├── core/          # Session lifecycle management
│   ├── file/          # Upload/download orchestration
│   ├── note/          # Note sharing
│   └── participant/   # Participant management + settings
├── billing/           # Stripe webhook processing
├── subscription/      # Subscription & plan enforcement
├── user/              # User identity resolution
├── storage/           # Supabase storage client (signed URLs)
└── websocket/         # STOMP event broadcasting
```

---

## Data Model (overview)

```mermaid
erDiagram
  users ||--o{ subscriptions : has
  users ||--o{ drop_session : owns
  drop_session ||--o{ drop_session_participants : includes
  drop_session_participants ||--o{ drop_session_note : writes
  drop_session ||--o{ drop_file : contains
  drop_session_participants ||--o{ drop_file_download : downloads
  drop_file ||--o{ drop_file_download : has

  users {
    UUID id PK
    UUID supabase_user_id
    VARCHAR email
    BOOLEAN guest
    VARCHAR guest_id
    TIMESTAMPTZ created_at
  }

  subscriptions {
    UUID id PK
    UUID user_id FK
    VARCHAR stripe_customer_id
    VARCHAR stripe_subscription_id
    VARCHAR plan_code
    VARCHAR status
    TIMESTAMPTZ current_period_end
    BOOLEAN cancel_at_period_end
  }

  drop_session {
    UUID id PK
    UUID owner_id FK
    VARCHAR code
    TIMESTAMPTZ created_at
    TIMESTAMPTZ expires_at
    TIMESTAMPTZ ended_at
    VARCHAR status
    VARCHAR end_reason
  }

  drop_session_participants {
    UUID id PK
    UUID drop_session_id FK
    UUID user_id FK
    VARCHAR role
    TIMESTAMPTZ joined_at
    TIMESTAMPTZ disconnected_at
    BOOLEAN auto_download
  }

  drop_file {
    UUID id PK
    UUID drop_session_id FK
    UUID uploader FK
    VARCHAR storage_path
    VARCHAR original_name
    BIGINT size_bytes
    TIMESTAMPTZ created_at
  }

  drop_file_download {
    UUID id PK
    UUID file_id FK
    UUID participant_id FK
    TIMESTAMPTZ downloaded_at
  }

  stripe_webhook_events {
    UUID id PK
    VARCHAR stripe_event_id
    VARCHAR type
    BOOLEAN livemode
    TIMESTAMPTZ received_at
    TIMESTAMPTZ processed_at
    VARCHAR status
    INTEGER attempt_count
    TIMESTAMPTZ next_retry_at
    TEXT last_error
    TEXT payload
    TEXT sig_header
  }
```

Notes:
- `stripe_webhook_events` supports idempotency + audit + retries
- Unique constraints enforce business invariants (session codes, event ids, one subscription per user)
