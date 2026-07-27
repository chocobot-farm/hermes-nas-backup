# Security findings

Date: 2026-07-16

Revised: 2026-07-27 for the current Proxmox VE / Proxmox Backup Server /
Synology deployment.

Status: Revised. The original findings remain valid and, with one exception,
remain unremediated in production. The system-level target is now the
[unified backup safety specification](unified-backup-safety-spec.md), which
covers both the cold PVE/PBS image path and the live Synology/Restic
application path. The [native Synology container deployment
specification](native-container-spec.md) remains accepted for the Synology
client itself and its requirements are subsumed by the unified specification.
The [hostile-source backup architecture](hostile-source-backup-spec.md) remains
a rejected alternative whose low-complexity controls may still be incorporated.

Deployed state: the Synology backup client in service is the `origin/main`
Compose deployment. `make-it-safe` adds only this documentation set and a
pointer to it in the README, so the running code and the branch under review
are the same. None of the Priority 1 through Priority 4 recommendations below
have been applied. The one material improvement since the original review came
from outside this repository: Hermes now runs as a virtual machine whose
complete image is backed up independently of the guest.

## Scope

This document assesses how the Hermes backup system protects its credentials
and its recovery points. The original review covered only the Synology Restic
pull client. This revision extends it to the surrounding deployment, because
that deployment now determines most of the residual risk.

The review covered:

- `setup-nas.sh`, `docker-compose.yaml`, `Dockerfile`, `backup.sh`,
  `server/hermes-backup-stream`, and `README.md` at `origin/main`, which is the
  deployed revision;
