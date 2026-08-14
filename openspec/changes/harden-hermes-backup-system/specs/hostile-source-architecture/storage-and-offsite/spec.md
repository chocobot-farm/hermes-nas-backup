## Purpose

Reduces the NAS to an encrypted-object storage service that cannot read what it holds and cannot be asked to delete it by the routine backup identity, and defines the independent off-site repository that makes the system survivable when the site is not.

## ADDED Requirements

### Requirement: The NAS acts only as an encrypted storage service

The NAS SHALL act only as an encrypted-object storage service. It SHALL NOT receive the Hermes SSH private key, either Restic repository password, the off-site credential, a source export in plaintext outside the encrypted Restic protocol, or routine backup deletion authority.

#### Scenario: NAS credential inventory

- **WHEN** the NAS is inspected
- **THEN** it holds no repository decryption password, no Hermes private key, and no off-site credential

#### Scenario: Plaintext export on a NAS volume

- **WHEN** a design would write the source export to a NAS volume outside the encrypted Restic protocol
- **THEN** it is refused

### Requirement: The local repository is served append-only over authenticated TLS

The local repository SHOULD use the official Restic REST server with equivalent settings to:

```text
--append-only
--private-repos
--tls
--tls-min-ver 1.3
--max-size DEPLOYMENT_QUOTA
```

Authentication SHALL use a high-entropy source-specific credential stored as a supported password verifier and SHALL NOT be disabled. The service SHALL be reachable only from the backup gateway and approved maintenance network; use a certificate validated by the gateway; run as a dedicated non-root identity; use a read-only container root filesystem when containerized; drop all capabilities not proven necessary; enable `no-new-privileges`; pin its image by digest; mount only its repository, authentication, TLS, and bounded runtime paths; and write access logs containing no credential or repository password. The repository share SHALL NOT be exposed through SMB, NFS, FTP, WebDAV, Synology Drive, media indexing, or normal user shares.

#### Scenario: Routine endpoint refuses deletion

- **WHEN** the routine append-only endpoint receives an overwrite or delete request
- **THEN** it is rejected

#### Scenario: Certificate is validated by the client

- **WHEN** the gateway connects to the repository endpoint
- **THEN** it validates the presented certificate and refuses an unvalidated endpoint

### Requirement: Local recovery points are protected by immutable snapshots

Where supported, the repository SHALL reside on Btrfs and use immutable Synology snapshots, with an initial minimum protection window of 14 days and a longer window selected from measured capacity and incident-detection time. This requirement applies to the PBS datastore shared folder as well as to the Restic repository. Snapshots of the PBS datastore SHALL be scheduled outside the PBS verification and garbage-collection windows, the shared folder SHALL keep the recycle bin disabled so garbage collection actually releases space, and the quota SHALL be sized for both the repository and its snapshot retention.

Local immutable snapshots are defense in depth; the system SHALL NOT be considered compliant until a protected off-site copy exists.

#### Scenario: Administrator deletes a repository

- **WHEN** NAS root or a DSM administrator deletes repository content covered by an immutable snapshot inside its window
- **THEN** the snapshot cannot be deleted and history remains recoverable

#### Scenario: Garbage collection does not release space

- **WHEN** garbage collection removes chunks a snapshot still references
- **THEN** the quota already accounts for the retained capacity

### Requirement: Deletion authority is a separate, temporary endpoint

The routine REST endpoint SHALL remain append-only. Any delete-capable maintenance endpoint SHALL use a different authentication credential, be disabled by default, be unreachable from Hermes and the gateway routine identity, be enabled only during an approved maintenance window, be restricted to the maintenance workstation network address, not run concurrently with backup ingestion, and be disabled and verified closed when maintenance finishes. The repository password SHALL remain on the maintenance workstation and SHALL NOT be persisted on the NAS for maintenance.

#### Scenario: Maintenance endpoint left open

- **WHEN** a maintenance window ends
- **THEN** the delete-capable endpoint is disabled and verified closed, and routine append-only behavior is confirmed

#### Scenario: Ingestion during maintenance

- **WHEN** a scheduled backup would ingest while the maintenance endpoint is open
- **THEN** the two are prevented from running concurrently

### Requirement: An off-site repository exists in an independent failure domain

The off-site target SHALL be in a different physical and administrative failure domain from Hermes, the gateway, and the NAS; receive client-side encrypted backup content; have no repository decryption password; enforce append-only writes or provider-controlled immutable snapshots; retain protected recovery points longer than the expected compromise-detection interval; support export or recovery without proprietary decryption software; and provide monitoring or an API sufficient to verify recent writes and protection status.

The initial minimum immutable window is 30 days. The target SHOULD retain daily recovery points for 90 days and monthly recovery points for at least one year, subject to confirmed recovery requirements and capacity.

Acceptable implementation classes include a managed Restic or Borg service with append-only access and client-held encryption keys; a snapshot-enabled rsync.net account whose ZFS snapshots are immutable to the client credential; a second independently administered Restic REST server with immutable underlying storage; or a second Proxmox Backup Server for the hypervisor profile. A plain writable SFTP, SMB, NFS, or object-store repository without independent immutability SHALL NOT satisfy this requirement.

#### Scenario: Writable remote target without immutability

- **WHEN** a plain writable SFTP, SMB, NFS, or object-store repository is proposed as the off-site copy
- **THEN** it is rejected as non-compliant

#### Scenario: Off-site immutability survives client compromise

- **WHEN** the routine off-site client credential is assumed compromised
- **THEN** protected remote recovery points remain, according to a tested or contractually verified procedure

#### Scenario: Least disruptive path for the image repository

- **WHEN** the image path needs an off-site copy
- **THEN** a native PBS sync or backup-copy job to a remote datastore is used rather than introducing a new backup product
