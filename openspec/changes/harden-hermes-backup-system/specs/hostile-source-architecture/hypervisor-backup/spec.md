## Purpose

Defines the hypervisor-controlled image backup that gives the system a recovery path whose scheduling, credentials, and encryption keys live entirely outside the hostile guest, and records which of those requirements the current deployment already meets.

## ADDED Requirements

### Requirement: The guest is isolated and excluded from backup control

Hermes SHALL run as a guest without access to the management or backup networks. Backup scheduling SHALL occur outside the guest, and backup credentials and client-side encryption keys SHALL remain outside the guest.

Deployment state: scheduling and credential separation are satisfied — PVE runs the backup job daily and PBS runs prune, verification, and garbage collection on its own schedules, and the PBS API token and AES key exist only on PVE and in operator recovery copies. Network isolation is not satisfied; the guest shares one flat LAN with PVE, PBS, and DSM.

#### Scenario: Guest reachability test

- **WHEN** reachability is tested from the guest
- **THEN** PVE management, PBS, DSM, and the NFS export are unreachable at the network layer

#### Scenario: Scheduling and keys are external

- **WHEN** the backup schedule and key material are located
- **THEN** both exist outside the guest, and the guest holds neither the PBS API token nor the image encryption key

### Requirement: The routine hypervisor identity cannot delete history

The normal hypervisor backup token SHALL be able to create backups but SHALL NOT have prune or datastore administration permission. Retention SHALL run on the backup server rather than under the backup token.

Deployment state: satisfied — the token is privilege-separated with `DatastoreBackup` on one datastore path, and retention runs on PBS.

#### Scenario: Token attempts a destructive operation

- **WHEN** the routine backup token attempts prune or datastore administration
- **THEN** the operation is refused

### Requirement: The backup server verifies, separates windows, and runs off the protected host

The backup server SHALL perform scheduled verification, including periodic re-verification of older snapshots. Backup, prune, verification, and garbage collection SHALL NOT overlap. The backup server SHALL run on a host other than the one it protects.

Deployment state: satisfied — weekly verification re-verifies snapshots on a 30-day interval, the four windows are separated, and PBS runs on the NAS rather than on the PVE host.

#### Scenario: Older snapshot develops corruption

- **WHEN** a snapshot that verified previously is re-verified on its interval
- **THEN** later storage corruption is detected rather than assumed absent

#### Scenario: Loss of the protected host

- **WHEN** the PVE host is lost
- **THEN** the recovery points survive because the backup server does not run on it

### Requirement: The datastore has an enforced ceiling and protected snapshots

The datastore SHALL have an enforced capacity ceiling and be monitored against it, using the storage platform's own shared-folder accounting as the authority. The datastore SHALL have protected snapshots or equivalent protection against deletion by a compromised administrator.

Deployment state: capacity is partially satisfied — a DSM shared-folder quota is set, but Synology NFS reports whole-volume statistics so PBS displays free space that does not exist. Protected snapshots are not satisfied.

#### Scenario: Capacity alert uses the real authority

- **WHEN** capacity is alerted on
- **THEN** DSM shared-folder usage is the value used, not PBS-reported free space

#### Scenario: Administrator deletes datastore content

- **WHEN** a compromised administrator deletes datastore content within the protection window
- **THEN** protected snapshots preserve it

### Requirement: An off-site copy protects image backups

An off-site synchronization or backup-copy job SHALL protect the image backups in another failure domain.

Deployment state: not satisfied; the datastore has no independent off-site copy.

#### Scenario: Site loss

- **WHEN** the site holding the datastore is lost
- **THEN** image recovery points remain available from the independent failure domain

### Requirement: Image restores are isolated, inspected, and never concurrent with production

A restored VM SHALL first boot in an isolated network with no Internet, production, backup, or management access. Recovery operators SHALL inspect the restored system before authorizing production connectivity. Production and a credential-identical restored guest SHALL NOT run online concurrently.

Deployment state: satisfied in procedure — the restore drill requires a new VM ID, no start-after-restore, no live restore, a unique MAC, disabled autostart, and a disconnected NIC before first boot.

#### Scenario: Restored clone carries duplicate identity

- **WHEN** a restored clone holding duplicate messaging-gateway tokens, OAuth credentials, host keys, and service identity is considered for connection
- **THEN** it is kept offline while production runs, and a changed MAC address is not accepted as a substitute

### Requirement: Consistency does not depend on guest cooperation

Guest-agent quiescing MAY improve consistency but SHALL NOT be treated as a security boundary, because a hostile guest can interfere with it. The deployment uses stop mode rather than guest-agent snapshot mode, accepting a short daily outage and a full-disk reread in exchange for consistency independent of the guest. A later move to snapshot mode for speed SHALL be recorded as accepting guest-influenced consistency.

#### Scenario: Snapshot mode is proposed

- **WHEN** a change to guest-agent snapshot mode is proposed
- **THEN** the trade is recorded as accepting guest-influenced consistency

#### Scenario: Hostile guest controls its own disk contents

- **WHEN** the consistency guarantee is stated
- **THEN** it covers removing the guest from the backup control plane and explicitly does not claim that guest data is truthful

### Requirement: The image encryption key has two external recovery copies

The image encryption key SHALL have at least two recovery copies outside the hypervisor and outside the NAS, because generating a new key does not decrypt existing snapshots.

Deployment state: satisfied — the key JSON is held in a password manager and on encrypted offline media.

#### Scenario: Hypervisor hardware loss

- **WHEN** the hypervisor is rebuilt after hardware loss
- **THEN** an external copy of the existing key JSON is uploaded and old snapshots decrypt, and generating a new key is not offered as a workaround
