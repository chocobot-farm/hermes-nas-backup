## Context

Hermes is an Internet-connected AI agent. It runs as an Ubuntu VM on a dedicated Proxmox VE host; PVE performs a daily stop-mode whole-VM backup encrypted with AES-256-GCM before it leaves the host, writing it to a Proxmox Backup Server VM under Synology Virtual Machine Manager whose datastore is a Btrfs shared folder exported over NFSv4.1. Alongside that, a Synology one-shot container pulls a forced-command Hermes export over restricted SSH and streams it into an encrypted Restic repository. Both paths are in service; the second one is exactly the `main` Compose deployment in this repository.

A security review dated 2026-07-16, revised 2026-07-27 for this deployment, assessed `setup-nas.sh`, `docker-compose.yaml`, `Dockerfile`, `backup.sh`, `server/hermes-backup-stream`, and the surrounding PVE/PBS/Synology deployment. Its findings, and the three specifications written against them, are the source material for this change. No secret values were read during that review; statements about the live hosts are inferences from the deployment guide and repository contents, and items marked as verification steps must be confirmed against the running systems.

Two structural facts frame every decision below.

The first is favorable. The original review had no good answer to a compromised Hermes source, because everything the backup system knew about Hermes came from an exporter running inside Hermes. A cold, guest-independent image backup now exists, so there is a recovery path that does not depend on the guest telling the truth. This is the single largest security improvement in the system and it did not come from hardening the Restic client.

The second is unfavorable. Every recovery point now lives on one Synology NAS. The PBS datastore and the Restic repository sit on the same device, almost certainly the same volume, under the same administrators, and the NAS also hosts the PBS VM and serves the NFS export beneath its datastore. The system has two backup paths and one failure domain.

### Threat model

| Threat | Current protection | Remaining risk | Recommended control |
| --- | --- | --- | --- |
| Accidental Git commit | `.env` and `secrets/` are ignored | Copies, renamed files, or other tools may still capture them | Store secrets outside the working tree and scan commits |
| Secret included in image | Secrets are runtime bind mounts | A malicious or modified build can read host-mounted secrets when run | Root-own deployment inputs and review/pin the installed image |
| Other ordinary DSM user | POSIX modes are `0700`/`0600` | Synology ACLs may add or inherit access; the current owner is an interactive user | Dedicated identity, dedicated share, explicit DSM ACL review |
| Compromised setup user's account | Container is non-root | Root Task Scheduler later executes user-controlled Compose and scripts | Root-owned installed deployment; no scheduled execution from the working tree |
| Removed disks | Restic and PBS content are encrypted | The Restic password and SSH private key are plaintext on the same unencrypted storage | Encrypted secret share or encrypted volume; key material kept separately |
| Powered-off NAS theft | File modes and content encryption | Local plaintext secrets can be recovered; locally auto-unlocked encryption may reduce separation | External Key Manager store, manual unlock, or remote KMIP |
| Powered-on NAS theft | File modes and disk encryption | Mounted files are readable by live root; both repositories leave with the device | Rapid revocation, remote KMIP, network isolation, off-site copy |
| Live DSM root compromise | Container hardening limits the container | Host root controls mounts, files, Docker, VMM, the NFS export, and both repositories | Cannot be solved locally; use external trust and off-site immutability |
| Compromised backup container | Read-only filesystem, dropped capabilities, non-root UID | The running backup must receive both credentials and can potentially exfiltrate them | Trusted root-owned image/code, restricted egress, short runtime, rotation capability |
| Compromised Hermes guest | Forced-command key; PVE controls scheduling and encryption from outside the guest | The guest can still emit malicious, incomplete, or oversized application output | Prefer the cold PBS image when guest output is suspect; bound export size and duration |
| Compromised PVE host | Nothing in this repository | PVE holds the client AES key and the PBS token, and can delete its own backup groups | Treat PVE as a trusted control plane; keep retention on PBS; keep an off-site copy PVE cannot reach |
| Compromised PBS VM | Least-privilege datastore ACLs; content is encrypted | The PBS VM holds a read/write NFS mount of the whole datastore and can destroy chunks | Protected NAS snapshots; NFS restricted to the PBS address; network segmentation |
| LAN host spoofing the PBS address | NFS export restricted to one IP; `AUTH_SYS` | Numeric identities are client-asserted, so an impostor on the LAN gets read/write to the datastore | Backup VLAN or firewall enforcement, Kerberos-secured NFS if supported, protected snapshots |
| Ransomware or destructive NAS administrator | Restic and PBS encrypt content | The Restic client has delete access and runs retention; NAS root can destroy both repositories | Immutable snapshots and remote append-only or off-site storage |
| Loss of the NAS as a whole | None | Both the machine-level and application-level recovery paths are lost together | Off-site encrypted PBS sync as the first milestone |
| Silent backup failure | Restic creates no snapshot on exporter failure | A correct refusal to snapshot is indistinguishable from success without monitoring | External alerting on last-success age for both paths |
| Lost credential or encryption key | README calls for an offline Restic password copy | The PVE AES key, PBS token, and any Synology encryption key are now equally load-bearing | A recovery register plus two independent copies of each secret |

