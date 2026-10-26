# KEP-1466: SLSA build provenance for Kueue release artifacts

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Terminology](#terminology)
  - [User Stories](#user-stories)
    - [Story 1: A cluster admin cannot tell which commit built the Kueue image they are running](#story-1-a-cluster-admin-cannot-tell-which-commit-built-the-kueue-image-they-are-running)
    - [Story 2: A platform team wants to reject Kueue images without build provenance at admission time](#story-2-a-platform-team-wants-to-reject-kueue-images-without-build-provenance-at-admission-time)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [A Sigstore outage can delay staging builds and releases](#a-sigstore-outage-can-delay-staging-builds-and-releases)
    - [Anyone who can act as the build identity can sign false provenance](#anyone-who-can-act-as-the-build-identity-can-sign-false-provenance)
    - [An unattested artifact can be promoted during the transition period](#an-unattested-artifact-can-be-promoted-during-the-transition-period)
    - [Consumers may assume L1 provenance gives stronger guarantees than it does](#consumers-may-assume-l1-provenance-gives-stronger-guarantees-than-it-does)
- [Design Details](#design-details)
  - [Attested artifacts](#attested-artifacts)
  - [Build identity](#build-identity)
  - [Attestation step](#attestation-step)
  - [Provenance content](#provenance-content)
  - [Failure handling](#failure-handling)
  - [Promotion](#promotion)
  - [Release candidates](#release-candidates)
  - [Changes in other repositories](#changes-in-other-repositories)
  - [Interactions](#interactions)
  - [Test Plan](#test-plan)
    - [Prerequisite testing updates](#prerequisite-testing-updates)
    - [Unit tests](#unit-tests)
    - [Integration tests](#integration-tests)
    - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha (v0.21)](#alpha-v021)
    - [Beta (v0.22)](#beta-v022)
    - [Stable (v0.23)](#stable-v023)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Future work ideas](#future-work-ideas)
- [Appendix: SLSA, Sigstore, and the image promoter](#appendix-slsa-sigstore-and-the-image-promoter)
<!-- /toc -->

## Summary

This KEP adds signed SLSA v1 build provenance to Kueue's staging images and Helm charts.
Consumers can then verify which source commit and build produced the artifacts promoted to the production registry.

## Motivation

Staging artifacts carry no build provenance today.
Promoted images carry a promotion signature, but no record of the build that produced them.
A promotion signature answers "was this artifact promoted through the official pipeline?".
It does not answer "which commit built it?".
The image promoter can carry build provenance to the production registry, but only when the staging artifact already carries it.
Release consumers (cluster admins, platform teams, and security auditors) cannot verify the build origin of Kueue images.

### Goals

- Every promoted image and Helm chart carries a Sigstore-signed SLSA v1 build provenance attestation on staging.
- The attestation survives promotion to the production registry.
- Release candidates are covered by the same mechanism.
- Consumers can verify provenance with a documented signing identity and issuer.
- A failed attestation step fails the build visibly.

### Non-Goals

- Provenance for GitHub release assets (`kubectl-kueue` binaries, rendered manifests, raw Helm chart tarballs). These are future work.
- SLSA Build Level 2 or higher.
- Changes to Kueue's Kubernetes APIs, configuration, or controller behavior.
- Attesting images that are not promoted to the production registry, such as periodic build images.
- Retroactive attestation of past releases.

## Proposal

After each image and chart push, the staging build signs a SLSA v1 build provenance attestation as an OCI referrer.
The signature uses the build's own workload identity.
The image promoter verifies the attestation when present and carries it to the production registry, next to the promotion signature.
Consumers verify the attestation against the build identity and the public Sigstore transparency log.

### Terminology

- **SLSA** (Supply-chain Levels for Software Artifacts): a framework that defines levels of build provenance assurance. This KEP targets SLSA Build L1, which requires a signed provenance statement from the build system.
- **Sigstore**: an open-source project providing tools and infrastructure for signing and verifying software artifacts using short-lived certificates and a public transparency log.
- **Staging artifact**: an image or Helm chart published to the staging Artifact Registry by a postsubmit build.
- **Promoted artifact**: a staging artifact copied to the production registry by the image promoter.
- **Build provenance**: a signed statement of which source, build system, and invocation produced an artifact.
- **Promotion signature**: a signature added by the image promoter at copy time. It attests the copy, not the build.
- **Build identity** (`kueue-sa`): the service account that runs the staging build. Its email address is the Sigstore signing identity.
- **OCI referrer**: an artifact attached to an existing image or chart via the OCI referrers API, allowing tools to discover signatures, attestations, and SBOMs associated with a digest without modifying the original artifact.
- **cosign**: a Sigstore tool for signing and verifying container images and other OCI artifacts using short-lived certificates and a public transparency log.
- **Fulcio / Rekor**: Sigstore components. Fulcio issues short-lived signing certificates tied to a workload identity (OIDC token). Rekor is the public transparency log where certificates and signatures are recorded and made auditable.

### User Stories

#### Story 1: A cluster admin cannot tell which commit built the Kueue image they are running

An admin wants to confirm that a running Kueue image was built from a known commit.
They verify the attestation of the image digest and read the source commit from the predicate:

```bash
cosign verify-attestation \
  --type slsaprovenance1 \
  --certificate-identity kueue-sa@k8s-staging-images.iam.gserviceaccount.com \
  --certificate-oidc-issuer https://accounts.google.com \
  registry.k8s.io/kueue/kueue@sha256:<digest>
```

The predicate records the source and the commit:

```json
{
  "buildDefinition": {
    "externalParameters": {
      "source": {
        "uri": "git+https://github.com/kubernetes-sigs/kueue@refs/tags/v0.21.0",
        "digest": { "gitCommit": "<commit-sha>" }
      }
    }
  },
  "runDetails": {
    "builder": { "id": "https://cloudbuild.googleapis.com/projects/k8s-staging-images/serviceAccounts/kueue-sa@k8s-staging-images.iam.gserviceaccount.com/cloudbuild.yaml" }
  }
}
```

#### Story 2: A platform team wants to reject Kueue images without build provenance at admission time

A platform team runs a policy controller in their clusters.
They admit Kueue images only when a provenance attestation from the Kueue build identity is present:

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: kueue-provenance
spec:
  images:
    - glob: "registry.k8s.io/kueue/**"
  authorities:
    - keyless:
        identities:
          - issuer: https://accounts.google.com
            subject: kueue-sa@k8s-staging-images.iam.gserviceaccount.com
      attestations:
        - name: build-provenance
          predicateType: slsaprovenance1
```

### Notes/Constraints/Caveats

- The level achieved is SLSA Build L1.
  The build writes its own provenance predicate, and build steps are not isolated from each other.
- Attestations attach to the multi-arch index digest that the image promoter references.
  Platform-specific manifests do not carry their own attestation.

### Risks and Mitigations

#### A Sigstore outage can delay staging builds and releases

A staging build cannot sign or verify while Sigstore is unavailable.
Rerun the postsubmit after the outage.
Until the promoter policy requires provenance, a missing attestation delays the attestation but does not block promotion.

#### Anyone who can act as the build identity can sign false provenance

Access to the service account is limited to staging project administrators and the Cloud Build pipeline.
Consumers must pin both the signing identity and the issuer when they verify.
Consumers must not treat L1 provenance as tamper-proof.

#### An unattested artifact can be promoted during the transition period

The promoter policy starts in warn mode, so an unattested artifact can still be promoted.
Until the policy requires provenance, the in-build self-check is the enforcement point.

#### Consumers may assume L1 provenance gives stronger guarantees than it does

The consumer documentation, tracked separately, states the SLSA level and its limits.

## Design Details

This KEP changes no Kueue API, configuration, or controller behavior.

### Attested artifacts

| Artifact | Type | Phase |
|---|---|---|
| kueue | Image | 1 (this KEP) |
| kueueviz-backend | Image | 1 (this KEP) |
| kueueviz-frontend | Image | 1 (this KEP) |
| kueue-populator | Image | 1 (this KEP) |
| kueue-priority-booster | Image | 1 (this KEP) |
| kueue chart | Helm chart | 1 (this KEP) |
| kueue-populator chart | Helm chart | 1 (this KEP) |
| kueue-priority-booster chart | Helm chart | 1 (this KEP) |
| Periodic build images (debug, importer, ray-mini) | Image | Not attested |
| GitHub release assets | Binary and manifest | Future work |

### Build identity

The staging build runs as a dedicated service account, `kueue-sa`, in the staging project.
The account already has Artifact Registry write access and OIDC token-creator permissions.
The build obtains a Fulcio identity certificate from its ambient Google identity, so it needs no extra credentials.
Consumers verify against the account's email address and the Google OIDC issuer.

### Attestation step

1. After the image and chart pushes complete, resolve each artifact's digest from its tag.
2. Write a SLSA v1 predicate that records the source repository, git ref, git commit, builder identity, and build invocation ID.
3. Sign the predicate as a Sigstore bundle OCI referrer, using the build's ambient workload identity.
4. Verify the attestation against the same digest and identity as an in-build self-check.
   Fail the build if verification fails.

A local dry-run mode writes the predicate without signing.

### Provenance content

The predicate has this shape. Values in angle brackets vary per build:

```json
{
  "buildDefinition": {
    "buildType": "https://github.com/kubernetes-sigs/kueue/RELEASE.md#staging-image-provenance-v1",
    "externalParameters": {
      "source": {
        "uri": "git+https://github.com/kubernetes-sigs/kueue@refs/tags/v0.21.0",
        "digest": { "gitCommit": "<commit-sha>" }
      }
    },
    "resolvedDependencies": [
      { "uri": "git+https://github.com/kubernetes-sigs/kueue", "digest": { "gitCommit": "<commit-sha>" } }
    ]
  },
  "runDetails": {
    "builder": { "id": "https://cloudbuild.googleapis.com/projects/k8s-staging-images/serviceAccounts/kueue-sa@k8s-staging-images.iam.gserviceaccount.com/cloudbuild.yaml" },
    "metadata": { "invocationId": "https://console.cloud.google.com/cloud-build/builds/<build-id>?project=<project-number>" }
  }
}
```

### Failure handling

A failed attestation or self-check step fails the build with a non-zero exit code.
The artifact stays in the staging registry without an attestation.
Rerunning the postsubmit signs the artifact again.
The earlier push stays in the registry, but the promoter does not promote it, because promotion acts only on digests that the current run references.

### Promotion

The image promoter verifies the build attestation when present and carries it to the production registry.
The promotion signature is added after the copy and is unaffected.
The promoter policy for Kueue starts in `warn` mode.
After one full release cycle, the policy moves to `require`.

### Release candidates

Release candidate tags trigger the same postsubmit build.
Release candidates are attested without extra configuration.
Consumers verify release candidate images the same way as release images.

### Changes in other repositories

| Repository | Change | Required for |
|---|---|---|
| kubernetes/test-infra | Submit the postsubmit build as `kueue-sa` | Signing with the Kueue-specific identity |
| kubernetes/k8s.io | Add a Kueue provenance policy in `warn` mode to the promoter manifest | Carrying attestations to production |
| kubernetes/k8s.io | Change the policy to `require` after one release | Stable graduation |

These changes are not part of this KEP's implementation PR.
They are tracked as external follow-ups.

### Interactions

- SBOM and OpenVEX release steps: unchanged. They run in a separate GitHub Actions workflow.
- Promotion signature: unchanged. It is added at copy time and exists alongside the build provenance.
- Periodic builds: not covered, because those images are not promoted.
- Consumer documentation and the release checklist: tracked by the docs sub-issue.
- GitHub release asset provenance: future work.

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

#### Prerequisite testing updates

None.

#### Unit tests

Predicate generation is tested offline.
The local dry-run mode writes the predicate JSON without signing, so it can be checked without Cloud Build access.

#### Integration tests

Not applicable. This KEP adds no controller code.

#### e2e tests

Not applicable for automated CI. Coverage comes from:

1. The in-build self-check.
2. A manual `cosign verify-attestation` on the first staging digest after merge.
3. A check on the first promoted release candidate.

### Graduation Criteria

#### Alpha (v0.21)

- The staging postsubmit attests all 8 artifacts (5 images and 3 Helm charts) on every build.
- The in-build self-check passes.
- The local dry-run mode produces the predicate without signing.
- The kubernetes/test-infra change that submits the build as `kueue-sa` has landed.

#### Beta (v0.22)

- One full release and at least one release candidate have verifiable provenance on the production registry.
- Consumer documentation and the release checklist update have merged.
- The promoter policy for Kueue is in `warn` mode.

#### Stable (v0.23)

- The promoter policy for Kueue requires provenance.
- All provenance-related issues filed during v0.21 and v0.22 have been resolved or explicitly deferred with a tracking issue.
- Phase 2 scope (GitHub release assets) is decided.

## Implementation History

- 2026-10-26: Initial KEP draft.

## Drawbacks

- The build depends at runtime on the public Sigstore infrastructure (Fulcio and Rekor). A Sigstore outage can delay builds.
- Rekor entries are public. Build metadata (source URI, commit, and invocation ID) becomes visible to anyone.
- Each release branch needs one more build step to maintain.

## Alternatives

**Cloud Build native provenance.**
Cloud Build produces a KMS-signed attestation stored in Artifact Analysis, not as a Sigstore OCI referrer.
The image promoter does not read Artifact Analysis records, so the attestation would not reach the production registry.

**goreleaser.**
It replaces the image build tooling instead of adding to it.
This is out of scope.

**tejolote.**
It is an external observer that wraps build steps.
It adds operational complexity.
At Build L1, the simpler in-build pattern is sufficient.

**Building and attesting images in GitHub Actions.**
Kueue's images are built by Cloud Build, not GitHub Actions.
This would duplicate the multi-arch build in a different environment.

**Relying on the promotion signature only.**
The promotion signature confirms the copy path, not the build.
It cannot show which source commit produced the image.

**Attesting by tag under the default build identity.**
The default build identity is shared by all projects in the staging GCP organization, not only Kueue.
A tag-based attestation is also not reproducible when the tag is reused across builds of the same commit.
It would also use SLSA v0.2 predicates, which are superseded.

## Future work ideas

- Phase 2: provenance for GitHub release assets (`kubectl-kueue` binaries, rendered manifests, and Helm chart tarballs), using GitHub's native attestation API.
- SLSA Build L2 and L3, through hermetic builds or a dedicated signing service.
- Attesting periodic build images (debug, importer, ray-mini).
- Publishing SBOMs as OCI referrers next to the provenance attestations.

## Appendix: SLSA, Sigstore, and the image promoter

SLSA (Supply-chain Levels for Software Artifacts) defines build levels.
Build L1 requires a signed provenance statement from the build system.
Build L2 also requires that a hosted build system generates the provenance, isolated from user-controlled code.
Kueue's staging Cloud Build meets L1.
The build steps write and sign the predicate with the build's own service account.

Sigstore keyless signing uses Fulcio and Rekor (see Terminology above), instead of a long-lived key.
Consumers verify by checking the certificate identity and issuer, without holding a key.

The image promoter copies images from staging to the production registry.
It verifies the attestations that are present and forwards them to the destination as OCI referrers.
A per-project provenance policy in the promoter manifest sets whether missing attestations produce a warning or a failure.
