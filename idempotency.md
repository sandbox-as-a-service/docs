```mermaid
sequenceDiagram
  participant C as Client
  participant S as API
  participant DB as Postgres

  C->>S: POST /vote (Idem-Key=K)
  S->>DB: BEGIN; INSERT vote; COMMIT
  DB-->>S: commit ok
  S--X C: response lost / timeout / server crash after commit

  Note over C: Client thinks it failed, retries…

  C->>S: POST /vote (Idem-Key=K)
  S->>DB: BEGIN; INSERT vote; COMMIT
  DB-->>S: (without guard) ✅ another row inserted
```
