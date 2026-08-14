## Purpose

States the foundational assumption that Hermes is a hostile tenant rather than a server that might eventually be compromised, fixes the authority each component holds, and defines the source-side contract that limits what possession of a pull key can accomplish.

## ADDED Requirements

### Requirement: Hermes is treated as a hostile tenant

The design SHALL assume Hermes can execute attacker-influenced code, make arbitrary outbound Internet connections, modify its application data before a backup, modify files owned by its runtime identity, return malicious, inconsistent, incomplete, or unbounded backup output, detect and interfere with in-guest backup activity, stop responding or deliberately fail a backup, and retain information it can read during normal operation.

Hermes SHALL NOT be given a repository encryption password, a repository transport credential, a credential capable of deleting, pruning, or expiring backups, a credential for the NAS, backup gateway, hypervisor, or off-site service, access to a backup scheduler or monitoring success signal, or a network route to repository or management endpoints.

#### Scenario: Guest holds no backup authority

- **WHEN** the guest's credentials and reachable endpoints are enumerated
- **THEN** none of the prohibited credentials or routes is present

#### Scenario: The system does not claim to make hostile data truthful

- **WHEN** the guarantees of the architecture are stated
- **THEN** they cover protecting previously captured recovery points, detecting missing or anomalous future backups, and providing an independent machine-level recovery path, and explicitly exclude proving that data supplied after source compromise is honest

#### Scenario: Fallback pressure to relocate credentials

- **WHEN** an operational problem would be solved by moving repository credentials onto Hermes
- **THEN** the change is refused

### Requirement: Component authority is explicitly bounded

The following trust and authority assignments SHALL hold:

| Component | Trust and authority |
| --- | --- |
| Hermes | Hostile data producer; no backup authority |
| Hypervisor (PVE) | Trusted to isolate Hermes, capture machine state outside the guest, and hold the client-side image encryption key |
| Backup gateway | Trusted to schedule pulls, hold routine secrets, bound hostile output, and encrypt data |
| Image backup server (PBS) | Trusted for storage, integrity verification, and retention; has no image decryption key |
| Synology NAS | Untrusted for confidentiality; trusted only for local storage availability within its snapshot window |
| NAS NFS export | Trusted only to the extent that address-based `AUTH_SYS` trust holds on the storage network |
| Local repository service | Accepts authenticated append-only Restic traffic; has no repository decryption key |
| Off-site provider | Untrusted for confidentiality; trusted to enforce contracted immutability or append-only behavior |
| Maintenance workstation | Highly trusted, normally offline or isolated; temporarily holds deletion authority |
| Monitoring service | Trusted to report observed backup outcomes; has no repository decryption or deletion authority |
| Recovery operator | Trusted to authorize and inspect restores before restored content reaches production |

The component that can both read current Hermes data and decrypt repository history SHALL have a smaller attack surface and a stricter network policy than both Hermes and the NAS, and SHALL NOT run general workloads, browsers, or agents.

#### Scenario: Highest-value host is kept single-purpose

- **WHEN** a general workload, browser, or agent is proposed for the host holding both source-read capability and history decryption keys
- **THEN** the proposal is refused

#### Scenario: NAS authority is bounded by encryption but not by availability

- **WHEN** NAS root or DSM administrator authority is assessed
- **THEN** it is recorded as reaching every local recovery point on both paths, prevented from reading image content by client-side encryption, and not prevented from destroying it

### Requirement: The source exposes one restricted key entry

The source SHALL normally expose exactly one SSH public key entry for the pull client; a second entry MAY coexist temporarily during a tested rotation. The entry SHALL use an equivalent of:

```text
from="GATEWAY_IP",restrict,command="/usr/local/libexec/hermes-backup/export" ssh-ed25519 ...
```

The entry SHALL be unique to this source, restrict the accepted source address, use `restrict`, force one absolute command, ignore or reject `SSH_ORIGINAL_COMMAND`, prohibit shell, PTY, forwarding, agent forwarding, X11, and user RC files, and be independently revocable. The account SHOULD have no password and no unrelated interactive or application access.

#### Scenario: Key from an unexpected address

- **WHEN** the key is presented from an address other than the permitted source
- **THEN** authentication is refused

#### Scenario: Only the forced command runs

- **WHEN** the key is used with any supplied command
- **THEN** the forced absolute command runs and the supplied command is rejected

### Requirement: The exporter and its helpers are integrity-protected

The forced-command executable and every executable helper it invokes SHALL reside outside a normal user's writable home directories, be owned by root, not be writable by the Hermes runtime identity, use absolute paths, not load executable code or dependencies from user-writable locations, treat application-owned configuration and data as untrusted inputs, and be installed from an identified, reviewed release artifact.

The exporter MAY run with the Hermes application identity when required to read data; root ownership of the exporter protects the protocol implementation and does not make application-owned data trustworthy.

#### Scenario: Exporter under a user-writable path

- **WHEN** the forced command resolves into a path the Hermes runtime identity can rewrite, including a symlink into a user-owned checkout
- **THEN** the deployment does not satisfy this requirement, regardless of change-management controls on that checkout

#### Scenario: Helper loaded from a user-writable location

- **WHEN** the exporter would load an executable or dependency from a user-writable location
- **THEN** the invocation is refused

### Requirement: The stream protocol is fixed and self-describing

On success, stdout SHALL contain only one uncompressed TAR stream, and status and diagnostic output SHALL go to stderr. The stream SHALL contain:

```text
RESTORE.txt
hermes/hermes.zip
```

The exporter SHALL use the supported Hermes backup interface, omit transient lock, WAL, and shared-memory files, describe in `RESTORE.txt` only the archives actually present, fail nonzero if required data is absent or inconsistent, clean its temporary data on normal exit and signals, and emit no secret value. Additional tool state MAY later be added under its own top-level directory, documenting how it is captured consistently and without weakening these requirements.

No source-provided checksum, signature, status string, or manifest SHALL be treated as proof that hostile source data is truthful.

#### Scenario: Diagnostics never contaminate stdout

- **WHEN** the exporter emits status or error text
- **THEN** it appears on stderr and the stdout stream remains a single valid TAR

#### Scenario: Source-supplied integrity claim

- **WHEN** the source includes its own checksum or signature in the stream
- **THEN** it is not used as evidence of truthfulness by any consumer of the backup
