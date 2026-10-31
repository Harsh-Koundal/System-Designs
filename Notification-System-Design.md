<img width="6853" height="4636" alt="notification-system-design" src="https://github.com/user-attachments/assets/a6cf0735-abca-4e21-a10e-4a72f71048ce" />

# Notification System Design

## 1. Overview

This design describes a distributed, event-driven notification platform
that delivers notifications through three channels:

-   **Email**
-   **In-app notifications**
-   **Push notifications**

The system separates notification creation from notification delivery.
Applications publish notification events, while independent
channel-specific consumers process and deliver them asynchronously.

The main goals are:

-   **Scalability:** process large notification volumes by adding
    consumers.
-   **Reliability:** retry transient failures and isolate permanently
    failing messages.
-   **Fault isolation:** a failure in email delivery should not stop
    in-app or push delivery.
-   **Duplicate protection:** avoid processing the same event more than
    once where possible.
-   **Provider protection:** rate-limit calls to external notification
    providers.
-   **Observability:** make failed, delayed, and successfully processed
    notifications traceable.

> The diagram uses generic service, broker, queue, worker, database, and
> provider symbols. The exact vendor/product represented by each icon is
> not specified, so this document explains the logical architecture
> rather than assuming a particular cloud provider.

## 2. Architecture at a glance

``` text
Users / Application Services
          |
          v
   API / Event Producers
          |
          v
 Notification Event Publisher
          |
          v
      Message Broker
          |
          +--------------------+---------------------+
          |                    |                     |
          v                    v                     v
      Email Queue          In-App Queue          Push Queue
          |                    |                     |
          v                    v                     v
      Email Worker         In-App Worker          Push Worker
          |                    |                     |
          v                    v                     v
   Email Provider          Notifications DB      Push Provider
          |                                          |
          v                                          v
      Recipient                                Device / Browser

Each channel queue has its own:
- Idempotency / duplicate check
- Rate limiter
- Retry handler with exponential backoff
- Dead Letter Queue (DLQ)
- Maximum retry policy
```

The left side of the supplied diagram represents multiple application
services or event sources. They feed a notification publishing layer and
then a central message broker. The broker routes work to separate queues
for email, in-app, and push delivery.

## 3. Main components

### 3.1 Application services / event producers

These are the business services that generate events requiring a
notification.

Examples:

-   A user registers.
-   A password is reset.
-   An order is placed or shipped.
-   A payment succeeds or fails.
-   A report is generated.
-   An administrator invites a new user.

Business services should publish an event describing what happened. They
should not directly call every notification provider. This keeps
business logic independent from delivery infrastructure.

Example event:

``` json
{
  "eventId": "evt_01JABC123",
  "eventType": "ORDER_PLACED",
  "userId": "user_123",
  "channels": ["EMAIL", "IN_APP", "PUSH"],
  "occurredAt": "2026-10-31T12:00:00Z",
  "data": {
    "orderId": "ORD-1001",
    "amount": 2500,
    "currency": "INR"
  }
}
```

The event should contain identifiers and business data needed to create
the notification. Avoid putting secrets or unnecessary personal
information in broker messages.

### 3.2 Notification API / publisher

The publisher accepts notification requests or business events and
prepares them for asynchronous processing.

Responsibilities:

1.  Validate the event and its requested channels.
2.  Validate the recipient or resolve recipients.
3.  Attach a unique event/notification identifier.
4.  Apply a schema or contract version.
5.  Publish the event to the broker.
6.  Return an acknowledgement once the event has been accepted according
    to the broker's durability guarantees.

For critical business updates, consider the **transactional outbox
pattern**. The business service writes its business change and an outbox
event in the same database transaction. A publisher later moves the
outbox event to the broker. This avoids the dual-write problem where the
database commit succeeds but event publishing fails.

### 3.3 Message broker

The broker decouples event producers from notification consumers.

Its responsibilities include:

