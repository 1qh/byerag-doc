# backups-pg-dump-and-restore-drill

Daily `pg_dump` of the Convex-backing Postgres volume. Dump file encrypted at rest, written to a separate disk path on the host. Restore drill runs monthly on a fresh compose stack.

## Mechanism

`apps/backend/scripts/backup.sh`:

```
TS=$(date -u +%Y%m%dT%H%M%SZ)
docker compose exec -T postgres pg_dump -U convex -d convex_self_hosted -Fc \
  | age -r "$BACKUP_AGE_PUBKEY" -o "$BACKUP_DIR/byerag-$TS.dump.age"
```

Cron entry on the host runs the script daily at 03:00 local. Retention: 30 daily + 12 monthly snapshots.

## Restore drill

`apps/backend/scripts/restore-drill.sh`:

1. Spin a parallel compose stack with a different project name (`byerag-restore-test`).
2. Pipe `age --decrypt` of the latest dump into `pg_restore -d convex_self_hosted` against the parallel Postgres.
3. Boot the parallel Convex backend pointed at the restored DB.
4. Assert: `messages` count matches the production DB ± N rows (where N is the in-flight window during dump).
5. Tear down the parallel stack.

Runs monthly via cron. Failure → `ledger.jsonl` + Convex `auditLogs` entry + (P6+) alerting.

## Beats

- **Volume snapshot (LVM / ZFS)**: assumes specific filesystem; doesn't survive host migration.
- **Logical replication to a hot standby**: more ops surface; overkill for internal team.
- **Convex's own backup tool**: managed-Convex has it; self-host does not, currently.

## Real cost

- Dump duration: O(DB size) — small at internal scale.
- Disk space for retained dumps.
- One restore drill failure → backups are useless until fixed.

## Gotcha for Claude

- `pg_dump` against a live DB needs `--no-tablespaces`; Convex doesn't use tablespaces but flag is defensive.
- Encryption: `age` is the chosen tool (single-binary, modern, reviewed). `BACKUP_AGE_PUBKEY` lives in operator memory; private key on offline media (operator backup).
- Restore-drill failure on count mismatch isn't necessarily a regression — high-write windows can cause drift. Allow `±5%` row count tolerance before alerting.
- Backup target disk: separate from Postgres data disk. Same disk failure mode = both gone.
- `_storage` blobs are NOT in the Postgres dump unless Convex stores them inline; self-host Convex stores blobs in a separate storage backend (default: local FS). Backup script must also tar that dir per `apps/backend/scripts/backup.sh` step 2.
