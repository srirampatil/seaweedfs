# Lifecycle policy processing subsystem

The user understands the complete lifecycle system:

1. Rule evaluation: EvaluateAction gates each action kind differently — ExpirationDays uses ModTime+TTL, ExpirationDate uses calendar date, NoncurrentDays uses SuccessorModTime (from ExtNoncurrentSinceNs), NewerNoncurrent uses NoncurrentIndex, ExpiredDeleteMarker requires sole-survivor marker, AbortMPU uses init time.

2. Five-component architecture: Engine (compiles rules into snapshots with delay-grouped indexes) → Reader (single meta-log subscription fanning to per-shard channels) → Router (matches events to rules, produces Match tuples with DueTime + Identity) → DailyRun (orchestrates shards, cursors, replay + walker, dispatch via LifecycleDelete RPC) → S3 API Server (identity-CAS, object-lock check, dispatch to S3 operations).

3. Two-pronged evaluation: event-driven meta-log replay for time-based rules + periodic bucket walker for bootstrap, ExpirationDate, ExpiredDeleteMarker, and NewerNoncurrent rules.

4. Sharding: 16 shards with per-shard cursors (TsNs, RuleSetHash, PromotedHash, LastWalkedNs). Parallel replay with shared rate.Limiter for dispatch throttling.

5. Identity CAS: EntryIdentity {MtimeNs, Size, HeadFid, ExtendedHash} prevents dispatching on modified objects — server re-fetches before acting.

6. Server-side dispatch: versioning-aware (Enabled→delete marker, Suspended→delete null+marker, Off→delete), noncurrent version deletion, MPU abort, object-lock checking.

7. Fast-path: volume TTL can be stamped at write time for simple Expiration.Days rules on non-versioned buckets.