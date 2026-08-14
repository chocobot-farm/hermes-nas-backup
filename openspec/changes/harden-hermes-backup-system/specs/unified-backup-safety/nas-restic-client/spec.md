## Purpose

Defines the Synology-side application backup client: how its image is published and pinned, how its one-shot containers are created and scheduled, how its secrets are held, and how it bounds and accounts for output from a potentially hostile source.

## ADDED Requirements

### Requirement: The client image is published by a protected workflow and deployed by digest

The target image SHOULD be published as `ghcr.io/OWNER/hermes-nas-backup` by a protected release workflow. Releases SHALL publish a semantic version, a source-commit tag, and an immutable manifest digest. Synology SHALL deploy the digest and never a moving `latest` tag. The workflow SHALL run tests, ShellCheck, an image smoke test, vulnerability policy, SBOM generation, and artifact attestation before release. Workflow actions and the base image SHALL be pinned to reviewed immutable revisions. Runtime secrets SHALL NOT enter image layers, build arguments, workflow logs, or registry metadata. A public package is preferred so Synology needs no long-lived GHCR credential; a private package requires a separate read-only package token.

#### Scenario: Deployment references a digest

- **WHEN** the deployed container configuration is inspected
- **THEN** its image reference is a `sha256:` manifest digest, not a mutable tag

#### Scenario: Release gate fails

- **WHEN** tests, ShellCheck, the smoke test, the vulnerability policy, SBOM generation, or attestation fails
- **THEN** no deployable image is published

### Requirement: Production runs as fixed, stopped, hardened one-shot containers

Synology SHOULD replace scheduled `docker-compose run` from the Git checkout with fixed, stopped containers created from the verified image digest:

| Container | Mode | Secret/config mounts |
| --- | --- | --- |
| `hermes-backup-daily` | `backup` | Restic password, SSH key, `known_hosts` |
| `hermes-backup-check` | `check` | Restic password only |
| `hermes-backup-prune` | `prune` | Restic password only |

Production containers SHALL run as fixed non-root UID/GID `65532:65532` after confirming those IDs are unused on the NAS, use a read-only root filesystem, drop all capabilities, enable `no-new-privileges`, have no Docker socket, host device, privileged mode, published port, or auto-restart, use bounded tmpfs for `/tmp` with `noexec,nosuid,nodev`, mount only the repository and operation-specific secret/config files, and remain stopped between one-shot runs.

#### Scenario: Maintenance containers hold no SSH key

- **WHEN** the check or prune container is inspected
- **THEN** no SSH private-key mount is present

#### Scenario: Container hardening is verified

- **WHEN** a production container is inspected
- **THEN** it runs as `65532:65532`, its root filesystem rejects writes, its effective capabilities are empty, `no-new-privileges` is active, no port is published, no Docker socket or host device is mounted, and auto-restart is disabled

### Requirement: Scheduling starts existing containers and propagates failure

DSM Task Scheduler SHALL start existing containers by fixed name using an absolute Docker path. It SHALL NOT build an image, pull a moving tag, run Compose from a Git checkout, or execute an application script writable by an interactive DSM user. The scheduler command SHALL attach/wait and propagate the container exit code.

#### Scenario: No scheduled root task evaluates the working tree

- **WHEN** the scheduled task definitions are reviewed
- **THEN** none references a Git checkout, Compose file, build context, or user-writable script

#### Scenario: Deliberate failure notifies the operator

- **WHEN** a container is made to exit nonzero on purpose
- **THEN** DSM reports the task as failed and sends the configured abnormal-task notification, and this is demonstrated before migration is accepted

### Requirement: Secrets are file mounts from a dedicated protected location

The SSH private key and Restic password SHALL be individual read-only file mounts from a dedicated encrypted Synology location. That location SHALL be denied to unrelated DSM users and excluded from SMB, NFS, FTP, WebDAV, Synology Drive, indexing, and unrelated synchronization or backup jobs. `known_hosts` is not secret but is integrity-sensitive and SHALL be root-owned and non-writable by the container identity. Secret values SHALL NOT appear in environment variables, Docker commands, image metadata, logs, Git, or monitoring payloads. The repository SHALL be a separate shared folder from the secret store. At least two independent recovery copies of the Restic password and Synology encryption recovery material SHALL exist outside the NAS.

#### Scenario: Container inspection reveals no secret value

