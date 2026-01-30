Db2 to AWS Migration & Self-Service DBA Platform – Detailed Design

This document consolidates a near-zero-downtime Db2 migration strategy and a governed self-service
database operations platform. It incorporates native Db2 backup/restore using S3, CDC using AWS DMS
or IBM InfoSphere (Q Replication), and a GitOps-driven DBA governance model.

===============
PART 1 – Near-Zero Downtime Db2 Migration
===============

1. Constraints & Goals
---------------------
Production (8TB):
- ≤ 30 minutes downtime
- Zero data loss
- Rollback ≤ 15 minutes
- 100% data validation

Data Warehouse (40TB):
- Complete migration ≤ 48 hours
- Minimal analytics disruption
- Ongoing Production → Warehouse replication

Source: Db2 on AIX
Target: AWS-hosted Db2 (RDS for Db2 or Db2 on EC2)

===============
2. Tooling Options
-----------------
Initial Load:
- Native Db2 BACKUP / RESTORE
- db2move / EXPORT / LOAD
- Amazon S3 (staging)

CDC:
- AWS Database Migration Service (DMS)
- IBM InfoSphere Data Replication (IIDR / Q Replication)

===============
3. High-Level Architecture
-------------------------
Source Db2 (AIX)
 → Native Backup → S3 → Restore to Target Db2
 → CDC (DMS / IIDR) → Target Db2 (Production)
 → CDC → Target DW (staging schema)

===============
4. Phase 0 – Preparation
-----------------------
- Inventory schemas, tables, PKs, indexes, LOBs
- Run db2look for DDL extraction
- Validate Db2 version compatibility
- Enable archive logging
- Take baseline full backup and store in S3
- Provision AWS Db2 with adequate CPU, memory, io2/gp3 IOPS
- Configure IAM role for RDS with S3 read access

===============
5. Phase 1 – Initial Full Load
-----------------------------
Option A – Native Backup/Restore (Preferred):
- Take ONLINE backup on source
- Upload encrypted backup to S3
- Restore into RDS Db2 using RDS stored procedures

Option B – Parallel Export/Load:
- Use db2move or EXPORT
- Split large tables by PK ranges
- Parallel LOAD into target

===============
6. Phase 2 – Continuous CDC
--------------------------
Option 1 – AWS DMS:
- Configure replication instance
- Full load disabled (CDC-only)
- Log-based CDC from Db2 AIX → Target Db2

Option 2 – IBM IIDR / Q Replication:
- Capture agent on source
- Apply agent near target
- Enterprise-grade, low-latency replication

Monitor:
- Replication lag
- Log read rates
- Error handling

===============
7. Validation – 100% Data Accuracy
---------------------------------
Structural:
- Schema compare (db2look source vs target)

Row-level:
- COUNT(*) per table
- Deterministic checksum per table (PK ordered, chunked)

Tools:
- Python + ibm_db
- Parallel workers
- Store results as audit artifacts

===============
8. Cutover Runbook (≤ 30 minutes)
--------------------------------
- Enable maintenance mode (read-only)
- Stop batch & ETL jobs
- Wait for CDC lag = 0
- Final checksum on critical tables
- Promote target to read-write
- Update DNS / connection strings (TTL < 60s)
- Smoke tests
- Resume traffic

===============
9. Rollback (≤ 15 minutes)
-------------------------
- No writes allowed during cutover
- Flip DNS / connection back to source
- Resume writes on original DB
- Target remains read-only

===============
10. Data Warehouse (40TB) Migration
----------------------------------
- Parallel unload from source (prefer standby/snapshot)
- Stage to S3 (or Snowball if network limited)
- Parallel LOAD into DW
- Disable indexes during load
- Rebuild indexes, RUNSTATS
- Apply CDC delta
- Validate key tables

===============
11. Post-Migration Replication
------------------------------
- Continuous CDC from Production → DW
- Apply to staging schema
- Merge during off-peak hours
- Maintain async replication to avoid analytics impact

===============
PART 2 – Self-Service Database Operations Platform
===============

12. Goal
--------
Enable developers to run approved SQL safely while DBAs retain governance, auditability, and security.

===============
13. Architecture
----------------
Developer → Git (SQL as Code)
 → CI Pipeline (Lint, Dry-run, Tests)
 → DBA Approval Gate
 → Flyway Execution
 → Audit Logs (S3 / ELK / Audit DB)

===============
14. Workflow
-------------
1. Developer creates SQL migration or task
2. Opens PR with Jira link & rollback
3. CI runs:
   - SQL lint
   - Dangerous statement detection
   - EXPLAIN & impact analysis
   - Test DB execution
4. DBA reviews & approves
5. Manual prod approval job
6. Flyway executes using service account
7. Audit record stored

===============
15. Flyway Standards
-------------------
- Versioned migrations: VYYYYMMDD__desc.sql
- Repeatable scripts for maintenance
- flyway validate enforced
- Rollback scripts mandatory where applicable

===============
16. Governance & Safety
----------------------
- Branch protection & required DBA approvers
- Secrets in AWS Secrets Manager / Vault
- No direct prod DB access for developers
- Time-window enforcement for heavy ops
- Emergency rollback runbook owned by DBA

===============
17. Audit Logging
----------------
Captured:
- Commit hash
- SQL file
- Author
- CI job ID
- Start/end time
- Rows affected
- Result

Stored:
- S3 (immutable)
- Central audit DB or ELK

===============
18. Benefits
------------
- Ticket turnaround: hours → minutes
- 60–70% DBA workload reduction
- Full compliance & traceability
- Safer, faster database changes

===============
19. 90-Day Implementation Roadmap
--------------------------------
Weeks 0–2: Repo, Flyway baseline, CI runners
Weeks 2–4: Static checks, test DB automation
Weeks 4–6: Pilot teams
Weeks 6–10: Production gates, Jira integration
Weeks 10–12: Audit hardening & training
