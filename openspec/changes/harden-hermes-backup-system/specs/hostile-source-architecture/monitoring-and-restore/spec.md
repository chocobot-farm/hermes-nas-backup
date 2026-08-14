## Purpose

Defines observation and recovery for a system whose data source is hostile: monitoring that Hermes cannot influence, the availability objectives the design is measured against, and restore procedures that treat every stored byte as attacker-supplied.

## ADDED Requirements

### Requirement: Monitoring is controlled outside the source

Monitoring SHALL be controlled outside Hermes. Hermes SHALL NOT be able to submit or overwrite the authoritative backup success status. Because both paths are designed to fail closed and both fail quietly, a path that stops producing recovery points SHALL be detectable without any signal from the source.

#### Scenario: Source asserts success

- **WHEN** Hermes emits a backup success signal
- **THEN** the authoritative status is unchanged by it

#### Scenario: Silent stop on either path

- **WHEN** a path stops producing recovery points without reporting a failure
- **THEN** the condition is detected by external monitoring rather than at restore time

### Requirement: Image-path alert conditions are covered

The image path SHALL alert on: hypervisor backup job failure or a run that did not start; backup-server verification failure or a snapshot that failed re-verification; garbage collection or prune failure; an unexpected change in the backup group owner; loss of the NFS mount inside the backup server, or a datastore path that is a local directory rather than the mounted export; shared-folder usage crossing its alert threshold, which is the real quota authority; and a backup-server TLS fingerprint that no longer matches the value pinned in the hypervisor.

#### Scenario: Datastore silently becomes a local directory

- **WHEN** the datastore path resolves to a local directory rather than the mounted export
- **THEN** an alert is raised before backups accumulate outside the intended storage

#### Scenario: Pinned fingerprint no longer matches

- **WHEN** the presented TLS fingerprint differs from the pinned value
- **THEN** an alert is raised

### Requirement: Application-path alert conditions are covered

Operators SHALL be alerted for: source connection or exporter failure; export timeout or size-limit violation; export size outside the configured baseline; a changed SSH host key; local repository failure; off-site repository failure; different TAR digests between repository snapshots; failure to clean the spool; overlapping backup attempts; repository quota pressure; missing or expired immutable protection; repository check failure; missed schedule; and restore-test failure.

#### Scenario: Repository snapshots disagree

- **WHEN** the local and off-site snapshots for one run record different TAR digests
- **THEN** an alert is raised

#### Scenario: Spool is not cleaned

- **WHEN** the spool remains after a run terminates
- **THEN** an alert is raised

### Requirement: Defined operational questions are answerable

Operators SHALL be able to determine: last attempted and last successful pull time; gateway version and configuration version; source exporter exit status; source byte count and gateway-computed SHA-256; snapshot identifier for each repository; last successful write observed at each storage target; immutable protection horizon; last check and prune result; and last successful application and image restore test.

Logs SHALL use UTC timestamps and SHALL NOT contain secret values, complete process environments, private keys, or authorization headers.

#### Scenario: Operator reconstructs a run

- **WHEN** an operator investigates a specific run
- **THEN** its exporter exit status, byte count, computed digest, and per-repository snapshot identifiers are all available

### Requirement: Availability objectives are defined and measured

Unless superseded by an approved deployment-specific recovery plan, the initial objectives SHALL be:

| Objective | Initial target |
| --- | --- |
| Application backup RPO | 24 hours |
| VM image backup RPO | 24 hours |
| Detection of a missed backup | 36 hours |
| Local application restore initiation | 4 hours after declared incident |
| Local image restore initiation | 4 hours after declared incident |
| Off-site-only restore initiation | 8 hours after declared incident |
| Local immutable window | At least 14 days |
| Off-site immutable window | At least 30 days |

These are service objectives, not guarantees; the deployment owner SHALL confirm they are adequate for Hermes.

#### Scenario: RPOs met while detection is not

- **WHEN** both RPOs are met but no external monitoring exists
- **THEN** the system is reported as meeting its targets only until it silently stops, and the detection objective is recorded as unmet

#### Scenario: Daily outage cost of stop-mode capture

- **WHEN** stop mode takes the guest offline for each image backup
- **THEN** the outage is accepted as the price of guest-independent consistency, and reboot persistence is verified so the guest reliably returns and its messaging gateway reconnects

### Requirement: Every restore treats repository contents as hostile

Application restores SHALL:

1. restore into a disposable, isolated environment;
2. use an unprivileged identity;
3. have no network access during extraction and initial inspection;
4. prohibit device creation and privilege restoration;
5. reject absolute paths and path traversal;
6. handle symlinks without allowing writes outside the restore root;
7. avoid preserving source ownership, setuid, setgid, capabilities, or unsafe extended attributes;
8. inspect the TAR and embedded Hermes ZIP before invoking application import;
9. run malware and policy scans appropriate to the environment;
10. validate representative Hermes behavior; and
11. require explicit operator approval before any recovered data reaches production.

Application restores SHALL import into a disposable isolated VM, never into the running production guest. VM image restores SHALL initially boot on an isolated recovery network, and a restored instance SHALL NOT receive production credentials or Internet access until inspected and approved.

#### Scenario: Archive contains hostile metadata

- **WHEN** the restored archive contains absolute paths, traversal, device nodes, setuid bits, or unsafe extended attributes
- **THEN** they are rejected or neutralized and nothing is written outside the restore root

#### Scenario: Offline inspection of a restored image

- **WHEN** a restored VM image is inspected before reconnection
- **THEN** firmware and bootloader, root filesystem mount, expected configuration, failed system units, and the presence of expected binaries and services are all reviewed, and operator judgment authorizes reconnection

#### Scenario: Restoring a compromised image

- **WHEN** the restored image may carry the compromise it was taken with
- **THEN** offline inspection before reconnection is what separates recovery from reinfection, and no automated step substitutes for it

### Requirement: Restores are exercised on a defined cadence

At least monthly, automation SHALL restore and validate a non-sensitive canary. At least quarterly, operators SHALL perform a representative isolated application restore from each repository. At least annually, operators SHALL exercise the complete disaster-recovery procedure, including offline recovery secrets and off-site-only recovery.

The annual exercise SHALL include restoring an image snapshot onto a rebuilt hypervisor host using the saved key material, because that is the path a real hardware loss takes. Auto-generating a new key during recovery SHALL NOT be offered as a workaround.

#### Scenario: Annual exercise on a rebuilt host

- **WHEN** the annual exercise runs
- **THEN** it restores onto a rebuilt hypervisor using the saved key JSON, and an unreadable or missing key copy is discovered by the drill rather than by an incident

#### Scenario: Monthly canary fails

- **WHEN** the monthly canary restore fails to validate
- **THEN** an alert is raised
