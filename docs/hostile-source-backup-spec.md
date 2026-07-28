# Hostile-source Hermes backup architecture specification

Status: Rejected as a whole. Partially realized by the current deployment.

Date: 2026-07-16

Revised: 2026-07-27 for the current Proxmox VE / Proxmox Backup Server /
Synology deployment.

Related assessment: [Security findings](security-findings.md)

System-level target:
[Unified backup safety specification](unified-backup-safety-spec.md)

Deployment of record: [Securely Deploying Hermes Agent in a Proxmox
Homelab](https://github.com/chocobot-farm/plume-pilot/blob/main/docs/deployment/0001-hermes-homelab-pve-pbs-synology.md)

Decision record: This document remains a rejected alternative. It does not
supersede the accepted [native Synology container
deployment](native-container-spec.md) or the unified specification. Its
dedicated backup-gateway architecture was not built. Its hypervisor profile
was. This revision records which of its requirements the deployment now
satisfies, which it does not, and which are no longer worth building.

## 1. Purpose

This specification records an alternative backup architecture for Hermes when the
source is an Internet-connected AI agent and therefore cannot be trusted with
backup credentials, backup policy, backup scheduling, or network access to
backup storage.

The design uses an independent trusted backup gateway to pull an
application-consistent export from Hermes, encrypt it, and write the same export
to two protected repositories. The Synology NAS is a storage service, not the
backup client. When Hermes is deployed as a virtual machine on a hypervisor that
supports image backup, a hypervisor-controlled image backup is also required.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in
this document describe implementation requirements.

### 1.1 Relationship to the current deployment

When this document was written, Hermes ran on physical hardware and the only
backup path was a Synology container pulling an application export over SSH.
The hypervisor profile in section 6.2 was hypothetical. It is now deployed.

Hermes runs as an Ubuntu VM on a dedicated Proxmox VE host. PVE performs a
daily Stop-mode whole-VM backup, encrypts it with AES-256-GCM before it leaves
the PVE host, and writes it to a Proxmox Backup Server VM running under
Synology Virtual Machine Manager, whose datastore is a Btrfs shared folder
exported to the PBS VM over NFSv4.1. The Synology Restic pull client continues
to run unchanged alongside it.

That deployment supplies the property this document was written to obtain: a
recovery path whose scheduling, credentials, and encryption keys live outside
the guest and do not depend on guest cooperation. It supplies that property
without the dedicated backup gateway proposed here.

The dedicated gateway of sections 6.1 and 9 is therefore no longer the
recommended direction. The requirements that matter — out-of-guest control,
bounded hostile output, credential separation, independent failure domains, and
isolated restore — are carried forward by the
[unified specification](unified-backup-safety-spec.md), which treats PVE/PBS as
the machine-level control plane and the Synology client as the application-level
path.

| Requirement class | Section | Current state |
| --- | --- | --- |
| Backup control outside the guest | 3.3, 12 | Satisfied by PVE for the image path; the application path is scheduled outside the guest but on the NAS, which goal 3.3 also excludes |
| Repository credentials outside the guest | 3.1, 8 | Satisfied on both paths |
| Content encrypted before reaching storage | 3.6 | Satisfied on both paths |
| Routine identity cannot prune or delete | 3.7, 13 | Satisfied for PBS; not satisfied for the local Restic repository |
| Guest cannot route to backup or management endpoints | 3.2, 7 | Not satisfied; the deployment is a flat LAN |
| Bounded, size- and time-limited hostile export | 3.5, 9.5 | Not satisfied; the application export is unbounded |
| Per-run manifest binding the exported bytes | 9.5 | Not satisfied; carried into the unified specification |
| Independent off-site failure domain | 3.8, 11 | Not satisfied; both repositories live on one NAS |
| Immutable local recovery points | 10.3 | Not satisfied; neither datastore has protected snapshots |
| Failure detected without a guest signal | 3.10, 14 | Not satisfied; neither path is externally monitored |
| Isolated restore of hostile content | 3.11, 15 | Satisfied in procedure for VM images; untested for application restores |

Sections marked not satisfied are findings against the deployment, not
proposals to build the gateway. They are tracked in the
[security findings](security-findings.md) and specified in the unified
document.

### 1.2 Terminology

The deployment guide uses "gateway" for the Hermes messaging gateway, the
always-on process that connects Hermes to Discord, Telegram, or Slack. This
document uses "backup gateway" for the proposed independent pull host. The two
are unrelated, and the messaging gateway runs inside the hostile guest.

## 2. Foundational assumption

Hermes MUST be treated as a hostile tenant, not merely as a server that might
eventually be compromised.

The design MUST assume Hermes can:

- execute attacker-influenced code;
- make arbitrary outbound Internet connections;
- modify its application data before a backup;
- modify files owned by its runtime identity;
- return malicious, inconsistent, incomplete, or unbounded backup output;
- detect and interfere with in-guest backup activity;
- stop responding or deliberately fail a backup; and
- retain information it can read during normal operation.

Hermes MUST NOT be given:

- a Restic or other repository encryption password;
- a repository transport credential;
- a credential capable of deleting, pruning, or expiring backups;
- a credential for the NAS, backup gateway, hypervisor, or off-site service;
- access to a backup scheduler or monitoring success signal; or
- a network route to repository or management endpoints.

No backup architecture can force a hostile source to produce truthful future
data. The system protects previously captured recovery points, detects missing
or anomalous future backups, and provides an independent machine-level recovery
path. It does not prove that data supplied after source compromise is honest.

## 3. Goals

The implementation MUST:

1. Keep all repository credentials and encryption keys outside Hermes.
2. Prevent Hermes from initiating network connections to backup components.
3. Schedule and initiate backups from a system outside Hermes and the NAS.
4. Preserve the application-consistent Hermes export.
5. Treat all source output as hostile, opaque, and potentially unbounded.
6. Encrypt backup content before it reaches either storage provider.
7. Give the normal backup path no ability to overwrite or delete existing
   local recovery points.
8. Maintain a second encrypted copy in an independent off-site failure domain
   with append-only or immutable protection.
9. Separate routine backup credentials from maintenance credentials.
10. Detect backup failure without trusting a signal emitted by Hermes.
11. Test restoration in an isolated environment that treats restored content
    as hostile.
12. Support credential rotation, incident response, rollback, and migration
    from the current NAS-pull implementation.

The implementation SHOULD:

1. Run Hermes in an isolated virtual machine. **Done.**
2. Add an image-level backup controlled by the hypervisor. **Done.**
3. Use Secure Boot, full-disk encryption, and TPM-bound credentials on the
   backup gateway where supported.
4. Randomize the application-backup start time within a documented window.
5. Use independent encryption keys for the local and off-site repositories.
6. Preserve at least 30 days of immutable off-site recovery points.

## 4. Non-goals

This design does not:

- make new data trustworthy after Hermes has been compromised;
- prevent Hermes from destroying or exfiltrating data it can already read;
- protect backup secrets from live root compromise of the backup gateway;
- make a same-NAS snapshot an off-site backup;
- allow unattended pruning from the normal backup identity;
- require the gateway to interpret or extract the source TAR stream;
- rely on a backup success message sent by Hermes; or
- make repository checks a substitute for full restore tests.

## 5. Security boundaries

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

Compromise of the backup gateway is the most serious technical compromise in
the baseline design because the gateway can pull current Hermes data and decrypt
repository history. The gateway MUST therefore have a smaller attack surface
and stricter network policy than both Hermes and the NAS.

In the deployed hypervisor profile the equivalent position is held by PVE: it
reads Hermes disk blocks in plaintext and holds the AES key that decrypts image
history. PVE MUST therefore be treated as the highest-value host in the system
and MUST NOT run general workloads, browsers, or agents alongside the
hypervisor role.

The NAS now occupies a second concentrated position, because it hosts the PBS
VM, serves the NFS export that PBS writes to, and stores the Restic repository.
NAS root or DSM administrator authority reaches every local recovery point on
both paths. Client-side encryption keeps that authority from reading image
content; it does not keep it from destroying it.

## 6. Target architecture

### 6.1 Baseline application-backup architecture

```text
                         constrained SSH pull
Hermes AI agent  <--------------------------------  Dedicated backup gateway
no backup secrets                                  scheduler, keys, bounded spool
                                                               |
                                      same export, independent encrypted writes
                                               +---------------+---------------+
                                               |                               |
                                               v                               v
                                  Synology rest-server              Off-site repository
                                  append-only                       immutable or append-only
                                  no repository decrypt key         no repository decrypt key
                                  immutable Btrfs snapshots         independent failure domain

       Isolated maintenance workstation ---> retention, checks, and restore
       External monitoring service --------> observes gateway/storage results
```

The gateway MUST pull one export into a bounded spool and MUST back up those
exact bytes to both repositories. It MUST NOT request two independent exports
for the two targets because a hostile source could return different content.

### 6.2 Deployed hypervisor profile

This profile is no longer hypothetical. It is the deployment of record.

```text
Hermes VM (hostile guest)
   |
   | disk state captured outside guest control, guest stopped cleanly
   v
Proxmox VE host ---- AES-256-GCM client-side encryption ----> PBS VM on Synology
   |                                                              |
   | holds the only copy of the image decryption key              | NFSv4.1
   v                                                              v
Password manager + encrypted offline media              Btrfs shared folder
                                                        (no off-site copy today)
```

The implemented arrangement satisfies the following requirements of this
section:

1. The Hermes guest possesses neither the PBS API token nor the image
   encryption key.
2. The PVE identity is a privilege-separated API token holding `DatastoreBackup`
   on a single datastore path. It can create and restore its own backup groups
   and cannot administer, prune, or arbitrarily delete the datastore.
3. Retention runs on PBS, so the routine backup identity never needs deletion
   authority.
4. PVE pins the PBS TLS certificate fingerprint, so the storage endpoint is
   authenticated rather than merely reachable.
5. Backup content is encrypted on PVE before transmission. PBS and the NAS
   store ciphertext and hold no decryption key.
6. PBS verifies chunk hashes, authenticated-encryption metadata, and manifests
   without the decryption key, so integrity checking is performed by a
   component that cannot read what it checks.
7. Backup, prune, verification, and garbage collection run in separate,
   non-overlapping windows.
8. PBS runs on the NAS rather than on the PVE host it protects, so loss of the
   PVE host does not take the recovery points with it.

The following requirements of this specification are **not** satisfied by the
deployment:

- The PBS datastore has no independent off-site copy, and no protected or
  immutable snapshot on the NAS shared folder. Sections 10.3 and 11 remain
  unmet for the image path as well as the application path.
- The NFS export is authorized by the PBS VM's address under `AUTH_SYS`.
  Anything that can occupy that address on the storage network can write or
  delete datastore chunks. Encryption prevents reading them; it does not
  prevent destroying them.
- The image encryption key is unattended on PVE by necessity, so live PVE root
  yields both current guest plaintext and image history. This mirrors the
  gateway-compromise position described in section 5 and section 17.4.

Stop-mode backup gives a stronger consistency guarantee than the
crash-consistent image this document originally assumed: PVE performs an
orderly guest shutdown before capturing the disks, so the filesystem is clean
rather than merely frozen. That strengthens the image path but does not replace
the application export. The image is a whole-machine artifact of a machine that
may be compromised, whereas the application export is granular, portable, and
restorable without rebuilding a VM. The two recovery paths address different
failures and both remain required.

A hostile guest can still influence what Stop-mode captures, because it
controls what is on disk at shutdown. Image backup removes the guest from the
backup control plane; it does not make guest data truthful.

### 6.3 Constrained fallback, as deployed

The NAS-initiated pull described here as a temporary fallback is what runs in
production today, alongside the hypervisor profile in section 6.2. It is not
compliant with the target architecture of section 6.1, because live DSM root
can obtain both source-read capability and the Restic repository decryption
key.

The judgment has changed since this document was written. With an independent,
guest-independent image path now in place, replacing the Synology pull client
with a dedicated backup gateway buys a smaller improvement than closing the
gaps listed in section 1.1 — an off-site copy, protected local history, bounded
export, and external monitoring. The
[unified specification](unified-backup-safety-spec.md) therefore retains the
Synology client and hardens it, rather than building the gateway.

The controls in section 9 that do not require a separate host — bounded spool
size and runtime, a per-run manifest, fail-closed behavior, and separation of
backup from prune and check — remain applicable to the Synology client and
SHOULD be adopted there.

Repository credentials MUST NOT be moved onto Hermes as a fallback.

## 7. Network policy

Network policy MUST default to deny and implement at least this matrix:

| Source | Destination | Permitted service | Requirement |
| --- | --- | --- | --- |
| Backup gateway | Hermes | SSH on configured port | Required |
| Backup gateway | Synology repository endpoint | HTTPS | Required |
| Backup gateway | Off-site repository endpoint | Provider-specific TLS/SSH | Required |
| Backup gateway | Monitoring endpoint | HTTPS or approved protocol | Required |
| Backup gateway | DNS, NTP, update mirrors | Approved infrastructure only | Optional and allowlisted |
| Maintenance workstation | Administrative repository endpoint | TLS/SSH during maintenance window | Conditional |
| Hypervisor | Dedicated backup server | Product-specific backup network | Conditional |
| Hermes | Any backup or management component | None | Explicitly denied |
| NAS | Hermes | None | Explicitly denied in target design |
| Public Internet | Gateway, NAS, hypervisor, or backup server management | None | Explicitly denied |

Hermes MAY retain its required Internet access on a separate application
network. That network MUST have no route to the backup, storage, or management
networks. DNS names alone are not a security boundary.

The Hermes guest MUST NOT be able to reach the hypervisor management address,
backup server, NAS repository endpoint, gateway, monitoring control plane, or
off-site repository endpoint.

### 7.1 Deployed network policy

The deployment places every component on one flat LAN: the PVE management
interface, the Hermes guest, DSM, and the PBS VM share a single subnet with no
segmentation between them. Management ports are protected by restricting them
to the admin LAN or a VPN and by not forwarding them from the Internet, but
the guest sits inside that same LAN.

The matrix above is therefore not implemented. In the deployed topology the
hostile guest can reach the PVE management interface, the PBS interface, DSM,
and the NFS export at the network layer. It is stopped by authentication, not
by routing.

Closing this gap does not require the gateway architecture. The concrete
requirements for the deployment are:

| Source | Destination | Permitted service | Requirement |
| --- | --- | --- | --- |
| PVE host | PBS VM | HTTPS on the PBS API port | Required |
| PBS VM | NAS NFS export | NFSv4.1 from the single reserved PBS address | Required |
| Synology backup client | Hermes guest | SSH on the configured port | Required |
| Admin workstation or VPN | PVE and PBS management ports | HTTPS | Required |
| PVE, PBS, NAS | Monitoring endpoint | HTTPS or approved protocol | Required and currently absent |
| Hermes guest | Its own Internet services | Outbound only | Required |
| Hermes guest | PVE management, PBS, DSM, NFS export, or NAS repository | None | Explicitly denied and currently permitted |
| Public Internet | PVE, PBS, DSM, guest SSH, or the Hermes dashboard | None | Explicitly denied |

Hermes SHOULD be moved to its own VLAN or bridge with outbound Internet access
and no route to the management or storage subnets. Until then, the guest's
inability to reach backup infrastructure rests entirely on credentials it does
not hold, which is one control rather than two.

The NFS export MUST remain restricted to the single reserved PBS VM address.
Because `AUTH_SYS` trusts client-supplied numeric identities, that address
restriction and the network boundary are the export's only real authorization.
Widening the export to the LAN would give any host on it write and delete
access to the image datastore.

## 8. Hermes source contract

### 8.1 Dedicated identity and forced command

The source MUST normally expose exactly one SSH public key entry for the backup
gateway. A second entry MAY coexist temporarily during the tested rotation
procedure in section 17.1.
It MUST use an equivalent of:

```text
from="GATEWAY_IP",restrict,command="/usr/local/libexec/hermes-backup/export" ssh-ed25519 ...
```

The entry MUST:

- be unique to this source;
- restrict the accepted source address;
- use `restrict`;
- force one absolute command;
- ignore or reject `SSH_ORIGINAL_COMMAND`;
- prohibit shell, PTY, forwarding, agent forwarding, X11, and user RC files;
  and
- be independently revocable.

The account SHOULD have no password and no unrelated interactive or application
access.

### 8.2 Exporter integrity

The forced-command executable and every executable helper it invokes MUST:

- reside outside a normal user's writable home directories;
- be owned by root;
- not be writable by the Hermes runtime identity;
- use absolute paths;
- not load executable code or dependencies from user-writable locations;
- treat application-owned configuration and data as untrusted inputs;
  and
- be installed from an identified, reviewed release artifact.

The current paths under `/home/anton/.local/bin`, now inside the Hermes guest,
do not meet this requirement. Virtualizing Hermes did not change this: the
exporter still runs from a directory the Hermes runtime identity can rewrite,
so the guest controls the protocol implementation as well as the data.

The exporter MAY run with the Hermes application identity when required to read
data. Root ownership of the exporter protects the protocol implementation; it
does not make application-owned data trustworthy.

### 8.3 Stream protocol

On success, stdout MUST contain only one uncompressed TAR stream. Status and
diagnostic output MUST go to stderr.

The stream MUST contain:

```text
RESTORE.txt
hermes/hermes.zip
```

The exporter MUST:

1. use the supported Hermes backup interface;
2. omit transient lock, WAL, and shared-memory files as currently documented;
3. describe in `RESTORE.txt` only the archives actually present;
4. fail nonzero if required data is absent or inconsistent;
5. clean its temporary data on normal exit and signals; and
6. emit no secret value.

Additional tool state MAY later be added under its own top-level directory in
the stream. Any such addition MUST document how it is captured consistently and
MUST NOT weaken the requirements above.

No source-provided checksum, signature, status string, or manifest may be
treated as proof that hostile source data is truthful.

## 9. Backup gateway

### 9.1 Isolation

The gateway MUST be a separate operating-system instance from Hermes and the
NAS. It MUST NOT be administered by Hermes or run inside a runtime Hermes can
control.

A dedicated physical device is preferred. A VM is acceptable only when its
hypervisor is outside Hermes's trust boundary and Hermes cannot access the
management plane.

The gateway MUST NOT run general user workloads, AI agents, web browsing, email,
or unrelated Internet-facing services.

### 9.2 Deployment integrity

Gateway executables, service units, configuration, and parent directories MUST
be root-owned and not writable by the runtime identity.

Deployable binaries or images MUST be selected by immutable version and digest.
The implementation SHOULD publish an SBOM and provenance. Automatic updates
MAY download candidates, but activation MUST follow a reviewed rollout and
rollback procedure.

### 9.3 Runtime identity and sandbox

The scheduled backup MUST run under a dedicated, non-interactive identity. It
MUST receive only the filesystem and network access required for the backup.

A systemd implementation SHOULD use applicable controls such as:

- `NoNewPrivileges=yes`;
- `PrivateTmp=yes`;
- a restrictive `ProtectSystem` setting;
- explicit `ReadWritePaths` for spool and state;
- `ProtectHome=read-only` or stricter;
- a closed capability bounding set;
- memory, process, and runtime limits; and
- systemd credentials for secret delivery.

The final sandbox MUST be tested against required SSH, spool, Restic, DNS, and
certificate operations rather than copied blindly from this list.

### 9.4 Secrets

The gateway holds:

- one source-specific SSH private key;
- one local repository password;
- one off-site repository password;
- one local repository authentication credential;
- one off-site repository authentication credential; and
- integrity-sensitive TLS CA or SSH host-key material.

The two repositories MUST use independent encryption passwords and independent
transport credentials.

Secret values MUST NOT appear in environment variables, command-line
arguments, logs, images, Git, monitoring payloads, or crash reports. Programs
SHOULD consume credentials from file descriptors, systemd credentials, or
root-only files.

At-rest gateway credentials SHOULD be protected by full-disk encryption and,
where operationally acceptable, TPM-bound systemd credentials. These controls
do not protect against live gateway root.

At least two independent offline recovery copies of each repository recovery
secret MUST exist outside Hermes, the gateway, and the NAS.

### 9.5 Bounded spool

The gateway MUST receive one source export into a private spool before writing
either repository.

The spool MUST:

- reside on an encrypted filesystem or size-appropriate tmpfs;
- be accessible only to the backup runtime identity and root;
- have a configured maximum byte size;
- have a configured maximum runtime;
- fail the backup if the source exceeds either limit;
- never parse, list, decompress, or extract the source TAR;
- survive long enough to attempt both independent repository writes; and
- be unlinked after both attempts or after a terminal failure; and
- rely on the encrypted spool filesystem, rather than overwrite-based deletion,
  to protect residual storage blocks.

The receiver MUST distinguish a complete source exit from truncation at the
size limit. Silently accepting the first `MAX_BYTES` of an oversized source is
not compliant.

After a successful pull, the gateway MUST generate its own manifest containing
at least:

- schema version;
- UTC pull start and finish times;
- source identifier;
- byte length;
- SHA-256 digest of the exact TAR bytes;
- exporter exit status;
- verified SSH host-key fingerprint; and
- gateway release identifier.

The TAR and gateway manifest MUST be backed up together to both repositories.
The two repository snapshots MUST refer to the same TAR digest.

### 9.6 Backup behavior

The scheduled gateway identity MUST be able to create snapshots but MUST NOT be
able to forget, prune, overwrite, or delete existing local repository objects.

For each scheduled run, the gateway MUST:

1. acquire a non-blocking run lock;
2. verify required credentials and trusted host material are readable;
3. pull one bounded export from Hermes;
4. require a zero SSH/exporter exit status;
5. require a nonempty export within the configured size range;
6. generate the gateway manifest;
7. back up the TAR and manifest to the local append-only repository;
8. independently back up the same TAR and manifest to the off-site target;
9. record the snapshot identifier and result of each target;
10. report partial success as an alert and overall nonzero result;
11. remove the spool; and
12. release the run lock.

Failure of one repository MUST NOT prevent an attempt to write the other after
a valid source export has been captured.

The backup MUST NOT run retention or pruning after snapshot creation.

## 10. Synology storage service

### 10.1 Role

The NAS MUST act only as an encrypted-object storage service for the target
architecture. It MUST NOT receive:

- the Hermes SSH private key;
- either Restic repository password;
- the off-site credential;
- a source export in plaintext outside the encrypted Restic protocol; or
- routine backup deletion authority.

### 10.2 Rest server

The local repository SHOULD use the official Restic REST server with equivalent
settings to:

```text
--append-only
--private-repos
--tls
--tls-min-ver 1.3
--max-size DEPLOYMENT_QUOTA
```

Authentication MUST use a high-entropy source-specific credential stored as a
supported password verifier. Authentication MUST NOT be disabled.

The service MUST:

- be reachable only from the backup gateway and approved maintenance network;
- use a certificate validated by the gateway;
- run as a dedicated non-root identity;
- use a read-only container root filesystem when containerized;
- drop all capabilities not proven necessary;
- enable `no-new-privileges`;
- pin its image by digest;
- mount only its repository, authentication, TLS, and bounded runtime paths;
  and
- write access logs that contain no credential or repository password.

The repository share MUST NOT be exposed through SMB, NFS, FTP, WebDAV,
Synology Drive, media indexing, or normal user shares.

### 10.3 Local immutability

Where supported, the repository MUST reside on Btrfs and use immutable
Synology snapshots. The initial minimum protection window is 14 days. A longer
window SHOULD be selected from measured capacity and incident-detection time.

This requirement applies to the PBS datastore shared folder as well as to the
Restic repository. Both now live on Btrfs shared folders on the same NAS and
neither has protected snapshots. NAS root or a DSM administrator can therefore
delete either repository's history outright.

Snapshots of the PBS datastore MUST be scheduled outside the PBS verification
and garbage-collection windows, and the shared folder MUST keep the recycle bin
disabled so that PBS garbage collection actually releases space. Snapshot
retention consumes capacity that garbage collection would otherwise return, so
the quota MUST be sized for both.

Local immutable snapshots are defense in depth. The system is not compliant
until a protected off-site copy exists.

### 10.4 Maintenance access

The routine REST endpoint MUST remain append-only.

Any delete-capable maintenance endpoint MUST:

- use a different authentication credential;
- be disabled by default;
- be unreachable from Hermes and the gateway routine identity;
- be enabled only during an approved maintenance window;
- be restricted to the maintenance workstation network address;
- not run concurrently with backup ingestion; and
- be disabled and verified closed when maintenance finishes.

The repository password remains on the maintenance workstation; it MUST NOT be
persisted on the NAS for maintenance.

## 11. Off-site repository

The off-site target MUST:

- be in a different physical and administrative failure domain from Hermes,
  the gateway, and the NAS;
- receive client-side encrypted backup content;
- have no repository decryption password;
- enforce append-only writes or provider-controlled immutable snapshots;
- retain protected recovery points longer than the expected compromise
  detection interval;
- support export or recovery without proprietary decryption software; and
- provide monitoring or an API sufficient to verify recent writes and
  protection status.

The initial minimum immutable window is 30 days. The target SHOULD retain daily
recovery points for 90 days and monthly recovery points for at least one year,
subject to confirmed recovery requirements and capacity.

Acceptable implementation classes include:

- a managed Restic or Borg service with append-only access and client-held
  encryption keys;
- a snapshot-enabled rsync.net account whose ZFS snapshots are immutable to
  the client credential;
- a second independently administered Restic REST server with immutable
  underlying storage; or
- a second Proxmox Backup Server for the hypervisor profile.

A plain writable SFTP, SMB, NFS, or object-store repository without independent
immutability does not satisfy this requirement.

The deployment currently has no off-site copy on either path. Both the PBS
datastore and the Restic repository reside on the same Synology NAS, which also
hosts the PBS VM. Loss, theft, ransomware, administrative error, or DSM
compromise at that one site removes every recovery point the system has. This
is the largest structural gap in the deployment and it is not mitigated by
client-side encryption, which provides confidentiality rather than availability.

For the image path, PBS supports native sync or backup-copy jobs to a remote
datastore, which is the least disruptive way to satisfy this section without
introducing a new backup product.

## 12. Hypervisor-level backup

This section is implemented. Each requirement below is stated as a requirement
first and annotated with its state in the deployment of record.

1. Hermes MUST run as a guest without access to the management or backup
   networks. **Not satisfied.** The guest shares one flat LAN with PVE, PBS,
   and DSM; see section 7.1.
2. Backup scheduling MUST occur outside the guest. **Satisfied.** PVE runs the
   backup job daily at 04:00 and PBS runs prune, verification, and garbage
   collection on its own schedules.
3. Backup credentials and client-side encryption keys MUST remain outside the
   guest. **Satisfied.** The PBS API token and the AES key exist only on PVE
   and in operator recovery copies.
4. The normal hypervisor backup token MUST be able to create backups but MUST
   NOT have prune or datastore administration permission. **Satisfied.** The
   token is privilege-separated with `DatastoreBackup` on one datastore path,
   and retention runs on PBS.
5. The backup server MUST perform scheduled verification. **Satisfied.** A
   weekly verification job re-verifies snapshots on a 30-day interval, which
   matters because a snapshot healthy last month can develop storage corruption
   later.
6. An off-site synchronization or backup-copy job MUST protect the image
   backups in another failure domain. **Not satisfied.** See section 11.
7. Backup, prune, verification, and garbage collection MUST NOT overlap.
   **Satisfied.** The four windows are separated, which matters on
   HDD-backed NFS where verification and garbage collection can run for hours.
8. The datastore MUST have an enforced capacity ceiling and be monitored
   against it. **Partially satisfied.** A DSM shared-folder quota is set, but
   Synology NFS reports whole-volume statistics, so PBS displays free space
   that does not exist. DSM shared-folder usage is the quota authority and MUST
   be the value that is alerted on.
9. The datastore MUST have protected snapshots or equivalent protection against
   deletion by a compromised administrator. **Not satisfied.** See section 10.3.
10. A restored VM MUST first boot in an isolated network with no Internet,
    production, backup, or management access. **Satisfied in procedure.** The
    restore drill requires a new VM ID, no start-after-restore, no live
    restore, a unique MAC, disabled autostart, and a disconnected NIC before
    first boot.
11. Recovery operators MUST inspect the restored system before authorizing
    production connectivity. **Satisfied in procedure.**
12. Production and a credential-identical restored guest MUST NOT run online
    concurrently. **Satisfied in procedure.** The restored clone carries
    duplicate messaging-gateway tokens, OAuth credentials, host keys, and
    service identity; a changed MAC address is not a substitute for keeping one
    of them offline.

Guest-agent quiescing MAY improve consistency but MUST NOT be treated as a
security boundary because a hostile guest can interfere with it. The deployment
uses Stop mode rather than guest-agent snapshot mode, which avoids depending on
guest cooperation for consistency at the cost of a short outage and a
full-disk reread each run. If the deployment later moves to Snapshot mode for
speed, that trade MUST be recorded as accepting guest-influenced consistency.

The image encryption key MUST have at least two recovery copies outside PVE and
outside the NAS, because generating a new key does not decrypt existing
snapshots. The deployment keeps the key JSON in a password manager and on
encrypted offline media, which satisfies section 9.4's requirement for two
independent offline recovery copies for this secret.

## 13. Retention and pruning

Routine backup identities MUST NOT possess deletion authority.

The image path satisfies this differently from the way this document
originally proposed, and acceptably so. Rather than deferring deletion to an
isolated maintenance workstation, PBS performs its own prune and garbage
collection under its own identity, and the PVE backup token holds no deletion
right at all. The separation that matters — the identity that writes backups
cannot destroy them — is preserved.

The deployed image retention keeps the last 3 snapshots, 7 daily, 4 weekly, and
6 monthly, pruned daily at 00:00, with garbage collection weekly. Prune removes
snapshot references; garbage collection later removes unreferenced chunks;
verification reads chunks to detect corruption. All three are distinct
operations and MUST be scheduled apart from each other and from the backup
window.

That retention keeps roughly six months of image recovery points, which exceeds
the 14-day minimum below but MUST be checked against measured NAS capacity
rather than assumed, since Synology NFS misreports free space to PBS.

For the Restic path, retention and pruning MUST run only from the isolated
maintenance workstation through a temporary full-access path. The operator MUST
inspect repository activity and backup anomalies before enabling deletion.

Restic repositories receiving append-only writes MUST use time-window retention
options rather than only count-based options. An initial policy is:

```text
keep all snapshots within 14 days
keep daily snapshots within 90 days
keep weekly snapshots within 1 year
keep monthly snapshots within 5 years
```

The exact policy is a deployment decision, but it MUST use applicable
`--keep-within*` options and MUST preserve at least one known-good recovery point
older than the maximum credible detection delay.

Every destructive maintenance run MUST:

1. confirm the normal ingest job is stopped;
2. confirm current immutable snapshots or off-site recovery points exist;
3. run `forget --dry-run` first;
4. review unexpected source times, sizes, tags, and snapshot density;
5. run a repository check appropriate to repository size and risk;
6. record the approved deletion plan;
7. run retention and pruning;
8. verify repository health afterward; and
9. close the maintenance endpoint and confirm routine append-only behavior.

## 14. Monitoring

Monitoring MUST be controlled outside Hermes. Hermes MUST NOT be able to submit
or overwrite the authoritative backup success status.

Nothing outside the two backup systems currently answers the question "when did
each path last succeed". Both paths are designed to fail closed, which is
correct, and both fail quietly, which is not survivable. A failed exporter
produces no Restic snapshot; a failed PVE or PBS task reports only where
someone looks. Two paths that both fail silently are worse than one path that
is watched, and this is the cheapest unmet requirement in the system.

The image path adds these alert conditions:

- PVE backup job failure or a run that did not start;
- PBS verification failure or a snapshot that failed re-verification;
- PBS garbage collection or prune failure;
- an unexpected change in the PBS backup group owner;
- loss of the NFS mount inside the PBS VM, or a datastore path that is a local
  directory rather than the mounted export;
- DSM shared-folder usage crossing its alert threshold, which is the real quota
  authority; and
- a PBS TLS fingerprint that no longer matches the value pinned in PVE.

Operators MUST be alerted for:

- source connection or exporter failure;
- export timeout or size-limit violation;
- export size outside the configured baseline;
- a changed SSH host key;
- local repository failure;
- off-site repository failure;
- different TAR digests between repository snapshots;
- failure to clean the spool;
- overlapping backup attempts;
- repository quota pressure;
- missing or expired immutable protection;
- repository check failure;
- missed schedule; and
- restore-test failure.

Operators MUST be able to determine:

- last attempted and last successful pull time;
- gateway version and configuration version;
- source exporter exit status;
- source byte count and gateway-computed SHA-256;
- snapshot identifier for each repository;
- last successful write observed at each storage target;
- immutable protection horizon;
- last check and prune result; and
- last successful application and image restore test.

Logs MUST use UTC timestamps and MUST NOT contain secret values, complete
process environments, private keys, or authorization headers.

## 15. Restore security

Every restore MUST treat repository contents as hostile because Hermes supplied
the original bytes.

Application restores MUST:

1. restore into a disposable, isolated environment;
2. use an unprivileged identity;
3. have no network access during extraction and initial inspection;
4. prohibit device creation and privilege restoration;
5. reject absolute paths and path traversal;
6. handle symlinks without allowing writes outside the restore root;
7. avoid preserving source ownership, setuid, setgid, capabilities, or unsafe
   extended attributes;
8. inspect the TAR and embedded Hermes ZIP before invoking application import;
9. run malware and policy scans appropriate to the environment;
10. validate representative Hermes behavior; and
11. require explicit operator approval before any recovered data reaches
    production.

Application restores MUST import a Restic-recovered Hermes export into a
disposable isolated VM, never into the running production guest.

VM image restores MUST initially boot on an isolated recovery network. A
restored Hermes instance MUST NOT receive production credentials or Internet
access until inspected and approved.

The deployed restore drill satisfies this: it restores to a new VM ID with
start-after-restore and live restore disabled, assigns a unique MAC, disables
autostart, and disconnects the virtual NIC before first boot. Offline
inspection covers firmware and bootloader, root filesystem mount, expected
Hermes configuration, failed system units, and the presence of the Hermes
binaries and gateway service.

Where the drill later connects the restored clone to test real messaging and
integrations, it does so by stopping production first, testing the clone, then
disconnecting the clone and returning production to service. That sequence is
acceptable because the restored guest carries duplicate messaging-gateway
tokens, OAuth credentials, host keys, and sessions, and running both online
would let two identities act as Hermes at once. The rule is absolute: two
credential-identical guests MUST NOT be online together, and the test VM MUST
NOT be deleted until production is confirmed running again.

Restoring a VM image of a possibly compromised guest restores the compromise
along with the data. Offline inspection before reconnection is what separates
recovery from reinfection, and it is the operator's judgment that does the
work, not the restore procedure.

At least monthly, automation MUST restore and validate a non-sensitive canary.
At least quarterly, operators MUST perform a representative isolated
application restore from each repository. At least annually, operators MUST
exercise the complete disaster-recovery procedure, including offline recovery
secrets and off-site-only recovery.

The annual exercise MUST include restoring an image snapshot onto a rebuilt PVE
host using the saved AES key JSON, because that is the path a real PVE hardware
loss takes and it is the path where an unreadable or missing key copy is
discovered too late. Auto-generating a new key during recovery MUST NOT be
offered as a workaround; a new key cannot decrypt existing snapshots.

## 16. Availability objectives

Unless superseded by an approved deployment-specific recovery plan, the initial
objectives are:

| Objective | Initial target | Deployed state |
| --- | --- | --- |
| Application backup RPO | 24 hours | Met by the daily Restic pull |
| VM image backup RPO | 24 hours | Met by the daily 04:00 Stop-mode backup |
| Detection of a missed backup | 36 hours | Not met; no external monitoring exists |
| Local application restore initiation | 4 hours after declared incident | Untested |
| Local image restore initiation | 4 hours after declared incident | Demonstrated by the restore drill |
| Off-site-only restore initiation | 8 hours after declared incident | Not achievable; no off-site copy exists |
| Local immutable window | At least 14 days | Not met on either repository |
| Off-site immutable window | At least 30 days | Not met |

Both RPOs are met and neither detection objective is, which is the
characteristic failure mode of an unmonitored backup system: it meets its
targets until it silently stops, and then reports nothing.

The image path also carries an availability cost the application path does not.
Stop mode shuts the guest down for each backup, so Hermes is briefly offline
daily and its messaging gateway reconnects afterward. That outage is the price
of consistency that does not depend on the guest, and reboot persistence MUST
be verified so the guest reliably returns.

These are service objectives, not guarantees. The deployment owner MUST confirm
that they are adequate for Hermes.

## 17. Credential rotation and incidents

### 17.1 Hermes pull key

Rotation MUST:

1. create a new gateway key;
2. install a second restricted public-key entry on Hermes;
3. verify the source host key independently;
4. complete a bounded test export and backup;
5. remove the old public-key entry; and
6. securely retire the old private key.

### 17.2 Repository encryption key

Normal password rotation MUST add and test a new repository key before removing
the old key. If a decrypted repository master key may have been exposed, the
repository MUST be replaced or copied into a new repository with new master key
material. Changing only the password does not revoke a leaked master key.

### 17.3 Suspected Hermes compromise

Operators MUST:

- preserve pre-compromise immutable recovery points;
- suspend destructive maintenance;
- distrust all backups created after the earliest credible compromise time;
- isolate Hermes from backup and management networks;
- rotate the Hermes pull key after rebuilding or remediating the source;
- restore first into isolation; and
- compare application-level and image-level recovery points where available.

Hermes compromise alone does not require rotating repository credentials because
Hermes never receives them. Rotate them if evidence indicates the gateway or
maintenance workstation was also exposed.

In the deployed system, guest compromise does not require rotating the PBS API
token or the PVE image encryption key either, because the guest never holds
them. It does require treating every image snapshot taken after the earliest
credible compromise time as containing the compromise, and identifying the
newest snapshot that predates it. That is the recovery point, and it is why
retention must exceed the credible detection delay. Credentials that lived
inside the guest — messaging-gateway tokens, OAuth grants, model-provider API
keys, and the guest's own SSH host keys — MUST be rotated regardless, and they
will also be present in any restored image.

### 17.4 Suspected gateway compromise

Operators MUST assume source confidentiality and all repository encryption keys
are compromised. They MUST revoke transport credentials, disable ingest, rotate
the Hermes pull key, preserve immutable storage, create new repositories with
new master keys, rebuild the gateway from trusted media, and resume only after
an isolated restore and integrity review.

### 17.5 Suspected PVE compromise

PVE occupies the gateway's position in the deployed system. Operators MUST
assume that current guest plaintext, the PBS API token, and the image
encryption key are all compromised, because PVE holds all three.

Operators MUST revoke the PBS token, preserve existing snapshots, rebuild PVE
from verified installation media, and treat every image snapshot created after
the earliest credible compromise time as suspect. Rotating the image encryption
key protects future snapshots only; it does not withdraw an exposed key from
snapshots already written, which is why an attacker who copied the key retains
the ability to decrypt any datastore copy they also obtained.

### 17.6 Suspected PBS compromise

Operators MUST assume the datastore can be read as ciphertext, destroyed, or
corrupted, but not decrypted, because PBS holds no image encryption key.

Operators MUST preserve the underlying shared folder and any snapshots of it
before rebuilding, revoke the PBS token, rebuild the PBS VM, reattach the
existing datastore path without initializing or erasing it, recreate the user,
token, and datastore ACLs, and verify snapshots before trusting them. Verify
rather than assume: PBS can validate chunk and manifest integrity without the
decryption key, so integrity checking survives the rebuild.

### 17.7 NAS compromise

Operators MUST assume the local repository can be destroyed or corrupted but
not decrypted unless the gateway or maintenance workstation was also
compromised. They MUST preserve off-site recovery, revoke NAS transport
credentials, rebuild the storage service, and repopulate it from a verified
source or off-site repository.

NAS compromise is now the most damaging non-PVE compromise in the deployment.
DSM administrator authority reaches the PBS VM, the NFS export beneath its
datastore, and the Restic repository. Both local recovery paths can be
destroyed by one attacker with one set of credentials, and today there is no
off-site copy to repopulate from. Until section 11 is satisfied, this incident
has no recovery procedure — only an inventory of what was lost.

## 18. Migration from the current implementation

Superseded. The gateway migration below was never executed and is retained as a
record. Step 10 — virtualize Hermes and add the hypervisor-level backup path —
was executed on its own and turned out to deliver most of the value of the
whole sequence. The migration that now applies is in the
[unified specification](unified-backup-safety-spec.md).

The ordering principle still holds and applies to that migration: migration
MUST avoid a gap in verified protection. The deployed PVE/PBS and Synology
paths MUST both remain operational until their replacements have passed backup
and restore acceptance tests.

Original recommended sequence:

1. Keep the current NAS-pull backup operational.
2. Provision and harden the independent gateway.
3. Provision a new append-only local repository service with no decryption key.
4. Provision the immutable off-site repository.
5. Install the root-owned source exporter and helper outside the user home.
6. Generate new gateway-to-Hermes and repository transport credentials.
7. Initialize independent local and off-site repositories with independent
   encryption keys.
8. Run a bounded pull and write the same spool to both targets.
9. Perform isolated restores from both new repositories.
10. If possible, virtualize Hermes and add the hypervisor-level backup path.
11. Observe at least two successful scheduled cycles and one missed-backup
    alert test.
12. Disable the old DSM backup, check, and prune tasks.
13. Revoke the old NAS-to-Hermes SSH key.
14. Remove the Restic password and Hermes private key from the NAS after the
    rollback window.
15. Make the old repository read-only and retain it according to the approved
    migration policy.
16. Remove the NAS Git checkout, Compose deployment, obsolete containers, and
    obsolete secret files only after recovery requirements are satisfied.

The old repository MUST NOT be reinitialized or destructively migrated. New
repositories are preferred because the old master key and historical exposure
cannot be withdrawn by a password change.

## 19. Verification and acceptance criteria

The target architecture is accepted only when all applicable checks pass.

### 19.1 Trust boundaries

- [ ] Hermes contains no repository password or transport credential.
- [ ] Hermes cannot route to the gateway, NAS repository endpoint, off-site
      endpoint, hypervisor management, or backup server.
- [ ] The NAS contains no repository decryption password or Hermes private key.
- [ ] The gateway is a separate operating-system trust domain from Hermes and
      the NAS.
- [ ] The normal gateway identity cannot delete or prune local snapshots.
- [ ] Maintenance credentials are absent from the routine gateway service.

### 19.2 Source behavior

- [ ] SSH accepts only the gateway key from the gateway address.
- [ ] Shell, PTY, forwarding, user RC, and arbitrary commands are rejected.
- [ ] Exporter and helpers are root-owned and outside user-writable paths.
- [ ] Exporter diagnostics never contaminate stdout.
- [ ] Exporter or SSH failure creates no snapshot.
- [ ] The exported Hermes archive is nonempty and validates after restore.

### 19.3 Hostile-output handling

- [ ] The gateway never extracts or parses source TAR content during backup.
- [ ] Empty source output fails.
- [ ] Nonzero exporter exit fails.
- [ ] Timeout fails without creating a successful snapshot.
- [ ] Oversized output fails rather than being silently truncated.
- [ ] Spool permissions and encryption meet this specification.
- [ ] Both repositories receive the same gateway-computed TAR digest.
- [ ] A deliberately malformed TAR is stored safely and rejected or contained
      by the isolated restore procedure.

### 19.4 Storage

- [ ] Local repository authentication is enabled and TLS is verified.
- [ ] The normal local endpoint rejects overwrite and delete operations.
- [ ] The repository is not exposed by general NAS file services.
- [ ] Local immutable snapshots meet the approved protection window.
- [ ] The off-site copy is in an independent failure domain.
- [ ] Off-site immutability survives compromise of the routine client
      credential, according to a tested or contractually verified procedure.
- [ ] Repository quota exhaustion generates an alert before backup failure.

### 19.5 Operations and recovery

- [ ] A failed local write still attempts the off-site write.
- [ ] A failed off-site write reports partial success as failure requiring
      action.
- [ ] Missing-backup monitoring does not depend on a Hermes signal.
- [ ] Retention uses approved `--keep-within*` rules from the maintenance
      workstation.
- [ ] Destructive maintenance requires a reviewed dry run.
- [ ] Application restore succeeds from each repository in isolation.
- [ ] A restored VM boots in isolation when the hypervisor profile is present.
- [ ] Offline recovery copies of both repository secrets have been tested.
- [ ] Incident runbooks cover Hermes, gateway, NAS, and maintenance-workstation
      compromise separately.

### 19.6 Hypervisor profile, as deployed

Checked items were verified against the deployment of record. Unchecked items
are open findings.

- [x] The guest holds neither the PBS API token nor the image encryption key.
- [x] The PVE backup identity is a privilege-separated token limited to
      `DatastoreBackup` on one datastore path.
- [x] The PBS TLS fingerprint is pinned in PVE and the storage shows active.
- [x] The image encryption key fingerprint is visible in PVE and the key file
      is root-only.
- [x] Two recovery copies of the image encryption key exist outside PVE and the
      NAS, one of them offline.
- [x] A backup task log confirms encryption was enabled for the stored
      snapshot.
- [x] PBS shows the snapshot as encrypted and owned by the intended token
      identity.
- [x] Manual verification of a snapshot completes with zero errors and without
      the decryption key.
- [x] Retention runs on PBS, not under the PVE backup token.
- [x] Backup, prune, verification, and garbage collection windows do not
      overlap.
- [x] PBS runs on a host other than the PVE host it protects.
- [x] The PBS datastore is a mounted NFS export, not an empty local
      mount-point directory.
- [x] The NFS mount is `hard` with synchronous writes and correct numeric
      ownership.
- [x] The NFS export is restricted to the single reserved PBS VM address.
- [x] A restore to a new VM ID completed and booted with its NIC disconnected.
- [x] The guest returns and its messaging gateway reconnects after a Stop-mode
      backup.
- [ ] The guest cannot reach PVE management, PBS, DSM, or the NFS export at the
      network layer.
- [ ] The PBS datastore shared folder has protected snapshots covering the
      approved window.
- [ ] An off-site sync or backup-copy job protects image snapshots in an
      independent failure domain.
- [ ] Backup, verification, and garbage-collection outcomes are reported to a
      monitor outside PVE, PBS, and the guest.
- [ ] DSM shared-folder usage, not PBS free-space reporting, drives the
      capacity alert.
- [ ] A rebuild-PVE-and-restore drill using the saved key JSON has been
      completed.

## 20. Required implementation deliverables

Superseded, in the same way as section 18: these deliverables describe the
gateway architecture that will not be built. Items 1, 2, 4, 5, 9, 10, 11, 12,
and 14 remain applicable to the Synology client and are carried into the
[unified specification](unified-backup-safety-spec.md). Items 3, 6, 7, 8, and
13 are gateway-specific and are retained only as a record.

Original list. Implementation would have been incomplete until the repository
contained:

1. A root-owned Hermes exporter installation package or script.
2. A versioned exporter stream contract.
3. A gateway service and timer definition.
4. A bounded spool receiver with tests for timeout, truncation, and oversize.
5. Gateway manifest schema and validation tooling.
6. Local and off-site Restic backup configuration with independent secrets.
7. A pinned and hardened Synology rest-server deployment.
8. Network firewall and routing instructions for every component.
9. External monitoring configuration and alert tests.
10. Maintenance, retention, pruning, and endpoint-opening procedures.
11. Safe application and VM restore procedures.
12. Credential rotation and incident-response runbooks.
13. Migration and rollback instructions from the current Compose deployment.
14. Automated tests covering the acceptance criteria that can be tested in CI.

## 21. Open deployment decisions

### 21.1 Decided

- Hermes is virtualized as an Ubuntu LTS guest on a dedicated Proxmox VE host.
- Image backup uses Proxmox Backup Server, not a commercial product.
- PBS runs as a VM under Synology Virtual Machine Manager, deliberately off the
  PVE host it protects.
- The PBS datastore is a Btrfs shared folder exported over NFSv4.1 to the PBS
  VM's reserved address, rather than a large virtual disk, so it can be
  reattached to a replacement PBS VM.
- Image backups use daily Stop mode, accepting a short daily guest outage and a
  full-disk reread in exchange for consistency that does not depend on the
  guest.
- Image content is encrypted client-side on PVE with an unattended
  auto-generated AES-256-GCM key, with recovery copies in a password manager
  and on encrypted offline media.
- Image retention, verification, and garbage collection run on PBS on separate
  schedules.
- The dedicated backup gateway of sections 6.1 and 9 will not be built.

### 21.2 Still open

- whether Hermes moves to its own VLAN or bridge, and when;
- off-site provider and mechanism for the image path, most likely a PBS remote
  sync or backup-copy target;
- off-site provider for the application path, and whether both paths share one;
- protected-snapshot schedule and retention for both shared folders, and the
  quota headroom that snapshot retention requires;
- local repository and datastore quotas, and the alert thresholds against DSM
  shared-folder usage;
- monitoring destination and on-call owner for both paths;
- expected and maximum application export size, runtime, and bandwidth;
- source backup account and exact data-read permissions;
- confirmation that the deployed image retention meets the required detection
  window; and
- acceptable maintenance, migration, and rollback windows.

These choices may strengthen the design but MUST NOT change the prohibition on
placing repository credentials, deletion authority, or backup control on
Hermes.

## 22. Primary references

- [Securely Deploying Hermes Agent in a Proxmox Homelab](https://github.com/chocobot-farm/plume-pilot/blob/main/docs/deployment/0001-hermes-homelab-pve-pbs-synology.md),
  the deployment of record
- [Restic append-only security and retention](https://restic.readthedocs.io/en/stable/060_forget.html#security-considerations-in-append-only-mode)
- [Restic REST server](https://github.com/restic/rest-server)
- [Restic repository and password handling](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html)
- [OpenSSH authorized-key restrictions](https://man.openbsd.org/sshd.8)
- [Proxmox VE backup and restore](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)
- [Proxmox Backup Server features](https://www.proxmox.com/en/products/proxmox-backup-server/features)
- [Proxmox Backup Server permissions](https://pbs.proxmox.com/docs/user-management.html)
- [Proxmox Backup Server client-side encryption](https://pbs.proxmox.com/docs/backup-client.html#encryption)
- [Proxmox Backup Server storage and maintenance](https://pbs.proxmox.com/docs/storage.html)
- [Synology immutable snapshots](https://kb.synology.com/en-us/DSM/help/SnapshotReplication/snapshots?version=7)
- [Synology DSM NFS permissions](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/file_share_privilege_nfs?version=7)
- [Borg append-only mode](https://borgbackup.readthedocs.io/en/stable/usage/notes.html#append-only-mode-forbid-compaction)
- [BorgBase documentation](https://docs.borgbase.com/)
- [rsync.net immutable snapshots](https://www.rsync.net/products/ransomware.html)