- **WHEN** `docker inspect` output and container environment are examined
- **THEN** they contain paths only, and no Restic password, private key, or registry token value

#### Scenario: Secret share is invisible to file services

- **WHEN** the secret location's DSM configuration is reviewed
- **THEN** it is not exposed through any file service, indexing, or unrelated synchronization or backup job

#### Scenario: Unlock profile is a recorded decision

- **WHEN** the unlock profile is chosen between manual unlock, an external key manager, and remote KMIP
- **THEN** the choice is recorded together with its availability consequences, and the record states that no local unlock design protects mounted secrets from live DSM root

### Requirement: Backup runs are bounded and fail closed

The backup mode SHALL:

1. acquire a non-blocking run lock;
2. verify repository, password, SSH key, and trusted-host material;
3. enforce a whole-run timeout covering SSH, stream transfer, and Restic finalization;
4. reject empty or undersized output;
5. reject oversized output without accepting a silently truncated stream;
6. invoke Restic with `--stdin-from-command` so a nonzero SSH/exporter exit creates no snapshot;
7. use stable `--host`, `--tag`, and `--stdin-filename` values;
8. compute the SHA-256 digest and byte length of the received stream as it passes to Restic;
9. emit a run manifest describing what was received;
10. report the created snapshot identifier; and
11. clean tmpfs material on all handled exits.

The live stream SHOULD continue directly into Restic rather than being written as a plaintext NAS archive. Initial minimum/maximum byte and duration limits SHALL be replaced with measured values plus documented headroom.

#### Scenario: Exporter failure creates no snapshot

- **WHEN** the SSH connection or the exporter exits nonzero
- **THEN** no Restic snapshot is created and the run exits nonzero

#### Scenario: Stalled source is terminated

- **WHEN** the source connects and then stalls past the whole-run timeout
- **THEN** the run is terminated, no snapshot is created, and the run exits nonzero

#### Scenario: Oversized stream is not silently truncated

- **WHEN** the source emits more than the configured maximum byte count
- **THEN** the run distinguishes the oversized stream from a stream exactly at the maximum, creates no snapshot, and exits nonzero

#### Scenario: Overlapping run is refused

- **WHEN** a second backup run starts while one holds the run lock
- **THEN** the second run fails safely without operating on the repository concurrently

### Requirement: Every attempt produces a retained run manifest

Each backup attempt SHALL produce one manifest record containing at least: schema version; UTC start and finish times; source identifier and verified SSH host-key fingerprint; exporter/SSH exit status; received byte length; SHA-256 digest of the received stream; deployed image digest; and the resulting Restic snapshot identifier or the reason no snapshot was created.

The manifest SHALL be emitted for failed attempts as well as successful ones, SHALL be retained outside the Restic repository, SHALL be durable for at least the snapshot retention period, and SHALL NOT contain secret values. The manifest SHALL NOT be carried in Restic tags, because retention groups by host and tag and a per-run tag value would place every snapshot in its own retention group.

#### Scenario: Failed attempt leaves evidence

- **WHEN** an attempt produces no snapshot
- **THEN** a manifest record is still written and retained, giving the attempt a trace that the repository itself cannot provide

#### Scenario: Digest is computed on the NAS over received bytes

- **WHEN** the digest and byte length are recorded
- **THEN** they are computed on the NAS over the bytes actually received, are never taken from a source-supplied value, and their computation neither buffers the stream to a plaintext NAS file, suppresses the exporter's exit status, nor prevents Restic from cancelling the snapshot on producer failure

#### Scenario: Restored snapshot re-hashes to its recorded digest

- **WHEN** a snapshot is restored and re-hashed
- **THEN** it matches the digest recorded in that run's manifest, which is evidence about transfer rather than about the truthfulness of the source data

### Requirement: Retention, check, and prune are separated

Daily backup MAY run `forget` only after a successful snapshot and SHALL NOT run `prune`. Weekly check and prune SHALL run in separate non-overlapping windows. Check and prune containers SHALL NOT receive the Hermes SSH key. Retention SHALL select the same stable host and tag used during backup. A full-data check and representative restore SHALL occur periodically; metadata checks alone are insufficient.

#### Scenario: Retention is skipped after a failed backup

- **WHEN** a backup attempt fails
- **THEN** no `forget` runs in that invocation

#### Scenario: Retention selects the stable host and tag

- **WHEN** retention runs
- **THEN** it selects snapshots by the same `--host` and `--tag` values used at backup time
