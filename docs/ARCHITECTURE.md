# Architecture Redesign

## 1) Current Architecture

```text
10M users
  |
  v
Single Node.js Express server
  - menu API
  - order placement
  - synchronous payment processing   <- Failure 3: connection hold amplification
  - delivery tracking
  - promo validation
  - static images and assets         <- Failure 5: NIC saturation
  |
  v
Single PostgreSQL database
  - max_connections = 100           <- Failure 1: pool exhausts at ~394 RPS
  - no read replicas                 <- Failure 1 and write/read contention
  - no index on orders.user_id

No CDN                              <- Failure 5: static traffic hits origin
No load balancer                    <- single point of failure
No cache                            <- every menu request hits DB
No auto-scaling                     <- compute saturation becomes outage
```

## 2) New Architecture

```text
10M users
  |
  v
CloudFront CDN
  - caches images, JS, CSS, and public menus
  - TTL: 5 minutes for menus, longer for immutable assets
  - origin shield for app traffic reduction
  |
  v
Application Load Balancer
  - SSL termination
  - health checks
  - rate limiting by IP / WAF policy
  |
  +--------------------+--------------------+--------------------+
  |                    |                    |                    |
  v                    v                    v                    v
Node.js instance      Node.js instance      Node.js instance      Node.js instance
  - stateless           - stateless           - stateless           - stateless
  - auto-scale on CPU   - auto-scale on CPU   - auto-scale on CPU   - auto-scale on CPU
  - target: 4 to 20 instances depending on load
  |
  v
Redis / ElastiCache
  - caches restaurant menus with 5 minute TTL
  - stores promo counter with atomic SETNX / INCR pattern
  - absorbs most read traffic from PostgreSQL
  |
  +----------------------------+----------------------------+
  |                            |                            |
  v                            v                            v
PgBouncer              PostgreSQL primary           PostgreSQL read replicas
  - small physical       - writes only               - menu / browse reads
    connection pool      - order writes              - order history reads
  - many app sessions    - strict write path
  |
  v
SQS payment queue
  - order write succeeds fast
  - payment processing becomes async
  - DLQ for failed or retryable messages
  |
  v
Payment worker service
  - consumes SQS messages
  - calls Razorpay / PayU asynchronously
  - updates payment status after completion
```

## 3) Component Justification Table

| Component | Failure It Prevents | How It Prevents It |
|---|---|---|
| CloudFront CDN | Failure 5: Static asset NIC saturation | Serves images and static bundles from edge locations so Node.js never carries the bulk asset load |
| Application Load Balancer | Single point of failure and unbounded traffic | Terminates TLS, health-checks instances, and stops routing to unhealthy nodes |
| Node.js autoscaled fleet | Failure 2: Event loop saturation | Spreads request load across multiple stateless instances instead of one saturated process |
| Redis cache | Failure 1: PostgreSQL pool exhaustion | Keeps high-frequency menu reads out of the database and reduces connection pressure |
| Redis atomic promo lock | Failure 4: Promo race condition | Uses an atomic counter update so only one request can consume a promo budget unit at a time |
| PgBouncer | Failure 1: Connection exhaustion | Multiplexes many app sessions onto a small number of real PostgreSQL connections |
| PostgreSQL read replicas | Read/write contention | Sends browse and history reads away from the primary so writes stay fast |
| SQS payment queue | Failure 3: Payment call amplification | Moves payment processing off the request path so DB connections are released immediately |
| Payment worker service | Payment timeout amplification | Handles slow gateway calls asynchronously and isolates them from customer-facing latency |
