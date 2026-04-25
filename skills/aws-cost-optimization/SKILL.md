---
name: aws-cost-optimization
description: AWS cost analysis and optimization knowledge for RDS, EC2 Auto Scaling, Reserved Instances, and Savings Plans. Use when reviewing AWS spend, purchasing RIs, or analyzing scaling behavior.
---

# AWS Cost Optimization — ap-southeast-2

## Account Structure

Three unique AWS accounts (multiple profiles point to same accounts):

| Account | Profiles |
|---|---|
| prod | prod, production, default, dr |
| stg | ass_stg, staging, stg, demo |
| bi | bi, ass_bi |
| lab | ass_lab, lab |

Profile `personal` is excluded from analysis by convention.

---

## RDS Instances

All instances: Aurora Standard MySQL, ap-southeast-2, encrypted, no MultiAZ.

### prod account
- `prod-mysql80-instance-1`
- `prod-mysql80-instance-2`
- `prod-mysql80-marketplace-blackhole-instance-1`
- `prod-mysql80-marketplace-blackhole-instance-2`

### stg account
- `stg-mysql8`
- `stg-mysql8-ro`
- `demo-mysql8-t4`
- `metabase-private-small-mysql8`

---

## Monthly RDS Spend starting from first day of 3 months ago

| Account | Jan | Feb | Mar | Apr (partial) |
|---|---|---|---|---|
| prod | $2,127 | $2,024 | $2,414 | $1,372 |
| stg | $447 | $473 | $881 | $714 |

---

## Reserved Instances — Key Facts

### RDS RIs have size flexibility (within same instance class type)
Source: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithReservedDBInstances.html

- A `db.r7g.large` RI **can** cover a `db.r7g.xlarge` (same class type, different size)
- A `db.r7g.large` RI **cannot** cover a `db.r6g.large` (different class type)
- Flexibility is based on **normalized units**:

| Size | Normalized units (Single-AZ) |
|---|---|
| medium | 2 |
| large | 4 |
| xlarge | 8 |
| 2xlarge | 16 |

### Practical example for this environment
7x `db.r7g.large` RIs = 28 normalized units, which covers:
- 2x `db.r7g.xlarge` = 16 units
- 3x `db.r7g.large` = 12 units
- Total = 28 units ✅ — exact match, no waste

### Supported engines for size flexibility
RDS for MySQL, MariaDB, PostgreSQL, Db2, Oracle BYOL. Also applies to Aurora — see Aurora docs.

### Last expired RIs (prod account)
- `db.r7g.large` aurora-mysql — expired Mar 2025 (ri-2025-03-19)
- `db.t4g.medium` aurora-mysql — expired Mar 2025 (ri-2025-03-12)
- All instances have been on-demand since March 2025

---

## Useful CLI Snippets

### List all RDS instances across profiles
```bash
for profile in prod ass_stg bi; do
  echo "=== $profile ==="
  aws rds describe-db-instances \
    --profile "$profile" \
    --region ap-southeast-2 \
    --query 'DBInstances[*].{ID:DBInstanceIdentifier,Class:DBInstanceClass,Engine:Engine,Status:DBInstanceStatus,Encrypted:StorageEncrypted}' \
    --output table
done
```

### Monthly RDS cost per account
```bash
aws ce get-cost-and-usage \
  --profile prod \
  --time-period Start=2026-01-01,End=2026-04-17 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Relational Database Service"]}}' \
  --query 'ResultsByTime[*].{Start:TimePeriod.Start,Cost:Total.UnblendedCost.Amount}' \
  --output table
```

### Check active/expired RDS RIs
```bash
aws rds describe-reserved-db-instances \
  --profile prod \
  --region ap-southeast-2 \
  --query 'ReservedDBInstances[*].{ID:ReservedDBInstanceId,Class:DBInstanceClass,State:State,End:StartTime}' \
  --output table
```
