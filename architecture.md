# Email Delivery Architecture

```mermaid
sequenceDiagram
    autonumber

    participant APP as Application (Creates Email)
    participant DB as SQL DB (EmailQueue + ThrottleConfig)
    participant ES as Email Sender Background Service
    participant RT as RedisThrottler
    participant R as Redis (Counters)
    participant S3 as S3 / Blob Storage
    participant Mail as Mail Provider

    %% --- EMAIL CREATION ---
    APP->>DB: INSERT email data INTO plateform.EmailQueue

    %% --- WORKER LOOP ---
    loop Every short interval
        ES->>DB: PopEmailQueueBatch(batchSize, instanceId)
        DB-->>ES: Pending email rows (ScheduledAt ≤ NOW)
    end

    %% --- PROCESS EACH EMAIL ---
    opt For each email row
        ES->>RT: RequestThrottleCheck - RequestPermitAsync(clientId)

        %% --- LOAD CONFIG ---
        RT->>DB: Load ThrottleConfig (global + clientId)
        DB-->>RT: Return ThrottleConfig list

        %% --- FETCH COUNTERS ---
        RT->>R: MGET global + client counters
        R-->>RT: Current counters (or nil)

        %% --- THROTTLE DECISION ---
        RT->>RT: Compute and validate limits
        alt Throttled
            RT-->>ES: PERMIT = false
            ES->>DB: UpdateEmailStatus(Pending, ScheduledAt = NOW + buffer time (may be 10s))
            ES-->>ES: Skip email
        else Allowed
            RT-->>ES: PERMIT = true

            %% --- FETCH EMAIL BODY ---
            ES->>S3: GetObjectAsStringAsync(row.S3Key)
            S3-->>ES: email body

            %% --- SEND EMAIL ---
            ES->>Mail: SendEmail(from, to, subject, body)

            %% --- MAIL RESPONSE HANDLING ---
            alt Success
                ES->>DB: UpdateEmailStatus(Sent)
            else TransientFailure
                ES->>ES: backoff = computeBackoff(attemptCount)
                ES->>DB: UpdateEmailStatus(Pending, ScheduledAt = backoff)
            else PermanentFailure
                ES->>DB: UpdateEmailStatus(DeadLetter)
            end
        end
    end

```


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
    autonumber
    participant DB as SQL Database
    participant EmailSender as Email Sender Background Service
    participant Redis as Redis Throttler
    participant S3 as S3 Storage
    participant Mail as Mail Provider

    loop Every short interval
        EmailSender->>DB: PopEmailQueueBatch(batchSize, instanceId)
        DB-->>EmailSender: Pending email rows (ScheduledAt ≤ NOW)
    end

    opt For each email row
        EmailSender->>Redis: RequestPermit(clientId)
        Redis-->>EmailSender: Allowed / Throttled

        alt Throttled
            EmailSender->>DB: UpdateEmailStatus(status=Pending, scheduledAt=Now+10s)
            EmailSender-->>EmailSender: Skip processing
        else Allowed
            EmailSender->>S3: GetObjectAsStringAsync(row.S3Key)
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
    participant EmailSender
    participant ConfigStore as ThrottleConfigStore (DB)
    participant Redis

    EmailSender->>ConfigStore: Load global config (email/global)
    ConfigStore-->>EmailSender: {LimitPerMin, Weight}

    EmailSender->>ConfigStore: Load client config (email/clientId)
    ConfigStore-->>EmailSender: {LimitPerMin, Weight}

    EmailSender->>Redis: Run Lua Script<br/>INCR global + client counters
    Redis-->>EmailSender: 1 (allowed) or 0 (blocked)
```

## 4. Sequence Diagram — Retry / Backoff Logic
```mermaid
sequenceDiagram
    autonumber

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