-   Buffering events during traffic spikes.
-   Routing events to the appropriate channel queues or topics.
-   Allowing consumers to process messages independently.
-   Providing acknowledgement and redelivery semantics.
-   Supporting horizontal scaling through consumer groups or competing
    consumers.

Possible technologies include:

  -----------------------------------------------------------------------
  Technology                          Typical use
  ----------------------------------- -----------------------------------
  Apache Kafka                        Durable event streams and
                                      high-throughput event distribution

  RabbitMQ                            Queue-based routing and
                                      acknowledgement workflows

  AWS SNS + SQS                       Managed pub/sub fan-out and durable
                                      queues

  Google Pub/Sub                      Managed asynchronous messaging
  -----------------------------------------------------------------------

The diagram resembles a broker with separate channel queues. The exact
broker and routing configuration should be selected based on throughput,
ordering, retention, and operational requirements.

### 3.4 Channel-specific queues

The broker routes notification jobs into three independent queues.

  -----------------------------------------------------------------------
  Queue                   Purpose                 Typical consumer action
  ----------------------- ----------------------- -----------------------
  Email Queue             Deliver email messages  Render template and
                                                  call email provider

  In-App Queue            Create in-product       Persist notification in
                          notifications           the database

  Push Notification Queue Send device/browser     Call push provider such
                          alerts                  as FCM or APNs
  -----------------------------------------------------------------------

**Why separate queues?**

Each channel has different latency, throughput, provider limits, and
failure patterns. If an email provider is down, email messages can
accumulate and retry without blocking in-app or push processing.

Separate queues also allow independent scaling. For example, if email
traffic is much higher than push traffic, more email consumers can be
started without changing the other channels.

### 3.5 Channel workers / consumers

A worker consumes a message from its channel queue and performs the
channel-specific delivery.

#### Email worker

1.  Read the email job.
2.  Resolve recipient and template.
3.  Render the subject and body using event data.
4.  Check idempotency.
5.  Apply provider rate limits.
6.  Call the email provider.
7.  Record the delivery result.
8.  Acknowledge the message only after the required processing succeeds.

#### In-app worker

1.  Read the in-app job.
2.  Check idempotency.
3.  Create a notification record in the database.
4.  Optionally publish a real-time event over WebSocket or Server-Sent
    Events.
5.  Acknowledge the queue message.

The database is the durable source of truth for in-app notifications.
Real-time delivery is a convenience; users should still see the
notification when they reconnect or refresh.

#### Push worker

1.  Read the push job.
2.  Resolve the user's active device tokens.
3.  Check idempotency.
4.  Apply provider rate limits.
5.  Send through a push provider, for example Firebase Cloud Messaging
    (FCM) or Apple Push Notification service (APNs).
6.  Handle invalid or expired device tokens.
7.  Record the result and acknowledge the message when appropriate.

One user may have multiple devices, so a push job can fan out to
multiple device tokens. Track each device delivery separately if
per-device status matters.

## 4. Reliability mechanisms

The diagram shows three important mechanisms around each channel's
processing path: idempotency, rate limiting, and retry handling.

### 4.1 Idempotency / duplicate check

Message brokers and consumers can deliver a message more than once. A
worker may successfully send a notification but crash before
acknowledging the message. The broker can then redeliver it.

Without duplicate protection, the user might receive the same
notification twice.

Use a stable idempotency key, for example:

``` text
notification:{eventId}:{userId}:{channel}
```

Before performing the side effect, the worker checks whether that key
has already been processed.

A typical database design is:

``` sql
CREATE TABLE notification_deliveries (
    id BIGSERIAL PRIMARY KEY,
    event_id VARCHAR(100) NOT NULL,
    user_id VARCHAR(100) NOT NULL,
    channel VARCHAR(20) NOT NULL,
    status VARCHAR(30) NOT NULL,
    provider_message_id VARCHAR(255),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (event_id, user_id, channel)
);
```

The unique constraint prevents duplicate delivery records. In a real
implementation, coordinate the idempotency state transition and provider
call carefully. A database check alone cannot guarantee exactly-once
delivery to an external provider. Use provider-supported idempotency
keys where available, and design for at-least-once message processing.

