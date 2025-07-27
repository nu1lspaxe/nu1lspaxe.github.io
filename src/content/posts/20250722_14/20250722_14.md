---
title: RabbitMQ AMQP 0-9-1 Internals
published: 2025-07-22
description: "A breakdown of how AMQP 0-9-1 delivers messages via exchanges, routing keys, and bindings."
image: "./rabbitmq.jpg"
tags: ["Message Queue", "RabbitMQ"]
category: "Message Queue"
draft: false
lang: ""
---

## Overview

![example-routing](./_1.png)

The core dataflow concept follows this sequence (from left to right) : Publisher ➝ Exchange ➝ Queue ➝ Consumer

1. **Messsages** are sent to the **exchange** by **publishers**
2. The **exchange** routes messages to the appropriate **queues** based on **binding** (i.e., routing rules)
3. **Consumers** receive messages from the queues in one of two ways :

   - **Push-based** (subscribed)
   - **Pull-based** (fetch on demand)

Exchange, Queue, and Binding are collectively referred to as _AMQP entities_, and the orange rabbit section in the diagram represents the **Broker**.

## Exchanges and Exchange Types

Exchanges receive messages and determine how to route them. Based on the _exchange type_ and _bindings_, messages are routed to zero or more queues.

AMQP 0-9-1 brokers provide four types of exchanges :

| Exchange type    | Default pre-declared names              |
| ---------------- | --------------------------------------- |
| Direct exchange  | (Empty string) and amq.direct           |
| Fanout exchange  | amq.fanout                              |
| Topic exchange   | amq.topic                               |
| Headers exchange | amq.match (and amq.headers in RabbitMQ) |

The **default exchange** is a **direct exchange** with no name (i.e, an empty string), and it is predefined by the broker. When you create a queue and give it a name (e.g, `search-indexing-online`), the broker automatically binds that queue to the default exchange using a **routing key** (also known as **binding key**) with the same name.

### Direct Exchange

- A **direct exchange** routes messsages to queues based on the message's routing key
- It's commonly used for unicast and multicast message delivery

![direct-exchange](./direct-exchange.png)

1. A **queue** is bound to the exchange using a **routing key** `K`
2. When a new message with routing key `R` arrives at the direct exchange, the exchange routes it to any queue where `K == R`
3. If multiple queues are bound to the exchange with the same routing key `K`, the exchange delivers the message to **all queues** that satisfy `K == R`

### Fanout Exchange

- A **fanout exchange** sends messages to **all queues** bound to the exchange, ignoring the routing key
- It's commonly used for broadcast scenarios
- For example use cases, see the [document](https://www.rabbitmq.com/tutorials/amqp-concepts#exchange-fanout)

![fanout-exchange](./fanout-exchange.png)

### Topic Exchange

- A **topic exchange** routes messages to one or more queues based on **pattern-matching** routing keys
- It's commonly used in publish/subscribe models and is frequently applied in multicast message delivery
- For example use cases, see the [document](https://www.rabbitmq.com/tutorials/amqp-concepts#exchange-topic)

![topic-exchange](./topic-exchange.png)

### Headers Exchange

- A **headers exchange** doesn't use routing keys; instead, it matches messages based on headers
- It allows for flexible, pattern-like matching using headers, which can be strings, integers, or even hashes (dictionaries)

![headers-exchange](./headers-exchange.png)

## Queues

A **queue** stores messages that are delivered to **consumers**.

In AMQP 0-9-1, queues can be categoriezed as either durable or transient. A durable queue stores its metadata on disk, while a transient queue stores it metadata in memory.

## Bindings

A **binding** defines the rules that an exchange uses to route messages to queues based on the routing key (similar to a filter).
If no matching queue is found, the message will either be dropped or returned to the publisher, depending on the user's configuration.

## Connections and Channels

- AMQP 0-9-1 uses a long-lived TCP connection
- It adopts a [channel-based design](https://www.rabbitmq.com/docs/channels), which allows multiple channels to share a single TCP connection
- Channels are isolated from each other and are identified by a channel ID, which helps the broker and client distinguish between them
