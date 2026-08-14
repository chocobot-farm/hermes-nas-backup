## Purpose

Defines how the system is changed and recovered over time: deliberate image upgrades with a rollback window, credential rotation that never leaves the system without a working path, incident response per compromised component, and the service objectives the whole design is measured against.

## ADDED Requirements

### Requirement: Image upgrades are deliberate and reversible

Upgrade SHALL pause schedules, pull and verify the new digest, create candidate containers, run snapshots/check/test backup, preserve the old digest and stopped containers during a rollback window, then switch fixed production names and resume scheduling. Restic repository-format compatibility SHALL be reviewed before upgrade.

#### Scenario: Rollback within the window

- **WHEN** a newly deployed digest misbehaves inside the rollback window
- **THEN** schedules are disabled, the recorded prior digest is restored to the fixed production names, validation runs, and scheduling resumes

#### Scenario: Repository-format migration is caught before deployment

- **WHEN** a candidate Restic version could write an incompatible repository format
- **THEN** the incompatibility is identified before deployment, because a code rollback cannot undo a repository-format migration

### Requirement: SSH key rotation adds before it removes

Rotation SHALL add a second restricted public key, test a complete backup with the new private key, remove the old authorization, verify rejection of the old key, and update recovery records. The private key SHALL originate on the NAS secret store or another approved final trusted host.

#### Scenario: Old key is verified rejected

- **WHEN** the old authorization has been removed
- **THEN** an attempt to use the old private key is refused, and the new key continues to complete backups

### Requirement: Restic key rotation preserves access at every step

Rotation SHALL add and test a new Restic key before removing the old key, atomically update the mounted password file, test snapshots/check/restore, and update both protected recovery copies. If decrypted repository data or its master key may have been exposed, a new repository with new master material SHALL be created; password rotation alone is insufficient.

#### Scenario: Suspected master-key exposure

- **WHEN** decrypted repository data or the master key may have been obtained by an attacker
- **THEN** a new repository with new master key material is created and verified snapshots are migrated, rather than only changing the password

#### Scenario: Password file is swapped atomically

- **WHEN** the mounted password file is replaced
- **THEN** the replacement is atomic and a subsequent snapshots/check/restore test succeeds

### Requirement: The PVE encryption key is recovery material, not a rotating credential

The PVE client encryption key SHALL be treated as stable recovery material. The exact existing AES JSON key SHALL be copied and tested; it SHALL NOT be rotated merely to match routine token rotation. If it is compromised, a new encrypted backup lineage SHALL be established and the old key preserved only as long as old backups remain required.

#### Scenario: Routine token rotation does not touch the AES key

- **WHEN** the PBS API token is rotated on its normal schedule
- **THEN** the client encryption key is left unchanged

#### Scenario: Exposed encryption key

- **WHEN** the AES key is believed exposed
- **THEN** a new key is generated for future backups, the old key is retained while old snapshots must remain restorable, and the exposure window is treated as a data-disclosure incident

### Requirement: Each component compromise has a defined response

The following responses SHALL be defined, owned, and exercised:

| Incident | Required response |
| --- | --- |
| Suspected Hermes compromise | Stop destructive retention, preserve pre-compromise points, isolate the guest, prefer a pre-compromise PBS restore, inspect application exports as hostile |
| Suspected Synology compromise | Revoke the source pull key, suspend Restic and PBS maintenance, preserve/offline the off-site copy, rebuild NAS/PBS services, rotate NAS-held credentials |
| Suspected PVE compromise | Revoke the PBS token, preserve PBS/NAS and off-site history, rebuild PVE, restore the existing client key only after trust is re-established |
| Suspected PBS compromise | Revoke the API token, preserve NAS snapshots and the off-site copy, rebuild PBS, safely reattach the datastore, verify before reconnecting PVE |
| Repository corruption | Stop prune/GC, preserve storage snapshots, identify the last verified point, recover from a protected snapshot or the off-site copy |
| NAS loss | Rebuild/reattach PBS from off-site data, upload the existing PVE AES key, restore Hermes isolated, then reconstruct the granular Restic service as needed |

#### Scenario: Suspected guest compromise

- **WHEN** Hermes compromise is suspected
- **THEN** destructive retention stops, pre-compromise recovery points are preserved, the newest snapshot predating the earliest credible compromise time is identified as the recovery point, and guest-held credentials are rotated regardless

#### Scenario: Procedures have named owners

- **WHEN** upgrade, rollback, rotation, and incident procedures are reviewed
- **THEN** each has a named owner and has been exercised at least once

### Requirement: Initial service objectives are defined and reviewable

Unless replaced by measured requirements, the system SHALL be measured against:

| Objective | Initial target |
| --- | --- |
| Cold VM backup RPO | 24 hours |
| Live application backup RPO | 24 hours |
| Missed-backup detection | 36 hours |
| Local machine restore initiation | 4 hours |
| Off-site-only restore initiation | 8 hours |
| Local protected-history window | At least 14 days |
| Off-site protected-history window | At least 30 days |
| Application restore drill | Quarterly |
| Full off-site disaster-recovery drill | Annually |

#### Scenario: Objective is unobservable

- **WHEN** an objective cannot be evidenced from monitoring or drill records
- **THEN** compliance is reported as unobserved rather than as met

#### Scenario: Measured values replace initial targets

- **WHEN** measured recovery requirements become available
- **THEN** they replace the initial targets, and the replacement is recorded

### Requirement: A secret-free recovery card exists

A recovery card SHALL exist covering component addresses, the datastore name, the NFS export and mount path, where the PBS TLS fingerprint and PBS token identity are recorded, where each recovery key copy is held, and the restore isolation procedure. It SHALL contain no secret values.

#### Scenario: Recovery card is used in a drill

- **WHEN** a drill is performed using only the recovery card and the protected key copies
- **THEN** the drill completes, and any missing information is added to the card
