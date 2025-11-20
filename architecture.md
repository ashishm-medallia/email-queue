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
    autonumber

    participant ES as EmailSender
    participant RT as RedisThrottler
    participant TCS as ThrottleConfigStore (DB)
    participant R as Redis

    ES->>RT: RequestThrottleCheck - RequestPermitAsync(clientId, priority)
    
    RT->>TCS: Load throttle config (global + client)
    TCS-->>RT: Return ThrottleConfig list

    RT->>R: Get counters for all keys (global + client)
    R-->>RT: Current counters (or nil)

    Note over RT: Compute finalWeight<br/>priority weight applied here

    RT->>RT: Check counters vs limit<br/>If ANY exceeded → throttle

    RT->>R: Lua Script<br/>INCR key by finalWeight<br/>TTL=60s
    R-->>RT: 0 (throttled) OR 1 (allowed)

    alt throttle exceeded
        RT-->>ES: ThrottleDecision = REJECT<br/>retry_after = 60 sec
    else allowed
        RT->>R: Lua Script<br/>Increment counters atomically<br/>Set TTL = 60 sec
        R-->>RT: OK
        RT-->>ES: ThrottleDecision = ALLOW
    end
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
        Sender->>DB: UpdateEmailStatus(Failed)
    else Success
        Sender->>DB: UpdateEmailStatus(Sent)
    end
```
