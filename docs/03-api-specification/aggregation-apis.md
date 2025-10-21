# Aggregation APIs

## Overview

Real-time balance aggregation APIs for users, businesses, and global totals using CQRS pattern with Redis caching.

## Endpoints

### 1. User Balance Aggregate
**GET /v1/aggregates/users/{user_id}/balance**

Returns sum of all wallet balances for a user across all currencies.

### 2. Business Balance Aggregate
**GET /v1/aggregates/businesses/{business_id}/balance**

Returns sum of all wallet balances for a business (all users under that business).

### 3. Global System Balance
**GET /v1/aggregates/global/balance**

Admin-only endpoint showing total system balances. Requires `SYSTEM_ADMIN` role.

## Caching Strategy

- **Cached Balances**: Updated via Kafka events (eventual consistency)
- **TTL**: 5 minutes
- **Force Refresh**: `?refresh=true` query parameter triggers recomputation

See [Data Flows](../01-architecture/data-flows.md) for aggregation implementation details.
