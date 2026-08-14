## Purpose

Defines the contract by which the Synology client obtains an application-consistent Hermes export over restricted SSH: how the pull key is authorized, how the exporter is installed so the guest runtime account cannot rewrite it, and what the emitted stream contains.

## ADDED Requirements

### Requirement: The pull key is restricted to one forced command

The Synology pull key SHALL be unique to Hermes and authorized with the equivalent of:

```text
from="NAS_SOURCE_IP",restrict,command="/usr/local/libexec/hermes-backup/export" ssh-ed25519 PUBLIC_KEY synology-hermes-backup
```

The authorization SHALL force one absolute command, reject a nonempty `SSH_ORIGINAL_COMMAND`, prohibit shell, PTY, forwarding, agent forwarding, X11, and user RC files, restrict the observed source address where stable addressing permits, and be independently revocable.

#### Scenario: Arbitrary command is rejected

- **WHEN** a client presents the backup key and requests any command other than the forced one
- **THEN** the request is refused, the forced command is not executed with the supplied command, and no backup data is emitted to stdout

#### Scenario: Interactive access is refused

- **WHEN** a client presents the backup key and requests a shell, PTY, port forwarding, agent forwarding, or X11
- **THEN** the request is refused

#### Scenario: Key is revoked independently

- **WHEN** the backup authorization is removed
- **THEN** unrelated administrator keys in the same `authorized_keys` file remain intact and functional

### Requirement: The client authenticates the source host strictly

The client SHALL use `BatchMode=yes`, `IdentitiesOnly=yes`, `StrictHostKeyChecking=yes`, and a separately provisioned `known_hosts` file. Host-key fingerprints SHALL be verified out of band.

#### Scenario: Unknown or changed host key

- **WHEN** the source presents a host key that does not match the provisioned `known_hosts` entry
- **THEN** the connection fails, no export is transferred, and the run exits nonzero

### Requirement: The exporter is root-owned and not writable by the Hermes runtime account

The deployment SHALL replace the user-writable `/home/anton/.local/bin/hermes-backup-stream` protocol entrypoint with root-owned files equivalent to:

```text
/usr/local/libexec/hermes-backup/                 root:root 0755
/usr/local/libexec/hermes-backup/export           root:root 0755
/home/anton/.cache/hermes-backup/                 anton     0700
```

The wrapper SHALL use fixed absolute executable paths, SHALL NOT trust environment overrides for executables or helpers, SHALL remain non-writable by the Hermes runtime account, SHALL reserve stdout exclusively for the TAR stream, SHALL send status and errors to stderr, SHALL create private temporary directories, SHALL install cleanup traps before creating sensitive temporary data, and SHALL fail nonzero on any incomplete or inconsistent export.

#### Scenario: Runtime account cannot replace the protocol

- **WHEN** the Hermes runtime account attempts to write the exporter, a protocol helper, or any of their parent directories
- **THEN** the write is refused

#### Scenario: Environment override is ignored

- **WHEN** an environment variable naming an alternative executable or helper path is presented to the forced command
- **THEN** the exporter uses its fixed absolute paths regardless

#### Scenario: Temporary plaintext is removed on any exit

- **WHEN** the exporter exits normally, fails, or receives a handled termination signal
- **THEN** its temporary export data is removed

### Requirement: Source-side installation is idempotent and reversible

A repository-supplied idempotent Ansible playbook SHOULD install and remove these files and manage only a marked backup block in `authorized_keys`. It SHALL preserve unrelated authorization entries and SHALL support two restricted keys during rotation.

#### Scenario: Repeat run reports no changes

- **WHEN** the playbook is applied a second time with unchanged inputs
- **THEN** it reports no changes

#### Scenario: Rotation with two active keys

- **WHEN** a second restricted public key is added during rotation
- **THEN** both keys are authorized with their own restricted entries until the old entry is removed

### Requirement: The stream is one TAR with a fixed, documented layout

On success the forced command SHALL emit one uncompressed TAR containing:

```text
RESTORE.txt
hermes/hermes.zip
```

The exporter SHALL invoke the supported Hermes backup command, require a nonempty Hermes archive, exclude lock, WAL, SHM, runtime, and reinstallable environment files, emit restore instructions describing only the archives actually present, and remove temporary plaintext after success, failure, or handled termination. Additional tool state MAY be added later under its own top-level directory; any such addition SHALL document its own consistency method and SHALL NOT weaken these guarantees.

#### Scenario: Empty application archive

- **WHEN** the Hermes backup command produces an empty archive
- **THEN** the exporter exits nonzero and emits no TAR stream on stdout

#### Scenario: Restore instructions match stream contents

- **WHEN** the stream is inspected
- **THEN** `RESTORE.txt` describes only the archives actually present in the TAR

### Requirement: The NAS treats the stream as untrusted bytes

The NAS SHALL treat the stream as untrusted bytes during ingestion and SHALL NOT extract it as part of the scheduled backup path. No source-provided checksum, signature, status string, or manifest SHALL be treated as proof that source data is truthful.

#### Scenario: Scheduled backup does not extract the stream

- **WHEN** a scheduled backup ingests the export
- **THEN** the TAR is never parsed, listed, decompressed, or extracted during the backup run

#### Scenario: Malformed TAR is stored without interpretation

- **WHEN** the source emits a deliberately malformed TAR within the accepted byte bounds and exits zero
- **THEN** the bytes are stored as an ordinary snapshot and the malformation is contained by the isolated restore procedure rather than by the ingestion path