### Findings that drive this change

- **Critical — root runs a deployment controlled by an interactive user.** The setup script creates `.env` under the cloned project and makes the project user the owner of the adjacent secrets directory; the README then directs a root Task Scheduler job to change into that project and execute Compose. An attacker who can write the project can replace `docker-compose.yaml`, point a bind mount at another host path, replace `backup.sh` before a rebuild, alter `.env`, or change the Dockerfile or entrypoint. Because Compose is launched as root, container hardening cannot repair this trust-boundary error, and NAS root now reaches the PBS datastore, the NFS export configuration, and VMM as well.
- **High — both repositories occupy a single failure domain.** `/volume1/proxmox_backups` and `/volume1/Backups/restic-hermes` are on the same NAS and, by the paths, the same volume. Encrypted data that has been deleted is simply deleted.
- **High — the NAS is now a hypervisor, an NFS server, and the backup target.** Each role is defensible alone; together, a compromise of any one reaches data belonging to the others.
- **High — the NFS export relies on address trust and `AUTH_SYS`.** Client-side encryption reduces this to an integrity and availability problem rather than a confidentiality one, which is still sufficient to destroy the cold recovery path.
- **High — plaintext credentials on unencrypted NAS storage**, **High — Synology ACLs are not explicitly controlled**, and **High — local repository history remains destructible**.
- **Medium** — the two paths can collide in time (the Restic pull can hit a stopped or restarting guest during the PVE window); the Restic share has no quota; the container receives both long-lived credentials; the SSH key grants the complete exported dataset; the exporter executes from a user-writable environment; unattended keys have no passphrase by explicit trade-off; recovery material now spans six independent secrets; rotation and incident procedures are undefined; and backup health is not observable outside the guest.
- **Low** — `known_hosts` is classified as a secret when it is integrity-sensitive rather than confidential; the PBS TLS fingerprint is the same kind of value.

### Controls that already work and must survive this change

Credentials are excluded from Git and the image, mounted read-only, and consumed as files rather than environment values. The container is non-root with a read-only root filesystem, all capabilities dropped, `no-new-privileges`, and a `noexec,nosuid,nodev` tmpfs. SSH host-key verification is strict and the client key is restricted by source address, forced command, and `restrict`. The generated Restic password has ample entropy. The source archive streams directly into Restic so no plaintext archive lands on a NAS volume, `--stdin-from-command` lets Restic cancel the snapshot on producer failure, and retention is attempted only after a successful backup. On the cold path, capture happens outside the guest, content is encrypted before the storage platform sees it, the PVE identity is a privilege-separated token that cannot administer the datastore, PBS verifies integrity without holding the decryption key, and restore drills isolate the clone.

## Goals / Non-Goals

### Goals

1. Preserve independent machine-level and application-level recovery paths.
2. Keep PBS credentials, the PVE client encryption key, Restic credentials, and NAS credentials out of the Hermes guest.
3. Keep backup scheduling outside the Hermes guest.
4. Make the cold PVE/PBS backup the trustworthy recovery path when guest-provided application output is suspect.
5. Preserve the existing application-consistent Hermes stream.
6. Remove mutable Git checkout and Compose files from scheduled root execution on Synology.
7. Make source-side backup protocol files root-owned and non-writable by the Hermes runtime account.
8. Pin the Synology backup image by immutable digest and retain its current runtime hardening.
9. Bound live-export duration and size, and ensure producer failure creates no Restic snapshot.
10. Separate backup, check, and prune execution so each operation receives only required mounts.
11. Protect local repositories against destructive changes with NAS snapshots where supported.
12. Maintain at least one encrypted copy in a failure domain independent of the Synology NAS.
13. Monitor outcomes outside Hermes and exercise real isolated restores.
14. Support deliberate upgrades, rollback, credential rotation, and incident response.