Suggested statuses:

-   `PENDING`
-   `PROCESSING`
-   `SENT`
-   `DELIVERED` (only when provider delivery confirmation is available)
-   `FAILED`
-   `RETRYING`

Do not treat `SENT` and `DELIVERED` as equivalent. A provider accepting
a request does not necessarily mean the recipient received or viewed it.

### 4.2 Rate limiter (Token Bucket)

A rate limiter controls how quickly workers call a downstream provider.

For example, an email provider may allow only a certain number of
requests per second. Sending too quickly can result in throttling or
temporary blocks.

The diagram specifies a **Token Bucket** algorithm.

How it works:

1.  A bucket has a maximum token capacity.
2.  Tokens are replenished at a configured rate.
3.  Each outgoing request consumes a token.
4.  If no token is available, the worker waits or reschedules the job.

Example configuration:

``` text
Bucket capacity: 100 tokens
Refill rate:     20 tokens / second
Request cost:    1 token
```

Use a shared/distributed limiter if multiple worker instances must
respect one provider-wide quota. A per-process limiter alone can exceed
the provider limit when the number of instances increases.

Rate-limit independently by provider, channel, tenant, or recipient
where appropriate.

### 4.3 Retry handler -- Exponential backoff

Not every delivery failure is permanent.

Temporary failures include:

-   Network timeout.
-   Provider unavailable.
-   HTTP 429 (rate limited).
-   HTTP 500 or 503.
-   Temporary DNS or connection errors.

The retry handler schedules another attempt after an increasing delay.

A common exponential backoff formula is:

``` text
delay = min(maxDelay, baseDelay * 2^attempt)
```

Add random jitter to avoid many workers retrying simultaneously:

``` text
actualDelay = random(0, delay)
```

Example retry schedule (illustrative):

  Attempt     Approximate delay
  --------- -------------------
  1                   2 seconds
  2                   4 seconds
  3                   8 seconds
  4                  16 seconds
  5                  32 seconds

Respect provider `Retry-After` headers when present. Retry only
transient failures. Invalid email addresses, malformed payloads, or
permanently invalid device tokens should normally be marked as permanent
failures rather than retried indefinitely.

### 4.4 Maximum retry policy

The diagram shows a **Max Retry** stage associated with each channel's
DLQ.

Configure a maximum attempt count or maximum message age. When the
policy is exhausted, stop retrying and route the message to that
channel's DLQ.

Example:

``` text
Maximum attempts: 5
Maximum message age: 24 hours
```

These values are examples, not requirements. Choose them according to
notification urgency and provider behavior.

### 4.5 Dead Letter Queue (DLQ)

A DLQ stores messages that could not be processed successfully after the
retry policy is exhausted, or that failed permanently.

There should be a separate DLQ for each channel:

-   Email DLQ
-   In-App DLQ
-   Push DLQ

This separation makes troubleshooting easier. A broken email template
should not mix with invalid push-token failures.

Each DLQ message should retain:

-   Original event or notification ID.
-   Channel and recipient reference.
-   Failure reason and error category.
-   Number of attempts.
-   First and last failure timestamps.
-   Correlation/trace ID.
-   Relevant provider response metadata, with secrets and sensitive data
    removed.

DLQ messages should be inspectable and replayable through a controlled
operational process. Fix the underlying issue before replaying.
Replaying should preserve the original idempotency key unless
intentionally creating a new notification.

## 5. End-to-end processing flows

### 5.1 Successful email delivery

``` text
Application
    |
    v
Publish notification event
    |
    v
Message Broker
    |
    v
Email Queue
    |
    v
Email Worker
    |
    v
Idempotency check
    |
    v
Rate limiter
    |
    v
Render email template
    |
    v
Email Provider
    |
    v
Persist result / acknowledge message
```

If the provider accepts the request, record the provider response and
mark the attempt as sent. Update to delivered only if a delivery webhook
or equivalent confirmation is received.

