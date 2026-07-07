---
title: Rust - Distributed Messaging and Asynchronous Communication
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Distributed Messaging and Asynchronous Communication

Covers the crates and conventions for **inter-process / inter-service** messaging in Rust: brokers (Kafka, RabbitMQ, NATS, SQS/SNS), MQTT for IoT, in-process async channels, message serialization, and the reliability patterns (idempotency, retry/backoff, dead-letter, outbox) the ecosystem has settled on.

> The async messaging world is **firmly tokio-based**. Almost every broker client is `tokio`-native; mixing in another runtime (async-std, smol) is possible but uncommon. Pick tokio unless you have a specific reason not to.

## 1. Broker clients — the recognized options

### Kafka

| Crate | Notes |
|---|---|
| [`rdkafka`](https://crates.io/crates/rdkafka) (a.k.a. `rust-rdkafka`) | The de facto Kafka client. Wraps **librdkafka** (C). Mature, feature-complete, the choice of most production deployments. Async bindings on top of librdkafka's background threads. |
| [`kafka`](https://crates.io/crates/kafka) | Pure-Rust, but **less actively maintained**; only suitable for simple workloads. |

```rust
use rdkafka::config::ClientConfig;
use rdkafka::producer::{FutureProducer, FutureRecord};

let producer: FutureProducer = ClientConfig::new()
    .set("bootstrap.servers", "broker1:9092")
    .set("message.timeout.ms", "5000")
    .create()?;

producer.send(
    FutureRecord::to("orders")
        .payload(&serde_json::to_string(&order)?)
        .key(&order.id.to_string()),
    Timeout::After(Duration::from_secs(5)),
).await?;
```

### RabbitMQ / AMQP

| Crate | Notes |
|---|---|
| [`lapin`](https://crates.io/crates/lapin) | Pure-Rust AMQP 0.9.1 client, async on tokio. The recommended choice. |
| [`amiquip`](https://crates.io/crates/amiquip) | Synchronous AMQP 0.9.1; simpler but no longer actively developed. |
| [`amqp`](https://crates.io/crates/amqp) | Older; avoid for new projects. |

### NATS

| Crate | Notes |
|---|---|
| [`async-nats`](https://crates.io/crates/async-nats) | Official async client from the Synadia / nats.io team. The only choice for serious NATS in Rust. |

```rust
let nats = async_nats::connect("nats://localhost:4222").await?;
nats.publish("orders.created", serde_json::to_vec(&order)?.into()).await?;
```

### AWS SQS / SNS

| Crate | Notes |
|---|---|
| [`aws-sdk-sqs`](https://crates.io/crates/aws-sdk-sqs) | Official AWS SDK. |
| [`aws-sdk-sns`](https://crates.io/crates/aws-sdk-sns) | Official AWS SDK. |

### MQTT (IoT)

| Crate | Notes |
|---|---|
| [`rumqttc`](https://crates.io/crates/rumqttc) | Pure-Rust MQTT 3.1.1/5 client; the recommended choice. |
| [`paho-mqtt`](https://crates.io/crates/paho-mqtt) | Wraps the Eclipse Paho C library; useful when you need Paho's specific feature set. |

### Redis (Pub/Sub + Streams)

The [`redis`](https://crates.io/crates/redis) crate (and `redis::aio`) supports Pub/Sub and Streams; popular for lightweight messaging when Kafka/NATS are overkill.

## 2. Message serialization

Use [`serde`](https://crates.io/crates/serde) with the appropriate format adapter (cross-reference Section 23):

| Format | Crate | When |
|---|---|---|
| JSON | `serde_json` | Default for web-facing services; human-readable. |
| Avro | `apache-avro` | Schema-registry-driven streaming (Kafka). |
| Protocol Buffers | `prost` | Strongly-typed, compact, multi-language. |
| Cap'n Proto | `capnp` | Zero-copy, very fast. |
| MessagePack | `rmp-serde` | Compact JSON replacement. |
| bincode | `bincode` | Rust-to-Rust only; very compact. |

**Schema evolution rules** (Avro/Protobuf): always make fields optional, never reuse a field number/tag, never change a type incompatible — these are the same rules as in any language; document them in your `proto`/`.avsc` files.

## 3. Delivery semantics and idempotency

**Assume at-least-once delivery.** Most brokers (Kafka, RabbitMQ, SQS, NATS with jetstream) redeliver on consumer failure or network blip. Design consumers to be **idempotent**.

### ✅ Idempotent consumer with a stable deduplication key

```rust
async fn handle(msg: Message) -> Result<(), Error> {
    let dedup = format!("{}:{}", msg.event_type, msg.id);

    // 1. Have we already processed this?
    if processed_store.contains(&dedup).await? {
        msg.ack().await?;       // already done — ack and move on
        return Ok(());
    }

    // 2. Apply the side effect
    apply(&msg).await?;

    // 3. Record the dedup key in the SAME transaction as the side effect,
    //    or in the outbox (§6) — never as a separate step.
    processed_store.insert(&dedup).await?;

    // 4. Acknowledge only after (2) and (3) persist.
    msg.ack().await?;
    Ok(())
}
```

### ❌ Acknowledging before the work is durable

```rust
// ❌ If the process dies between ack and apply, the message is lost.
msg.ack().await?;
apply(&msg).await?;
```

## 4. Retry and backoff

| Crate | Notes |
|---|---|
| [`backoff`](https://crates.io/crates/backoff) | Exponential backoff with jitter, the de facto crate. Exposes both a synchronous and an async API. |
| [`tokio-retry`](https://crates.io/crates/tokio-retry) | tokio-native retry with strategies (exponential, fixed, custom). |

```rust
use backoff::{ExponentialBackoffBuilder, future::retry};

let bo = ExponentialBackoffBuilder::new()
    .with_initial_interval(Duration::from_millis(100))
    .with_max_interval(Duration::from_secs(10))
    .with_max_elapsed_time(Some(Duration::from_secs(60)))
    .build();

retry(bo, || async {
    match service.call().await {
        Ok(v) => Ok(v),
        // permanent errors stop retrying:
        Err(ServiceError::Invalid) => Err(backoff::Error::permanent(ServiceError::Invalid)),
        // transient errors retry:
        Err(e @ ServiceError::Timeout) => Err(backoff::Error::transient(e)),
    }
}).await?;
```

**Conventions:**

- Always **jitter** backoff to avoid correlated retry storms.
- Set a **max elapsed time** so poison messages do not retry forever.
- Combine with a **dead-letter queue** (next section) for messages that exhaust their retries.

## 5. Dead-letter queues (DLQ)

Route unprocessable messages to a separate queue rather than silently dropping them. The implementation depends on the broker:

- **RabbitMQ:** configure a DLX on the queue; messages that fail or expire land there.
- **Kafka:** use a separate "DLQ" topic; the consumer explicitly produces to it on permanent failure.
- **SQS:** the source queue's redrive policy points to a DLQ with a `maxReceiveCount`.
- **NATS JetStream:** advisories + a dead-letter subject.

```rust
match handle(&msg).await {
    Ok(()) => msg.ack().await?,
    Err(Error::Invalid(e)) => {
        // permanent — move to DLQ, do not retry
        dlq_producer.send(msg.into_dlq(e)).await?;
        msg.ack().await?;       // ack the original to stop redelivery
    }
    Err(e) => {
        // transient — let the broker redeliver (or use backoff::Error::transient)
        msg.nack_with_delay(Duration::from_secs(2)).await?;
    }
}
```

## 6. The outbox pattern (transactional publish)

The **dual-write problem**: you write to your database, then publish to the broker. If the publish fails, your DB write is committed but the message is lost; if the DB rolls back, your broker may have already published. The recognized solution is the **transactional outbox**:

1. Inside the DB transaction, insert the event into an `outbox` table.
2. Commit the transaction — now the event is durable.
3. A separate **relay** (in-process task or a sidecar) reads the `outbox` table and publishes to the broker.
4. Only after a successful broker `ack` does the relay mark the row as published.

```rust
// Inside a sqlx transaction
let mut tx = pool.begin().await?;
sqlx::query!(
    "INSERT INTO orders (id, total) VALUES ($1, $2)",
    order.id, order.total
).execute(&mut *tx).await?;

sqlx::query!(
    "INSERT INTO outbox (aggregate_id, event_type, payload) VALUES ($1, $2, $3)",
    order.id, "OrderCreated", serde_json::to_value(&order)?
).execute(&mut *tx).await?;

tx.commit().await?;     // atomic: both rows or neither

// A separate task reads outbox rows and publishes, then marks them sent.
```

Crates that can help structure this: [`outbox-processor`](https://crates.io/crates/outbox-processor) (early) or a hand-rolled relay — there is no dominant crate here; the pattern is the standard, not a particular library.

## 7. Ordering and partitioning

- **Kafka:** ordering is per-partition; pick a stable partition key (`order.user_id`) so related events land in the same partition and are delivered in order.
- **NATS JetStream:** subject-based partitioning.
- **FIFO queues in SQS:** `MessageGroupId` provides ordering within a group.
- **Trade-off:** stricter ordering reduces parallelism. Partition by a key that gives you the ordering you need and no more.

## 8. In-process async channels (tokio::sync)

When "messaging" means communication between tasks in the same process, the canonical primitives are in `tokio::sync`:

| Primitive | Use |
|---|---|
| `tokio::sync::mpsc` | Multi-producer, single-consumer channel — the default. |
| `tokio::sync::broadcast` | Multi-producer, multi-consumer — every subscriber sees every message. |
| `tokio::sync::watch` | Single-producer, multi-consumer where only the latest value matters (config updates). |
| `tokio::sync::oneshot` | One value, one time (request/reply). |
| [`flume`](https://crates.io/crates/flume) | Sync+async hybrid channel, often faster than std/mpsc for some workloads. |
| [`async-channel`](https://crates.io/crates/async-channel) | Bound async channel used by async-std/smol ecosystem. |

```rust
use tokio::sync::mpsc;

let (tx, mut rx) = mpsc::channel::<Order>(100);

tokio::spawn(async move {
    while let Some(order) = rx.recv().await {
        process(order).await;
    }
});

tx.send(order).await?;       // backpressures at capacity
```

> Bounded channels (`mpsc::channel(capacity)`) provide **backpressure**; unbounded (`mpsc::unbounded_channel`) does not — prefer bounded in production.

## 9. AsyncAPI contracts

[AsyncAPI](https://www.asyncapi.com/) is to event-driven APIs what OpenAPI is to REST. The Rust ecosystem support is nascent:

- There is no first-class Rust code generator comparable to `prost` for Protobuf or the OpenAPI generators.
- The convention is to use AsyncAPI documents as **documentation** of your message schemas, and to generate the message types from Protobuf/Avro schemas (which AsyncAPI references).

## 10. When this section does not apply

For Rust projects that do not communicate across process boundaries (libraries, single-binary CLIs, embedded firmware), this section is **not applicable** — note it explicitly with rationale, as in Section 25.

## Completion checklist

- [x] Dominant broker clients documented side by side (Kafka, RabbitMQ, NATS, SQS/SNS, MQTT, Redis).
- [x] Message format and schema-evolution conventions documented (serde + adapters).
- [x] Consumer idempotency and delivery-semantics conventions documented with ✅/❌.
- [x] Ordering and partitioning conventions documented.
- [x] Reliability patterns (DLQ, retry/backoff) documented.
- [x] Outbox / transactional-publish pattern documented.
- [x] In-process async channels (tokio::sync, flume) documented.
- [x] AsyncAPI tooling state documented.

### References

- rdkafka — https://docs.rs/rdkafka
- lapin — https://docs.rs/lapin
- async-nats — https://docs.rs/async-nats
- rumqttc — https://docs.rs/rumqttc
- AWS SDK for Rust — https://aws.github.io/aws-sdk-rust/
- `backoff` — https://docs.rs/backoff
- `tokio::sync` — https://docs.rs/tokio/#channels
- AsyncAPI — https://www.asyncapi.com/
- Transactional Outbox Pattern — https://microservices.io/patterns/data/transactional-outbox.html