### Non-Goals

This change does not attempt to prove that application data emitted after guest compromise is truthful; protect secrets from live root compromise of the system that legitimately uses them; treat encryption as protection from deletion; claim that two repositories on one NAS satisfy 3-2-1; require the application exporter to make a hostile guest trustworthy; require automatic deployment of newly published container images; require automatic pruning with the routine PBS backup token; or allow production and a credential-identical restored Hermes VM to run concurrently.

## Decisions

### Decision: three proposed spec sets are carried in parallel, not merged

The source material comprises three specifications that overlap heavily and differ in scope: a system-level unified proposal covering both paths, a proposal covering only the Synology client's native container deployment, and a proposal built around a dedicated backup gateway that treats the source as a hostile tenant. They are imported as three sibling capability trees rather than reconciled into one, so each remains readable as the coherent document it was written as, and none is presented here as accepted, rejected, or superseded. Where they conflict, the conflict is visible by comparing sibling capabilities rather than hidden by an editorial merge.

The consequence is deliberate duplication: bounded hostile output, off-site immutability, isolated restore, and out-of-guest monitoring each appear in more than one tree, stated in that tree's own terms.

### Decision: PVE/PBS is the machine-level control plane; no new gateway host is required for it

The property the gateway architecture was written to obtain — scheduling, credentials, and encryption keys outside the guest — is supplied by the deployed hypervisor profile. That leaves a genuine trade-off recorded rather than resolved here: building the dedicated gateway would additionally remove live DSM root from the position of being able to obtain both source-read capability and the Restic repository decryption key, at the cost of a new host, a new append-only storage service, and a migration; closing the four unmet gaps instead — off-site copy, protected local history, bounded export, external monitoring — costs less and addresses failure modes that are currently unmitigated. The `hostile-source-architecture` tree keeps the gateway design intact so that trade-off stays available rather than being lost.

The controls in that tree that do not require a separate host — bounded spool size and runtime, a per-run manifest, fail-closed behavior, and separation of backup from prune and check — apply to the Synology client as it stands.

### Decision: stop mode over guest-agent snapshot mode

An orderly stop invalidates the hypervisor's dirty bitmap, so each run may reread and checksum the whole virtual disk even though only changed chunks are uploaded, and the guest is briefly offline daily. That is accepted in exchange for consistency that does not depend on guest cooperation. Backup-window sizing is based on the full-read cost. Moving to snapshot mode for speed is permitted only as a recorded decision to accept guest-influenced consistency.

### Decision: the digest is computed on the NAS, inline, over received bytes

A source-supplied checksum proves nothing about a hostile source. The digest and byte length are computed on the NAS over the bytes actually received as they pass to Restic. Inserting that computation must not buffer the stream to a plaintext NAS file, suppress the exporter's exit status, or prevent Restic from cancelling the snapshot on producer failure. The resulting record is evidence about transfer — which bytes were received and stored, and whether a restored snapshot re-hashes to them — not evidence about truthfulness.

### Decision: the run manifest lives outside the repository and outside Restic tags

An attempt that produced no snapshot is precisely the event that leaves no trace in the repository, so the manifest must be emitted for failures too and retained outside the repository for at least the snapshot retention period. It must not be carried in Restic tags: retention groups by host and tag, so a per-run tag value would place every snapshot in its own retention group and defeat the retention policy.

### Decision: `known_hosts` and the PBS TLS fingerprint are integrity-sensitive, not confidential

Both are public values whose usefulness depends entirely on not being rewritable. They belong in root-owned configuration, non-writable by the runtime identity, changed only through a trusted administrative session after a known key or certificate change — never by accepting whatever the endpoint currently presents.

### Decision: monitoring comes first in the migration order

