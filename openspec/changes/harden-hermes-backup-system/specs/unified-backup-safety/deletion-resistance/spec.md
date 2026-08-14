## Purpose

Defines how recovery points survive deletion, corruption, and loss of the Synology NAS: protected local snapshots over both repository shares, capacity accounting that reflects snapshot retention, and at least one encrypted copy in an independent failure domain.

## ADDED Requirements

### Requirement: Both repositories are protected by local NAS snapshots

PBS and Restic SHALL reside in separate dedicated shared folders. Where supported, Synology protected or immutable Btrfs snapshots SHOULD cover both folders with schedules that do not overlap active PBS verification/garbage collection or Restic pruning. The protection window SHALL exceed the expected time to detect compromise or backup failure; fourteen days is a reasonable initial minimum, with the final period based on capacity and incident-detection requirements.

#### Scenario: Protected snapshot survives an attempted deletion

- **WHEN** an ordinary repository deletion is attempted against a share covered by a protected snapshot within its protection window
- **THEN** the snapshot cannot be deleted and the repository remains restorable from it

#### Scenario: Snapshot schedule avoids maintenance windows

- **WHEN** snapshot schedules are configured
- **THEN** they fall outside PBS verification and garbage collection and outside Restic prune windows

#### Scenario: Protection lapses

- **WHEN** a protected snapshot is absent or its protection window has expired
- **THEN** operators are alerted

### Requirement: Snapshot, repository, and off-site retention are separate policies

Snapshot retention, repository retention, and off-site retention SHALL be treated as separate policies. Capacity planning SHALL account for the fact that deleting data through PBS or Restic does not necessarily free NAS capacity while Btrfs snapshots retain the underlying blocks.

#### Scenario: Freed repository space is not returned

- **WHEN** PBS garbage collection or Restic prune removes data that a protected snapshot still references
- **THEN** the capacity plan already accounts for the retained blocks and the quota is sized for both the repository and its snapshot overhead

### Requirement: Both repository shares have enforced capacity ceilings

Each repository shared folder SHALL have a deployment-specific quota or another tested hard capacity limit, sized for its retention policy plus snapshot overhead. Capacity alerting SHALL use Synology shared-folder accounting rather than repository-reported free space, and SHOULD alert well before the hard limit is reached.

#### Scenario: Quota pressure is alerted before failure

- **WHEN** shared-folder usage crosses its configured threshold
- **THEN** operators are alerted before a backup fails for lack of space

#### Scenario: Repository-reported free space disagrees with DSM

- **WHEN** PBS reports abundant free space because Synology NFS reports whole-volume statistics
- **THEN** the DSM shared-folder quota remains the authority for capacity decisions and alerts

### Requirement: At least one encrypted copy exists outside the Synology failure domain

The system SHALL NOT be considered 3-2-1 compliant until an encrypted copy exists outside the Synology NAS and local site. The preferred first off-site milestone is a native PBS sync/copy path to a separately administered PBS or supported remote target. It SHALL preserve encrypted backup content and retain history long enough to survive the expected detection interval. Routine replication credentials SHOULD be unable to delete protected remote history. If application-level off-site recovery is added, it SHOULD use an append-only Restic-compatible endpoint or immutable storage with independent transport credentials, and the remote service SHALL NOT receive the repository decryption password.

#### Scenario: Whole-VM recovery without the production NAS

- **WHEN** a recovery drill excludes the production Synology NAS entirely
- **THEN** complete Hermes VM recovery succeeds from the off-site copy and the preserved encryption key

#### Scenario: Replication credential cannot delete remote history

- **WHEN** the routine replication credential attempts to remove protected remote recovery points
- **THEN** the deletion is refused

#### Scenario: Off-site replication goes stale

- **WHEN** the latest successful off-site copy ages past its expected interval
- **THEN** operators are alerted

#### Scenario: Application off-site copy does not delay the VM path

- **WHEN** off-site replication of the application repository is proposed
- **THEN** it is permitted as an improvement to granular recovery but does not delay establishing the off-site whole-VM path