### 5.2 Failed email delivery

``` text
Email Queue
    |
    v
Email Worker
    |
    v
Provider request fails
    |
    v
Retry Handler
    |
    +---- Retry budget remains? ---- Yes
    |                                |
    |                                v
    |                         Wait with backoff
    |                                |
    |                                v
    |                           Retry queue
    |                                |
    |                                v
    |                           Email Worker
    |
    No
    |
    v
Email DLQ
    |
    v
Alert / inspect / fix / controlled replay
```

The same pattern applies independently to in-app and push notifications.

### 5.3 In-app notification delivery

The in-app worker writes the notification into persistent storage.

Example table:

``` sql
CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(100) NOT NULL,
    event_id VARCHAR(100) NOT NULL,
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    type VARCHAR(50),
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (user_id, event_id)
);
```

The frontend can retrieve notifications through an API:

``` http
GET /api/notifications
GET /api/notifications/unread-count
PATCH /api/notifications/:id/read
PATCH /api/notifications/read-all
```

For real-time updates, the backend can additionally notify connected
clients through WebSockets or Server-Sent Events. The database remains
the durable record.

### 5.4 Push notification delivery

The push worker sends a message to the provider using a registered
device token.

Example logical payload:

``` json
{
  "token": "device-token",
  "notification": {
    "title": "Order confirmed",
    "body": "Your order ORD-1001 has been confirmed."
  },
  "data": {
    "eventId": "evt_01JABC123",
    "orderId": "ORD-1001"
  }
}
```

If a device token is permanently invalid, deactivate or remove it. Do
not keep retrying an invalid token.

## 6. Suggested database model

A production implementation can use the following logical entities.

### Notifications

Stores the user-facing notification, particularly for the in-app
channel.

  Field        Description
  ------------ ------------------------
  id           Unique notification ID
  user_id      Recipient
  event_id     Source event
  title        Notification title
  body         Notification content
  type         Notification category
  is_read      Read state
  created_at   Creation timestamp

### Notification deliveries

Tracks each channel-specific delivery attempt or final state.

  Field                 Description
  --------------------- --------------------------
  id                    Unique delivery ID
  event_id              Source event
  user_id               Recipient
  channel               EMAIL / IN_APP / PUSH
  status                Current delivery state
  attempt_count         Number of attempts
  provider_message_id   Provider's reference
  last_error            Sanitized failure reason
  next_retry_at         Scheduled retry time
  created_at            Creation timestamp
  updated_at            Last update

Use a unique constraint on the logical delivery identity, such as
`(event_id, user_id, channel)`, when that matches the product's
deduplication rules.

### Device tokens

Stores push registration information.

  Field        Description
  ------------ -----------------------------
  id           Device registration ID
  user_id      Owner
  token        Push token (store securely)
  platform     Android / iOS / Web
  is_active    Whether token can be used
  updated_at   Last registration update

## 7. Scaling strategy

The architecture supports horizontal scaling.

### Queue-based scaling

If queue lag increases, add more consumers for that channel.

For example:

``` text
Email Queue
    |
    +---- Email Worker 1
    +---- Email Worker 2
    +---- Email Worker 3
    +---- Email Worker 4
```

Scale email, in-app, and push workers independently.

### Partitioning and ordering

If using Kafka, partition by a stable key such as `userId` when per-user
ordering is required. More partitions can increase parallelism, but
ordering is generally guaranteed only within a partition.

If using RabbitMQ or cloud queues, select queue and consumer settings
that match the required ordering and acknowledgement semantics.

### Backpressure

Monitor queue depth and consumer lag. When downstream providers slow
down, queues buffer work and consumers should reduce their request rate
rather than overwhelming the provider.