Phase 1 is observability, ahead of every structural change. It is the cheapest phase, is independent of all others, changes no backup path, and no later gate can be trusted while failures are invisible.

### Synology at-rest encryption options

| Option | What it protects | What it costs |
| --- | --- | --- |
| A. Manually mounted encrypted shared folder, no mount-on-boot | Real human-controlled unlock boundary; strongest for a powered-off NAS or removed drives; no unattended local key path at boot | Backups fail after reboot until an administrator mounts it; still readable by live root; needs reliable operational monitoring. The NAS now also hosts the PBS VM, so unattended recovery after a power event matters more than it did |
| B. Encrypted shared folder with Synology Key Manager | Maintains unattended operation; protects removed data disks; an external key store removed after boot improves separation for a subsequently powered-off or moved NAS | Automatic unlock necessarily makes keys available to the NAS; a running, already-mounted system remains readable by root; theft outcomes depend on whether the key device is present |
| C. DSM 7.2+ encrypted volume with local Key Vault | Broad at-rest protection via LUKS/dm-crypt across shared folders, package data, and VM images | Model and DSM-version dependent; irreversible for the created volume; more operational scope and possible performance cost on a volume now carrying VM and datastore I/O; largely redundant for PBS content PVE already encrypted; a local available vault does not protect against live root |
| D. DSM 7.2+ encrypted volume with remote KMIP | Separates volume keys from the encrypted NAS; covers loss of the whole NAS better than a local vault; allows remote revocation of a lost client | Requires a second compatible NAS, certificates, monitoring, and recovery planning; failure or certificate expiry can prevent automatic unlock, which also stops the PBS VM and the cold path; unlocks the filesystem rather than supplying the Restic password |

Option B is the recommended balance for this NAS. If a second Synology NAS becomes available, using it as an off-site PBS sync target is a better first investment than using it as a KMIP key server: the former addresses the dominant risk, the latter a secondary one.

### Approaches considered and not adopted

- **Encrypting a secret with another local secret.** An encrypted private key plus a passphrase file beside it has the same effective host boundary as the original key. Encryption helps only when the unlocking authority is separated by human input, different hardware, a remote service, or a more restricted credential.
- **Treating client-side encryption as deletion resistance.** It solves confidentiality for the cold path completely and does nothing about deletion, corruption, quota exhaustion, or NAS loss.
- **Adding a second local repository instead of a second location.** Two repositories on one NAS satisfy neither the "different media" nor the "off-site" element of 3-2-1; the current deployment already demonstrates the limit.
- **Switching to Docker Compose `secrets`.** A file-backed Compose secret is still a host bind mount: it neither encrypts the source file at rest nor protects it from host root. Swarm secrets differ materially but add operational complexity and Synology supportability questions for a one-node system.
- **Moving the password into an environment variable.** Strictly worse than the current file-based design: environment values reach diagnostics, child processes, and container inspection.
- **An external password manager without solving bootstrap authentication.** A permanent unrestricted API token stored on the same NAS simply replaces the Restic password with another valuable secret. An external manager helps only when its client credential is scoped, revocable, short-lived, hardware-bound, or manually unlocked.
- **A dedicated non-interactive export account on the Hermes host by default.** The normal profile runs the exporter as the existing Hermes service account so it retains the data access a consistent export requires; a dedicated account is used only where its data access can be granted without broad sudo rights, unrelated group membership, or access to other application secrets.

### Target layout

