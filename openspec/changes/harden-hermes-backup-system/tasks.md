## 0. Record and verify the live baseline

- [ ] 0.1 Record current NAS paths, schedules, image ID, Restic repository, stable host/tag, and SSH fingerprints, without recording secret values.
- [ ] 0.2 Run the existing test suite.
- [ ] 0.3 Restore a current Restic snapshot and validate its Hermes ZIP.
- [ ] 0.4 Verify the latest cold PBS backup and restore it with networking disconnected.
- [ ] 0.5 Back up the PVE AES key, Restic password, and Synology recovery material to two protected external locations.
- [ ] 0.6 Confirm the Restic pull window does not overlap the PVE stop-mode backup or the guest restart.
- [ ] 0.7 **Gate**: both current recovery paths have successful restore evidence.

## 1. Make both paths observable

- [ ] 1.1 Stand up a monitoring destination outside Hermes, PVE, PBS, and the NAS, with a named on-call owner.
- [ ] 1.2 Report PVE backup, PBS verification, prune, and garbage-collection outcomes to it.
- [ ] 1.3 Report Synology task outcomes and the Restic client's exit status to it.
- [ ] 1.4 Alert on a missed schedule on either path, not only on a reported failure.
- [ ] 1.5 Alert on newest-snapshot age for both repositories, not only on task exit status.
- [ ] 1.6 Alert on Synology shared-folder usage crossing its threshold, using DSM accounting rather than PBS-reported free space.
- [ ] 1.7 Enable failure notifications on PVE, PBS, and DSM Task Scheduler.
- [ ] 1.8 Deliberately fail one job on each path and confirm the alert arrives.
- [ ] 1.9 **Gate**: an operator can state when each path last succeeded without logging into PVE, PBS, or the NAS, and a suppressed schedule raises an alert within the detection objective.

## 2. Protect the source protocol

- [ ] 2.1 Write the idempotent Hermes-host Ansible playbook, example inventory, and variable reference, with the validation that fails before mutation on bad input.
- [ ] 2.2 Write the versioned exporter stream contract (`RESTORE.txt` + `hermes/hermes.zip`, stdout reserved for TAR, stderr for diagnostics).
- [ ] 2.3 Install the root-owned exporter under `/usr/local/libexec/hermes-backup/` with the managed runtime parent.
- [ ] 2.4 Manage the marked `authorized_keys` block with `from=`, `restrict`, and the forced absolute command, preserving unrelated keys.
- [ ] 2.5 Add a second restricted key entry if key rotation is required.
- [ ] 2.6 Verify idempotence: second run with unchanged inputs reports no changes; check mode completes without unexpected mutation.
- [ ] 2.7 Verify rejection of shell, PTY, forwarding, user RC, and a supplied `SSH_ORIGINAL_COMMAND`, with no backup data on stdout.
- [ ] 2.8 Test a complete backup through the existing NAS client.
- [ ] 2.9 Remove the legacy authorization that invokes the user-writable exporter and confirm its absence.
- [ ] 2.10 **Gate**: arbitrary commands are rejected, the exporter path is not runtime-user-writable, and a producer failure creates no Restic snapshot.

## 3. Bound and account for hostile output

- [ ] 3.1 Enforce a whole-run timeout covering SSH, stream transfer, and Restic finalization.
- [ ] 3.2 Enforce minimum and maximum source byte counts without writing a plaintext spool to persistent storage.
- [ ] 3.3 Distinguish an oversized stream from one exactly at the maximum; never accept silent truncation.
- [ ] 3.4 Compute the SHA-256 digest and byte length on the NAS over the bytes actually received, inline, without buffering to a plaintext file or suppressing the producer's exit status.
- [ ] 3.5 Define the versioned run-manifest schema and emit one record per attempt, including failed attempts.
- [ ] 3.6 Retain manifests outside the Restic repository for at least the snapshot retention period, with no secret values and no per-run Restic tag.
- [ ] 3.7 Add failure tests: exporter failure, timeout, undersize, oversize, exactly-at-maximum, and overlapping runs.
- [ ] 3.8 Add a test that a restored snapshot re-hashes to the digest its manifest recorded.

## 4. Replace mutable NAS execution

