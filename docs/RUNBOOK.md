# Incident Runbook

## STEP 1: DETECT

Create these CloudWatch alarms:

| Metric | Threshold | Alert Type |
|---|---|---|
| ALB `HTTPCode_ELB_5XX_Count` | > 5% of requests for 2 minutes | Critical |
| RDS `DatabaseConnections` | > 80% of max_connections | Warning |
| EC2 `CPUUtilization` | > 80% for 3 minutes | Warning |
| EC2 `StatusCheckFailed` | > 0 for 1 minute | Critical |
| ElastiCache `CurrConnections` or `Evictions` | > 75% memory or rising evictions | Warning |
| SQS `ApproximateNumberOfMessagesVisible` | > 10,000 | Warning |
| p99 API latency | > 2 seconds | Critical |
| Promo budget remaining | < 10% | Warning |

## STEP 2: TRIAGE

Check in this order. Stop at the first red signal.

1. Check `RDS.DatabaseConnections`.
   - If red: root cause is DB pool pressure. Go to Step 3a.
2. Check `EC2.CPUUtilization` across all app instances.
   - If red on all nodes: root cause is compute saturation. Go to Step 3b.
3. Check ElastiCache `Evictions` and `CacheMissRate`.
   - If red: root cause is cache thrash or invalidation. Go to Step 3c.
4. Check SQS queue depth and DLQ count.
   - If red: root cause is payment worker backlog. Go to Step 3d.

If none of the above are red but users still report failure, inspect the ALB 5xx rate and then the payment gateway logs. That usually means an external dependency is timing out.

## STEP 3: RESPOND

### 3a. DB Pool Exhaustion

What to do:

1. Check active sessions:

```sql
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
```

2. Terminate idle-in-transaction sessions older than 30 seconds:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND query_start < NOW() - INTERVAL '30 seconds';
```

3. Increase PgBouncer pool size if spare DB capacity exists.
4. If the primary is still saturated, shift browse reads to replicas and reduce checkout traffic by throttling at the ALB.

Success signal:

- `DatabaseConnections` falls below 70% within 2 to 5 minutes
- ALB 5xx rate drops below 1%

Owner:

- `#dba-oncall`

### 3b. Compute Saturation

What to do:

1. Increase app capacity in the ASG or ECS service.

```text
Set desired capacity to 20 instances
```

2. Confirm new instances pass health checks.
3. If latency stays high, check whether Redis or SQS is also failing.

Success signal:

- EC2 CPU falls below 60% within 2 minutes
- p99 latency drops below 1 second

Owner:

- `#app-oncall`

### 3c. Redis Cache Miss Spike

What to do:

1. Check evictions:

```text
redis-cli INFO stats | grep evicted_keys
```

2. If evictions are climbing, add cache capacity or a shard.
3. If a deploy flushed the cache, allow warmup and watch DB read traffic.

Success signal:

- Cache hit rate returns above 80%
- DB read QPS drops back toward baseline

Owner:

- `#app-oncall`

### 3d. Payment Queue Backlog

What to do:

1. Check SQS and the DLQ.
2. Restart the payment worker service if it is crashing.

```text
aws ecs update-service --cluster swiggy-workers --service payment-worker --force-new-deployment
```

3. If the gateway is rate-limiting, pause new orders at the ALB or application layer and keep payment retries in the queue.

Success signal:

- Queue depth decreases steadily within 10 minutes
- DLQ growth stops

Owner:

- `#payments-oncall`

## STEP 4: ROLLBACK

Rollback criteria:

- ALB 5xx rate stays above 20% for more than 5 minutes
- CPU and DB metrics do not improve after scaling
- There was a recent application deploy and no database migration is involved

Rollback command:

```text
aws ecs update-service \
  --cluster swiggy-prod \
  --service api \
  --task-definition swiggy-api:PREVIOUS_STABLE_VERSION
```

Warning:

- Never roll back the database schema during an incident. Roll back application code only.

## STEP 5: POSTMORTEM TEMPLATE

Fill this in within 24 hours.

### Timeline

- Record the first alert time, first user impact, mitigation steps, and recovery time using CloudWatch timestamps.

### Root Cause

- State the single deepest technical cause. Example: `PgBouncer pool_size was too low for the payment hold time at peak traffic.`

### Impact

- Duration:
- Affected users:
- Failed requests:
- Estimated revenue loss:

### What Worked

- Note the actions that reduced impact and the metrics that improved.

### Action Items

| Item | Owner | Due Date | Status |
|---|---|---|---|
| Increase pool capacity | | | |
| Add cache warmup tests | | | |
| Add gateway timeout alerts | | | |
| Review promo atomicity | | | |
