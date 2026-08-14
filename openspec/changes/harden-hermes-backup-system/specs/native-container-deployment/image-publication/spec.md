## Purpose

Defines how the backup container image is built, identified, and distributed outside the NAS, so that deploying it is a review of a specific immutable artifact rather than an execution of whatever a Git working tree currently contains.

## ADDED Requirements

### Requirement: The image is published to GHCR with a documented visibility decision

The canonical image name SHOULD be `ghcr.io/OWNER/hermes-nas-backup`. The GHCR package SHOULD be public, because the image contains no private data and public visibility lets the NAS pull anonymously without a long-lived registry credential.

If policy requires a private image, the NAS SHALL use a dedicated classic personal access token with only `read:packages`; the token SHALL NOT hold `repo`, `write:packages`, or `delete:packages`; the token SHALL be treated as another NAS secret and rotated independently; registry authentication SHALL use password standard input rather than a command-line argument; and the operator SHALL determine how DSM stores registry credentials before accepting the residual risk.

#### Scenario: Public package needs no NAS credential

- **WHEN** the NAS pulls the published image
- **THEN** the pull succeeds without any stored registry credential

#### Scenario: Private package exception

- **WHEN** policy mandates a private package
- **THEN** a `read:packages`-only token is used, supplied on standard input, rotated independently, and the DSM credential-storage behavior is documented as accepted residual risk

### Requirement: Every release publishes a version, a commit tag, and a digest

Every release SHALL publish at least:

```text
ghcr.io/OWNER/hermes-nas-backup:vX.Y.Z
ghcr.io/OWNER/hermes-nas-backup:sha-GIT_COMMIT
ghcr.io/OWNER/hermes-nas-backup@sha256:IMAGE_DIGEST
```

The digest is the deployment identity; tags are discovery and human-facing labels only. Deployment SHALL NOT automatically follow `latest`. The workflow MAY publish `latest` for convenience, but NAS deployment documentation and commands SHALL use a digest.

#### Scenario: Deployment command references a digest

- **WHEN** a deployment command or record is reviewed
- **THEN** it names the image by `sha256:` digest

#### Scenario: A moving tag changes

- **WHEN** `latest` is repointed to a new image
- **THEN** no NAS container is replaced as a result

### Requirement: Publication is authorized, minimally permissioned, and pinned

Publishing SHALL occur only from an explicitly authorized release event, such as a protected semantic-version tag or an approved GitHub release. The publishing job SHALL declare minimum permissions:

```yaml
permissions:
  contents: read
  packages: write
  attestations: write
  id-token: write
```

The workflow SHALL publish using the repository-scoped `GITHUB_TOKEN` and SHALL NOT use a personal access token for normal image publication. All third-party and GitHub Actions SHALL be pinned to full commit SHAs; mutable major-version tags alone are insufficient.

#### Scenario: Publication from an unauthorized event

- **WHEN** a push occurs outside an authorized release event
- **THEN** no image is published

#### Scenario: Action pinning is verified

- **WHEN** the release workflow is reviewed
- **THEN** every action reference is a full commit SHA

### Requirement: The release pipeline completes defined stages before publishing

The pipeline SHALL complete these stages before publishing a deployable image:

1. check out the exact release commit;
2. run the repository test suite;
3. run ShellCheck against shell scripts;
4. build the requested platform image or multi-platform manifest;
5. run a vulnerability scan and fail according to the documented severity policy;
6. generate an SBOM;
7. push version and commit tags;
8. capture the pushed manifest digest;
9. generate a GitHub artifact attestation for the pushed digest; and
10. publish release notes containing the source commit, image digest, supported platforms, configuration-schema version, and known migration requirements.

The pipeline SHOULD also run a smoke test against the final image rather than only testing files in the checkout.

#### Scenario: Vulnerability policy fails

- **WHEN** the scan finds a finding above the documented severity policy
- **THEN** publication stops and no deployable image is produced

#### Scenario: Attestation verifies against source

- **WHEN** the published digest's attestation is verified
- **THEN** it matches the expected source repository and commit

### Requirement: Supply-chain inputs are pinned and free of secrets

The Dockerfile base image SHALL be pinned by digest, and automated dependency tooling SHOULD propose reviewed digest updates regularly. The image SHALL include OCI labels for at least the source repository, source revision, semantic version, build creation time, and license where applicable. No GitHub token, build credential, repository secret, SSH key, Restic password, or private test material SHALL be copied into an image layer or included in build arguments or persistent build environment variables.

#### Scenario: Image history contains no credential

- **WHEN** the published image's layer history and metadata are inspected
- **THEN** no secret or build credential is present

#### Scenario: Base image is pinned

- **WHEN** the Dockerfile is reviewed
- **THEN** its base image is referenced by digest