- [ ] 4.1 Add the protected GHCR release workflow with test, ShellCheck, build, vulnerability policy, SBOM, digest capture, attestation, and release notes.
- [ ] 4.2 Pin all workflow actions to full commit SHAs and the base image to a digest; add OCI labels.
- [ ] 4.3 Publish the first attested image and record its digest.
- [ ] 4.4 Confirm UID/GID 65532 are unused on the NAS.
- [ ] 4.5 Create the dedicated encrypted secret share and choose its unlock profile; create the root-owned `known_hosts` configuration location.
- [ ] 4.6 Create or select the separate Restic repository share; apply and verify DSM ACLs plus numeric POSIX ownership and modes on all three locations.
- [ ] 4.7 Verify the Hermes SSH host key out of band and install `known_hosts` root-owned and non-writable by the runtime identity.
- [ ] 4.8 Pull the image by digest and verify its attestation, source repository, and revision.
- [ ] 4.9 Write and apply the native `docker create` templates for daily, check, prune, init, and snapshots.
- [ ] 4.10 Disable the old Compose-based DSM tasks before testing the new containers against the existing repository.
- [ ] 4.11 Confirm the existing repository is recognized and not reinitialized.
- [ ] 4.12 Run snapshots, check, backup, and a representative restore test.
- [ ] 4.13 Record the absolute Docker CLI path and create DSM tasks that start containers by fixed name.
- [ ] 4.14 Validate DSM failure propagation and abnormal-task notification, adding an explicit wait-and-return-exit-code wrapper if `start --attach` does not propagate status.
- [ ] 4.15 Enable fixed-container schedules and observe at least two successful cycles.
- [ ] 4.16 **Gate**: no scheduled root task evaluates the Git checkout, Compose file, build context, or user-writable script.

## 5. Protect local history

- [ ] 5.1 Enable and verify protected Btrfs snapshots for the PBS datastore and Restic repository shares.
- [ ] 5.2 Set quotas on both shares sized for retention plus snapshot overhead, and alert at roughly 80% using DSM shared-folder accounting.
- [ ] 5.3 Separate backup, verification, garbage-collection, check, prune, and snapshot schedules, and alert on overlap.
- [ ] 5.4 Confirm the datastore share keeps the recycle bin and redundant compression disabled.
- [ ] 5.5 Constrain NFS at the network layer in addition to the DSM export rule, keeping the PBS address reserved.
- [ ] 5.6 **Gate**: a tested protected snapshot survives an attempted ordinary repository deletion and remains restorable.

## 6. Separate the guest from the backup network

- [ ] 6.1 Move the Hermes guest to its own VLAN or bridge with outbound Internet access and no route to PVE management, PBS, DSM, or NFS endpoints.
- [ ] 6.2 Preserve the Synology client's reach to the guest's SSH port, with no return path from the guest to the NAS.
- [ ] 6.3 Confirm from the guest that PVE management, PBS, DSM, the NFS export, and the Restic repository endpoint are unreachable.
- [ ] 6.4 Confirm from the guest that Hermes retains the Internet access its own operation requires.
- [ ] 6.5 Run one complete cycle on each backup path and one restore drill after the change.
- [ ] 6.6 **Gate**: isolation is enforced by routing as well as by credentials, and both paths still complete.

## 7. Establish independent recovery

- [ ] 7.1 Configure an encrypted off-site PBS sync or copy job in a different administrative and physical failure domain.
- [ ] 7.2 Ensure replication credentials cannot delete protected remote history.
- [ ] 7.3 Test a copy, verify it remotely, and perform an off-site-only isolated restore using the saved AES key on a rebuilt PVE host.
- [ ] 7.4 Optionally add append-only off-site application backups with independent transport credentials and no repository password at the remote service.
- [ ] 7.5 **Gate**: complete Hermes VM recovery succeeds without the production Synology NAS.

## 8. Retire the Compose deployment

- [ ] 8.1 Remove the old DSM tasks.
- [ ] 8.2 Remove obsolete containers and the local build cache.
- [ ] 8.3 Revoke obsolete SSH keys and confirm the old key is rejected.
- [ ] 8.4 Remove old plaintext secret copies.
- [ ] 8.5 Retain the old repository unchanged until its approved retention end, without reinitializing it.
- [ ] 8.6 Remove the NAS Git checkout from every privileged execution path.

