## Purpose

Defines the published image's externally observable runtime contract: the fixed unprivileged identity it runs as, the paths it may write, the one-shot modes it supports, and the privilege options every production container must carry.

## ADDED Requirements

### Requirement: The image runs as a fixed unprivileged identity

The published image SHALL use the fixed identity UID `65532` and GID `65532` unless a future specification revision changes it. The Dockerfile SHALL create or select that identity and end with an equivalent of `USER 65532:65532`. The entrypoint SHALL NOT start as root in order to implement a `PUID`/`PGID` ownership rewrite. Container creation SHOULD additionally set `--user 65532:65532` as defense in depth. NAS installation SHALL verify that UID and GID 65532 are not assigned to an unrelated local principal before changing file ownership.

#### Scenario: Process identity at runtime

- **WHEN** a production container runs
- **THEN** its process runs as `65532:65532` and never as root

#### Scenario: Identity collision on the NAS

- **WHEN** UID or GID 65532 is already assigned to an unrelated local principal
- **THEN** installation stops before changing file ownership

### Requirement: The filesystem contract is read-only with two writable paths

The image SHALL operate with a read-only root filesystem. The only writable paths available to a normal run SHALL be `/repository`, backed by the Restic repository bind mount, and `/tmp`, backed by an ephemeral tmpfs. The image SHALL NOT declare a persistent anonymous volume for `/tmp`. The image SHALL contain the empty mount-point parent directories `/run/secrets` and `/run/config`, owned by root and not writable by UID/GID 65532.

The default runtime paths are:

```text
RESTIC_REPOSITORY=/repository
RESTIC_PASSWORD_FILE=/run/secrets/restic_password
RESTIC_CACHE_DIR=/tmp/restic-cache
SSH_KEY_FILE=/run/secrets/hermes_ssh_key
SSH_KNOWN_HOSTS_FILE=/run/config/known_hosts
```

These defaults SHOULD be embedded in the image, and normal NAS configuration SHOULD not override them.

#### Scenario: Write outside the permitted paths

- **WHEN** the running container attempts to write anywhere other than `/repository` or `/tmp`
- **THEN** the write is refused by the read-only root filesystem

#### Scenario: Mount points exist under a read-only root

- **WHEN** secret and config files are mounted
- **THEN** `/run/secrets` and `/run/config` already exist in the image, root-owned and not writable by UID 65532, so the runtime need not synthesize them

### Requirement: The entrypoint is a one-shot process with defined modes

The entrypoint SHALL remain a one-shot process that exits after completing the requested operation. Supported modes are:

| Mode | Purpose | Required mounts |
| --- | --- | --- |
| `init` | Initialize an empty Restic repository | Repository RW, password RO |
| `backup` | Stream a Hermes backup and apply configured retention | Repository RW, password RO, SSH key RO, known_hosts RO |
| `snapshots` | List matching snapshots | Repository RW, password RO |
| `check` | Check repository metadata and configured data subset | Repository RW, password RO |
| `prune` | Remove unreferenced repository data | Repository RW, password RO |

Modes that do not use SSH SHALL NOT be given the SSH private-key mount.

#### Scenario: One-shot exit

- **WHEN** any mode completes
- **THEN** the process exits and no scheduler or long-running loop remains inside the container

#### Scenario: Repository initialization is not destructive

- **WHEN** `init` runs against a repository whose `config` already exists
- **THEN** the existing repository is neither overwritten nor reinitialized

### Requirement: The entrypoint fails closed and bounds its source

The entrypoint SHALL preserve `set -Eeuo pipefail` behavior, use arrays or equivalent safe argument construction, never log secret values, refuse unreadable required files, preserve strict SSH host-key checking, preserve Restic `--stdin-from-command` failure propagation, enforce configured minimum and maximum source byte counts without writing a plaintext spool to persistent storage, enforce a maximum duration for backup snapshot creation covering the source command and Restic finalization rather than only SSH connection establishment, distinguish an oversized stream from a successful stream at exactly the configured maximum, retain the repository lock preventing overlapping operations, clean temporary runtime files on normal exit and signals, and return nonzero for every failed backup, check, prune, initialization, or source command.

#### Scenario: Stream exactly at the maximum

- **WHEN** the source emits exactly `MAX_EXPORT_BYTES`
- **THEN** the run succeeds and creates a snapshot

#### Scenario: Stream above the maximum

- **WHEN** the source emits more than `MAX_EXPORT_BYTES`
- **THEN** the run detects the excess rather than silently truncating, creates no snapshot, and exits nonzero

#### Scenario: Unreadable required file

- **WHEN** a required password, key, or known-hosts file cannot be read
- **THEN** the run refuses to continue and exits nonzero

### Requirement: Production containers carry fixed privilege restrictions

All production containers SHALL be created with the equivalent of:

```text
--read-only
--cap-drop ALL
--security-opt no-new-privileges=true
--restart no
--user 65532:65532
```

They SHALL NOT use `--privileged`, the host PID or IPC namespace, a Docker socket mount, host devices, added Linux capabilities, a published TCP/UDP port, or host networking unless a documented compatibility test proves bridge networking cannot meet the requirement and a security review accepts the change.

The `/tmp` mount SHALL be an ephemeral tmpfs with an initial target configuration equivalent to `rw,noexec,nosuid,nodev,size=128m,mode=0700,uid=65532,gid=65532`. Its size SHALL be configurable at container creation because Restic may need more space for concurrent pack staging on larger repositories. No web portal is required, and auto-restart SHALL be disabled for these one-shot containers.

#### Scenario: Privilege inspection

- **WHEN** a production container is inspected
- **THEN** effective capabilities are empty, `no-new-privileges` is active, no port is published, no Docker socket or host device is mounted, and auto-restart is disabled

#### Scenario: Host networking is proposed

- **WHEN** host networking is proposed for a production container
- **THEN** it is used only after a documented compatibility test and an accepting security review

#### Scenario: Larger tmpfs for a larger repository

- **WHEN** Restic pack staging needs more than the initial 128 MiB
- **THEN** the tmpfs size is raised at container creation without weakening its `noexec,nosuid,nodev` options
