## Purpose

Defines the default-deny network policy that makes the guest's inability to reach backup infrastructure a property of routing rather than only of credentials, for both the target architecture and the flat-LAN topology currently in place.

## ADDED Requirements

### Requirement: Network policy defaults to deny

Network policy SHALL default to deny and implement at least this matrix:

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

Hermes MAY retain its required Internet access on a separate application network that has no route to the backup, storage, or management networks. DNS names alone SHALL NOT be treated as a security boundary. The Hermes guest SHALL NOT be able to reach the hypervisor management address, backup server, NAS repository endpoint, gateway, monitoring control plane, or off-site repository endpoint.

#### Scenario: Guest attempts to reach a backup component

- **WHEN** the guest opens a connection to any backup, storage, or management endpoint
- **THEN** the connection is refused by routing or firewall policy, not only by authentication

#### Scenario: Guest retains its own Internet access

- **WHEN** Hermes performs its normal outbound work
- **THEN** it succeeds from the application network, which holds no route to the backup, storage, or management networks

### Requirement: The deployed flat topology is corrected to an explicit matrix

The deployment currently places the PVE management interface, the Hermes guest, DSM, and the PBS VM on one subnet with no segmentation, so the guest is stopped by authentication rather than by routing. The concrete target for the deployment SHALL be:

| Source | Destination | Permitted service | Requirement |
| --- | --- | --- | --- |
| PVE host | PBS VM | HTTPS on the PBS API port | Required |
| PBS VM | NAS NFS export | NFSv4.1 from the single reserved PBS address | Required |
| Synology backup client | Hermes guest | SSH on the configured port | Required |
| Admin workstation or VPN | PVE and PBS management ports | HTTPS | Required |
| PVE, PBS, NAS | Monitoring endpoint | HTTPS or approved protocol | Required |
| Hermes guest | Its own Internet services | Outbound only | Required |
| Hermes guest | PVE management, PBS, DSM, NFS export, or NAS repository | None | Explicitly denied |
| Public Internet | PVE, PBS, DSM, guest SSH, or the Hermes dashboard | None | Explicitly denied |

Hermes SHOULD be moved to its own VLAN or bridge with outbound Internet access and no route to the management or storage subnets.

#### Scenario: One direction is preserved

- **WHEN** the guest is separated onto its own VLAN or bridge
- **THEN** the Synology client can still reach the guest's SSH port and the guest gains no return path to the NAS

#### Scenario: Both paths still complete after separation

- **WHEN** network separation is applied
- **THEN** one complete cycle on each backup path and one restore drill succeed afterwards

### Requirement: The NFS export stays restricted to one reserved address

The NFS export SHALL remain restricted to the single reserved PBS VM address. Because `AUTH_SYS` trusts client-supplied numeric identities, that address restriction and the network boundary are the export's only real authorization. Widening the export to the LAN SHALL NOT be done, and NFS SHOULD additionally be constrained at the network layer by a dedicated backup VLAN or firewall rules. Kerberos-secured NFS (`sec=krb5i` or `krb5p`) SHOULD be used if both ends support it; otherwise the deployment SHALL document that the export is protected by network position alone. Non-privileged ports and subfolder traversal SHALL NOT be enabled unless demonstrably required.

#### Scenario: Export widening is proposed

- **WHEN** widening the export beyond the reserved PBS address is proposed
- **THEN** it is refused, because it would grant any host on the LAN write and delete access to the image datastore

#### Scenario: Address spoofing on the storage network

- **WHEN** a host occupies or spoofs the reserved PBS address
- **THEN** the residual exposure is recorded as an integrity and availability problem — chunks can be deleted, truncated, or corrupted but not read — and protected snapshots are relied on as the control the NFS client cannot reach

#### Scenario: Reserved address is preserved

- **WHEN** addressing is changed on the storage network
- **THEN** the PBS VM address remains reserved so the export rule cannot silently begin matching a different host
