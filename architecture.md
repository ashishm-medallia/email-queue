# Email Delivery Architecture

## 1. Use Case Diagram — Email Sending + Throttling + Retry Flow
```mermaid
flowchart LR
    User(User/System Creates Email) --> EQ[(Email Queue Table)]

    EmailSender([Email Sender Background Service])
    Redis[(Redis Throttling Counters)]
    S3[(S3 Bucket / Blob Storage)]
    Mail(Mail Provider)
    DB[(SQL DB)]

    EmailSender -->|PopEmailQueueBatch| DB
    EmailSender -->|Throttling Check| Redis
    EmailSender -->|Fetch Body| S3
    EmailSender -->|Send Email| Mail
    EmailSender -->|UpdateStatus| DB
```

## 2.Sequence Diagram — Full Email Sending Process With Throttling + Retry
```mermaid
sequenceDiagram
    participant DB as SQL Database
    participant EmailSender as Email Sender Worker
    participant Redis as Redis Throttler
    participant S3 as S3 Storage
    participant Mail as SMTP/Mail API

    loop Every few ms
        EmailSender->>DB: PopEmailQueueBatch(batchSize, instanceId)
        DB-->>EmailSender: Email rows
    end

    opt For each email row
        EmailSender->>Redis: RequestPermit(clientId, priority)
        Redis-->>EmailSender: Allowed / Throttled

        alt Throttled
            EmailSender->>DB: UpdateEmailStatus(status=Pending, scheduledAt=Now+10s)
            EmailSender-->>EmailSender: Skip processing
        else Allowed
            EmailSender->>S3: GetObject(bodyKey)
            S3-->>EmailSender: Email body

            EmailSender->>Mail: SendEmail(from, to, subject, body)
            alt Success
                EmailSender->>DB: UpdateEmailStatus(status=Sent)
            else Transient Failure
                EmailSender->>DB: UpdateEmailStatus(status=Pending, scheduledAt=BackoffTime)
            else Permanent Failure
                EmailSender->>DB: UpdateEmailStatus(status=DeadLetter)
            end
        end
    end
```

## 3. Sequence Diagram — Throttling Logic Only
```mermaid
sequenceDiagram
    participant EmailSender
    participant ConfigStore as ThrottleConfigStore (DB)
    participant Redis

    EmailSender->>ConfigStore: Load global config (email/global)
    ConfigStore-->>EmailSender: {LimitPerMin, Weight}

    EmailSender->>ConfigStore: Load client config (email/clientId)
    ConfigStore-->>EmailSender: {LimitPerMin?, Weight?}

    EmailSender->>Redis: Run Lua Script<br/>INCR global + client counters
    Redis-->>EmailSender: 1 (allowed) or 0 (blocked)
```

## 4. Sequence Diagram — Retry / Backoff Logic
```mermaid
sequenceDiagram
    participant Sender as Email Sender
    participant DB as SQL DB

    Sender->>DB: PopEmailQueueBatch()
    DB-->>Sender: Email row with ScheduledAt <= now

    Sender->>Sender: Send attempt

    alt Transient Failure
        Sender->>Sender: computeBackoff(attempt)
        Sender->>DB: UpdateEmailStatus(Pending, scheduledAt = backoffTime)
    else Permanent Failure
        Sender->>DB: UpdateEmailStatus(DeadLetter)
    else Success
        Sender->>DB: UpdateEmailStatus(Sent)
    end
```
