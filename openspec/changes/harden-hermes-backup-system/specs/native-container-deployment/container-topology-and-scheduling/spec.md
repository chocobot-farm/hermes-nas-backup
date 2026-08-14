## Purpose

Defines the three fixed NAS containers, the mounts and settings each receives, how they are created as an explicit privileged operation, and how DSM Task Scheduler starts them without touching a Git checkout.

## ADDED Requirements

### Requirement: The daily container backs up with the full mount set

The daily container SHALL be named `hermes-backup-daily` and configured with:

```text
MODE=backup
HERMES_SSH_TARGET=user@host
SSH_PORT=22
SSH_CONNECT_TIMEOUT=20
SSH_SERVER_ALIVE_INTERVAL=30
SSH_SERVER_ALIVE_COUNT_MAX=3
MAX_BACKUP_SECONDS=14400
MIN_EXPORT_BYTES=10240
MAX_EXPORT_BYTES=107374182400
RESTIC_HOST=hermes-server
RESTIC_TAG=hermes
FORGET_AFTER_BACKUP=true
PRUNE_AFTER_BACKUP=false
KEEP_DAILY=7
KEEP_WEEKLY=5
KEEP_MONTHLY=12
```

Its mounts SHALL be the Restic repository at `/repository` read/write, the Restic password file at `/run/secrets/restic_password` read-only, the SSH private key at `/run/secrets/hermes_ssh_key` read-only, and the verified `known_hosts` at `/run/config/known_hosts` read-only.

#### Scenario: Daily run creates a snapshot and applies retention

- **WHEN** the daily container runs successfully
- **THEN** a Restic snapshot is created under the configured host and tag, `forget` runs afterwards, and `prune` does not

#### Scenario: Initial bounds are replaced by measured values

- **WHEN** successful export sizes and durations have been observed
- **THEN** `MIN_EXPORT_BYTES`, `MAX_EXPORT_BYTES`, and `MAX_BACKUP_SECONDS` are replaced with measured values plus documented headroom

### Requirement: Maintenance containers receive no SSH material

The check container SHALL be named `hermes-backup-check` with `MODE=check`, the configured `RESTIC_HOST` and `RESTIC_TAG`, and `CHECK_READ_DATA_SUBSET` (initially `5%`). The prune container SHALL be named `hermes-backup-prune` with `MODE=prune` and the configured host and tag. Both SHALL mount only the repository read/write and the Restic password file read-only, and SHALL NOT receive the SSH private key or known-hosts mount.

#### Scenario: Check container mounts

- **WHEN** `hermes-backup-check` is inspected
- **THEN** it has the repository and password mounts only

#### Scenario: Prune container mounts

- **WHEN** `hermes-backup-prune` is inspected
- **THEN** it has the repository and password mounts only

### Requirement: Administrative modes use temporary or dedicated containers

`init` and `snapshots` SHOULD use temporary manually created containers, or dedicated stopped containers, that receive only their required mounts.

#### Scenario: Snapshot listing without SSH material

- **WHEN** an operator lists snapshots
- **THEN** the container used receives the repository and password only

### Requirement: Containers are created once as an explicit privileged operation

Production containers SHOULD be created once with native `docker create` commands and then managed and observed through Synology Container Manager, because the documented UI does not expose every required hardening setting — particularly read-only root, tmpfs, and `no-new-privileges`. The creation command MAY be entered interactively or generated on a trusted workstation, but it SHALL NOT be stored in a normal user's writable scheduled script on the NAS.

The implementation documentation SHALL provide a command equivalent to this template, with placeholders resolved and the image specified by digest:

