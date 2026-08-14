## Purpose

Defines what configuration may live in Docker metadata, where the confidential and integrity-sensitive files live on the NAS, how repository storage is bounded, and how many independent recovery copies each recovery secret needs.

## ADDED Requirements

### Requirement: Only non-confidential configuration lives in environment metadata

Only non-confidential configuration MAY be stored in Docker environment metadata. Approved environment variables are:

```text
HERMES_SSH_TARGET
SSH_PORT
RESTIC_HOST
RESTIC_TAG
MODE
FORGET_AFTER_BACKUP
PRUNE_AFTER_BACKUP
KEEP_DAILY
KEEP_WEEKLY
KEEP_MONTHLY
CHECK_READ_DATA_SUBSET
SSH_CONNECT_TIMEOUT
SSH_SERVER_ALIVE_INTERVAL
SSH_SERVER_ALIVE_COUNT_MAX
MAX_BACKUP_SECONDS
MIN_EXPORT_BYTES
MAX_EXPORT_BYTES
```

The Restic password, the SSH private key or its passphrase, a GHCR token, a Synology encrypted-share key, a Synology volume recovery key, and credentials for an external secret manager SHALL NOT be environment-variable values. Variable names ending in `_FILE` contain paths, not secrets, and MAY use the image defaults.

#### Scenario: Environment carries only paths and settings

- **WHEN** container environment metadata is inspected
- **THEN** it contains approved settings and `_FILE` paths, and no secret value

#### Scenario: Password moved to an environment variable

- **WHEN** a configuration change would place the Restic password in an environment variable
- **THEN** it is refused in favour of Restic's password-file or password-command interface

### Requirement: Credentials live in a dedicated encrypted shared folder

The NAS SHALL provide a dedicated encrypted shared folder for application credentials, illustratively `/volume1/hermes-backup-secrets`, with the actual location installation-specific. The share SHALL have explicit DSM ACLs denying unrelated users and groups; SHALL NOT be exposed through SMB, NFS, FTP, WebDAV, Synology Drive, media indexing, or search indexing; SHALL NOT be included in an unrelated synchronization job; SHALL be excluded from backups that would copy plaintext mounted contents unless that backup has a separately reviewed encryption and recovery design; and SHALL use either manual unlock, an external Key Manager store, or remote KMIP according to the chosen availability profile.

Recommended runtime paths and modes are:

```text
/volume1/hermes-backup-secrets/runtime/           0500  65532:65532
/volume1/hermes-backup-secrets/runtime/id_ed25519 0400  65532:65532
/volume1/hermes-backup-secrets/runtime/restic     0400  65532:65532
```

File names are not a security boundary and MAY differ. The SSH public key MAY be stored outside the encrypted share. The private key SHALL be unique to this backup source and SHALL have an independently removable, restricted `authorized_keys` entry.

#### Scenario: ACL review covers every DSM access path

- **WHEN** the share's permissions are reviewed
- **THEN** local users, local groups, system internal users, inherited ACL entries, application permissions, advanced shared-folder permissions, file services, NFS rules, and indexing are all inspected rather than assuming `chmod 700` describes access

#### Scenario: A NAS-wide backup job reaches the share

- **WHEN** any backup or synchronization job would copy the mounted plaintext credentials
- **THEN** the share is excluded from that job unless the job has a separately reviewed encryption and recovery design

### Requirement: Integrity-sensitive configuration is root-owned and outside the secret share

`known_hosts` is not confidential but is integrity-sensitive. It SHOULD be stored outside the secret share in a root-owned configuration location, for example `/volume1/hermes-backup-config/known_hosts`. The file SHALL be readable by UID 65532 and not writable by that UID; an illustrative ownership and mode is `root:root 0444`. The host-key fingerprint SHALL be verified out of band before installation and whenever it changes.

#### Scenario: Runtime identity cannot rewrite trust material

- **WHEN** UID 65532 attempts to write `known_hosts`
- **THEN** the write is refused while reads continue to succeed

#### Scenario: Host key changes

- **WHEN** the source host key changes
- **THEN** the new fingerprint is verified out of band before `known_hosts` is updated

### Requirement: Repository storage is separate, access-restricted, and capacity-bounded

The repository SHALL be a separate shared folder from the credentials, for example `/volume1/Backups/restic-hermes`. UID/GID 65532 SHALL have the read, write, and traversal access Restic requires, and unrelated users and services SHOULD have none. The repository SHOULD reside on Btrfs where supported, and immutable snapshots SHOULD protect it for at least 7 to 14 days subject to model support and capacity planning. A second encrypted copy SHALL exist outside the NAS before the system is considered the sole reliable backup of the source data.

The repository shared folder SHALL have a deployment-specific quota or another tested hard capacity limit. Capacity monitoring SHOULD alert before usable space falls below the amount required for one maximum-sized export plus normal Restic pack staging and immutable-snapshot growth. The quota is a final containment boundary and does not replace the per-run byte limit.

#### Scenario: Repository share is unreachable by file services

- **WHEN** the repository share's exposure is reviewed
- **THEN** it is reachable by the backup identity and by no file service

#### Scenario: Capacity alert precedes the hard limit

- **WHEN** usable space approaches the amount needed for one maximum-sized export plus staging and snapshot growth
- **THEN** an alert is raised before the quota is enforced against a running backup

### Requirement: Every recovery secret has two independent copies outside the NAS

At least two independent recovery copies SHALL exist outside the NAS for the current Restic password, the Synology encrypted shared-folder key when used, the encrypted-volume recovery key when used, and remote-KMIP recovery information and certificate procedures when used. Recovery copies SHALL NOT share the same single failure domain as the NAS.

#### Scenario: Recovery register is complete

- **WHEN** the recovery register is reviewed
- **THEN** each listed item names where its two independent copies live and how to use it, and no copy is stored on the NAS or in a synchronization path sharing its failure domain

#### Scenario: Recovery copy is exercised

- **WHEN** a recovery drill is performed
- **THEN** it uses the external copies rather than the values held on the running system
