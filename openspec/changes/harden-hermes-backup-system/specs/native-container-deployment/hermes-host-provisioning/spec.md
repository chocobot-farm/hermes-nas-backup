## Purpose

Defines the automated, idempotent provisioning of the Hermes host's backup protocol: an Ansible playbook that installs the root-owned forced-command exporter, manages only its own block of SSH authorizations, and verifies the result without ever running a full export through the controller.

## ADDED Requirements

### Requirement: Provisioning runs from a trusted controller with a validated interface

Hermes-host setup SHALL be performed by a repository-supplied Ansible playbook invoked from a trusted administrative workstation with an equivalent of:

```bash
ansible-playbook -i INVENTORY ansible/hermes-host.yml
```

The playbook SHALL use privilege escalation only for the narrowly scoped tasks that require root. Inventory files SHALL NOT contain plaintext login passwords, sudo passwords, private SSH keys, or NAS secrets. Operator authentication SHOULD use an existing administrative SSH key plus interactive privilege escalation or another separately approved mechanism. The supported `ansible-core` version range SHALL be documented and tested; any non-core collection or role SHALL be declared in a requirements file and pinned to an exact reviewed version. The controller SHALL verify the Hermes SSH host key rather than enabling host-key checking bypasses.

The playbook interface SHALL define and validate at least:

| Variable | Purpose |
| --- | --- |
| `hermes_backup_account` | Existing account under which the forced command runs |
| `hermes_backup_authorized_keys` | List of public keys and their permitted NAS source IP or CIDR |
| `hermes_backup_hermes_bin` | Absolute path to the Hermes application executable |
| `hermes_backup_runtime_parent` | Private temporary-export parent owned by the runtime account |
| `hermes_backup_state` | `present` for installation or `absent` for managed removal |

The playbook SHALL fail before mutation when a required variable is missing, a path is not absolute, the runtime account does not exist, an application executable is unavailable to that account, a source restriction is malformed, or a supplied key is not an accepted public-key type.

#### Scenario: Invalid input stops before mutation

- **WHEN** a required variable is missing, a path is relative, the runtime account is absent, a source restriction is malformed, or a supplied key is not an accepted public-key type
- **THEN** the playbook fails before changing anything on the host

#### Scenario: NAS private key is never an input

- **WHEN** the playbook is run
- **THEN** only the NAS public key is supplied to it, and no NAS secret or private key appears in the inventory

#### Scenario: Package prerequisites without owning the application

- **WHEN** the playbook prepares the host
- **THEN** it installs or verifies ordinary operating-system packages such as Bash and TAR through the host package manager, and does not install, upgrade, or take ownership of Hermes itself

### Requirement: The installed source-side layout is root-owned

The initial installed layout SHALL be equivalent to:

```text
/usr/local/libexec/hermes-backup/                 root:root 0755
/usr/local/libexec/hermes-backup/export           root:root 0755
RUNTIME_ACCOUNT_HOME/.cache/hermes-backup/        runtime account 0700
```

The exporter, every protocol helper, and all of their parent directories SHALL be root-owned and not writable by the Hermes runtime account or unrelated users. The playbook SHALL install them from the same identified source revision as the deployment documentation. Updates SHALL be atomic and SHOULD retain the prior managed files long enough for rollback.

The forced command SHALL use fixed absolute paths for the Hermes executable, any protocol helper, the temporary parent, and operating-system tools; SHALL NOT accept environment-variable overrides for executable or helper paths; SHALL reject a nonempty `SSH_ORIGINAL_COMMAND`; SHALL send diagnostics to stderr and reserve stdout for the TAR stream; SHALL create each temporary directory with mode `0700` beneath the managed runtime parent; SHALL clean temporary content on normal exit and handled signals; and SHALL return nonzero when any application export, consistency check, or TAR stream operation fails.

#### Scenario: Runtime account cannot rewrite the protocol

- **WHEN** the Hermes runtime account attempts to modify the exporter, a helper, or a parent directory
- **THEN** the modification is refused

#### Scenario: Update and rollback

- **WHEN** the installed exporter is updated
- **THEN** the replacement is atomic and the prior managed files remain available for rollback

#### Scenario: Application executable stays under its own lifecycle

- **WHEN** the Hermes application is updated by its normal mechanism
- **THEN** the root-owned wrapper continues to fix how it is invoked without being elevated into ownership of the application, and the NAS-side source bounds remain required

### Requirement: SSH authorization is managed as a marked block

For every active NAS public key the playbook SHALL manage an entry equivalent to:

```text
from="NAS_SOURCE_IP_OR_CIDR",restrict,command="/usr/local/libexec/hermes-backup/export" ssh-ed25519 PUBLIC_KEY synology-hermes-backup
```

The playbook SHALL manage only a clearly marked Hermes-backup block in the account's `authorized_keys`, preserving unrelated administrator keys; SHALL set the account's `.ssh` directory to `0700` and `authorized_keys` to `0600` with correct ownership; SHALL ensure each managed public-key blob occurs exactly once; SHALL remove a legacy entry for the same key that invokes an exporter from a user-writable path before the deployment is accepted; SHALL permit two distinct restricted keys temporarily during rotation; and when `hermes_backup_state=absent` SHALL remove only its managed entries and root-owned exporter files, deleting the runtime parent only when it is empty and never deleting Hermes application data.

The public-key restriction is the authorization boundary. The playbook SHALL NOT grant this key shell, PTY, forwarding, agent forwarding, X11, user-RC, arbitrary command, sudo, or unrelated application access.

#### Scenario: Unrelated keys survive

- **WHEN** the playbook applies or removes its managed block
- **THEN** unrelated administrator keys in `authorized_keys` are unchanged

#### Scenario: Legacy user-writable authorization is removed

- **WHEN** a managed key still carries a legacy authorization pointing at a user-writable exporter
- **THEN** that entry is removed before the deployment is accepted

#### Scenario: Managed removal

- **WHEN** the playbook runs with `hermes_backup_state=absent`
- **THEN** its managed authorizations and root-owned files are removed, the runtime parent is deleted only if empty, and Hermes application data is untouched

### Requirement: Provisioning is idempotent and self-verifying

The playbook SHALL support repeatable normal execution and Ansible check mode. After a successful application, a second run with unchanged inputs SHALL report no changes. It SHALL verify at least:

1. installed file and parent-directory ownership and modes;
2. runtime-account execute access to the configured application paths;
3. runtime-parent ownership, mode, and temporary-file creation;
4. SSH daemon configuration syntax;
5. exactly one managed authorization per configured public key;
6. rejection of a supplied original SSH command without emitting backup data to stdout; and
7. absence of the legacy user-writable forced-command entry for each managed key.

The playbook SHALL NOT run a full export automatically, because that could emit substantial sensitive data through the Ansible controller; the end-to-end export is performed only through the NAS backup container.

#### Scenario: Second run is a no-op

- **WHEN** the playbook is applied again with unchanged inputs
- **THEN** it reports no changes

#### Scenario: Check mode

- **WHEN** the playbook runs in Ansible check mode against a provisioned host
- **THEN** it completes without reporting an unexpected mutation requirement

#### Scenario: No export through the controller

- **WHEN** verification runs
- **THEN** it confirms that a supplied original command is rejected without emitting backup data, and no full export is streamed through the Ansible controller