## 8. Failure scenarios and expected behavior

  -----------------------------------------------------------------------
  Failure                             Expected behavior
  ----------------------------------- -----------------------------------
  Email provider unavailable          Retry email jobs; do not block
                                      other channels

  Push provider throttles requests    Respect rate limit and retry later

  Worker crashes before               Broker may redeliver; idempotency
  acknowledgement                     prevents duplicate side effects
                                      where possible

  Invalid email address               Mark permanent failure and route to
                                      DLQ or failure state

  Invalid device token                Deactivate token; do not retry
                                      permanently invalid token

  In-app database unavailable         Retry the in-app job; eventually
                                      send to its DLQ

  Broker temporarily unavailable      Publisher retries; use an outbox
                                      for important events

  Malformed event                     Reject or quarantine; record enough
                                      metadata for diagnosis

  Queue backlog grows                 Scale consumers and investigate
                                      provider throughput
  -----------------------------------------------------------------------

## 9. Monitoring and observability

Track each channel independently.

Recommended metrics:

-   Notifications published per minute.
-   Queue depth and oldest-message age.
-   Consumer throughput.
-   Queue processing latency.
-   Success and failure rates by channel.
-   Retry count and retry exhaustion rate.
-   DLQ message count and age.
-   Provider latency and HTTP error codes.
-   Rate-limiter wait time.
-   Duplicate/idempotency hits.
-   In-app unread notification count.

Use structured logs with `eventId`, `notificationId`, `userId` (or a
safe pseudonymous identifier), `channel`, `attempt`, and `traceId`.

Create alerts for sustained queue growth, high provider failure rates,
DLQ growth, and unusually old pending notifications. Avoid logging raw
email content, device tokens, credentials, or other sensitive data.

## 10. Security considerations

-   Authenticate and authorize notification API requests.
-   Validate event schemas and allowed channel names.
-   Do not trust arbitrary recipient addresses or device tokens supplied
    by untrusted clients.
-   Store provider credentials in a secrets manager or protected
    environment configuration.
-   Encrypt sensitive data in transit and at rest.
-   Apply tenant-level access controls to notification history.
-   Sanitize provider errors before storing or logging them.
-   Enforce retention and deletion policies for notification data.
-   Protect replay and DLQ operations with role-based access and audit
    logs.

## 11. Important design trade-offs

### At-least-once vs. exactly-once delivery

Most practical queue-based systems provide at-least-once processing. A
message can be delivered more than once, so consumers must be
idempotent.

Exactly-once delivery to a human recipient cannot generally be
guaranteed: a provider may accept a message while the worker times out
before receiving the response. Use idempotency support from the provider
where possible, persist delivery state, and make duplicate effects
unlikely.

### Independent queues vs. one shared queue

Independent channel queues improve fault isolation and allow independent
scaling. They add operational complexity because each queue needs
monitoring, retry policies, and DLQ handling.

### Fan-out strategy

If one event requests all three channels, the publisher or broker
routing layer must create one logical delivery job per requested
channel. Track those jobs independently so one channel's success does
not incorrectly mark all channels successful.

## 12. Recommended implementation sequence

1.  Define the notification event schema and channel contracts.
2.  Implement the notification publisher and broker integration.
3.  Create separate queues for email, in-app, and push.
4.  Implement one worker per channel.
5.  Add delivery persistence and idempotency.
6.  Add provider-specific rate limiting.
7.  Add exponential backoff and maximum retry policies.
8.  Configure a DLQ for every channel.
9.  Implement in-app notification APIs and read/unread state.
10. Add push device-token registration and cleanup.
11. Add metrics, tracing, dashboards, and alerts.
12. Test provider outages, worker crashes, duplicate messages,
    throttling, and DLQ replay.

## 13. Summary

This design uses asynchronous messaging to separate business events from
notification delivery. A central broker distributes work into
independent email, in-app, and push queues. Each channel has its own
consumer and reliability controls: idempotency, rate limiting,
exponential-backoff retries, maximum retry limits, and a DLQ.

The result is a notification platform that can scale by channel,
tolerate provider outages, isolate failures, and provide operational
visibility into delivery outcomes.

**Key principle:** accept notification work quickly, process it
asynchronously, make consumers idempotent, and never allow one delivery
channel's failure to block the others.
