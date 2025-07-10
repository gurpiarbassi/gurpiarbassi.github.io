---
layout: post
title: "Latency Tuning"
published: true
date: 2025-07-10
categories: architecture performance
---

# Motivation

In this post, we'll explore some ways we can reduce latency.

## 1. **Use Cache to Serve Popular Data**

Cache frequently requested data in memory (e.g., Redis, Memcached) to avoid repeated database queries or expensive
computations.

*Quick win:* Can drastically reduce response times with minimal code changes.

---

## 2. **Compress the Payload**

Use GZIP, Brotli, or similar compression algorithms for HTTP requests/responses to reduce payload size.

*Quick win:* Often enabled via web server config (e.g., Nginx) or middleware.

---

## 3. **Group Requests**

Batch or bundle requests (e.g., GraphQL batching, REST aggregation) to minimize round-trips.

*Quick win:* Reduce client-server chattiness with minor frontend/backend adjustments.

---

## 4. **Use CDN to Keep Data Closer to the Users**

Content Delivery Networks (like Cloudflare, Akamai, etc.) serve static assets from edge locations near the user.

*Quick win:* Drop-in config for static content; reduces geographic latency.

---

## 5. **Use Connection Pooling**

Enable connection pooling for DBs and remote services to avoid overhead of establishing new connections.

*Quick win:* Often just a config change in your database/client settings.

---

## 6. **Use Database Indexes**

Add indexes on frequently queried fields to speed up read access.

*Moderate effort:* Requires DB schema knowledge, but high performance payoff.

---

## 7. **Reduce External Dependencies**

Eliminate or delay 3rd-party API calls (e.g., during checkout or login) to avoid blocking on slow services.

*Moderate effort:* Requires code refactoring or retry mechanisms.

---

## 8. **Add a Load Balancer to Distribute Traffic**

Distribute incoming traffic across multiple instances to avoid overloading any one server.

*Moderate effort:* Usually done at infra level (e.g., AWS ELB, NGINX).

---

## 9. **Use HTTP/2 for Parallel Request Multiplexing**

HTTP/2 enables multiple requests in a single TCP connection, reducing latency over HTTP/1.1.

*Moderate effort:* Enabled via web server settings; may need TLS.

---

## 10. **Scale Vertically (More CPU/RAM)**

Increase memory, CPU, or disk performance to handle more requests faster on a single machine.

*Quick to implement* but may have **cost implications**.

---

## 11. **Use Efficient Data Serialization Format**

Switch to compact, binary formats like Protobuf or Avro over JSON to reduce encoding/decoding time and payload size.

*More complex change:* Affects how data is structured and parsed.

---

## 12. **Use Message Queues for Background Tasks**

Offload time-consuming tasks (e.g., email sending, PDF generation) to queues like Kafka, RabbitMQ, or SQS.

*Heavier lift:* Requires architectural change and retry handling.

---

### 🏁 Summary: Prioritized by Impact/Effort

| Priority  | Rule                         | Effort | Impact |
|-----------|------------------------------|--------|--------|
| Quick Win | Use cache                    | Low    | High   |
| Quick Win | Compress the payload         | Low    | High   |
| Quick Win | Group requests               | Low    | Medium |
| Quick Win | Use CDN                      | Low    | High   |
| Quick Win | Use connection pooling       | Low    | High   |
| Medium    | Use database indexes         | Medium | High   |
| Medium    | Reduce external dependencies | Medium | Medium |
| Medium    | Add load balancer            | Medium | Medium |
| Medium    | Enable HTTP/2                | Medium | Medium |
| Quick     | Scale vertically             | Low    | Medium |
| Heavy     | Efficient data serialization | High   | High   |
| Heavy     | Use message queues           | High   | High   |
