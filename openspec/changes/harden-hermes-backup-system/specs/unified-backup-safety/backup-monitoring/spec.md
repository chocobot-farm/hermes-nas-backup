## Purpose

Defines out-of-guest observability for both backup paths, so that a path which silently stops producing recovery points is detected within the missed-backup objective rather than at the moment a restore is needed.

## ADDED Requirements

### Requirement: Authoritative monitoring runs outside the guest

Authoritative backup monitoring SHALL run outside the Hermes guest. Hermes-emitted success messages SHALL NOT determine backup health. The monitoring destination SHALL also be outside PVE, PBS, and the NAS, and SHALL have a named on-call owner.

#### Scenario: Guest cannot assert backup health

- **WHEN** the guest emits a success signal
- **THEN** the authoritative monitoring state is unchanged by it

#### Scenario: Operator answers last success without logging in

- **WHEN** an operator is asked when each backup path last succeeded
- **THEN** they can answer without logging into PVE, PBS, or the NAS

### Requirement: Missed schedules alert as loudly as failures

Operators SHALL be alerted on a missed schedule on either path, not only on a reported failure, so that a job which never ran is as visible as one that failed. A suppressed schedule SHALL raise an alert within the missed-backup detection objective.

#### Scenario: Suppressed schedule

- **WHEN** a scheduled PVE or Synology backup does not run at all
- **THEN** an alert is raised within the detection objective

#### Scenario: Alerting is proven by deliberate failure

- **WHEN** one job on each path is deliberately failed
- **THEN** the corresponding alert arrives at the monitoring destination

### Requirement: Defined alert conditions are covered

Operators SHALL be alerted for: missed PVE or Restic schedules; nonzero exporter, SSH, Restic, PVE, PBS, verification, prune, garbage-collection, or sync jobs; changed SSH or PBS fingerprints; live-export size or duration outside established bounds; loss of the NFS mount inside the PBS VM or a datastore path resolving to a local directory rather than the mounted export; an unexpected change in the owner of a PBS backup group; overlapping tasks; Synology shared-folder quota pressure; absent or expired protected snapshots; stale or failed off-site replication; and failed restore drills.

#### Scenario: Export size leaves its established bounds

- **WHEN** a live export's size or duration falls outside the established bounds
- **THEN** an alert is raised, and raising the bound requires review rather than an automatic unbounded retry

#### Scenario: NFS mount is lost inside the PBS VM

- **WHEN** the datastore path inside the PBS VM resolves to a local directory rather than the mounted export
- **THEN** an alert is raised

#### Scenario: Backup group ownership changes

- **WHEN** the owner of a PBS backup group changes unexpectedly
- **THEN** an alert is raised

### Requirement: Defined operational questions are answerable

Operators SHALL be able to determine: last attempt and last success for each backup path; PVE VMID, PBS datastore, backup mode, and backup snapshot time; the PVE client-encryption fingerprint but not the key value; deployed Restic image digest and container exit code; latest Restic snapshot ID, host, tag, size, and duration; the run manifest for any recent attempt including attempts that created no snapshot; the received byte length and stream digest trend across runs; last PBS verification, prune, and garbage-collection results; last Restic check and prune results; the current NAS protected-snapshot horizon; the latest successful off-site copy; and the latest successful isolated restore from each required path.

#### Scenario: Anomalous export trend is visible

- **WHEN** the received byte length changes abruptly or the stream digest repeats identically across runs
- **THEN** the trend is visible to operators from retained run data

#### Scenario: Encryption fingerprint without the key

- **WHEN** an operator inspects the recorded PVE client-encryption state
- **THEN** the fingerprint is available and the key value is not

### Requirement: Logs are UTC and free of secret material

Logs SHALL use UTC and SHALL NOT contain private keys, passwords, API token secrets, authorization headers, full process environments, or decrypted backup content.

#### Scenario: Log review finds no secret

- **WHEN** monitoring payloads and container logs are reviewed
- **THEN** they contain UTC timestamps and no secret value, environment dump, or decrypted backup content
