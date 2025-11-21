# Email Delivery Architecture

```mermaid
sequenceDiagram
    autonumber

    participant APP as Application (Creates Email)
    participant MQ as Message Broker
    participant DB as SQL DB (EmailQueue)

    participant ES as Email Sender (Consumer)
        participant S3 as S3 / Blob Storage
    participant Redis as Throttler (Redis + Local Bucket)
    participant Mail as Mail Provider

    %% --- EMAIL CREATION ---
    APP->>DB: INSERT INTO EmailQueue (meta, body/S3Key, status=Pending)
    APP->>MQ: Publish EmailMessage (Id, Meta, Body or S3Key, retryCount=0)

    %% --- MESSAGE ARRIVES TO CONSUMER ---
    MQ-->>ES: Deliver EmailMessage(Id, Meta, Body/S3Key, retryCount)

    %% --- THROTTLE CHECK ---
    ES->>Redis: RequestPermitAsync(clientId)
    Redis-->>ES: Allowed / Throttled

    alt Throttled
        ES->>DB: UpdateEmailStatus(Pending, ScheduledAt = NOW + shortDelay)
        ES->>MQ: Reschedule message (delay = shortDelay, retryCount++)
        ES-->>ES: Stop processing

    else Allowed

        %% --- LOAD EMAIL BODY ---
        alt Inline Body (<=250 KB)
            ES->>ES: body = message.Body
        else Body in S3 (>250 KB)
            ES->>S3: GetObjectAsStringAsync(S3Key)
            S3-->>ES: Email body
        end

        %% --- SEND EMAIL ---
        ES->>Mail: SendEmail(from, to, subject, body)

        %% --- SEND RESULT ---
        alt Success
            ES->>DB: UpdateEmailStatus(Sent)

        else Transient Failure
            ES->>ES: backoff = computeBackoff(retryCount)
            ES->>DB: UpdateEmailStatus(Pending, ScheduledAt = backoff)
            ES->>MQ: Reschedule message (delay = backoff, retryCount++)

        else Permanent Failure
            ES->>DB: UpdateEmailStatus(DeadLetter)
        end

    end

```


## 1. Use Case Diagram — Email Sending + Throttling + Retry Flow
```mermaid
flowchart LR
    User[Application] --> |Insert records| EQ[(SQL DB EmailQueue)]
    User --> | publish message |MQ((Message Broker))

    EmailSender([Email Sender])
    Redis[(Redis Throttling Counters)]
    LocalTB[[Local Token Bucket]]
    S3[(S3 Bucket / Blob Storage)]
    Mail(Mail Provider)

    MQ --> EmailSender
    EmailSender -->|Throttling Check | Redis
    LocalTB --> Redis
    Redis --> LocalTB
    EmailSender -->|If email body is large | S3
    EmailSender -->|Send email | Mail
    EmailSender -->|Update Status | EQ

```

## 2.Sequence Diagram — Full Email Sending Process With Throttling + Retry
```mermaid
sequenceDiagram
    autonumber
    participant MQ as Message Broker
    participant EmailSender as Email Sender
    participant DB as SQL DB (EmailQueue)
    participant S3 as S3 Storage
    participant Redis as Throttler(Redis + Local Bucket)
    participant Mail as Mail Provider

    loop On message arrival
        MQ-->>EmailSender: Deliver EmailMessage (Id, Meta, Body or S3Key, retryCount)

        %% --- THROTTLE CHECK (LOCAL + REDIS) ---
            EmailSender->>Redis: RequestPermitAsync(clientId)
            Redis-->>EmailSender: Allowed / Throttled

        alt Throttled
            EmailSender->>DB: UpdateEmailStatus(Pending, ScheduledAt = NOW + shortDelay)
            EmailSender->>MQ: Reschedule message with delay & retryCount++
            EmailSender-->>EmailSender: Skip processing
        else Allowed

            %% --- BODY LOAD (INLINE or S3) ---
            alt Body inline (<= 250 KB)
                EmailSender->>EmailSender: body = message.Body
            else Body in S3 (> 250 KB)
                EmailSender->>S3: GetObjectAsStringAsync(S3Key)
                S3-->>EmailSender: Email body
            end

            %% --- SEND EMAIL ---
            EmailSender->>Mail: SendEmail(from, to, subject, body)

            %% --- HANDLE SEND RESULT ---
            alt Success
                EmailSender->>DB: UpdateEmailStatus(Sent)
            else Transient Failure
                EmailSender->>EmailSender: backoffTime = computeBackoff(retryCount)
                EmailSender->>DB: UpdateEmailStatus(Pending, ScheduledAt = backoffTime)
                EmailSender->>MQ: Reschedule message with delay = backoffTime & retryCount++
            else Permanent Failure
                EmailSender->>DB: UpdateEmailStatus(DeadLetter)
            end
        end
    end
```

## 3. Sequence Diagram — Throttling Logic Only
```mermaid
sequenceDiagram
    autonumber
    participant Email Sender
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

    participant MQ as Message Broker
    participant Consumer as Email Sender
    participant DB as SQL DB (EmailQueue)

    MQ->>Consumer: Receive EmailMessage

    %% Consumer->>DB: LoadEmailRecord(emailId)
    %% DB-->>Consumer: Email row (status=Pending/Scheduled)

    Consumer->>Consumer: Attempt to Send Email

    alt Transient Failure OR Throttled
        Consumer->>Consumer: backoffTime = computeBackoff(retryCount)

        Consumer->>DB: UpdateEmailStatus(status = Pending, scheduledAt = backoffTime)

        Consumer->>MQ: Reschedule message with delay = backoffTime & retryCount++
    else Permanent Failure
        Consumer->>DB: UpdateEmailStatus(status = Failed)
    else Success
        Consumer->>DB: UpdateEmailStatus(Sent)
    end
```


