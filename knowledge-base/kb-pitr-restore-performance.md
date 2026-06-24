# KB: PITR restore performance

## Introduction

**Problem Statement:**
A Percona Backup for MongoDB PITR restore was extremely slow during oplog replay despite the host appearing underutilized. The snapshot phase completed quickly, but replaying the time-series oplog slices crawled at about 1.2x realtime, making restore impractically long.

**Environment:**
- Technology / Service: Percona Backup for MongoDB 2.12.0 / MongoDB (unspecified)
- Component: PITR oplog replay, time-series bucket collections (`*.system.buckets.data`)
- Severity: not specified

---

## Process

### 1. Identifying the Problem
A user reported a large (~20GB compressed) time-series database with daily logical snapshots and PITR enabled. The restore was run locally with no S3, no network, no encryption, yet PITR oplog replay was extremely slow.

### 2. Diagnosis
The restore metrics showed:

```bash
# Observed restore metrics
snapshot_recovery: 9m9s
oplog_replay_rate: ~1.2x realtime
estimated_restore_time: 17h for 20h of PITR slices
```

System utilization was low during restore:

- MongoDB CPU: ~7-9%
- PBM CPU: ~3-4%
- RAM available: ~23 GiB
- Swap: full but not actively swapping
- I/O wait: ~6%

Post-analysis showed the main delay was write latency, not compute. MongoDB logged slow time-series bucket writes with `majority` write concern and `waitForWriteConcernDurationMillis` commonly around 180-408ms.

This workload was applying many small `applyOps` operations against time-series bucket collections such as `*.system.buckets.data`.

### 3. Root Cause
The PITR restore was latency-bound and replay-order-bound. Oplog replay was stalled by small time-series bucket writes under majority write concern, causing write amplification and preventing PBM/Mongo from using full CPU or disk bandwidth.

### 4. Fix Applied
No single software fix was available in PBM 2.12.0 for oplog replay parallelism. The mitigation path included:

```bash
# Mitigation actions
# Close MongoDB Compass or any active collection scans during restore
# Compare with mongorestore replay behavior
mongorestore --replayOp /path/to/oplog.bson
```

Additional actions:

- Close MongoDB Compass to eliminate extra collection scans and counts.
- Confirm that the snapshot phase is already fast, focusing tuning on oplog replay only.
- Note that PBM had no obvious oplog replay parallelism knob in version 2.12.0.

### 5. Verification
Verification should focus on improved restore throughput and reduced write concern latency:

```bash
# Verification step
# Examine restore progress and MongoDB logs for waitForWriteConcernDurationMillis
```

If restore progress accelerates after removing external load or comparing with `mongorestore --replayOp`, it confirms the bottleneck is latency-bound oplog replay rather than CPU or I/O saturation.

---

## Conclusion

**Summary:**
A local PITR restore with Percona Backup for MongoDB was slow because oplog replay was bound by many small time-series bucket writes and majority write concern latency. The snapshot restore itself was quick, but ordered replay and time-series write amplification made the restore effectively unusable.

**Prevention:**
- Monitor time-series write latency and `waitForWriteConcernDurationMillis` during restores.
- Avoid running interactive tools like MongoDB Compass during critical restore operations.
- Test alternate restore paths such as `mongorestore --replayOp` for latency-sensitive workloads.

---

## Tags

`mongodb` `pitr` `pbm` `performance`
