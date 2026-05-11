# Failure Cascade Analysis

## 1) Traffic Simulation Math

Scenario inputs:

- Notification recipients: 180,000,000
- Expected click-through rate: 8%
- Users who open the app: 14,400,000
- Spike window: 60 seconds
- API calls per active user in the first minute: 3

Peak demand math:

```text
14,400,000 active users x 3 calls / 60 seconds = 720,000 RPS
```

If you use the narrower 10,000,000-user click cohort from the prompt's simplified example:

```text
10,000,000 users x 3 calls / 60 seconds = 500,000 RPS
```

Either way, the monolith is facing demand far above its single-instance capacity.

## 2) Component Capacity Numbers

Monolith hard limits:

- PostgreSQL `max_connections = 100`
- Node.js single-process event loop saturation: about 12,000 to 15,000 RPS on a single 1 CPU / 4 GB instance before the callback queue backs up
- Synchronous payment hold time: 200 ms to 2,000 ms per order request
- Node heap / RSS pressure: about 15,000 queued requests can push the process into OOM territory on a 4 GB host

DB connection exhaustion math:

Let non-payment requests hold a DB connection for 20 ms and payment requests hold a DB connection for 800 ms.

```text
Connections held = (0.70 x RPS x 0.020) + (0.30 x RPS x 0.800)
                 = 0.014 x RPS + 0.240 x RPS
                 = 0.254 x RPS
```

Pool exhaustion happens when held connections reach 100:

```text
0.254 x RPS = 100
RPS = 393.7
```

So the PostgreSQL pool exhausts at about 394 RPS, not thousands.

## 3) The Cascade

### Failure 1: PostgreSQL Connection Pool Exhaustion

- Severity: CRITICAL
- Trigger: about 394 RPS with 30% payment calls and 800 ms hold time
- User symptom: requests hang, then fail with 500s / upstream timeout errors
- Next failure: requests stack up in Node.js waiting for a free DB slot, which increases latency and memory use

### Failure 2: Node.js Event Loop Saturation

- Severity: CRITICAL
- Trigger: roughly 12,000 to 15,000 RPS on a single process
- User symptom: slow menus, delayed order placement, then full request timeout
- Next failure: callback backlog grows, memory rises, and the process can crash under load

### Failure 3: Synchronous Payment Call Amplification

- Severity: HIGH
- Trigger: concurrent with the DB pool problem; every payment request holds a connection for 200 ms to 2,000 ms
- User symptom: payment screens spin, then fail even though the checkout page loaded successfully
- Next failure: connection occupancy rises, so fewer requests can make progress in PostgreSQL

### Failure 4: Promo Code Race Condition

- Severity: HIGH
- Trigger: within seconds of concurrent promo redemption if validation and deduction are separate statements
- User symptom: users see success messages for discounts that should already be exhausted
- Next failure: discount budget is oversold and finance / support systems disagree with user-facing state

### Failure 5: Static Asset NIC Saturation

- Severity: HIGH
- Trigger: when restaurant images and static bundles are served by Node.js during the spike
- User symptom: image loads stall, app feels broken before API requests even complete
- Next failure: the same NIC that should carry API responses becomes saturated by asset traffic

### Failure 6: Monitoring Blindness

- Severity: HIGH
- Trigger: always present if there are no traces, no correlation IDs, and no structured logs
- User symptom: widespread failures without a clear root cause
- Next failure: MTTD and MTTR explode because the on-call engineer has to guess

## 4) Timeline

- T+0s: Push notification goes out to 180M users
- T+1s: Initial app opens start, and the first burst of menu requests hits Node.js
- T+3s: PostgreSQL connection pool reaches 100/100 and begins rejecting new work
- T+5s: Node.js request queue backs up; p99 latency jumps from tens of milliseconds to seconds
- T+8s: DB errors begin surfacing to users as order and menu requests time out
- T+10s: Payment calls are still holding connections, so checkout failures increase sharply
- T+12s: Promo redemptions oversell because read-then-write validation races under concurrency
- T+15s: Static assets continue to consume bandwidth and the NIC becomes a bottleneck
- T+18s: Memory pressure and backlog push the Node.js process toward OOM
- T+20s: The load balancer, if present, would mark the instance unhealthy; without one, the single endpoint stays unavailable
- T+45m: On-call identifies the broad class of failure only after manual log inspection and metric correlation
- T+2h: Service is restored after restart, connection cleanup, and traffic reduction