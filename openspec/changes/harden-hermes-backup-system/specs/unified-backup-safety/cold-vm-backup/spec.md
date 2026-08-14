## Purpose

Defines the cold, guest-independent whole-machine recovery path: Proxmox VE stops the Hermes VM, encrypts its disk image client-side, and stores it in a Proxmox Backup Server datastore hosted on the Synology NAS. This is the recovery path that remains trustworthy when guest-provided application output is suspect.

## ADDED Requirements

### Requirement: Backup scheduling and capture occur outside the guest

PVE SHALL schedule and perform the Hermes VM backup outside the guest. The production backup mode SHALL perform an orderly stop before capture; PVE MAY resume the VM once the stopped state is established and the background backup process has started. Guest-agent quiescing MAY improve behavior but SHALL NOT be treated as a security boundary.

#### Scenario: Scheduled cold backup runs without guest cooperation

- **WHEN** the daily PVE backup job fires
- **THEN** PVE stops the Hermes guest, captures its disks from outside the guest, and completes with `TASK OK` without requiring any in-guest backup agent to succeed

#### Scenario: Guest returns after a backup-induced stop

- **WHEN** a stop-mode backup completes and the guest is resumed
- **THEN** the guest boots and its messaging gateway reconnects, and this behavior is verified before the mode is accepted for production

#### Scenario: Backup window is sized for a full disk read

- **WHEN** backup-window and maintenance-separation planning is performed
- **THEN** it is based on the full virtual-disk read and checksum cost implied by an invalidated dirty bitmap, not on the size of the uploaded delta

#### Scenario: Moving to guest-influenced consistency is a recorded decision

- **WHEN** a change to guest-agent snapshot mode is proposed to recover incremental read speed
- **THEN** the change is recorded as a deliberate decision to accept guest-influenced consistency

### Requirement: The guest holds no backup authority and no route to backup infrastructure

The Hermes guest SHALL NOT hold the PBS API token or the PVE client encryption key. The guest network SHALL NOT reach PVE, PBS, or NAS management interfaces.

#### Scenario: Guest credential inventory is empty of backup material

- **WHEN** the guest filesystem and configuration are inspected
- **THEN** no PBS API token, PVE client encryption key, NAS credential, or retention/deletion authority is present

#### Scenario: Guest cannot route to backup endpoints

- **WHEN** reachability is tested from inside the guest
- **THEN** the PVE management interface, the PBS interface, DSM, the NFS export, and the Restic repository endpoint are all unreachable, while the guest retains the Internet access its own operation requires

### Requirement: PBS access is encrypted client-side and authenticated by pinned fingerprint

PVE SHALL use client-side encryption for the PBS storage entry and SHALL pin the PBS TLS certificate fingerprint obtained through a trusted administrative session. A fingerprint change SHALL be treated as an incident until explained by a known certificate replacement.

#### Scenario: Stored backup content is ciphertext to the storage platform

- **WHEN** a backup is written to PBS
- **THEN** the content is encrypted on the PVE host before transmission, PBS reports the snapshot as encrypted, and neither PBS nor the NAS possesses the decryption key

#### Scenario: Unexplained fingerprint change

- **WHEN** the PBS TLS fingerprint presented to PVE no longer matches the pinned value and no known certificate replacement has occurred
- **THEN** the mismatch is raised as a security incident rather than re-pinned to the currently presented value

### Requirement: The client encryption key is preserved, not regenerated

The exact existing AES JSON key SHALL have two protected recovery copies outside PVE, PBS, and the NAS, such as a password manager and encrypted offline media. Disaster recovery SHALL upload the existing key and SHALL NOT generate a replacement key when restoring old encrypted backups.

#### Scenario: Recovery from a rebuilt PVE host

- **WHEN** an operator restores an old encrypted snapshot onto a freshly installed PVE host
- **THEN** the preserved AES JSON key is uploaded and the restore succeeds, and no workflow offers key regeneration as a substitute

### Requirement: The PVE backup identity is least-privilege

The PBS API token used by PVE SHALL have only the permissions required to create and restore its owned backup groups. Retention and datastore administration SHOULD remain server-side on PBS. Token and user ACLs SHALL be verified as an effective least-privilege intersection.

#### Scenario: Routine token cannot destroy history

- **WHEN** the PVE backup token attempts datastore administration or deletion
- **THEN** the operation is refused, and retention continues to run under PBS's own identity

### Requirement: The PBS datastore is a restricted NAS-backed export

The datastore SHALL remain on a dedicated Synology Btrfs shared folder exported only to the PBS VM. The NFS export SHALL be restricted to the PBS VM's single reserved address, which SHALL NOT be allowed to change. NFS SHALL use a hard mount, synchronous-safe behavior, stable numeric ownership, and a mount arrangement that cannot silently write into an empty local mount point when NFS is absent. Shared-folder compression and Recycle Bin SHOULD remain disabled. Capacity SHALL be monitored using Synology shared-folder usage, not only PBS-reported filesystem capacity.

#### Scenario: Datastore path resolves to the mounted export

- **WHEN** the datastore path is inspected inside the PBS VM
- **THEN** it resolves to the mounted NFS export rather than to an empty local mount-point directory

#### Scenario: Export authorization is verified as the only effective control

- **WHEN** the export rule is reviewed
- **THEN** it names the single reserved PBS VM address, and the review records that `AUTH_SYS` trusts client-supplied numeric identities so this address restriction plus the network boundary are the export's only effective authorization

#### Scenario: Datastore identity is tested end to end

- **WHEN** the PBS `backup` identity is tested against the mounted datastore
- **THEN** create, write, rename, sync, and delete all succeed, and any ownership or mapping problem is corrected at the export rather than by widening file modes

#### Scenario: Capacity is reported by DSM accounting

- **WHEN** capacity is evaluated
- **THEN** Synology shared-folder usage is the authority, because Synology NFS reports whole-volume statistics that can show PBS free space the share quota will not permit

### Requirement: PBS is excluded from image backup and from file-level backup

The PVE backup job SHALL NOT include the PBS VM; its recovery path is rebuild-and-reattach rather than image restore, and reattaching SHALL NOT initialize or erase the existing datastore. The datastore SHALL NOT be captured by any ordinary file-level backup job that copies chunks as individual files.

#### Scenario: PBS VM is absent from the backup job

- **WHEN** the PVE backup job selection is reviewed
- **THEN** the PBS VM is not included

#### Scenario: Datastore reattachment preserves existing chunks

- **WHEN** a rebuilt PBS VM is attached to the existing datastore path
- **THEN** the datastore is reattached without initialization or erasure and its snapshots verify

### Requirement: PBS maintenance windows do not overlap

Backup, verification, prune, and garbage-collection windows SHALL NOT overlap. Initial cadence SHOULD be daily prune outside the backup window, daily cold backup, weekly verification with periodic reverification of older snapshots, and weekly garbage collection in a separate I/O window. Every job SHALL report failures, and the NAS SHALL retain capacity headroom for snapshots and maintenance.

#### Scenario: Overlapping maintenance is detected

- **WHEN** two of backup, verification, prune, and garbage collection are scheduled or observed to run concurrently
- **THEN** the overlap is reported to operators as a defect in the schedule

#### Scenario: Deployed schedule differs from the initial cadence

- **WHEN** a deployment adopts a different cadence
- **THEN** the change is permitted provided the windows remain non-overlapping, every job reports failures, and capacity headroom is retained