```text
PVE host                                   trusted control plane
  /etc/pve/priv/storage/<id>.enc           client AES key, root-only, mode 0600
  PBS API token secret                     privilege-separated, DatastoreBackup only

Hermes VM                                  untrusted data source
  /usr/local/libexec/hermes-backup/export  root-owned exporter, not guest-writable
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

Volume and share names may differ; the trust boundaries may not. The dedicated backup identity should have a stable UID/GID shared with the container, no interactive DSM/SSH/file-service login, no unrelated DSM applications or shares, read access only to its private key and Restic password, write access only to the Restic repository and necessary runtime locations, and no ability to modify installed scripts or deployment configuration. Task Scheduler may still need to invoke Docker as root; Docker access is root-equivalent, so the safer boundary is a root-owned fixed deployment rather than granting Docker access to an ordinary user.

## Risks / Trade-offs

- **Migration without a coverage gap.** The deployed PVE/PBS and Synology paths must both remain operational until their replacements pass backup and restore acceptance tests. The existing Restic repository must not be reinitialized. Mitigation: phase gates, a rollback window, and retention of the old repository until its approved retention end.
- **Daily guest outage.** Stop mode takes Hermes offline briefly each day and the messaging gateway must reconnect. Mitigation: verify reboot persistence and gateway reconnection explicitly as an acceptance item.
- **Schedule collisions between paths.** The Restic pull against a stopped or restarting guest fails legitimately and looks identical to a broken exporter. Mitigation: record every schedule in one place, keep the pull clear of the PVE window with margin, and alert on overlapping task execution rather than trusting the calendar.
- **Snapshot capacity.** Protected snapshots retain blocks that PBS garbage collection and Restic prune would otherwise release, so both shares can approach their quotas faster than repository size suggests. Mitigation: size quotas for repository plus snapshot retention, and alert on DSM shared-folder usage rather than repository-reported free space.
- **Network separation can break the pull.** Moving the guest to its own VLAN must preserve exactly one direction: the Synology client reaching the guest's SSH port, with no return path. Mitigation: place the phase after the container migration, and require one complete cycle on each path plus a restore drill afterwards.
- **Unattended keys have no passphrase.** An unattended task cannot answer a passphrase prompt, and storing the passphrase beside the key creates two locally readable secrets instead of one. The PVE AES key has the same property for the same reason. Mitigation: put the protection into storage encryption, ownership, ACLs, and the recovery copies, and into who can reach the host at all.
- **Digest accounting must not weaken failure propagation.** Adding inline hashing is the change most likely to accidentally swallow the producer's exit status. Mitigation: explicit failure tests for exporter failure, timeout, undersize, and oversize as acceptance items.
- **Duplication across three spec trees can drift.** Mitigation: the overlap is deliberate and each tree is self-consistent; any future reconciliation is a separate change rather than an edit during implementation.

## Migration Plan

Migration is incremental and reversible, in eight phases with explicit gates; the ordered task breakdown is in `tasks.md`.

0. Record and verify the live baseline, and prove both current paths can restore.
1. Make both paths observable, before anything structural changes.
2. Protect the source protocol with the root-owned exporter and managed authorization.
3. Replace mutable NAS execution with digest-pinned fixed containers.
4. Protect local history with snapshots, quotas, and separated maintenance windows.
5. Separate the guest from the backup network.
6. Establish independent off-site recovery.
7. Retire the Compose deployment only after the rollback window and all earlier gates pass.

## Open Questions

Deployment parameters to confirm from the target systems before implementation is finalized:

- Synology model, DSM and Container Manager versions, CPU architecture, and the absolute Docker CLI path.
- Whether UID/GID 65532 are unused and supported by the relevant DSM ACL path.
- Btrfs, immutable-snapshot, encrypted-shared-folder, encrypted-volume, and remote-KMIP support; whether a removable USB key store is acceptable; and whether manual unlock after reboot is acceptable now that the NAS also hosts the PBS VM.
- Whether the Restic repository and PBS datastore share one volume, and whether NAS snapshots are currently enabled on either share.
- Protected-snapshot schedule and retention for both shares, the quota headroom snapshot retention requires, and the alert thresholds against DSM shared-folder usage.
- Whether a second Synology NAS or an off-site target is available, and whether it is better used for PBS sync or as a KMIP key server; the off-site provider and mechanism for each path, and whether both paths share one.
- Monitoring destination and named on-call owner for both paths.
- Expected and maximum export size, runtime, and bandwidth; required `/tmp` size from observed Restic workload; maximum complete-backup duration and SSH liveness intervals.
- Hermes runtime account and application executable paths; the source backup account's exact data-read permissions.
- Trusted Ansible controller, inventory ownership, and privilege-escalation method.
- Public versus policy-mandated private GHCR visibility.
- Whether the LAN can be segmented for NFS and management traffic, and when the guest moves to its own VLAN or bridge.
- Confirmation that deployed image retention meets the required detection window, and acceptable maintenance, migration, and rollback windows.

These choices may strengthen the design but must not change the prohibition on placing repository credentials, deletion authority, or backup control on Hermes, nor the prohibition on passing secret values through container environment variables.
