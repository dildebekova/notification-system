# Notification System (Scalable Design)

## Overview

This project describes a notification system designed to handle
large-scale message delivery across multiple channels such as in-app
notifications, push alerts, and email.

The system focuses on reliability, scalability, and async processing so
that user-facing APIs stay fast even under heavy load.

------------------------------------------------------------------------

## High-Level Architecture

    Client Apps
       ↓
    API Layer (Authentication + Validation)
       ↓
    Message Dispatcher Service
       ↓
    Message Queue (Kafka / Redis Streams)
       ↓
    Background Workers
       ├── In-App Worker
       ├── Push Worker
       └── Email Worker
       ↓
    External Providers (FCM, APNs, SMTP)
       ↓
    Database + Cache Layer

------------------------------------------------------------------------

## Core Components

### 1. API Layer

Handles incoming requests from backend services or user actions.\
Responsibilities: - Input validation - Rate limiting - User preference
check - Sending tasks to queue

------------------------------------------------------------------------

### 2. Dispatcher Service

Acts as the brain of the system: - Determines which channels to use -
Applies deduplication rules - Publishes events into message queues

------------------------------------------------------------------------

### 3. Message Queue

Used for async processing: - Decouples API from delivery - Buffers
traffic spikes - Enables retry mechanisms

------------------------------------------------------------------------

### 4. Workers

Separate workers for each channel: - Push Worker → mobile
notifications - Email Worker → email delivery - In-App Worker → stores
notifications in DB

------------------------------------------------------------------------

## API Endpoints

### Create Notification

**POST /notifications**

``` json
{
  "user_id": "123",
  "event": "comment_added",
  "channels": ["push", "email"],
  "message": "Someone commented on your post"
}
```

Response:

``` json
{
  "status": "queued",
  "id": "notif_001"
}
```

------------------------------------------------------------------------

### Get Notifications

**GET /users/{id}/notifications**

Returns paginated list of notifications for UI display.

------------------------------------------------------------------------

### Update Preferences

**PUT /users/{id}/preferences**

Allows users to enable/disable channels and configure quiet hours.

------------------------------------------------------------------------

## Data Storage

### Notifications Table

Stores in-app notifications and history: - id - user_id - message -
type - status - created_at

Indexes: - user_id + created_at (fast feed loading)

------------------------------------------------------------------------

### User Preferences Cache

Stored in Redis: - notification settings - rate limiting counters -
deduplication keys

------------------------------------------------------------------------

## Scaling Approach

### Horizontal Scaling

-   API servers are stateless → can scale freely
-   Workers scaled independently per channel

### Partitioning

-   Messages distributed across queue partitions
-   Each worker group consumes independently

### Database Strategy

-   Read-heavy system → read replicas used
-   Partitioning by user_id for large datasets

------------------------------------------------------------------------

## Reliability Strategy

### Retry Policy

If delivery fails: - Retry with increasing delay - After max attempts →
move to failed queue

### Deduplication

Prevent duplicate notifications using: - Redis keys with expiration -
Idempotency checks in worker

### Failure Handling

-   Queue failure → messages preserved
-   Worker crash → automatic reprocessing
-   External service failure → retry + fallback

------------------------------------------------------------------------

## Trade-offs

### Push vs Polling

-   Push = real-time but complex
-   Polling = simpler and scalable → chosen: hybrid approach

------------------------------------------------------------------------

### Sync vs Async

-   Sync is simple but slow under load
-   Async improves performance and stability → chosen: async
    architecture

------------------------------------------------------------------------

## Summary

The system is built to remain fast, modular, and fault-tolerant under
high traffic.\
By separating API, queue, and workers, it avoids bottlenecks and allows
independent scaling of each component.

Even if traffic explodes, the queue absorbs pressure and keeps the
system alive.

(Yes, queues are basically emotional support systems for backend
services.)
