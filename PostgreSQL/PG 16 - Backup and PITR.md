# 16 — Backup and PITR

🔑 Two layers: logical dumps (`pg_dump`) for portability, physical base backup + WAL archive for point-in-time recovery.

Source: https://www.postgresql.org/docs/current/backup.html

## Logical: pg_dump / pg_restore
```bash
pg_dump -Fc -d app -f app.dump          # custom format (compressed, parallelizable)
pg_dump -Fd -j 8 -d app -f app.dir      # directory format, parallel
pg_restore -d app_new -j 8 app.dump
```
| Format | Restore |
|---|---|
| `-Fp` plain SQL | `psql -f` |
| `-Fc` custom | `pg_restore` (selective, parallel restore) |
| `-Fd` directory | `pg_restore -j N` (parallel dump + restore) |
| `-Ft` tar | `pg_restore` |

- ⚠️ `pg_dump` is per-database; for whole cluster use `pg_dumpall` (roles, tablespaces).
- 💡 Logical dumps are **portable across major versions**. Physical backups are not.

## Physical: pg_basebackup + WAL Archive
```bash
pg_basebackup -h primary -D /backups/base -Ft -z -P -X stream -c fast
```
- Streams the data directory + minimal WAL.
- For PITR, you also need an ongoing WAL archive.

```ini
# postgresql.conf on primary
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /wal-archive/%f && cp %p /wal-archive/%f'
```

## Point-in-Time Recovery (PITR)
1. Restore base backup to a fresh data directory.
2. Drop a `recovery.signal` file (or use `pg_basebackup -R`).
3. Configure recovery target.

```ini
# postgresql.conf on the restored node
restore_command       = 'cp /wal-archive/%f %p'
recovery_target_time  = '2026-04-28 12:00:00+00'
recovery_target_action = 'promote'
```
- Postgres replays WAL up to the target, then promotes.

## Higher-Level Tools
| Tool | Notes |
|---|---|
| **pgBackRest** | Parallel, compression, encryption, retention, S3. Production standard. |
| **Barman** | Catalog-based, hot-standby aware. |
| **WAL-G** | Cloud-native (S3/GCS/Azure), delta backups. |

## Verify or It Didn't Happen
- 💡 Restore drills on a schedule. An untested backup is a hope, not a backup.
- ⚠️ `pg_dump` doesn't capture replication slots, role passwords (without `--no-role-passwords` flip), or LSN — it's a logical snapshot only.

## Tags
[[PostgreSQL]] [[Backup]] [[PITR]] [[WAL]]
