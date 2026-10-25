# Implementation Plan: F15. Backup and Operations

**Prerequisites:**
- F01 Project Foundation implemented: config module, `ClockPort` and `FixedClock`, logger, Drizzle client and migrator, route descriptor and route builder, Awilix container, role entrypoint and healthcheck, Dockerfile, and the local production and e2e compose files
- PostgreSQL 17 client tools (`pg_dump`, `pg_restore`) in the backend runtime image, from the PGDG apt repository
- Environment variables: the F15 backup and heartbeat variables listed in the spec's component table, all with defaults; `BACKUP_CLOCK_FIXED_AT` and `BACKUP_RUN_ONCE` only for tests and manual runs
- Docker Engine with Compose v2 on the host for the restore script and the container-level checks

### Stage 1: Domain Rules and Persistence

**1. Configuration additions** - Extend the configuration module with the backup role, the backup schedule, retention, directory, and clock-override settings, and the heartbeat timing and runtime-directory settings. Retire the worker ready-marker variable and validate everything the same way as F01's variables.

**2. Operations domain** - Add the backup file-name value object, the daily due rule, the retention rule, the backup attempt record, and the heartbeat staleness rule, along with the repository interfaces and errors for backup attempts and the worker heartbeat.

**3. Operations schema and migration** - Define the backup status and worker heartbeat tables in the Drizzle schema module and generate the migration so F01's runner applies it on boot.

**4. Persistence adapters** - Implement the backup attempt and worker heartbeat repositories over Drizzle, covering the latest-success and latest-attempt reads and the single-row heartbeat upsert.

### Stage 2: Backup Pipeline

**5. Backup ports and adapters** - Define the dumper, storage, and heartbeat pause-switch ports. Implement the `pg_dump` adapter so credentials stay out of process arguments, the filesystem storage with temp-file commit by rename, the pinned clock, and the backup liveness file.

**6. Backup use cases** - Implement the single backup run (dump to temp, commit, measure, apply retention, record) with its failure path, which cleans up the temp file, records the failure, and logs the error. Then implement the scheduled tick that only runs a backup when one is due.

**7. Backup role and scheduler job** - Add the timer-driven scheduler job, which also supports the one-shot mode, and the backup role boot (connect, clear stale temp files, run the loop or a single tick, shut down gracefully). Wire them into the container and the role dispatcher, choosing the pinned clock only when the override is set.

### Stage 3: Worker Heartbeat and Status API

**8. Heartbeat use cases and job** - Implement heartbeat recording, which respects the pause switch, and the worker health verdict. Add the heartbeat timer job and start it from the worker role in place of the ready marker, with the PID file and both pause-switch implementations wired according to the test-routes flag.

**9. Role-aware healthcheck** - Extend the healthcheck entrypoint so the worker role reports unhealthy on a stale heartbeat and terminates the worker process once it passes the kill threshold, and the backup role checks its liveness file. The api check stays unchanged.

**10. Operations status endpoint** - Implement the status use case, handler, and route that combine the latest successful backup, the latest attempt's error, and the worker's last-seen time, and register the route so it appears in the OpenAPI document.

**11. Architecture gate update** - Teach the architecture check about the new interface jobs folder and register the backup entrypoint with the orphan checker, so the clean-arch gates accept the new modules.

### Stage 4: Containers and Operator Tooling

**12. Runtime image changes** - Install the PostgreSQL 17 client tools in the runtime stage and pre-create the backups and worker runtime directories owned by the non-root user.

**13. Compose wiring** - Add the backup service and the backups volume, mounted read-write for backup and read-only for the database, to the local production and e2e compose files. Give the worker an init process, the tmpfs runtime directory, and its healthcheck timings, and set the `unless-stopped` restart policy on api, worker, db, and backup.

**14. Restore script** - Write the repository-root restore script. It validates the requested archive before touching any service, stops the services that write to the database, restores in a single transaction, always restarts the services, waits for them to be healthy, and reports the documented exit codes.

**15. Operator documentation and environment template** - Document every new variable in `.env.example`, marking the clock override and one-shot mode as test or operator only, and add a backups and restore section to the README.
