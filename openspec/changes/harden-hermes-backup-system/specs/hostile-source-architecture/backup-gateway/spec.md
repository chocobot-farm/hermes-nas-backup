## Purpose

Defines an independent backup gateway that pulls one bounded export from the hostile source into a private spool and writes those exact bytes to two independently encrypted repositories, holding all routine backup secrets outside both the source and the NAS.

## ADDED Requirements

### Requirement: The gateway is an independent trust domain

The gateway SHALL be a separate operating-system instance from Hermes and the NAS. It SHALL NOT be administered by Hermes or run inside a runtime Hermes can control. A dedicated physical device is preferred; a VM is acceptable only when its hypervisor is outside Hermes's trust boundary and Hermes cannot access the management plane. The gateway SHALL NOT run general user workloads, AI agents, web browsing, email, or unrelated Internet-facing services.

#### Scenario: Gateway hosted on a reachable hypervisor

- **WHEN** the gateway is proposed as a VM
- **THEN** it is accepted only if its hypervisor is outside Hermes's trust boundary and Hermes cannot reach the management plane

#### Scenario: Unrelated workload proposed

- **WHEN** an unrelated Internet-facing service is proposed for the gateway
- **THEN** it is refused

### Requirement: Gateway deployment inputs are root-owned and version-pinned

Gateway executables, service units, configuration, and parent directories SHALL be root-owned and not writable by the runtime identity. Deployable binaries or images SHALL be selected by immutable version and digest. The implementation SHOULD publish an SBOM and provenance. Automatic updates MAY download candidates, but activation SHALL follow a reviewed rollout and rollback procedure.

#### Scenario: Runtime identity attempts to modify its own program

- **WHEN** the backup runtime identity attempts to write a gateway executable, unit, or configuration file
- **THEN** the write is refused

#### Scenario: Downloaded update activates itself

- **WHEN** an update is downloaded automatically
- **THEN** it is not activated until the reviewed rollout procedure runs

### Requirement: The scheduled backup runs under a sandboxed dedicated identity

The scheduled backup SHALL run under a dedicated, non-interactive identity and SHALL receive only the filesystem and network access required for the backup. A systemd implementation SHOULD use applicable controls such as `NoNewPrivileges=yes`, `PrivateTmp=yes`, a restrictive `ProtectSystem` setting, explicit `ReadWritePaths` for spool and state, `ProtectHome=read-only` or stricter, a closed capability bounding set, memory, process, and runtime limits, and systemd credentials for secret delivery. The final sandbox SHALL be tested against the required SSH, spool, Restic, DNS, and certificate operations rather than copied blindly.

#### Scenario: Sandbox is validated against real operations

- **WHEN** the sandbox configuration is applied
- **THEN** a full backup cycle including SSH, spool writes, Restic operations, DNS, and certificate validation is exercised successfully under it

### Requirement: The gateway holds separated secrets for two repositories

The gateway holds one source-specific SSH private key, one local repository password, one off-site repository password, one local repository authentication credential, one off-site repository authentication credential, and integrity-sensitive TLS CA or SSH host-key material. The two repositories SHALL use independent encryption passwords and independent transport credentials.

Secret values SHALL NOT appear in environment variables, command-line arguments, logs, images, Git, monitoring payloads, or crash reports; programs SHOULD consume credentials from file descriptors, systemd credentials, or root-only files. At-rest gateway credentials SHOULD be protected by full-disk encryption and, where operationally acceptable, TPM-bound systemd credentials; these controls do not protect against live gateway root. At least two independent offline recovery copies of each repository recovery secret SHALL exist outside Hermes, the gateway, and the NAS.

#### Scenario: One repository credential is compromised

- **WHEN** a single repository's transport credential or password is exposed
- **THEN** the other repository's encryption and transport remain unaffected because they are independent

#### Scenario: Secret appears in a diagnostic

- **WHEN** logs, monitoring payloads, or crash reports are reviewed
- **THEN** no credential value is present

### Requirement: The source export is received into a bounded private spool

The gateway SHALL receive one source export into a private spool before writing either repository. The spool SHALL reside on an encrypted filesystem or size-appropriate tmpfs; be accessible only to the backup runtime identity and root; have a configured maximum byte size; have a configured maximum runtime; fail the backup if the source exceeds either limit; never parse, list, decompress, or extract the source TAR; survive long enough to attempt both independent repository writes; be unlinked after both attempts or after a terminal failure; and rely on the encrypted spool filesystem rather than overwrite-based deletion to protect residual storage blocks.

The receiver SHALL distinguish a complete source exit from truncation at the size limit; silently accepting the first `MAX_BYTES` of an oversized source is not compliant.

#### Scenario: Oversized source

- **WHEN** the source exceeds the configured maximum byte size
- **THEN** the backup fails and the truncated prefix is not accepted as a successful export

#### Scenario: Spool is never interpreted

- **WHEN** the export is held in the spool
- **THEN** it is never parsed, listed, decompressed, or extracted by the gateway

#### Scenario: Spool removal after a terminal failure

- **WHEN** the run fails terminally
- **THEN** the spool is unlinked and residual blocks remain protected by the encrypted spool filesystem

### Requirement: The gateway generates its own manifest and backs it up with the export

After a successful pull the gateway SHALL generate its own manifest containing at least a schema version, UTC pull start and finish times, source identifier, byte length, SHA-256 digest of the exact TAR bytes, exporter exit status, verified SSH host-key fingerprint, and gateway release identifier. The TAR and gateway manifest SHALL be backed up together to both repositories, and the two repository snapshots SHALL refer to the same TAR digest.

#### Scenario: Digests diverge between repositories

- **WHEN** the two repository snapshots record different TAR digests
- **THEN** an alert is raised

#### Scenario: Manifest travels with the data

- **WHEN** either repository is restored
- **THEN** the gateway manifest is present alongside the TAR it describes

### Requirement: Scheduled runs fail closed and write both targets

The scheduled gateway identity SHALL be able to create snapshots but SHALL NOT be able to forget, prune, overwrite, or delete existing local repository objects. For each scheduled run the gateway SHALL:

1. acquire a non-blocking run lock;
2. verify required credentials and trusted host material are readable;
3. pull one bounded export from Hermes;
4. require a zero SSH/exporter exit status;
5. require a nonempty export within the configured size range;
6. generate the gateway manifest;
7. back up the TAR and manifest to the local append-only repository;
8. independently back up the same TAR and manifest to the off-site target;
9. record the snapshot identifier and result of each target;
10. report partial success as an alert and an overall nonzero result;
11. remove the spool; and
12. release the run lock.

Failure of one repository SHALL NOT prevent an attempt to write the other after a valid source export has been captured. The backup SHALL NOT run retention or pruning after snapshot creation. The gateway SHALL NOT request two independent exports for the two targets, because a hostile source could return different content.

#### Scenario: One repository is unreachable

- **WHEN** the local write fails after a valid export has been captured
- **THEN** the off-site write is still attempted, and the run reports partial success as a failure requiring action

#### Scenario: Routine identity attempts deletion

- **WHEN** the scheduled gateway identity attempts to forget, prune, overwrite, or delete a local repository object
- **THEN** the operation is refused

#### Scenario: Single pull feeds both targets

- **WHEN** both repositories are written
- **THEN** they receive the same captured bytes from one pull rather than two separate source exports
