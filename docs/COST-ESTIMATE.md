# AWS Cost Estimate

Assumptions:

- 720 hours per month
- Public on-demand prices approximated from AWS pricing pages
- Baseline represents steady-state traffic
- Peak represents a 4-hour World Cup surge

## 1) Baseline Monthly Cost

### EC2 for Node.js

- 4 x t3.medium at $0.0416 / hour

```text
$0.0416 x 4 x 720 = $119.81 / month
```

### RDS PostgreSQL Primary

- 1 x db.r6g.large at $0.182 / hour

```text
$0.182 x 720 = $131.04 / month
```

### RDS Read Replicas

- 2 x db.r6g.large at $0.182 / hour

```text
$0.182 x 2 x 720 = $262.08 / month
```

### ElastiCache Redis

- 3 x cache.r6g.large at $0.166 / hour

```text
$0.166 x 3 x 720 = $358.56 / month
```

### Application Load Balancer

- Base ALB charge: $16.20 / month
- LCU estimate: about $40 / month

```text
$16.20 + $40.00 = $56.20 / month
```

### CloudFront

- 10 TB monthly transfer = 10,000 GB
- $0.0085 / GB

```text
$0.0085 x 10,000 = $85.00 / month
```

### SQS

- 1,000,000 messages/day x 30 days = 30,000,000 messages/month
- $0.40 / million requests

```text
30 x $0.40 = $12.00 / month
```

### Baseline Total

```text
$119.81 + $131.04 + $262.08 + $358.56 + $56.20 + $85.00 + $12.00 = $1,024.69 / month
```

## 2) Peak Event Extra Cost

### Additional EC2 Capacity for 4 Hours

- Scale from 4 instances to 20 instances
- Extra 16 x t3.2xlarge at $0.3328 / hour

```text
$0.3328 x 16 x 4 = $21.30
```

### Temporary RDS Scale-Up for 4 Hours

- Upgrade primary to db.r6g.4xlarge at $1.027 / hour

```text
$1.027 x 4 = $4.11
```

### CloudFront Surge Transfer

- Extra 50 TB transfer = 50,000 GB
- $0.0085 / GB

```text
$0.0085 x 50,000 = $425.00
```

### Peak Extra Total

```text
$21.30 + $4.11 + $425.00 = $450.41
```

## 3) Business Justification

If the platform loses revenue at roughly ₹4.2 crore per minute during a major outage, then a 45-minute incident costs about ₹189 crore. Against that loss, a steady-state AWS bill of roughly $1,024.69 per month and a peak-night burst cost of about $450.41 is not a discretionary expense; it is the cheaper option by many orders of magnitude.
