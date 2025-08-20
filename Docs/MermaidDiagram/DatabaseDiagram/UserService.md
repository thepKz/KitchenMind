
# Auth Service (Đăng nhập, OAuth, Session)
```mermaid
erDiagram
  USERS {
    uuid id PK
    varchar email "unique, not null"
    varchar password_hash
    varchar full_name
    varchar phone
    text avatar_url
    boolean email_verified
    timestamp created_at
    timestamp updated_at
    boolean is_active
  }

  OAUTH_PROVIDERS {
    uuid id PK
    uuid user_id FK
    varchar provider
    varchar provider_id
    text access_token
    text refresh_token
    timestamp expires_at
    timestamp created_at
  }

  USER_SESSIONS {
    uuid id PK
    uuid user_id FK
    varchar token_hash
    jsonb device_info
    timestamp last_used
    timestamp created_at
    timestamp expires_at
  }

  USERS ||--o{ OAUTH_PROVIDERS : "linked"
  USERS ||--o{ USER_SESSIONS : "has"
  ```