- [Securely Deploying Hermes Agent in a Proxmox
  Homelab](https://github.com/chocobot-farm/plume-pilot/blob/main/docs/deployment/0001-hermes-homelab-pve-pbs-synology.md),
  the authoritative description of the current deployment;
- Proxmox VE client-side backup encryption, PBS API tokens and ACLs, PBS
  verification, pruning, and garbage collection;
- the Synology NFS export and the PBS datastore that consumes it;
- Synology DSM encrypted shared folders, encrypted volumes, Key Manager, KMIP,
  permissions, immutable snapshots, and account hardening;
- Restic password handling and append-only repository guidance;
- Docker Compose and Docker Swarm secret behavior; and
- OpenSSH `authorized_keys` restrictions.

No secret values were read or recorded during this review. Statements about the
live hosts are inferences from the deployment guide and repository contents;
items marked as verification steps must be confirmed against the running
systems.

## What changed since the original review

The original review assessed a single backup path: a Synology container pulling
an application export from a physical Hermes server over restricted SSH into a
local Restic repository. The current deployment adds a second, independent
path and moves the source onto virtualized infrastructure.

- Hermes runs as an Ubuntu VM on a dedicated Proxmox VE host.
- PVE performs a daily Stop-mode whole-VM backup at 04:00 and encrypts the
  content with AES-256-GCM before it leaves the PVE host.
- Proxmox Backup Server runs as a VM under Synology Virtual Machine Manager.
- The PBS datastore is an ordinary Synology Btrfs shared folder exported to the
  PBS VM over NFSv4.1.
- PVE authenticates to PBS with a privilege-separated API token restricted to
  `DatastoreBackup` on one datastore, and pins the PBS TLS fingerprint.
- PBS runs its own prune, verification, and garbage-collection schedules.
- The Synology Restic client continues to run unchanged from `main`.

Two consequences dominate this revision.

The first is favorable. The original review had no good answer to a compromised
Hermes source: everything the backup system knew about Hermes came from an
exporter running inside Hermes. A cold, guest-independent image backup now
exists, so there is a recovery path that does not depend on the guest telling
the truth. This is the single largest security improvement in the system and it
did not come from hardening the Restic client.

The second is unfavorable. Every recovery point now lives on one Synology NAS.
The PBS datastore and the Restic repository sit on the same device, almost
certainly the same volume, under the same administrators. The system has two
backup paths and one failure domain. Adding the second path improved recovery
from guest and PVE failures and did nothing for NAS loss, theft, ransomware, or
administrator compromise.

## Executive summary

The implementation retains the defense-in-depth controls identified in the
original review. Credentials are excluded from Git and the image, mounted
read-only, consumed as files rather than environment-variable values, and
copied to tmpfs only when the SSH client needs a key with suitable permissions.
The container runs as a non-root user with a read-only root filesystem, all
capabilities dropped, and `no-new-privileges`. SSH host-key verification is
strict, and the client key is restricted on the server by source address, a
forced command, and OpenSSH's `restrict` option. The generated Restic password
has ample entropy.

The new PVE/PBS layer is well constructed on its own terms. Backup content is
encrypted before it reaches PBS or the NAS, so the storage platform holds
ciphertext it cannot read. The PVE identity is a privilege-separated token
scoped to one datastore, which cannot administer or arbitrarily delete it. The
PBS TLS fingerprint is pinned. PBS verifies chunk and manifest integrity
without possessing the decryption key, which is a genuine separation of duties.
PBS runs off the host it protects. The restore procedure isolates a restored
clone before first boot, which correctly treats duplicated gateway tokens and
service identity as a hazard.

Three problems now dominate.

The first is unresolved from the original review and remains the most serious
authorization defect. Synology Task Scheduler is instructed to execute Compose
as root from a project directory controlled by the setup user. An attacker who
can modify `docker-compose.yaml`, `.env`, an invoked script, or the build
context turns the next scheduled root execution into arbitrary host access.
That now matters more than it did, because NAS root reaches the PBS datastore
as well as the Restic repository.

The second is new and structural. Both repositories share one NAS. No copy of
anything exists in an independent failure domain. The deployment is not 3-2-1
compliant, and no amount of encryption changes that: encryption provides
confidentiality, not immutability or availability.

The third is new and operational. Both backup paths are designed to fail
closed, which is correct, and both fail quietly, which is not survivable
without monitoring. A failed exporter produces no Restic snapshot; a failed
PVE or PBS task reports only where someone looks. Nothing outside the two
backup systems currently answers the question "when did each path last
succeed". Two paths that both fail silently are worse than one path that is
watched.

The recommended direction is unchanged in substance and reordered in priority:

1. Put backup-health alerting outside Hermes and confirm that both paths are
   producing recovery points.
2. Install a fixed copy of the application into a root-owned,
   non-user-writable deployment directory. Do not run scheduled root jobs from
   an ordinary Git working tree.
3. Establish at least one encrypted copy outside the Synology NAS.
4. Protect local history on both repositories with protected snapshots,
   quotas, and non-overlapping maintenance windows.
5. Store the private key and Restic password in a dedicated encrypted Synology
   shared folder with tightly restricted DSM ACLs and no file-service exposure,
   and run the container under a dedicated non-interactive identity.
6. Keep independent offline recovery copies of every recovery secret, which now
   number more than the Restic password alone.

## Threat model

The design should distinguish the following attackers and failures because no
single technique handles all of them. Rows marked "new" arise from the current
deployment rather than the original one.

| Threat | Current protection | Remaining risk | Recommended control |
| --- | --- | --- | --- |
| Accidental Git commit | `.env` and `secrets/` are ignored | Copies, renamed files, or other tools may still capture them | Store secrets outside the working tree and scan commits |
| Secret included in image | Secrets are runtime bind mounts | A malicious or modified build can read host-mounted secrets when run | Root-own deployment inputs and review/pin the installed image |
| Other ordinary DSM user | POSIX modes are `0700`/`0600` | Synology ACLs may add or inherit access; the current owner is an interactive user | Dedicated identity, dedicated share, explicit DSM ACL review |
| Compromised setup user's account | Container is non-root | Root Task Scheduler later executes user-controlled Compose and scripts | Root-owned installed deployment; no scheduled execution from working tree |
| Removed disks | Restic and PBS content are encrypted | The Restic password and SSH private key are plaintext on the same unencrypted storage | Encrypted secret share or encrypted volume; key material kept separately |
| Powered-off NAS theft | File modes and content encryption | Local plaintext secrets can be recovered; locally auto-unlocked encryption may reduce separation | External Key Manager store, manual unlock, or remote KMIP |
| Powered-on NAS theft | File modes and disk encryption | Mounted files are readable by live root; both repositories leave with the device | Rapid revocation, remote KMIP, network isolation, off-site copy |
| Live DSM root compromise | Container hardening limits the container | Host root controls mounts, files, Docker, VMM, the NFS export, and both repositories | Cannot be solved locally; use external trust and off-site immutability |
| Compromised backup container | Read-only filesystem, dropped capabilities, non-root UID | The running backup must receive both credentials and can potentially exfiltrate them | Trusted root-owned image/code, restricted egress, short runtime, rotation capability |
| Compromised Hermes guest (new severity) | Forced-command key; PVE controls scheduling and encryption from outside the guest | The guest can still emit malicious, incomplete, or oversized application output | Prefer the cold PBS image for recovery when guest output is suspect; bound export size and duration |
| Compromised PVE host (new) | Nothing in this repository | PVE holds the client AES key and the PBS token, and can delete its own backup groups | Treat PVE as a trusted control plane; keep retention on PBS, not on the PVE token; keep an off-site copy PVE cannot reach |
| Compromised PBS VM (new) | Least-privilege datastore ACLs; content is encrypted | The PBS VM holds a read/write NFS mount of the whole datastore and can destroy chunks | Protected NAS snapshots; NFS restricted to the PBS address; network segmentation |
| LAN host spoofing the PBS address (new) | NFS export restricted to one IP; `AUTH_SYS` | Numeric identities are client-asserted, so an impostor on the LAN gets read/write to the datastore | Backup VLAN or firewall enforcement, Kerberos-secured NFS if supported, protected snapshots |
| Ransomware or destructive NAS administrator | Restic and PBS encrypt content | The Restic client has delete access and runs retention; NAS root can destroy both repositories | Immutable snapshots and remote append-only or off-site storage |
| Loss of the NAS as a whole (new) | None | Both the machine-level and application-level recovery paths are lost together | Off-site encrypted PBS sync as the first milestone |
| Silent backup failure (new) | Restic creates no snapshot on exporter failure | A correct refusal to snapshot is indistinguishable from success without monitoring | External alerting on last-success age for both paths |
| Lost credential or encryption key | README calls for an offline Restic password copy | The PVE AES key, PBS token, and any Synology encryption key are now equally load-bearing | A recovery register plus two independent copies of each secret |

## Existing controls that should be retained

### Credentials are not committed or baked into the image

`.gitignore` excludes `.env` and the `secrets/` directory. The Compose
configuration provides the SSH key, known-hosts file, and Restic password as
read-only runtime bind mounts. The Dockerfile does not copy those values into an
image layer.

### Secret values are not placed in environment variables

The container receives paths such as `RESTIC_PASSWORD_FILE` and `SSH_KEY_FILE`,
not the secret values themselves. This avoids common leakage through process
environments, diagnostic output, and generated Compose configuration.

Restic officially supports `RESTIC_PASSWORD_FILE` and
`RESTIC_PASSWORD_COMMAND` for automated operation. A password file is a valid
automation mechanism when the host file and the process that consumes it are
appropriately protected.

### Runtime container hardening is good

The container:

- runs as a non-root UID/GID;
- has a read-only root filesystem;
- drops all Linux capabilities;
- enables `no-new-privileges`;
- uses a `tmpfs` with `noexec`, `nosuid`, and `nodev`;
- copies the SSH key into that tmpfs only for the duration of a run; and
- exits after the one-shot operation.

These controls limit accidental persistence and many container-level attacks.
They do not protect against hostile Compose configuration or a compromised
Docker/DSM administrator.

### SSH authentication is substantially constrained

The documented `authorized_keys` entry uses:

```text
from="NAS_IP",restrict,command="/absolute/path/hermes-backup-stream" ssh-ed25519 ...
```

OpenSSH's `restrict` option disables port, agent, and X11 forwarding, PTY
allocation, and execution of `~/.ssh/rc`. The forced command prevents this key
from selecting an arbitrary shell command. The `from=` condition further limits
where the key can be accepted.

The client also uses `BatchMode=yes`, `IdentitiesOnly=yes`, and
`StrictHostKeyChecking=yes`, with an explicitly verified known-hosts file.

### Restic password generation is strong

`setup-nas.sh` reads 48 bytes from `/dev/urandom` and base64-encodes them. This
provides approximately 384 bits of random input, far more than is needed to
resist password guessing. The weakness is storage location and access, not
password entropy.

### Backup streaming and failure handling are sound

The source archive is streamed directly into Restic, so no plaintext archive is
written to a NAS volume. `--stdin-from-command` lets Restic observe SSH/exporter
failure and cancel the snapshot. Retention is only attempted after a successful
backup.

### The cold path is captured outside the guest

PVE stops the Hermes VM, snapshots its disks, and uploads them without asking
the guest for cooperation or trusting anything the guest reports. Backup
scheduling, retention, and encryption all live outside the machine being backed
up. This is the property the original review identified as missing and it
should be preserved through any future change.

### Backup content is encrypted before it reaches the storage platform

PVE encrypts with AES-256-GCM on the PVE host. PBS and the Synology NAS store
ciphertext and never receive the decryption key. Neither a NAS administrator
nor a stolen disk yields guest plaintext from the cold path. This also makes
Synology shared-folder encryption largely redundant for the PBS datastore,
though not for the Restic client's plaintext credential files.

### The PVE identity is least-privilege and separately authenticated

`pve-backup@pbs!pve` is a privilege-separated API token holding
`DatastoreBackup` on a single datastore path. It can create and restore its own
backup groups but cannot administer the datastore. Retention runs on PBS, so
the routine token never needs deletion authority. Possessing the token does not
decrypt anything, and possessing the AES key grants no network access. The PBS
TLS fingerprint is pinned in PVE, so the storage endpoint is authenticated
rather than merely reachable.

### PBS verification does not require the decryption key

PBS validates chunk hashes, authenticated-encryption metadata, and manifests
without decrypting guest data. Integrity checking is therefore performed by a
component that cannot read the content it checks. Preserve this separation; do
not move the AES key onto PBS to make any operation more convenient.

### Restore drills isolate the clone

The restore procedure requires a new VM ID, disabled autostart, a disconnected
NIC before first boot, and an explicit rule against running production and a
credential-identical clone at the same time. This correctly treats duplicated
gateway tokens, OAuth credentials, host keys, and sessions as the real hazard,
rather than relying on a changed MAC address. The same reasoning applies to
application-level restores: import a Restic-recovered Hermes export into an
isolated VM, not into the running production guest.

## Findings

### Critical: root runs a deployment controlled by an interactive user

Unremediated. The setup script creates `.env` under the cloned project and makes
the project user the owner of the adjacent secrets directory. The README then
directs a root Task Scheduler job to change into that project and execute
Compose.

This creates a delayed privilege-escalation path. An attacker who can write the
project can, for example:

- replace or edit `docker-compose.yaml`;
- point a bind mount at another host path;
- replace `backup.sh` before rebuilding;
- alter `.env` paths and runtime settings;
- change the Dockerfile or image entrypoint; or
- arrange for a future setup/build invocation to execute hostile code.

Because Compose itself is launched as root, container hardening cannot repair
this trust-boundary error.

The consequence has grown. NAS root now also reaches the PBS datastore
directory, the NFS export configuration, and Virtual Machine Manager. A single
successful use of this path destroys or subverts both backup paths at once,
including the cold path that exists specifically to be independent of the
guest.

#### Recommendation

Create a root-owned installed deployment, for example:

```text
/volume1/hermes-backup-app/
```

The Task Scheduler command should invoke one fixed root-owned wrapper from this
location. The wrapper, Compose file, `.env`, scripts, and relevant parent
directories must not be writable by the development/setup user or ordinary DSM
accounts. Updating the deployment should be a separate explicit privileged
operation that copies reviewed files from the working tree.

The accepted target replaces Compose entirely with fixed one-shot containers
created from a digest-pinned published image, which removes the build context
and the Compose file from the NAS as well. Either form is acceptable; running
root against a mutable Git checkout is not.

The `.env` file does not currently contain secret values, but it still controls
security-sensitive mount paths and execution, so it must be treated as trusted
deployment configuration.

### High: both backup repositories occupy a single failure domain

The PBS datastore is `/volume1/proxmox_backups` on the Synology NAS. The Restic
repository is `/volume1/Backups/restic-hermes` on the same NAS and, by the path,
the same volume. The PBS VM that fronts the datastore also runs on that NAS
under Virtual Machine Manager.

The system therefore has two backup paths and one place where all recovery
points can be lost simultaneously: NAS hardware failure, volume corruption,
theft, fire, ransomware reaching DSM, or a destructive administrator action.

Encryption does not help here. Encrypted data that has been deleted is simply
deleted. The deployment guide states the requirement plainly and it has not yet
been met.

#### Recommendation

Establish an encrypted copy outside the NAS and outside the site. The preferred
first milestone is a native PBS sync or copy job to a separately administered
PBS instance or supported remote target, because that carries the whole-machine
recovery path rather than only the application bundle. Replication credentials
should not be able to delete protected remote history.

The Restic repository may also be replicated off-site for granular recovery,
but that must not delay the whole-VM off-site path. If it is added, prefer an
append-only Restic-compatible endpoint with transport credentials independent of
the NAS, and do not give the remote service the repository password.

Until an independent copy exists, treat every other recommendation in this
document as risk reduction rather than resilience.

### High: the NAS is now a hypervisor, an NFS server, and the backup target

The Synology NAS runs Virtual Machine Manager hosting a PBS VM, exports NFS to
that VM, runs Container Manager for the Restic client, and stores both
repositories along with the plaintext Restic password and SSH private key.

Each added role is defensible on its own. Together they mean that the machine
holding all recovery data now has substantially more attack surface than the
storage appliance the original review assessed, and that a compromise of any
one role reaches data belonging to the others.

#### Recommendation

Accept the topology but reduce its blast radius:

- keep DSM, VMM, and PBS management restricted to an admin LAN or VPN and never
  port-forwarded;
- treat the PBS VM as an internet-facing-class appliance for patching purposes
  even though it is LAN-only;
- keep the Restic client's credential share invisible to VMM, PBS, and all file
  services; and
- rely on protected NAS snapshots and the off-site copy, rather than on the
  NAS's own integrity, for deletion resistance.

Splitting roles across two devices would be materially better and is not
proportionate for this deployment today. Record the decision rather than
leaving it implicit.

### High: the NFS export relies on address trust and `AUTH_SYS`

The datastore export is restricted to the PBS VM's address, uses `AUTH_SYS`
security, and deliberately preserves numeric UIDs and GIDs without mapping.
This configuration is correct for making PBS work and is weak as an
authentication boundary.

`AUTH_SYS` accepts client-asserted numeric identities. The only real control is
the IP allowlist, which is unauthenticated on a flat LAN. Any host that can
occupy or spoof `192.168.10.31` obtains read/write access to the entire PBS
datastore.

Client-side encryption limits this to an integrity and availability problem
rather than a confidentiality one: the attacker cannot read guest data, but can
delete, truncate, or corrupt chunks, which is sufficient to destroy the cold
recovery path.

#### Recommendation

- Constrain NFS at the network layer, not only in the DSM export rule: a
  dedicated backup VLAN or firewall rules permitting NFS only between the PBS
  VM and the NAS.
- Keep the PBS VM address reserved so the export rule cannot silently start
  matching a different host.
- Use Kerberos-secured NFS (`sec=krb5i` or `krb5p`) if both ends support it;
  otherwise document that the export is protected by network position alone.
- Do not enable non-privileged ports or subfolder traversal unless
  demonstrably required.
- Treat protected Btrfs snapshots of the datastore share as the actual defense
  against destructive NFS access, because they are the only control the NFS
  client cannot reach.

### High: plaintext credentials reside on unencrypted NAS storage

Unremediated. The SSH private key and Restic password are stored as regular
files under the project's `secrets/` directory. Mode `0600` is useful against
ordinary local accounts, but it does not protect against offline disk access,
unencrypted NAS theft, live root, or accidental capture by a separate NAS
backup job.

#### Recommendation

Move only the confidential files to a dedicated encrypted shared folder. Do
not expose that share through SMB, NFS, FTP, WebDAV, Synology Drive, indexing,
or unrelated backup/synchronization jobs. Give normal users and groups no
access in DSM and verify both Synology ACLs and POSIX ownership/modes.

Keep the repository in a separate shared folder. The Restic repository is
already encrypted; putting the password beside it weakens the value of that
encryption in a disk-theft scenario.

The deployment guide's rule against recursively backing up the PBS datastore as
ordinary files applies with equal force to the credential share: any NAS-wide
copy job that reaches it converts a protected secret into an unprotected one.

### High: Synology ACLs are not explicitly controlled

Unremediated. DSM normally uses Windows ACLs, including inherited entries, in
addition to the Unix modes visible through `chmod`. The setup script applies
`chmod`, but it does not create a dedicated shared folder or inspect and
constrain DSM ACLs.

#### Recommendation

Create the secret location through DSM as its own shared folder and explicitly
configure its permissions. Do not assume `chmod 700` alone describes every DSM
access path. Review:

- local users;
- local groups;
- system internal users;
- inherited ACL entries;
- application permissions;
- advanced shared-folder permissions; and
- file services, NFS rules, and indexing.

Extend the same review to the two repository shares. The PBS datastore share
must be reachable by NFS from one address and by nothing else; the Restic
repository share must be reachable by the backup identity and by no file
service at all.

DSM administrators and root remain privileged and are not a boundary this
configuration can eliminate.

### High: local repository history remains destructible

The backup container has read/write/delete access to the local Restic
repository. Daily operation may run `forget`, and the maintenance mode runs
`prune`. On the cold path, PBS itself performs pruning and garbage collection
against a datastore it mounts read/write over NFS.

A compromised NAS, administrator, scheduler configuration, backup container, or
PBS VM can therefore destroy backup history even where content remains
confidential.

The PVE token's lack of deletion authority is a genuine mitigation for one
actor. It does not constrain PBS itself, the NFS client, or NAS root.

#### Recommendation

On supported Btrfs models, schedule protected or immutable snapshots of both
repository shared folders, ideally after successful backups, with at least the
Synology recommended 7-to-14-day protection period. Immutable snapshots cannot
be deleted during that protection window, which is precisely why they are the
only local control that survives the actors above.

Schedule those snapshots outside PBS verification and garbage collection and
outside Restic pruning, and account for the fact that deleting data through PBS
or Restic will not release NAS capacity while a snapshot still references the
blocks.

Also maintain another copy outside the NAS, as described above. For stronger
ransomware isolation on the application path, send Restic backups to a remote
REST server in append-only mode and perform `forget`/`prune` from a separate,
better-protected administrative identity. Restic explicitly recommends
separating append-only backup access from full-access maintenance.

Snapshots on the same volume are not a substitute for an off-site copy or a
second failure domain.

### Medium: the two backup paths can collide in time

PVE performs a Stop-mode backup of the Hermes VM daily at 04:00. Stop mode
shuts the guest down, starts a background QEMU process against the consistent
disks, and resumes the VM after a short outage.

The Synology Restic client pulls its export over SSH from that same guest. If
its daily task overlaps the PVE window, SSH fails against a stopped or
restarting VM, the exporter never runs, and no snapshot is created. The failure
is legitimate, correct, and indistinguishable from the exporter problem
described above.

PBS verification runs Saturday at 10:00 and garbage collection Sunday at 10:00,
both against NAS disks that also serve the Restic repository and its weekly
check and prune tasks. On HDD-backed storage these jobs run for hours and
contend with each other.

#### Recommendation

Record all schedules for PVE, PBS, DSM snapshots, and the Synology Restic tasks
in one place, and separate them explicitly:

- keep the Restic pull well clear of the PVE Stop-mode window and its guest
  restart, allowing margin for a backup that runs long;
- keep Restic check and prune off the PBS verification and garbage-collection
  days; and
- keep Btrfs snapshot schedules outside both.

Verify the separation after any schedule change or system update, and alert on
overlapping task execution rather than assuming the calendar still holds.

### Medium: the Restic repository share has no capacity ceiling

The PBS datastore share has an explicit DSM quota, which correctly prevents PBS
from consuming the whole volume. The Restic repository share has no equivalent
control in this repository or the deployment guide.

Both repositories sit on the same volume. Unbounded Restic growth can therefore
exhaust the capacity that the PBS quota was designed to protect, and a full
volume degrades both backup paths at once. Btrfs snapshots retaining freed
blocks make this easier to reach than raw repository size suggests.

Synology NFS additionally reports whole-volume statistics rather than the share
quota, so PBS can display abundant free space while DSM is close to enforcing a
much smaller limit.

#### Recommendation

Set an explicit quota on the Restic repository share sized for its retention
policy plus snapshot overhead. Treat DSM shared-folder usage as the authority
for both shares and alert at roughly 80% rather than at the hard limit. Include
snapshot-retained capacity in the calculation.

### Medium: the container receives both long-lived credentials

During a backup, the container can read the Restic password and SSH private key.
This is necessary for the current workflow. A malicious image or modified
script could send them elsewhere while it runs.

#### Recommendation

- Keep the installed image and all execution inputs root-owned.
- Do not automatically rebuild from an untrusted or user-writable working tree.
- Restrict NAS/container network egress to the Hermes SSH endpoint and required
  infrastructure where practical.
- Keep the container short-lived and preserve the existing read-only/tmpfs
  controls.
- Give the check and prune containers the repository and password but not the
  SSH private key; they have no reason to hold it.
- Treat Docker access as root-equivalent.

### Medium: the SSH key grants access to the complete exported dataset

The SSH restrictions prevent an interactive shell, but possession of the key
still grants the ability to invoke the forced exporter and receive the entire
backup stream when the source restriction is satisfied. The data is plaintext
in transit inside the SSH connection before Restic encrypts it on the NAS.

#### Recommendation

Use a separate key for every source and a dedicated source-side account such as
`hermes-backup-export`. Give it no unrelated application or interactive access.
Retain the forced command, `restrict`, and `from=` controls. Restrict the
exporter and its parent directories from modification by unrelated accounts
where practical.

Rotate the SSH key by installing and testing a new restricted public key before
removing the old one.

Note that this finding no longer carries the weight it did. Because PBS
captures the entire VM disk from outside the guest, the cold path already
contains everything the export contains and more, and it is the path to prefer
when guest-produced output is suspect. Effort spent making the guest-side
exporter trustworthy has a low ceiling; effort spent keeping the cold path
independent and recoverable does not.

### Medium: the exporter executes from a normal user's writable environment

The forced command points into `/home/anton/.local/bin`. This makes the exporter
part of the normal user's writable environment. A compromise of that account or
its home can change what future backups contain.

The deployed arrangement is a symlink from `~/.local/bin` into a user-owned Git
checkout that tracks `origin/main` and is updated by `git pull --ff-only`, with
documented checks that the checkout is clean, on `main`, and not group- or
world-writable. That is a real improvement in change management and traceability
and no improvement in trust: the checkout, the symlink, and the exporter all
remain writable by the Hermes account, so a compromised guest can still replace
what the forced command executes. The pull also adds an upstream dependency to
that path.

This does not let the restricted backup key choose a different command, but it
does weaken confidence that the forced command is the reviewed exporter.

#### Recommendation

Prefer a root-owned exporter path such as `/usr/local/libexec/`, installed by
the idempotent Ansible playbook the accepted specification requires, and not
writable by the Hermes runtime account. The exporter can still run with the
minimum privileges required to read and consistently snapshot Hermes data.

### Medium: no passphrase on unattended keys is an explicit tradeoff

An unattended task cannot answer an SSH key passphrase prompt. Encrypting the
key while storing its passphrase next to it merely creates two locally readable
secrets instead of one.

The PVE client encryption key has exactly the same property for the same
reason: scheduled backups must run without an operator. The deployment
accordingly stores it unprotected on PVE at mode `0600` and relies on the
password-manager and offline copies for recovery. That is the same tradeoff,
consistently applied, and it is reasonable.

#### Recommendation

For fully unattended operation, retain a separate unencrypted Ed25519 key with
the strong server-side restrictions above. Protect it through storage
encryption, ownership, ACLs, and capability restriction.

For the strongest protection at the cost of unattended recovery after reboot,
use a passphrase-protected key loaded manually into an agent or use a manually
mounted secret share. Hardware-backed keys that require user presence are
usually incompatible with an unattended scheduled backup.

Apply the same reasoning to PVE: accept the unattended AES key on the host, and
put the protection into the recovery copies and into who can reach PVE at all.

### Medium: recovery material now spans several independent secrets

The original review tracked one operational secret and one recovery copy. The
current deployment has more, and the loss of any one of them removes a recovery
path:

| Item | Held by | Loss consequence |
| --- | --- | --- |
| Restic repository password | NAS `secrets/` | Every application snapshot is unrecoverable |
| NAS SSH private key | NAS `secrets/` | Live export stops until a new key is authorized |
| PVE client AES JSON key | PVE `/etc/pve/priv/storage/` | Every encrypted PBS snapshot is unrecoverable |
| PBS API token secret | PVE storage config | PVE cannot back up or restore until reissued |
| PBS TLS fingerprint | PVE storage config | Storage goes inactive; recoverable by re-pinning |
| Synology encryption recovery key | DSM, if encryption is adopted | The secret store or volume cannot be unlocked |

The AES key deserves particular attention. Generating a new one does not
decrypt old snapshots, so losing every copy converts a healthy, verified,
fully replicated datastore into unusable ciphertext.

#### Recommendation

Maintain a written recovery register listing each item, where its copies live,
and how to use it — never the values themselves. Keep at least two independent
copies of each secret outside the NAS and outside PVE: a well-protected
password manager and an offline encrypted device or sealed physical record are
appropriate. Do not store the only recovery copy on the same NAS or in a
synchronization path sharing the same failure domain.

Test the recovery procedure periodically, including a full Restic restore to an
isolated destination and a PBS restore performed by uploading the saved AES key
to a freshly installed PVE rather than by reusing the running one.

### Medium: credentials need a defined rotation and incident procedure

The current documentation does not specify normal rotation or response to a
suspected leak, and the set of credentials has grown.

#### Recommendation

- SSH key compromise: install a new restricted key, verify it, remove the old
  `authorized_keys` entry, and replace the NAS private key.
- Restic password exposure without evidence of repository decryption: add and
  test a new Restic key, remove the old key, and update the protected password
  file.
- Suspected compromise of decrypted repository access: create a new repository
  with new key material and migrate/copy verified snapshots. Merely changing a
  password does not make data already decrypted by an attacker secret again.
- PBS token compromise: delete the token in PBS, issue a new one, update the
  PVE storage, and review datastore ACLs and task history for unexpected
  activity.
- PVE AES key exposure: existing snapshots remain readable to whoever holds the
  key and cannot be retroactively protected. Generate a new key for future
  backups, retain the old key for as long as old snapshots must be restorable,
  and treat the exposure window as a data-disclosure incident.
- PBS certificate change: re-pin the fingerprint through a trusted admin
  session, never by accepting whatever the storage currently presents.
- NAS theft: revoke the SSH public key immediately, revoke the PBS token,
  remove the NFS export rule, and, when remote KMIP is in use, remove the
  stolen NAS's KMIP client certificate.
- Preserve logs and verify repository integrity and restored content after any
  incident.

### Medium: backup health is not observable outside the guest

Both paths fail closed, which is correct, and both fail quietly, which is not
survivable without monitoring. Restic declines to snapshot when the exporter
fails, so a broken exporter, an unreachable guest, and a healthy idle system
all look alike from the repository. PVE and PBS report task results only where
someone looks.

The exporter's own preconditions make this concrete. It exits 127 if the Hermes
binary is missing and exits 1 if Hermes produces an empty archive, and in both
cases `--stdin-from-command` correctly creates no snapshot. Those guards are
right; without external alerting they convert a source-side change into an
unnoticed gap in recovery points.

Hermes must not be the thing that reports on backups of Hermes.

#### Recommendation

Establish the current state first:

```bash
sudo MODE=snapshots /usr/local/bin/docker-compose run --rm hermes-backup
```

Compare the newest snapshot time against the schedule, and confirm the cold
path independently in PBS from the newest snapshot in the `synology` datastore
and the last verification result.

Then alert outside the guest on:

- missed or failed PVE, PBS, and Synology scheduled tasks;
- age of the newest snapshot in each repository, not merely task exit status;
- failed verification, check, prune, or garbage-collection runs;
- changed SSH or PBS fingerprints;
- export size or duration outside established bounds;
- overlapping task execution;
- shared-folder quota pressure on both shares;
- absent or expired protected snapshots; and
- stale off-site replication once it exists.

An operator should be able to answer "when did each path last succeed" without
logging into either backup system.

### Low: `known_hosts` is classified as a secret

The SSH known-hosts file is not confidential, but its integrity is essential.
An attacker who can replace it and redirect the SSH target can undermine server
authentication.

The PBS TLS fingerprint in the PVE storage configuration is the same kind of
value: public, integrity-critical, and useless as a control if it can be
rewritten or re-accepted casually.

#### Recommendation

Move `known_hosts` into root-owned application configuration rather than the
encrypted secrets folder. Keep it non-writable by the runtime identity and
continue verifying fingerprints out of band during setup and rotation. Treat
the PBS fingerprint identically: change it only through a trusted admin session
after a known certificate change.

## Synology encryption options

Client-side encryption on PVE already protects the PBS datastore contents, so
these options now matter mainly for the Restic client's plaintext credential
files and, secondarily, for the Restic repository itself.

### Option A: manually mounted encrypted shared folder

Create a dedicated encrypted shared folder and do not enable mount-on-boot.
After every NAS reboot, an administrator manually imports or enters its key
before scheduled backups can run.

Advantages:

- Creates a real human-controlled unlock boundary.
- Strong protection for a powered-off NAS and removed drives.
- No unattended local key path is available at boot.

Tradeoffs:

- Backups fail after reboot until the folder is mounted.
- The mounted folder is still readable by live root.
- Requires reliable operational monitoring.

This is the strongest single-NAS option when human intervention after reboot is
acceptable. Note that the NAS now also hosts the PBS VM, so unattended recovery
after a power event has become more operationally important than it was.

### Option B: encrypted shared folder with Synology Key Manager

Synology Key Manager can store encrypted-shared-folder keys in a system
partition or external device. Only the machine-key cipher supports automatic
mounting on boot. Synology recommends an external device as the key-store
location and supports ejecting that device after boot.

Advantages:

- Maintains unattended operation.
- Protects removed data disks.
- An external key store that is removed after boot improves separation for a
  subsequently powered-off or moved NAS.

Tradeoffs:

- Automatic unlock necessarily makes keys available to the NAS.
- A running, already-mounted system remains readable by root.
- Theft scenarios depend on whether the key-store device is still present and
  whether the NAS is running.

This remains the recommended balance for this NAS.

### Option C: DSM 7.2+ encrypted volume with local Key Vault

Supported models can create an encrypted volume using Synology's Encryption Key
Vault. The volume automatically unlocks when its vault is available. Volume
encryption protects all data on that volume, not just the two application
secrets, and is irreversible for the created volume.

Advantages:

- Broad at-rest protection using LUKS/dm-crypt.
- Protects shared folders, package data, VMM images, and other contents on the
  encrypted volume.

Tradeoffs:

- Model and DSM-version dependent.
- More operational scope and possible performance cost than a small encrypted
  shared folder, and the volume now carries VM and datastore I/O.
- Largely redundant for PBS content that PVE already encrypted.
- A local available vault does not protect against live root.

### Option D: DSM 7.2+ encrypted volume with remote KMIP

On compatible Synology systems, the Encryption Key Vault can live on another
Synology NAS over KMIP. The encrypted-volume NAS automatically unlocks only
when the remote key server is available. Synology documents removal of the
client certificate on the key server as a way to prevent a lost NAS from
retrieving its key.

Advantages:

- Separates volume encryption keys from the encrypted NAS.
- Covers loss of the full NAS better than a local vault.
- Allows remote revocation of a lost client's access.

Tradeoffs:

- Requires a second compatible Synology NAS, certificates, monitoring, and
  recovery planning.
- Failure or certificate expiry can prevent automatic unlock, which now also
  stops the PBS VM and the cold backup path.
- A client that is already running with the volume unlocked remains exposed to
  live root.
- KMIP unlocks the filesystem; it is not an application secret manager that
  supplies the Restic password directly.

If a second NAS becomes available, using it as an off-site PBS sync target is a
better first investment than using it as a KMIP key server. The former
addresses the dominant risk; the latter addresses a secondary one.

## Why several tempting approaches do not solve the problem

### Encrypting a secret with another local secret

An encrypted private key plus a passphrase file beside it has the same effective
host boundary as the original private key. The same applies to encrypting the
Restic password with a locally stored decryption key.

Encryption helps only when the unlocking authority is separated through human
input, different hardware, a remote service, or a more restricted credential.

### Treating client-side encryption as deletion resistance

PVE encrypts backup content before PBS or the NAS sees it, which is genuinely
valuable and solves confidentiality completely for the cold path. It does
nothing about deletion, corruption, quota exhaustion, or NAS loss. Encrypted
chunks that a compromised NFS client removes are gone. Confidentiality and
availability need different controls; only snapshots and off-site copies
provide the second.

### Adding a second local repository instead of a second location

Two repositories on one NAS satisfy neither the "different media" nor the
"off-site" element of 3-2-1. The current deployment already demonstrates the
limit: PBS and Restic protect against different failures of the source and
against none of the same failures of the storage.

### Switching to ordinary Docker Compose `secrets`

Compose secret syntax is clearer about intent and grants per-service access,
but a file-backed Compose secret is still implemented as a host bind mount. It
does not encrypt the source file at rest and does not protect it from host root.
The current read-only file mounts already provide the relevant runtime behavior.

Docker Swarm secrets are different: Swarm stores them in an encrypted Raft log
and mounts decrypted values from memory into authorized services. Even then, a
live Swarm manager/host and the authorized running service remain trusted. The
operational complexity and Synology supportability must be weighed against the
limited benefit for this one-node system.

### Moving the password into an environment variable

This is worse than the current file-based design. Environment variables can be
included in diagnostic output, inherited by child processes, and exposed by
process and container inspection. Continue using Restic's password-file or
password-command interface.

### Using an external password manager without solving bootstrap authentication

Restic can call a command to obtain its password, so Vault or another external
secret service can be integrated. However, a completely unattended client must
still authenticate to that service. A permanent unrestricted API token stored
on the same NAS simply replaces the Restic password with another valuable
secret.

An external manager materially improves the design only when its client
credential is scoped, revocable, short-lived, hardware-bound, manually
unlocked, or otherwise less useful to an attacker than the secret it retrieves.

## Recommended target layout

```text
PVE host                                   trusted control plane
  /etc/pve/priv/storage/<id>.enc           client AES key, root-only, mode 0600
  PBS API token secret                     privilege-separated, DatastoreBackup only

Hermes VM                                  untrusted data source
  /usr/local/libexec/hermes-backup-stream  root-owned exporter, not guest-writable
  ~/.ssh/authorized_keys                   from=, restrict, forced absolute command

Synology NAS
  /volume1/hermes-backup-app/              root-owned installed deployment
    run-backup                             fixed Task Scheduler entrypoint
    known_hosts                            root-owned integrity-sensitive config
  /volume1/hermes-backup-secrets/          dedicated encrypted shared folder
    hermes_ed25519                         backup UID, mode 0400 or 0600
    restic_password                        backup UID, mode 0400 or 0600
  /volume1/Backups/restic-hermes/          Restic repository share, quota, no file services
  /volume1/proxmox_backups/                PBS datastore share, quota, NFS to PBS VM only
  /volume1/development/hermes-nas-backup/  optional Git working tree
                                           never executed by a scheduled root task

PBS VM
  /mnt/pbs-nfs                             hard NFSv4.1 automount, backup:backup, 0750

Independent failure domain                 currently missing
  encrypted PBS sync/copy target           off-site whole-VM recovery
```

The exact volume and share names can differ, but the trust boundaries should
not. Protected Btrfs snapshots should cover both repository shares.

The dedicated backup identity should:

- have a stable UID/GID shared with the container;
- have no interactive DSM, SSH, or file-service login;
- be denied unrelated DSM applications and shares;
- read only its private key and Restic password;
- write only the Restic repository and necessary runtime locations; and
- not own or be able to modify installed scripts or Compose configuration.

Task Scheduler may still need to invoke Docker as root. Docker access is
effectively root-equivalent, so the safer boundary is a root-owned fixed
deployment rather than granting Docker access to an ordinary user.

## Implementation priorities

Priorities are reordered from the original review. The addition of the PVE/PBS
path did not reduce the urgency of Priority 1, and it raised the urgency of the
off-site copy from "eventually" to "the largest single gap".

### Priority 0: confirm both paths work and can be seen to work

1. List Restic snapshots and compare the newest against the schedule.
2. Confirm the newest PBS snapshot and the last successful verification.
3. Confirm that the Restic pull window does not overlap the PVE Stop-mode
   backup or the guest restart.
4. Enable failure notifications on PVE, PBS, and DSM Task Scheduler.
5. Alert on newest-snapshot age for both paths, not only on task exit status.

### Priority 1: remove the root execution trust failure

1. Add an explicit privileged installation/update step.
2. Install reviewed application files into a root-owned directory, or move to
   digest-pinned native containers as the accepted specification requires.
3. Install a fixed root-owned scheduler wrapper.
4. Make the scheduler call only that wrapper with an absolute path.
5. Reject an installed deployment if a relevant file or parent directory is
   writable by the backup identity or ordinary users.

### Priority 2: establish an independent failure domain

1. Configure a native PBS sync or copy job to an off-site or separately
   administered target.
2. Ensure replication credentials cannot delete protected remote history.
3. Verify a restore that uses only off-site data and the saved AES key.
4. Optionally add append-only off-site Restic storage for granular recovery.

### Priority 3: protect local history

1. Enable protected or immutable Btrfs snapshots on both repository shares.
2. Set a quota on the Restic repository share and alert at roughly 80% on both.
3. Separate backup, verification, garbage-collection, check, prune, and
   snapshot schedules, and alert on overlap.
4. Constrain NFS at the network layer in addition to the DSM export rule.

### Priority 4: separate credentials and establish a dedicated identity

1. Create a dedicated encrypted shared folder.
2. Choose manual unlock, external Key Manager storage, or remote KMIP according
   to the required availability/security balance.
3. Move the two confidential credentials outside the project.
4. Move `known_hosts` to root-owned application configuration.
5. Review DSM ACLs and exclude the share from all file services and unrelated
   backup jobs.
6. Rotate the credentials after migration because their past exposure cannot be
   proven from repository state.
7. Create or reserve a non-interactive backup UID/GID, run the image with it,
   grant only the repository and credential access required, and remove
   reliance on the interactive setup user's UID/GID.
8. Withhold the SSH private key from the check and prune containers.
9. Build the recovery register and place two independent copies of every
   recovery secret outside both machines.

### Priority 5: harden the surrounding systems

On DSM:

- disable the default `admin` account;
- enforce MFA for administrators;
- enable Auto Block and Account Protection;
- limit DSM, VMM, SSH, NFS, and other service exposure with the DSM and network
  firewalls;
- install security updates promptly;
- run Security Advisor regularly;
- alert on failed scheduled tasks and suspicious logins; and
- treat all members of the administrators group as capable of accessing a
  mounted secret store and both repositories.

On PVE and PBS:

- restrict PVE 8006 and PBS 8007 to an admin LAN or VPN and never port-forward
  them;
- keep both on supported release channels and patch promptly;
- keep retention on PBS so the routine PVE token needs no deletion authority;
- keep the AES key off PBS; and
- repeat isolated restore drills after major architecture changes.

On the Hermes VM:

- use a dedicated export account and key;
- retain `from=`, `restrict`, and a forced absolute command;
- make the exporter root-owned and integrity-protected;
- restrict inbound SSH to the NAS where practical;
- do not expose the dashboard or gateway ports directly; and
- monitor use of the backup key.

## Information status

The original review listed unknowns. Most are now answered by the deployment
guide.

Known:

- Hermes runs as a VM on PVE with a daily encrypted Stop-mode backup to PBS.
- PBS runs on the Synology NAS with an NFS-backed Btrfs datastore.
- The NAS supports Btrfs, VMM, NFSv4.1, and shared-folder quotas.
- The PBS datastore share has a quota, checksums, no recycle bin, and no
  redundant compression.
- The Synology Restic client still runs the `main` Compose deployment.
- No off-site or second-failure-domain copy exists.

Still to confirm before implementation:

- Synology model and DSM version, and therefore encrypted-shared-folder,
  encrypted-volume, remote-KMIP, and immutable-snapshot support;
- whether the Restic repository and PBS datastore share one volume;
- whether NAS snapshots are currently enabled on either repository share;
- whether a second Synology NAS or an off-site target is available, and whether
  it is better used for PBS sync or as a KMIP key server;
- whether a removable USB key store is acceptable;
- whether manual unlock after reboot is acceptable now that the NAS also hosts
  the PBS VM; and
- whether the LAN can be segmented for NFS and management traffic.

These choices determine whether the final profile is manual/high-security,
single-NAS unattended, or remote-vault unattended.

## Primary references

- [Hermes homelab deployment: PVE, PBS, and
  Synology](https://github.com/chocobot-farm/plume-pilot/blob/main/docs/deployment/0001-hermes-homelab-pve-pbs-synology.md)
- [Proxmox VE backup and restore](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)
- [PBS client-side AES encryption](https://pbs.proxmox.com/docs/backup-client.html#encryption)
- [PBS user management and API tokens](https://pbs.proxmox.com/docs/user-management.html)
- [PBS backup storage and maintenance](https://pbs.proxmox.com/docs/storage.html)
- [Synology DSM NFS permissions](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/file_share_privilege_nfs?version=7)
- [Synology: Manage Encrypted Shared Folders](https://kb.synology.com/en-au/DSM/help/DSM/AdminCenter/file_share_key_manager)
- [Synology: Create and manage an encrypted volume](https://kb.synology.com/en-global/DSM/help/DSM/StorageManager/volume_create_volume)
- [Synology: Set up a remote KMIP key server](https://kb.synology.com/en-us/DSM/tutorial/How_do_I_set_up_KMIP_server)
- [Synology Volume Encryption white paper](https://kb.synology.com/en-eu/WP/Synology_Volume_Encryption_White_Paper/3)
- [Synology: Assign Shared Folder Permissions](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/file_share_privilege?version=6)
- [Synology: Manage Advanced Shared Folder Permissions](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/file_share_privilege_asp?version=7)
- [Synology: Snapshot Replication and immutable snapshots](https://kb.synology.com/en-us/DSM/help/SnapshotReplication/snapshots?version=7)
- [Synology NAS security guidance](https://kb.synology.com/en-us/DSM/tutorial/How_to_add_extra_security_to_your_Synology_NAS)
- [Docker: Manage secrets securely in Compose](https://docs.docker.com/compose/how-tos/use-secrets/)
- [Docker: Compose trust model](https://docs.docker.com/compose/trust-model/)
- [Docker: Swarm secrets](https://docs.docker.com/engine/swarm/secrets/)
- [Restic: Preparing a new repository and supplying passwords](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html)
- [Restic: Scripting environment and password commands](https://restic.readthedocs.io/en/stable/075_scripting.html)
- [Restic: Append-only repositories and maintenance separation](https://restic.readthedocs.io/en/stable/060_forget.html)
- [OpenSSH: `authorized_keys` restrictions](https://man.openbsd.org/OpenBSD-current/man8/sshd.8)
