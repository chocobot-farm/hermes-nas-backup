## Purpose

Defines how recovery points are restored without reinfecting production or letting two credential-identical Hermes instances act at once, and how often restores must actually be exercised.

## ADDED Requirements

### Requirement: A restored VM boots isolated

A restored Hermes VM SHALL use a new unused VMID and SHALL have **Start after restore** and **Live restore** disabled. Before first boot the operator SHALL disable autostart, disconnect the virtual NIC (`link_down=1` or equivalent), ensure no route to the Internet, production, NAS, PVE/PBS management, or backup networks, and validate through the PVE console.

#### Scenario: First boot with no network path

- **WHEN** a restored VM boots for the first time
- **THEN** its NIC is disconnected, autostart is disabled, and it has no route to the Internet, production, NAS, or management networks

#### Scenario: Restored image of a compromised guest

- **WHEN** the restored image may contain a compromise
- **THEN** offline inspection is completed and an operator explicitly approves before any production connectivity is granted

### Requirement: Production and a restored clone never run online concurrently

Production and a restored clone SHALL NOT run online concurrently, because they contain duplicated Hermes, OAuth, gateway, and messaging credentials. To test service functionality the operator SHALL stop production first, enable only the restored guest, complete the test, stop and disconnect the restore, restart production, and verify production reconnection before deleting the test VM.

#### Scenario: Functional test of a restored clone

- **WHEN** a restored clone must be tested against real messaging and integrations
- **THEN** production is stopped first, the clone is tested, the clone is stopped and disconnected, production is restarted and confirmed running, and only then is the test VM deleted

#### Scenario: MAC change is not accepted as separation

- **WHEN** a proposal relies on a changed MAC address to run both instances online
- **THEN** it is refused, because duplicated gateway tokens, OAuth grants, host keys, and sessions are the hazard

### Requirement: Application restores treat repository content as hostile

Restic content originated in the guest and SHALL be treated as untrusted. Restores SHALL run into a disposable isolated environment as an unprivileged identity. Before import the procedure SHALL reject absolute paths and path traversal, prevent device creation, setuid/setgid restoration, capabilities, and unsafe ownership, handle symlinks without permitting writes outside the restore root, inspect the TAR and embedded Hermes ZIP, and keep networking disabled until inspection is complete.

#### Scenario: Archive contains a traversal path

- **WHEN** the restored TAR contains an absolute path or a `..` traversal
- **THEN** the entry is rejected and nothing is written outside the restore root

#### Scenario: Restore target is disposable and isolated

- **WHEN** an application restore is performed
- **THEN** it imports into a disposable isolated environment under an unprivileged identity, never into the running production guest

### Requirement: Restores are exercised on a defined cadence

A representative application restore SHALL be performed at least quarterly. Off-site-only recovery using protected key copies SHALL be exercised at least annually. A monthly lightweight canary restore is recommended. A repository check alone SHALL NOT be treated as a substitute for restoring and validating representative Hermes content.

#### Scenario: Quarterly application drill

- **WHEN** a quarter passes without a representative application restore
- **THEN** the missed drill is reported to operators

#### Scenario: Annual off-site-only exercise

- **WHEN** the annual disaster-recovery exercise runs
- **THEN** it restores using only off-site data and the protected key copies, including uploading the preserved AES key to a rebuilt PVE host rather than reusing the running one
