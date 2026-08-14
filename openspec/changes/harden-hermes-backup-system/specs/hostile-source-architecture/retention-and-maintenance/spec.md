## Purpose

Separates the authority that writes backups from the authority that destroys them, and defines the retention policy and the reviewed procedure that every destructive maintenance run must follow.

## ADDED Requirements

### Requirement: Routine backup identities hold no deletion authority

Routine backup identities SHALL NOT possess deletion authority. The separation that matters is that the identity which writes backups cannot destroy them; it MAY be achieved either by deferring deletion to an isolated maintenance workstation or by having the backup server perform its own retention under its own identity while the routine write token holds no deletion right.

#### Scenario: Write identity attempts deletion

- **WHEN** a routine backup identity attempts to forget, prune, or expire recovery points
- **THEN** the operation is refused

#### Scenario: Server-side retention satisfies the separation

- **WHEN** the backup server performs its own prune and garbage collection while the write token holds no deletion right
- **THEN** the separation requirement is met

### Requirement: Prune, garbage collection, and verification are distinct scheduled operations

Prune removes snapshot references, garbage collection later removes unreferenced chunks, and verification reads chunks to detect corruption. All three SHALL be scheduled apart from each other and from the backup window. Retention that appears to exceed the minimum protection window SHALL be checked against measured storage capacity rather than assumed, because the storage platform may misreport free space to the backup server.

#### Scenario: Retention exceeds the minimum on paper

- **WHEN** a retention policy nominally keeps more history than the minimum window requires
- **THEN** it is validated against measured shared-folder capacity before being relied on

#### Scenario: Two maintenance operations overlap

- **WHEN** any two of prune, garbage collection, verification, and backup are scheduled to overlap
- **THEN** the schedule is corrected

### Requirement: Application-path retention runs from an isolated maintenance path

For the application repository, retention and pruning SHALL run only from the isolated maintenance workstation through a temporary full-access path. The operator SHALL inspect repository activity and backup anomalies before enabling deletion.

Repositories receiving append-only writes SHALL use time-window retention options rather than only count-based options. An initial policy is:

```text
keep all snapshots within 14 days
keep daily snapshots within 90 days
keep weekly snapshots within 1 year
keep monthly snapshots within 5 years
```

The exact policy is a deployment decision, but it SHALL use applicable `--keep-within*` options and SHALL preserve at least one known-good recovery point older than the maximum credible detection delay.

#### Scenario: Count-based retention only

- **WHEN** a proposed policy expresses retention only as snapshot counts
- **THEN** it is replaced by time-window `--keep-within*` rules

#### Scenario: Detection delay exceeds recent history

- **WHEN** compromise is detected later than the newest retained recovery points can predate
- **THEN** the retention policy is corrected so at least one known-good point older than the maximum credible detection delay is preserved

### Requirement: Destructive maintenance follows a reviewed procedure

Every destructive maintenance run SHALL:

1. confirm the normal ingest job is stopped;
2. confirm current immutable snapshots or off-site recovery points exist;
3. run `forget --dry-run` first;
4. review unexpected source times, sizes, tags, and snapshot density;
5. run a repository check appropriate to repository size and risk;
6. record the approved deletion plan;
7. run retention and pruning;
8. verify repository health afterward; and
9. close the maintenance endpoint and confirm routine append-only behavior.

#### Scenario: Dry run reveals an anomaly

- **WHEN** the dry run shows unexpected source times, sizes, tags, or snapshot density
- **THEN** deletion does not proceed until the anomaly is explained

#### Scenario: No protected copy exists

- **WHEN** neither a current immutable snapshot nor an off-site recovery point can be confirmed
- **THEN** the destructive run does not proceed

#### Scenario: Maintenance completes

- **WHEN** retention and pruning finish
- **THEN** repository health is verified, the maintenance endpoint is closed, and routine append-only behavior is confirmed
