# Scalable Notification System

A high-performance, fault-tolerant notification system designed to deliver **100M+ notifications per day** across multiple channels including push, email, and in-app messaging.

---

## 🚀 Overview

Modern applications require real-time, reliable notification delivery at scale. This system is built to handle massive traffic while keeping APIs fast, decoupled, and resilient under failure.

Key design goals:

* High throughput (100M+ notifications/day)
* Low API latency
* Horizontal scalability
* Fault tolerance and retry safety
* Multi-channel delivery (Push, Email, In-App)

---

## 🧠 High-Level Architecture

```
Client Apps
   ↓
API Layer (Auth + Validation)
   ↓
Dispatcher Service
   ↓
Message Queue (Kafka / Redis Streams)
   ↓
Background Workers
   ├── Push Worker (FCM / APNs)
   ├── Email Worker (SMTP / Providers)
   └── In-App Worker (DB storage)
   ↓
Database + Cache Layer
```

---

## ⚙️ Core Components

### 1. API Layer

Handles incoming requests from client apps or backend services.

Responsibilities:

* Input validation
* Rate limiting
* User preference checks
* Publishing events to queue

---

### 2. Dispatcher Service

The decision engine of the system.

Responsibilities:

* Determines delivery channels
* Applies deduplication rules
* Enriches notification payload
* Publishes events to message queue

---

### 3. Message Queue

Ensures asynchronous and reliable processing.

Benefits:

* Decouples API from workers
* Handles traffic spikes gracefully
* Enables retries and backpressure control

---

### 4. Workers

Specialized consumers for each channel:

* **Push Worker** → Mobile notifications via FCM/APNs
* **Email Worker** → Email delivery via SMTP providers
* **In-App Worker** → Stores notifications in database

---

## 🌐 API Endpoints

### Create Notification

`POST /notifications`

```json
{
  "user_id": "123",
  "event": "comment_added",
  "channels": ["push", "email"],
  "message": "Someone commented on your post"
}
```

Response:

```json
{
  "status": "queued",
  "id": "notif_001"
}
```

---

### Get Notifications

`GET /users/{id}/notifications`

Returns paginated notifications for UI display.

---

### Update Preferences

`PUT /users/{id}/preferences`

Allows users to configure:

* Enabled channels
* Quiet hours
* Notification categories

---

## 🗄️ Data Storage Design

### Notifications Table (PostgreSQL)

```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id VARCHAR(255) NOT NULL,
  message TEXT NOT NULL,
  type VARCHAR(50) NOT NULL,
  channel VARCHAR(50) NOT NULL,
  status VARCHAR(20) DEFAULT 'pending',
  priority SMALLINT DEFAULT 1,
  metadata JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  delivered_at TIMESTAMP WITH TIME ZONE,
  read_at TIMESTAMP WITH TIME ZONE
);
```

### Indexes

```sql
CREATE INDEX idx_notifications_user_created
ON notifications(user_id, created_at DESC);

CREATE INDEX idx_notifications_status
ON notifications(status) WHERE status != 'sent';

CREATE INDEX idx_notifications_priority
ON notifications(priority, created_at) WHERE priority > 2;
```

---

## ⚡ Caching Layer (Redis)

### User Preferences Cache

Key: `user_prefs:{user_id}`

```json
{
  "channels": {
    "push": true,
    "email": false,
    "in_app": true
  },
  "quiet_hours": {
    "start": "22:00",
    "end": "08:00",
    "timezone": "UTC"
  }
}
```

TTL: 24 hours

---

### Deduplication Store

Key: `dedup:{user_id}:{event_hash}`

Purpose:

* Prevent duplicate notifications
* Enforce idempotency

TTL: 1 hour

---

## 📈 Scaling Strategy

### Horizontal Scaling

* API layer is stateless → fully scalable
* Workers scale independently per channel

---

### Partitioning

* Queue partitions distribute load
* Workers consume independently

---

### Database Scaling

* Read replicas for heavy read traffic
* Partitioning by `user_id`

---

### Sharding Strategy

**Shard Key:** `user_id (hashed)`

```text
hash(user_id) % 256 → shard_id
```

* 256 virtual shards mapped to physical nodes
* Each shard ~78K users
* Each shard handles ~390K writes/day

Benefits:

* Even distribution
* Easy scaling
* Hot-user isolation support

---

## 🔁 Reliability Strategy

### Retry Policy

* Exponential backoff retries
* Max retry attempts before failure queue

---

### Failure Handling

* Queue ensures message durability
* Workers auto-recover on crash
* External API failures handled with retry + fallback

---

### Deduplication

* Redis-based idempotency keys
* Worker-level duplicate checks

---

## ⚖️ Design Trade-offs

### Push vs Polling

* Push → real-time, complex
* Polling → simpler, scalable
* **Chosen:** Hybrid approach

---

### Sync vs Async

* Sync → simple but slow
* Async → scalable and fast
* **Chosen:** Async architecture

---

### Consistency Model

| Type                 | Use Case           | Trade-off      |
| -------------------- | ------------------ | -------------- |
| Strong Consistency   | Security, payments | Higher latency |
| Eventual Consistency | Likes, comments    | Temporary lag  |

**Final choice:** Hybrid model

---

## 🧾 Summary

This system is designed to:

* Handle massive scale without breaking
* Keep APIs fast under heavy load
* Isolate failures using queues and workers
* Scale each component independently

Even under extreme traffic spikes, the queue absorbs pressure and keeps the system stable.

Because yes, queues are basically emotional support systems for backend architecture.
