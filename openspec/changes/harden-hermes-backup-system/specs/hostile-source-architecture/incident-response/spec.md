## Purpose

Defines credential rotation and the per-component compromise procedures for a system where each trusted component holds a different subset of the recovery material, so that a compromise triggers exactly the response its blast radius requires.

## ADDED Requirements

### Requirement: The source pull key rotates without a coverage gap

Rotation SHALL create a new client key, install a second restricted public-key entry on the source, verify the source host key independently, complete a bounded test export and backup, remove the old public-key entry, and securely retire the old private key.

#### Scenario: Rotation is completed

- **WHEN** rotation finishes
- **THEN** a backup with the new key has succeeded, the old entry is absent, and the old private key is retired

### Requirement: Repository key rotation distinguishes password change from key replacement

Normal password rotation SHALL add and test a new repository key before removing the old key. If a decrypted repository master key may have been exposed, the repository SHALL be replaced or copied into a new repository with new master key material; changing only the password SHALL NOT be treated as revoking a leaked master key.

#### Scenario: Master key may be leaked

- **WHEN** exposure of the decrypted master key is suspected
- **THEN** a new repository with new master material is created and verified snapshots are migrated

### Requirement: Suspected source compromise preserves and distrusts

Operators SHALL preserve pre-compromise immutable recovery points, suspend destructive maintenance, distrust all backups created after the earliest credible compromise time, isolate Hermes from backup and management networks, rotate the pull key after rebuilding or remediating the source, restore first into isolation, and compare application-level and image-level recovery points where available.

Source compromise alone SHALL NOT require rotating repository credentials, because the source never receives them; they SHALL be rotated if evidence indicates the gateway or maintenance workstation was also exposed. Credentials that lived inside the guest — messaging-gateway tokens, OAuth grants, model-provider API keys, and the guest's own SSH host keys — SHALL be rotated regardless, and they will also be present in any restored image.

#### Scenario: Identifying the recovery point

- **WHEN** the earliest credible compromise time is established
- **THEN** the newest snapshot predating it is identified as the recovery point, which is why retention must exceed the credible detection delay

#### Scenario: Repository credentials after guest compromise

- **WHEN** only the guest is known to be compromised
- **THEN** repository credentials are not rotated, while guest-held credentials are rotated regardless

### Requirement: Suspected gateway compromise assumes total secret exposure

Operators SHALL assume source confidentiality and all repository encryption keys are compromised. They SHALL revoke transport credentials, disable ingest, rotate the source pull key, preserve immutable storage, create new repositories with new master keys, rebuild the gateway from trusted media, and resume only after an isolated restore and integrity review.

#### Scenario: Gateway rebuild

- **WHEN** the gateway is rebuilt
- **THEN** ingest resumes only after new repositories with new master keys exist and an isolated restore and integrity review have passed

### Requirement: Suspected hypervisor compromise assumes plaintext and key exposure

The hypervisor occupies the gateway's position in the deployed system. Operators SHALL assume that current guest plaintext, the backup-server API token, and the image encryption key are all compromised. They SHALL revoke the API token, preserve existing snapshots, rebuild the hypervisor from verified installation media, and treat every image snapshot created after the earliest credible compromise time as suspect.

Rotating the image encryption key SHALL be understood to protect future snapshots only; it does not withdraw an exposed key from snapshots already written.

#### Scenario: Attacker copied the encryption key

- **WHEN** the image encryption key is assumed copied
- **THEN** the response records that any datastore copy the attacker also obtained remains decryptable to them, and a new key is established for future backups

### Requirement: Suspected backup-server compromise assumes destruction, not disclosure

Operators SHALL assume the datastore can be read as ciphertext, destroyed, or corrupted, but not decrypted, because the backup server holds no image encryption key. They SHALL preserve the underlying shared folder and any snapshots of it before rebuilding, revoke the API token, rebuild the server, reattach the existing datastore path without initializing or erasing it, recreate the user, token, and datastore ACLs, and verify snapshots before trusting them.

#### Scenario: Reattaching after rebuild

- **WHEN** the rebuilt server is attached to the existing datastore path
- **THEN** the datastore is neither initialized nor erased, and integrity is verified without the decryption key before the snapshots are trusted

### Requirement: NAS compromise has a defined response and a stated precondition

Operators SHALL assume the local repository can be destroyed or corrupted but not decrypted unless the gateway or maintenance workstation was also compromised. They SHALL preserve off-site recovery, revoke NAS transport credentials, rebuild the storage service, and repopulate it from a verified source or off-site repository.

Because a single DSM administrator authority reaches the backup-server VM, the NFS export beneath its datastore, and the application repository, both local recovery paths can be destroyed by one attacker with one set of credentials. Until an off-site copy exists, this incident SHALL be recorded as having no recovery procedure — only an inventory of what was lost.

#### Scenario: NAS compromise without an off-site copy

- **WHEN** the NAS is compromised and no off-site copy exists
- **THEN** the response is recorded as loss inventory rather than recovery, which is the stated justification for establishing the independent failure domain

#### Scenario: NAS compromise with an off-site copy

- **WHEN** an off-site copy exists
- **THEN** it is preserved and taken offline, NAS transport credentials are revoked, the storage service is rebuilt, and the repository is repopulated from the verified off-site data