## 9. Operational deliverables

- [ ] 9.1 Write the Synology encrypted-secret, ACL, UID/GID, scheduler, quota, and protected-snapshot instructions.
- [ ] 9.2 Write the PVE/PBS schedule, encryption-key recovery, verification, and isolated-restore instructions.
- [ ] 9.3 Write the off-site PBS copy and off-site-only recovery instructions.
- [ ] 9.4 Write the monitoring and alert verification instructions.
- [ ] 9.5 Write the guest network separation instructions, including the reachability tests that prove isolation without breaking the Synology pull.
- [ ] 9.6 Write the secret-free recovery card template covering component addresses, datastore name, NFS export and mount path, where the PBS TLS fingerprint and token identity are recorded, where each recovery key copy is held, and the restore isolation procedure.
- [ ] 9.7 Write the upgrade, rollback, credential rotation, incident response, and migration runbooks, and assign named owners.
- [ ] 9.8 Write the restore-test instructions and schedule the drill cadence: monthly canary, quarterly representative application restore, annual off-site-only disaster-recovery exercise.
- [ ] 9.9 Build the recovery register listing every recovery secret, where its two independent external copies live, and how to use it.
- [ ] 9.10 Replace initial byte, duration, and quota limits with measured values plus documented headroom.

## 10. Surrounding system hardening

- [ ] 10.1 DSM: disable the default `admin` account, enforce administrator MFA, enable Auto Block and Account Protection, restrict service exposure with the DSM and network firewalls, install updates promptly, run Security Advisor regularly, and alert on failed tasks and suspicious logins.
- [ ] 10.2 PVE/PBS: restrict 8006 and 8007 to an admin LAN or VPN and never port-forward them, stay on supported release channels, keep retention on PBS, keep the AES key off PBS, and repeat isolated restore drills after major architecture changes.
- [ ] 10.3 Hermes VM: use a dedicated export key, retain `from=`, `restrict`, and the forced absolute command, keep the exporter root-owned and integrity-protected, restrict inbound SSH to the NAS where practical, do not expose dashboard or gateway ports directly, and monitor use of the backup key.

## 11. Acceptance

- [ ] 11.1 Cold recovery checklist passes: `TASK OK`; PBS holds ciphertext without the AES key; the backup token has no datastore administration authority; the TLS fingerprint is pinned and the storage reports active; verification reports zero errors; the datastore path is the mounted export; the export is restricted to the reserved PBS address; the PBS VM is excluded from the backup job; restore to a new VMID boots with networking disconnected; production and restored clones are never online concurrently; the preserved AES key has been tested in an isolated recovery procedure.
- [ ] 11.2 Live application recovery checklist passes: the root-owned forced command rejects shell, PTY, forwarding, and arbitrary commands; Hermes export validation succeeds; exporter failure, timeout, undersize, and oversize create no snapshot; every attempt including a deliberately failed one produces a retained manifest; a restored snapshot re-hashes to its recorded digest; scheduled containers are digest-pinned and do not execute the Git checkout; check and prune containers have no SSH key mount; a representative Hermes restore succeeds in isolation.
- [ ] 11.3 Storage and operations checklist passes: both repositories have protected NAS snapshots; quota and free-space alerts use DSM shared-folder accounting; an encrypted whole-VM copy exists outside the Synology failure domain; off-site-only restore succeeds; two protected external copies exist for every recovery key; monitoring detects missed schedules and deliberate failures; an operator can state when each path last succeeded without logging into PVE, PBS, or the NAS; the guest cannot route to backup infrastructure; both paths and a restore drill still succeed after network separation; the guest returns and reconnects its messaging gateway after a backup-induced stop; a secret-free recovery card exists and has been used in a drill; upgrade, rollback, rotation, and incident procedures have named owners.
- [ ] 11.4 CI and image checklist passes: tests and lint pass before publication; the required platform image exists; deployment records reference a digest; the base image is digest-pinned; workflow actions are SHA-pinned; the workflow uses minimum `GITHUB_TOKEN` permissions; the vulnerability policy passes; an SBOM is available; the attestation verifies against the expected source repository and commit; image history contains no secret or build credential.
