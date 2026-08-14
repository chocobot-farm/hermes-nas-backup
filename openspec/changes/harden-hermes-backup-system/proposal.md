## Why

Hermes is an Internet-connected AI agent that must be treated as a potentially compromised data source, yet the deployed backup system on `main` still runs scheduled root jobs from a mutable Git checkout on the NAS, keeps every recovery point on one Synology NAS, bounds nothing about the guest-produced export, and reports nothing about its own health. A security review of the deployment (2026-07-16, revised 2026-07-27) found one critical, six high, and nine medium findings; none of its Priority 1–4 recommendations have been applied. Both backup paths fail closed and both fail quietly, so the system's compliance with every recovery-point objective is currently unobserved rather than demonstrated.

## What Changes

This change folds the three proposed backup specifications and the security review into OpenSpec as one change with three parallel spec sets. All three sets are proposed; none is presented here as accepted or superseded.

- **Cold machine path** — Proxmox VE performs an out-of-guest, client-side-encrypted, stop-mode whole-VM backup to a Proxmox Backup Server whose datastore is a NAS-backed NFS export; guest isolation, TLS-fingerprint pinning, least-privilege tokens, and non-overlapping maintenance become normative.
- **Live application path** — the user-writable `/home/anton/.local/bin/hermes-backup-stream` forced-command entrypoint is replaced by a root-owned exporter under `/usr/local/libexec/hermes-backup/`, installed by an idempotent Ansible playbook, with a versioned TAR stream contract. **BREAKING** for the current `authorized_keys` entry and exporter path.
- **NAS client** — scheduled `docker-compose run` from a Git checkout is replaced by fixed, stopped, digest-pinned one-shot containers (`backup`, `check`, `prune`) started by DSM Task Scheduler. **BREAKING** for the deployed Compose workflow and its DSM tasks.
- **Hostile-output bounds** — whole-run timeout, minimum/maximum byte bounds, inline SHA-256/length accounting, and a per-run manifest emitted for failed attempts as well as successful ones.
- **Deletion resistance** — protected/immutable Btrfs snapshots over both repository shares, quotas driven by DSM shared-folder accounting, and at least one encrypted copy in a failure domain independent of the Synology NAS.
- **Observability** — authoritative backup monitoring outside Hermes, PVE, PBS, and the NAS, alerting on missed schedules rather than only on reported failures.
- **Restore safety** — isolated, non-concurrent VM restores and untrusted-content application restores, with scheduled drills.
- **Alternative architecture** — the dedicated-backup-gateway design (bounded spool, append-only REST endpoint, maintenance-workstation-only deletion, dual independent repositories) is carried in full as its own proposed spec set alongside the other two.
- Long-form narrative in `docs/` is removed; OpenSpec becomes the home for these requirements and the README points at it.

## Capabilities

### New Capabilities

Unified system-level proposal:

- `unified-backup-safety/cold-vm-backup`: PVE/PBS stop-mode image backup, guest isolation, client-side encryption, NAS-backed datastore, and PBS maintenance windows.
- `unified-backup-safety/live-application-export`: restricted SSH authorization, root-owned installed exporter, and the TAR stream contract.
- `unified-backup-safety/nas-restic-client`: digest-pinned one-shot Synology containers, secret handling, bounded hostile-output backup behavior, run manifest, retention, check, and prune.
- `unified-backup-safety/deletion-resistance`: protected NAS snapshots, capacity accounting, and an independent off-site failure domain.
- `unified-backup-safety/backup-monitoring`: out-of-guest monitoring, alert conditions, operator-answerable questions, and log hygiene.
- `unified-backup-safety/restore-safety`: isolated VM restore, untrusted application restore, and drill cadence.
- `unified-backup-safety/operational-lifecycle`: upgrade, rollback, credential rotation, incident response, and initial service objectives.

Native Synology container deployment proposal:

- `native-container-deployment/image-publication`: GHCR registry policy, release identifiers, workflow permissions, pipeline stages, and supply-chain requirements.
- `native-container-deployment/image-runtime-contract`: fixed UID/GID, read-only filesystem contract, entrypoint modes, and container privileges.
- `native-container-deployment/configuration-and-secrets`: approved environment variables, encrypted secret storage, integrity-sensitive configuration, repository storage, and recovery copies.
- `native-container-deployment/hermes-host-provisioning`: Ansible playbook interface, installed source-side layout, managed SSH authorization, and idempotence verification.
- `native-container-deployment/container-topology-and-scheduling`: daily/check/prune container definitions, creation method, and DSM Task Scheduler model.
- `native-container-deployment/lifecycle-and-operations`: image update, rollback, credential rotation, and monitoring/operations for the container deployment.

Hostile-source architecture proposal:

- `hostile-source-architecture/source-trust-model`: hostile-tenant assumption, trust boundaries, and the Hermes source contract.
- `hostile-source-architecture/network-policy`: default-deny matrix for the target architecture and for the deployed topology.
- `hostile-source-architecture/backup-gateway`: gateway isolation, deployment integrity, sandboxed runtime identity, secrets, bounded spool, and dual-target backup behavior.
- `hostile-source-architecture/storage-and-offsite`: NAS-as-storage-service role, append-only REST server, local immutability, maintenance access, and off-site repository requirements.
- `hostile-source-architecture/hypervisor-backup`: hypervisor-controlled image backup requirements and their deployment state.
- `hostile-source-architecture/retention-and-maintenance`: separation of write and delete authority, time-window retention, and destructive-maintenance procedure.
- `hostile-source-architecture/monitoring-and-restore`: gateway/storage monitoring, availability objectives, and hostile-content restore security.
- `hostile-source-architecture/incident-response`: pull-key and repository-key rotation plus per-component compromise procedures.

### Modified Capabilities

None. `openspec/specs/` is currently empty, so every capability above is new.

## Impact

- **Repository**: `docs/*.md` removed in favour of these artifacts; README pointer updated. Implementation deliverables include an Ansible playbook, a GHCR release workflow, `docker create` templates, a run-manifest schema, and runbooks — none of which exist yet.
- **Deployed code**: `docker-compose.yaml`, `setup-nas.sh`, and `server/hermes-backup-stream` are all affected; `backup.sh` gains timeout, byte-bound, digest, and manifest behavior.
- **Infrastructure**: PVE, PBS, DSM Task Scheduler, Synology shared folders and ACLs, NFS export, guest VLAN, and an as-yet-unselected off-site target and monitoring destination.
- **Migration constraint**: the deployed PVE/PBS and Synology paths must both remain operational until their replacements pass backup and restore acceptance tests; the existing Restic repository must not be reinitialized.
