## Purpose

Defines how the container deployment is updated, rolled back, and rotated without coupling those operations to each other, and what an operator must be able to observe about it day to day.

## ADDED Requirements

### Requirement: Image deployment is deliberate

The NAS SHALL NOT automatically replace a running or stopped container merely because a tag changed. An update procedure SHALL:

1. disable or pause the related scheduled tasks;
2. pull the new image by digest;
3. verify its attestation, source repository, source revision, and digest;
4. create candidate containers with new temporary names and the reviewed settings;
5. run at least `snapshots` and an appropriate check or test backup;
6. preserve the old stopped containers and image during a rollback window;
7. replace or rename containers so scheduled tasks again reference the fixed production names;
8. re-enable scheduling; and
9. record the new deployed digest.

Changing image versions SHALL NOT require copying or re-encoding the secret values.

#### Scenario: Candidate is validated before promotion

- **WHEN** a new digest is deployed
- **THEN** candidate containers pass `snapshots` and a check or test backup before the fixed production names are switched

#### Scenario: Update leaves secrets untouched

- **WHEN** the image version changes
- **THEN** no secret value is copied, re-encoded, or re-entered

### Requirement: Rollback is possible within a defined window

Rollback SHALL be possible by disabling schedules, restoring the prior container names or recreating them from the recorded prior digest, running a validation, and re-enabling schedules. Repository-format compatibility SHALL be reviewed before deploying a Restic version that could write an incompatible format.

#### Scenario: Rollback to the recorded digest

- **WHEN** the new deployment misbehaves inside the rollback window
- **THEN** the prior digest's containers are restored, validated, and rescheduled

#### Scenario: Format migration blocks rollback

- **WHEN** a Restic version would migrate the repository format
- **THEN** the review happens before deployment, because a code rollback cannot undo the migration

### Requirement: Credential rotation is independent of image deployment

SSH rotation SHALL create a new key in the encrypted share; add its public key to `hermes_backup_authorized_keys` alongside the old key and apply the Hermes-host playbook; create or update a candidate daily container to mount the new key; test a complete backup; remove the old key from `hermes_backup_authorized_keys`, apply the playbook, and verify its authorization is absent; and securely retire the old private key and update recovery records.

Restic rotation SHALL create a new strong password; add and test a new Restic key; update the mounted password file atomically; test snapshots, check, and restore access; remove the old Restic key when appropriate; and update offline recovery copies. Suspected access to decrypted repository contents SHALL require migration to a new repository with new key material.

#### Scenario: Rotation without an image change

- **WHEN** either credential is rotated
- **THEN** the deployed image digest is unchanged

#### Scenario: Old authorization is confirmed gone

- **WHEN** SSH rotation completes
- **THEN** the old key's authorization is verified absent and a backup with the new key succeeds

### Requirement: Operators can observe the deployment's state

DSM SHALL notify operators when any scheduled container exits abnormally. Operators SHALL be able to determine the last start and finish time, the deployed image digest, the container exit code, the last successful Restic snapshot ID, the current repository check status, the last prune result, immutable snapshot status, and second-copy status.

Operators SHALL be alerted when a run exceeds its duration or byte limit and when repository capacity approaches the configured quota. Initial limits SHALL be replaced with values derived from observed successful exports plus documented headroom; raising a limit after an alert requires review rather than an automatic retry with no bound.

#### Scenario: Limit is exceeded

- **WHEN** a run exceeds its duration or byte limit
- **THEN** operators are alerted, and any subsequent increase of the limit is a reviewed decision rather than an automatic retry

#### Scenario: Deployment state is answerable

- **WHEN** an operator is asked what is deployed and when it last worked
- **THEN** the image digest, last exit code, last snapshot ID, check and prune results, snapshot protection status, and second-copy status are all available

### Requirement: Logs are bounded and free of secret material

Logs SHALL NOT contain the Restic password, private-key content, registry tokens, Synology recovery keys, or complete environment or container inspection output sent to untrusted destinations. Container logs SHOULD have a bounded retention policy, and backup completion and failure logs SHOULD use UTC timestamps.

#### Scenario: Log review

- **WHEN** container logs and any exported diagnostics are reviewed
- **THEN** they contain no secret value or full inspection dump, and use UTC timestamps

### Requirement: Restores are tested rather than inferred from checks

The restore procedure SHALL be tested periodically. A repository check alone SHALL NOT be treated as a substitute for restoring and validating representative Hermes content.

#### Scenario: Check passes but restore is untested

- **WHEN** only repository checks have been run over a review period
- **THEN** the deployment is not considered verified until a representative restore has succeeded and its contents validated
