# Swiggy is Down - Scale Simulation and Incident Architecture

This repository documents how a SwiftEats-style monolith fails under a World Cup Final promo spike, and what to build instead.

## Scenario

India vs Pakistan, World Cup Final, 8 PM IST. A 50% promo goes to 180 million users. Roughly 14 million open the app, and the backend is a single Node.js server talking to a single PostgreSQL database.

## Document Summary

| Document | What it Contains |
|---|---|
| [docs/FAILURE-CASCADE.md](docs/FAILURE-CASCADE.md) | Traffic math, hard capacity limits, failure triggers, and the incident timeline |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | The current monolith, the redesigned multi-tier architecture, and the component justification table |
| [docs/COST-ESTIMATE.md](docs/COST-ESTIMATE.md) | Baseline and peak AWS cost calculations using real instance types and formulas |
| [docs/RUNBOOK.md](docs/RUNBOOK.md) | A 5-step incident runbook with detection, triage, response, rollback, and postmortem template |

## Key Findings

- PostgreSQL is the first hard stop: with 30% payment traffic and 800 ms hold time, the pool exhausts at about 394 RPS.
- The simplified launch spike still reaches 500,000 RPS, which is far beyond a single Node.js process.
- Synchronous payment calls turn a short checkout path into an 800 ms connection hostage situation.
- Static assets served by Node.js can saturate the origin NIC before the API itself fully collapses.
- Without traces, correlation IDs, and structured logs, the outage becomes a guessing game instead of an incident.

## Architecture Overview

The redesigned system moves static traffic to CloudFront, fronts dynamic traffic with an ALB, scales Node.js horizontally, caches hot reads in Redis, pools database connections with PgBouncer, and pushes payments onto SQS-backed workers. The result is a system where the database and payment gateway are no longer on the user-facing request path.

## Tech Stack Context

This analysis covers Node.js, PostgreSQL, Redis, AWS CloudFront, AWS ALB, AWS SQS, RDS, ElastiCache, and EC2.