```bash
docker create \
  --name hermes-backup-daily \
  --user 65532:65532 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=128m,mode=0700,uid=65532,gid=65532 \
  --cap-drop ALL \
  --security-opt no-new-privileges=true \
  --restart no \
  --network bridge \
  --mount type=bind,src=REPOSITORY_PATH,dst=/repository \
  --mount type=bind,src=PASSWORD_PATH,dst=/run/secrets/restic_password,readonly \
  --mount type=bind,src=SSH_KEY_PATH,dst=/run/secrets/hermes_ssh_key,readonly \
  --mount type=bind,src=KNOWN_HOSTS_PATH,dst=/run/config/known_hosts,readonly \
  --env MODE=backup \
  --env HERMES_SSH_TARGET=USER_AT_HOST \
  --env SSH_PORT=22 \
  --env SSH_CONNECT_TIMEOUT=20 \
  --env SSH_SERVER_ALIVE_INTERVAL=30 \
  --env SSH_SERVER_ALIVE_COUNT_MAX=3 \
  --env MAX_BACKUP_SECONDS=14400 \
  --env MIN_EXPORT_BYTES=10240 \
  --env MAX_EXPORT_BYTES=107374182400 \
  --env RESTIC_HOST=hermes-server \
  --env RESTIC_TAG=hermes \
  --env FORGET_AFTER_BACKUP=true \
  --env PRUNE_AFTER_BACKUP=false \
  --env KEEP_DAILY=7 \
  --env KEEP_WEEKLY=5 \
  --env KEEP_MONTHLY=12 \
  IMAGE_REFERENCE_AT_DIGEST
```

Equivalent templates SHALL be provided for `check` and `prune`, omitting their unneeded mounts and environment variables. The final implementation MAY add resource and log limits after testing but SHALL NOT weaken the required security options.

#### Scenario: Creation command is not left in a user-writable script

- **WHEN** the NAS is reviewed after container creation
- **THEN** no normal user's writable scheduled script contains the creation command

#### Scenario: UI fallback documents lost controls

- **WHEN** containers are created entirely through Container Manager's UI because the DSM version cannot be driven otherwise
- **THEN** any inability to configure read-only root, tmpfs, capability dropping, or `no-new-privileges` is explicitly documented, and secrets remain file mounts rather than environment variables

### Requirement: DSM starts existing containers by fixed name

DSM Task Scheduler SHALL start existing containers by their fixed names and SHALL NOT run Compose, build an image, pull an unreviewed tag, or evaluate files from a user-writable project directory. The intended command shape is:

```bash
ABSOLUTE_DOCKER_PATH start --attach hermes-backup-daily
```

Equivalent weekly tasks start `hermes-backup-check` and `hermes-backup-prune`. The deployment procedure SHALL discover and record the actual Docker CLI path on the target DSM version and use that absolute path in Task Scheduler.

#### Scenario: Docker CLI path is recorded

- **WHEN** the scheduled task is created
- **THEN** it invokes the discovered absolute Docker path rather than relying on `PATH`

#### Scenario: Scheduler never builds or pulls

- **WHEN** the scheduled task definitions are reviewed
- **THEN** none builds an image, runs Compose, pulls a moving tag, or executes a user-writable script

### Requirement: Scheduled runs propagate exit status and notify

Before production scheduling, an acceptance test SHALL demonstrate that a successful container exit is reported as task success, a deliberately failed container exit is reported as task failure, stdout/stderr or Docker logs contain actionable diagnostics, and DSM sends the configured abnormal-task notification. If `docker start --attach` on the installed DSM/Docker version does not propagate the container's exit status, the scheduled command SHALL also wait for the container and explicitly return its recorded exit code, and any such command SHALL remain stored inside DSM Task Scheduler or another root-controlled location.

#### Scenario: Exit status does not propagate natively

- **WHEN** the installed Docker version does not propagate the container exit code through `start --attach`
- **THEN** the scheduled command waits for the container and returns its recorded exit code, and that command lives in a root-controlled location

#### Scenario: Failure notification reaches the operator

- **WHEN** a scheduled container exits nonzero
- **THEN** DSM reports task failure and the configured notification reaches the operator

### Requirement: Schedules are separated and containers stay stopped between runs

Recommended initial scheduling is a daily backup during a quiet window, a weekly repository check outside the backup window, a weekly prune outside both other windows, and an immutable Synology snapshot after the normal backup completion window. The shared repository lock remains mandatory, and a concurrent task SHALL fail safely rather than operate on the repository simultaneously. Containers SHALL remain stopped between runs; a long-running cron process inside the image is out of scope.

#### Scenario: Concurrent task hits the repository lock

- **WHEN** two repository operations are triggered at once
- **THEN** the second fails safely without operating on the repository concurrently

#### Scenario: Containers between runs

- **WHEN** no scheduled run is active
- **THEN** all production containers are stopped
