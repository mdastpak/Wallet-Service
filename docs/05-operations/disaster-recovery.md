# Disaster Recovery

## Overview

Comprehensive disaster recovery strategy ensuring business continuity with minimal data loss.

---

## Recovery Objectives

- **RPO** (Recovery Point Objective): 0 seconds (synchronous replication)
- **RTO** (Recovery Time Objective): < 5 minutes (automated failover)

---

## Backup Strategy

### Database Backups

**Oracle Data Guard**:
- **Primary**: Active database (us-east-1)
- **Standby**: Synchronous replica (us-west-2)
- **Failover**: Automatic with Data Guard Broker
- **Frequency**: Continuous replication

**Snapshot Backups**:
- **Full**: Daily at 2 AM UTC
- **Incremental**: Every 6 hours
- **Retention**: 30 days local, 365 days archived (S3 Glacier)

---

### Application State

**Stateless Services**: No state to backup (Kubernetes redeploys)

**Redis Data**:
- **Persistence**: AOF + RDB snapshots
- **Replication**: Redis Sentinel (3-node cluster)
- **Backup**: Hourly RDB snapshots to S3

---

## Failover Procedures

### Automated Failover (Database)

1. Oracle Data Guard detects primary failure
2. Standby promoted to primary (< 1 minute)
3. DNS updated to new primary endpoint
4. Application connections re-established

### Application Failover (Kubernetes)

1. Liveness probe detects pod failure
2. Kubernetes kills unhealthy pod
3. New pod scheduled in healthy node/AZ
4. Service mesh routes traffic to healthy pods

---

## Testing

- **DR Drill**: Quarterly full failover test
- **Backup Restoration**: Monthly verification
- **RTO/RPO Validation**: Measured during drills

---

## Recovery Procedures

See [Runbooks](./runbooks.md) for detailed recovery playbooks.
