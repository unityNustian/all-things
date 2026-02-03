# Designing a Highly Scalable URL Shortener: A Comprehensive System Design

Designing a URL shortener like Bitly or TinyURL is a classic system design challenge that touches on distributed systems, database sharding, caching, and security. This document provides a detailed breakdown of the architecture, trade-offs, and concrete implementation strategies.

---

## 1. Overview and Core Goals

The primary purpose of a URL shortener is to map a long URL to a unique, compact alias. When a user visits the short URL, the system redirects them to the original destination.

### Primary Goals
*   **Low Latency**: Redirects should ideally occur in under 50–100 ms to ensure a seamless user experience.
*   **High Availability**: The redirection service is mission-critical; it must be resilient to regional failures.
*   **Scalability**: The system must handle billions of URLs and millions of redirects per second.
*   **Abuse Protection**: Prevention of phishing, malware, and spam is essential for maintaining the service's reputation.

---

## 2. Requirements

### Functional Requirements
*   **URL Shortening**: Generate a unique short alias for any given long URL.
*   **Redirection**: Efficiently redirect users from the short link to the original URL.
*   **Custom Aliases**: Allow users to define "vanity" URLs (e.g., `bit.ly/my-awesome-link`).
*   **Analytics**: Track click data (geo-location, referrer, device type, timestamp).
*   **Expiration**: Support automatic deletion of links after a specified duration.

### Non-Functional Requirements
*   **Read-Heavy Nature**: The ratio of reads (redirects) to writes (creations) is typically 100:1 or higher.
*   **Consistency**: Redirect lookups must be strongly consistent (or highly available with low replication lag).
*   **Durability**: Once a link is created, it should not be lost.

---

## 3. Core API Design

A clean, RESTful API is the foundation of the service.

| Method | Endpoint | Description | Payload Example |
| :--- | :--- | :--- | :--- |
| **POST** | `/v1/shorten` | Creates a shortened URL | `{ "url": "https://example.com/long-path", "alias": "custom", "expiry": "2026-12-31" }` |
| **GET** | `/:shortId` | Redirects to target URL | N/A |
| **GET** | `/:shortId/stats` | Returns click analytics | N/A |
| **DELETE** | `/:shortId` | Deletes/disables a link | N/A |

---

## 4. Short Key Generation Strategies

Choosing how to generate the unique `shortId` is the most critical design decision.

### Strategy A: Sequential IDs with Base62 Encoding
This involves using a central counter (or a distributed ID generator like Snowflake) and converting the numeric ID into a Base62 string (a–z, A–Z, 0–9).

*   **Pros**: Guaranteed uniqueness, compact keys, easy to shard by numeric range.
*   **Cons**: Predictable URLs (users can guess the next link), requires a distributed counter.

### Strategy B: Random Token Generation
Generate a random 6–8 character string and check the database for collisions.

*   **Pros**: Non-predictable, easier to generate in a decentralized manner.
*   **Cons**: Collision probability increases as the database grows; requires a "check-then-insert" loop.

### Strategy C: Hash-based (MD5/SHA)
Hash the long URL and take the first N characters of the Base62-encoded hash.

*   **Pros**: Idempotent (same URL always gets the same short link).
*   **Cons**: High collision rate with truncation; handling custom aliases requires a separate flow.

### Key Length vs. Capacity (Base62)
| Length | Total Unique Keys | Scale Reference |
| :--- | :--- | :--- |
| 4 chars | 62⁴ ≈ 14.7 Million | Small/Internal tools |
| 5 chars | 62⁵ ≈ 916 Million | Medium-scale service |
| 6 chars | 62⁶ ≈ 56.8 Billion | Global-scale (Bitly/TinyURL) |

---

## 5. Data Modeling and Storage

The data model is simple, but the storage strategy must account for massive scale.

### Minimal Schema
*   `short_id` (PK, VARCHAR): The unique alias.
*   `target_url` (TEXT): The destination URL.
*   `owner_id` (UUID): Reference to the creator.
*   `created_at`, `expires_at` (TIMESTAMP): Lifecycle management.
*   `disabled` (BOOLEAN): For abuse/takedown handling.

### Storage Choices
*   **SQL (PostgreSQL/MySQL)**: Best for consistency and relational features (e.g., user management). Use **sharding by `hash(short_id)`** to distribute load.
*   **NoSQL (DynamoDB/Cassandra)**: Ideal for extreme write throughput and horizontal scaling. DynamoDB’s TTL feature is perfect for link expiration.

---

## 6. Caching and Latency Optimization

To achieve < 100ms redirects, caching is non-negotiable.

### Caching Layers
1.  **L1 (Application Cache)**: Store the most popular 1% of links in memory (LRU cache) within the service process.
2.  **L2 (Redis/Memcached)**: A distributed cache cluster. Since short URLs are immutable, we can use a long TTL.
3.  **Edge Redirection**: Use Cloudflare Workers or Lambda@Edge to perform the redirect at the CDN level, reducing latency to < 10ms for global users.

### HTTP Redirect Types
*   **301 (Permanent)**: Browsers cache the redirect. Reduces server load but makes analytics gathering difficult.
*   **302 (Found/Temporary)**: Browsers do not cache. Every click hits the server, allowing for 100% accurate analytics. **Recommended for most commercial shorteners.**

---

## 7. Analytics Pipeline

Analytics logging must be decoupled from the redirect path to avoid blocking the user.

1.  **Event Capture**: The redirect worker sends a "click event" to a message queue (e.g., **Apache Kafka** or **AWS SQS**).
2.  **Stream Processing**: **Apache Flink** or **Kafka Streams** aggregates data in real-time (e.g., "clicks per minute").
3.  **Long-term Storage**: Store raw events in an OLAP database like **ClickHouse**, **BigQuery**, or **Snowflake** for complex reporting.

---

## 8. Security and Abuse Prevention

A URL shortener is a prime target for malicious actors.

*   **URL Scanning**: Integrate with services like Google Safe Browsing to check target URLs for malware/phishing.
*   **Rate Limiting**: Implement per-IP and per-API-key limits on the `/shorten` endpoint to prevent bulk spam.
*   **Blacklisting**: Maintain a list of forbidden domains (e.g., `*.onion`, internal IPs).
*   **Quarantine**: Suspicious links should redirect to a warning page rather than the final destination.

---

## 9. Concrete Scale Example (Worked Example)

Assume we want to support **1 Billion URLs** with **100,000 Redirects per Second (RPS)**.

### Storage Calculation
*   Metadata per URL: ~200 bytes.
*   Total Raw Storage: 1B * 200 bytes = **200 GB**.
*   With indices and replication (3x): **~600 GB - 1 TB**.

### Throughput & Caching
*   100k RPS is high for a single DB.
*   **Cache Hit Ratio (90%)**: 90k RPS served from Redis, 10k RPS hitting the DB.
*   **Redis Memory**: To cache 10 million "hot" links: 10M * 200 bytes = **2 GB**. This easily fits in a small Redis cluster.

---

## 10. Summary of Trade-offs

| Feature | Option A | Option B | Recommendation |
| :--- | :--- | :--- | :--- |
| **Key Gen** | Sequential (Compact) | Random (Private) | **Random** for public services to prevent scraping. |
| **Database** | SQL (ACID) | NoSQL (Scale) | **NoSQL** for the mapping table; SQL for user data. |
| **Redirect** | 301 (Faster) | 302 (Better Data) | **302** to ensure analytics are captured. |
| **Latency** | Centralized | Edge/CDN | **Edge** for global reach if budget allows. |
