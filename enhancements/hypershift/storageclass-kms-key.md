---
title: storageclass-kms-key
authors:
  - "@devguyio"
reviewers:
  - "@csrwng, for HyperShift architecture and API design"
  - "@muraee, for HyperShift AWS platform and propagation chain"
  - "@celebdor, for HyperShift platform and storage integration"
approvers:
  - "@csrwng"
  - "@enxebre"
api-approvers:
  - "@enxebre"
creation-date: 2026-06-09
last-updated: 2026-06-15
status: provisional
tracking-link:
  - https://issues.redhat.com/browse/OCPSTRAT-1679
see-also:
  - "/enhancements/storage/aws-ebs-csi-driver-sts.md"
replaces: []
superseded-by: []
---

# StorageClass KMS Key for AWS Hosted Control Planes

## Summary

This enhancement adds an optional `storageKMSKeyARN` field to the AWS platform spec
of the HyperShift `HostedCluster` API. When set, the Hosted Cluster Config Operator
(HCCO) propagates the key ARN to the `ClusterCSIDriver` resource in the guest cluster,
causing the cluster-storage-operator to configure the default StorageClass to encrypt
new EBS volumes with the customer-specified AWS KMS key. This closes a parity gap
between ROSA classic and ROSA Hosted Control Planes (HCP) for storage encryption.

## Motivation

ROSA classic clusters allow customers to configure KMS encryption for volumes created
by the default StorageClass. ROSA HCP clusters cannot currently expose this capability,
even though the underlying CSI operator already supports it via
`ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN`. HyperShift simply does not
expose the field in the `HostedCluster` API or propagate it to the guest cluster.

Self-managed HyperShift on AWS has the same gap: operators cannot configure default
StorageClass encryption on day 1 or change it on day 2.

Related prior work: [openshift/enhancements#1163](https://github.com/openshift/enhancements/pull/1163)
(AWS EBS CSI driver KMS support), which established the CSI-level mechanism this
enhancement builds on.

### User Stories

- As a ROSA HCP cluster administrator, I want to specify a KMS key ARN when creating
  a cluster so that all PVCs provisioned by the default StorageClass are encrypted with
  my organization's key, not AWS-managed encryption, so that I can meet my organization's
  data-at-rest encryption requirements.

- As a ROSA HCP cluster administrator, I want to update the KMS key ARN on a running
  cluster (day-2 operation) so that new PVCs use an updated or rotated KMS key without
  requiring cluster recreation.

- As a self-managed HyperShift operator on AWS, I want to specify a KMS key for the
  default StorageClass at cluster creation time via the `hypershift` CLI so that I can
  enforce encryption standards from day 1.

- As a cluster operations engineer, I want a dedicated status condition on the
  `HostedCluster` that reports whether the configured KMS key is valid and accessible,
  so that I can detect and remediate KMS permission issues before they affect application
  workloads.

### Goals

- Add `storageKMSKeyARN` to `AWSPlatformSpec` (shared by `HostedCluster` and
  `HostedControlPlane`) as an optional, mutable string field.
- Propagate the field from `HostedCluster` → `HostedControlPlane` → `ClusterCSIDriver`
  in the guest cluster.
- Report key validity via a dedicated `ValidAWSStorageKMSConfig` condition on the
  `HostedCluster` status.
- Expose `--storage-volumes-kms-key` in the `hcp create cluster aws` command and the
  `hypershift create cluster aws` developer CLI.
- Apply identically to ROSA HCP and self-managed HyperShift on AWS.
- Preserve backward compatibility: clusters without the field continue to use
  AWS-managed encryption with no behavioral change.

### Non-Goals

- **ROSA CLI (`rosa create cluster`) and Hybrid Cloud Console integration** are out of
  scope for this enhancement. ROSA service layer tooling integration is a downstream
  concern handled by the ROSA product team and will be tracked separately after the
  upstream API is established.
- **Per-StorageClass KMS key granularity.** This enhancement targets only the default
  StorageClass managed by the cluster-storage-operator. The CSI operator's
  `ClusterCSIDriver.DriverConfig` applies to the driver globally; per-StorageClass
  granularity would require CSI operator changes, which are out of scope.
- **Re-encrypting existing PVs.** Updating `storageKMSKeyARN` changes the key used for
  new PVCs only. Existing volumes retain their original encryption; re-encryption of
  existing volumes requires a customer-managed migration.
- **NodePool root volume encryption.** Root volume encryption is configured separately
  on `NodePool.spec.platform.aws.rootVolume.encryptionKey` and is not affected by this
  enhancement.
- **Etcd encryption.** Etcd KMS key management is handled by the existing
  `ValidAWSKMSConfig` mechanism using a dedicated `AWSKMSRoleARN` role. This
  enhancement does not affect that code path.

## Proposal

When a customer sets `spec.platform.aws.storageKMSKeyARN` on a `HostedCluster`, the
following propagation chain executes automatically:

1. The HC controller mirrors the field to `HostedControlPlane.spec.platform.aws.storageKMSKeyARN`,
   following the existing mirroring pattern for other `AWSPlatformSpec` fields.
2. The HCCO validates the key by assuming the `StorageARN` role (already provisioned for
   storage credential management) and calling AWS KMS `Encrypt` with a test payload.
3. The HCCO sets the `ValidAWSStorageKMSConfig` condition on `HostedControlPlane` status;
   the HC controller bubbles it to `HostedCluster` status.
4. The HCCO storage reconciliation writes `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN`
   to the guest cluster via its guest cluster client. The cluster-storage-operator (CSO)
   reads this value and configures the default StorageClass with the KMS key ID.
5. New EBS volumes provisioned via the StorageClass carry the KMS encryption.

When the field is cleared, the HCCO clears `DriverConfig.AWS.KMSKeyARN`, the CSO reverts
the default StorageClass to AWS-managed encryption, and the condition transitions to
`Unknown` (key not configured).

### Workflow Description

**Actors:**
- **Cluster administrator:** A human operator who creates or manages `HostedCluster`
  resources (ROSA HCP customer or self-managed HyperShift operator).
- **HC controller:** The HyperShift HostedCluster controller running on the management
  cluster.
- **HCCO:** The Hosted Cluster Config Operator (Hosted Control Plane Operator subsystem)
  running in the HCP namespace on the management cluster.
- **CSO:** The cluster-storage-operator, running in the guest cluster.

**Day-1 (cluster creation with KMS key):**

1. The cluster administrator runs:
   ```bash
   hcp create cluster aws \
     --storage-volumes-kms-key arn:aws:kms:us-east-1:123456789012:key/mrk-abc123 \
     ...
   ```
   or sets `spec.platform.aws.storageKMSKeyARN` directly on the `HostedCluster` manifest.
2. The HC controller creates the `HostedControlPlane` with the field mirrored.
3. The HCCO validates the key:
   - If the `StorageARN` role can assume and KMS `Encrypt` succeeds → condition `True/AsExpected`.
   - If role assumption fails → condition `False/InvalidIAMRole`.
   - If KMS call fails (key disabled, deleted, or wrong region) → condition `False/AWSError`.
4. The HCCO writes `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` in the guest cluster.
5. The CSO configures the default StorageClass with `parameters.kmsKeyId` set to the ARN.
6. New PVCs created from the default StorageClass produce EBS volumes encrypted with the
   customer's key.

**Day-2 (key rotation):**

1. The administrator updates `HostedCluster.spec.platform.aws.storageKMSKeyARN` to a new
   ARN.
2. The HC controller updates the `HostedControlPlane` field; the HCCO re-validates and
   updates `ClusterCSIDriver`.
3. The CSO updates the default StorageClass. There is a brief period between CSO
   reconciliation cycles where the StorageClass references neither the old nor the new
   key. This gap is safe on OpenShift 4.13+ because the CSO update is non-disruptive
   (in-place StorageClass mutation).
4. New PVCs use the new key. Existing PVs retain encryption with the original key. If the
   original key is later disabled in AWS, volumes encrypted with it become inaccessible.
   The administrator is responsible for key lifecycle management.

**Day-2 (key removal):**

1. The administrator sets `storageKMSKeyARN` to empty string.
2. The HCCO clears `DriverConfig.AWS.KMSKeyARN`; the CSO reverts the default StorageClass
   to AWS-managed encryption.
3. The `ValidAWSStorageKMSConfig` condition transitions to `Unknown/StatusUnknown`.

### API Extensions

#### New field: `storageKMSKeyARN` on `AWSPlatformSpec`

Added to the existing `AWSPlatformSpec` struct, which is shared between `HostedCluster`
and `HostedControlPlane`:

```go
// StorageKMSKeyARN is the AWS KMS key ARN used to encrypt PVCs created by
// the default StorageClass in the guest cluster. Both KMS key ARNs
// (arn:...:key/...) and alias ARNs (arn:...:alias/...) are accepted.
// When empty, the default StorageClass uses AWS-managed encryption.
// The StorageARN role in AWSRolesRef must have kms:Encrypt and kms:Decrypt
// permissions on the specified key.
// Updating this field does not re-encrypt existing volumes; only newly
// created PVCs use the new key.
// +optional
StorageKMSKeyARN string `json:"storageKMSKeyARN,omitempty"`
```

The field is validated with a CEL `XValidation` rule accepting both KMS key ARNs and
alias ARNs:

```text
^arn:(aws|aws-cn|aws-us-gov):kms:[a-z0-9-]+:[0-9]{12}:(key|alias)/[a-zA-Z0-9:/_-]+$
```

A `MaxLength=2048` marker is applied (matching the downstream `ClusterCSIDriver` CRD
constraint). The regex extends the existing `ValidAWSKMSConfig` etcd key pattern to
include the `alias/` ARN resource type and the broader alias name character set
(`:`, `/`, `_` are valid in alias names).

#### New condition type: `ValidAWSStorageKMSConfig`

A new condition type is added to the HyperShift API, following the established pattern
of `ValidAWSKMSConfig` (etcd encryption):

| Status | Reason | Trigger | Example Message |
|--------|--------|---------|-----------------|
| `Unknown` | `StatusUnknown` | Field not configured | `"Storage KMS is not configured"` |
| `True` | `AsExpected` | Key validated successfully | `"Storage KMS key is valid and accessible"` |
| `False` | `AWSError` | KMS API call failed | `"KMS key arn:aws:kms:us-east-1:123456789012:key/abc-def is not accessible: KMSKeyDisabled. Verify the key is enabled and the StorageARN role has kms:Encrypt permission."` |
| `False` | `InvalidIAMRole` | Role assumption failed | `"Failed to assume StorageARN role for KMS key arn:aws:kms:us-east-1:123456789012:key/abc-def: AccessDenied. Verify the StorageARN role in AWSRolesRef has kms:Encrypt and kms:Decrypt permissions on this key."` |

The condition is registered in `ExpectedHCConditions` with `ConditionUnknown` for clusters
where `storageKMSKeyARN` is not configured.

Error messages include the failing ARN, the AWS error code, and a remediation step so
administrators can diagnose the issue from the condition alone without consulting operator
logs.

#### CLI flag: `--storage-volumes-kms-key`

A new flag is added to the AWS cluster creation command, consistent with the existing
`--root-volume-kms-key` naming convention:

```text
--storage-volumes-kms-key string
    AWS KMS key ARN (arn:...:key/...) or alias ARN (arn:...:alias/...) used
    to encrypt PVCs created by the default StorageClass. If omitted, PVCs use
    AWS-managed encryption. The StorageARN role must have kms:Encrypt and
    kms:Decrypt permissions on the specified key.
```

The flag is bound through the shared options mechanism so it is exposed in both the HCP
CLI (`hcp create cluster aws`) and the developer CLI (`hypershift create cluster aws`)
automatically.

### Topology Considerations

#### Hypershift / Hosted Control Planes

All control plane components — HC controller, HCCO, CSO, CSI driver operators — run on
the management cluster in the HCP namespace. The HCCO uses a dual-client architecture:
a management cluster client for reading `HostedControlPlane` spec, and a guest cluster
client (via an injected guest kubeconfig) for writing `ClusterCSIDriver` and cloud
credential secrets. CSI DaemonSets run on guest cluster worker nodes.

The `StorageARN` role is an IRSA-style (IAM Roles for Service Accounts) role that the
HCCO already provisions for storage credential management (`ebs-cloud-credentials`).
KMS validation reuses this existing IRSA credential acquisition path — no new roles
or permissions are introduced beyond what the feature already requires.

#### Standalone Clusters

Not applicable. This enhancement targets only `HostedCluster` resources managed by
HyperShift.

#### Single-node Deployments or MicroShift

Not applicable.

#### OpenShift Kubernetes Engine (OKE)

Not applicable. Storage KMS configuration in standard OpenShift clusters is handled
via `ClusterCSIDriver` directly.

### Implementation Details/Notes/Constraints

**CEL validation:** The ARN regex must be verified with envtest across Kubernetes API
server versions 1.31–1.35 to ensure consistent CEL validation behavior and ratcheting
compatibility. The `api/CLAUDE.md` and `api/AGENTS.md` files in the HyperShift repository
document the kube-api-linter conventions; the final regex may require adjustment to pass
the linter.

**Alias ARN pass-through:** The CSI operator accepts alias ARNs and passes them verbatim
to AWS at volume creation time. AWS resolves aliases to the underlying key — HyperShift
does not need to resolve aliases itself.

**Validation timing:** The KMS probe runs in the HCCO reconcile loop (not on every API
request), following the same pattern as `ValidAWSKMSConfig` etcd validation in the CPO.
The initial condition is `Unknown`; it transitions to `True` or `False` after the first
reconcile that executes the probe.

**Self-managed AWS:** This design applies identically to both ROSA HCP and self-managed
HyperShift on AWS. The propagation chain and validation mechanism are platform-internal.

**Why HCCO instead of CPO for validation:** HCCO already owns `reconcileStorage`, the
`StorageARN` credential provisioning (`ebs-cloud-credentials`), and the guest cluster
client needed to write `ClusterCSIDriver`. Co-locating KMS validation in HCCO keeps all
storage concerns — reconciliation, credential provisioning, and key validation — in one
operator. The existing `ValidAWSKMSConfig` etcd validation in CPO uses a different role
(`AWSKMSRoleARN`) for a different domain and is not a precedent for placement here.

### Risks and Mitigations

**Risk: StorageClass gap during key rotation.**
During day-2 key rotation, there is a brief window between when the HCCO updates
`ClusterCSIDriver` and when the CSO reconciles the StorageClass. PVCs created during
this window may use AWS-managed encryption. This is safe on OCP 4.13+ where in-place
StorageClass mutations are supported. For clusters on earlier versions, key rotation
should be performed during low-traffic periods.
*Mitigation:* Document the behavior and the OCP version constraint. No code change needed.

**Risk: Key disabled after volumes are encrypted.**
If the customer disables or deletes the KMS key in AWS after volumes are encrypted,
those volumes become inaccessible. HyperShift cannot prevent this.
*Mitigation:* The `ValidAWSStorageKMSConfig` condition transitions to `False/AWSError`
on the next HCCO reconcile, alerting operators. Document the key lifecycle responsibility.

**Risk: IAM role misconfiguration.**
If the `StorageARN` role lacks `kms:Encrypt` / `kms:Decrypt` permissions on the key,
the KMS probe fails and the condition is `False/InvalidIAMRole`.
*Mitigation:* The condition message includes the failing ARN and a remediation hint.
The HCCO still writes the ARN to `ClusterCSIDriver` (validation does not gate
reconciliation, following the established `ValidAWSKMSConfig` pattern), so PVC
provisioning may fail at the CSI driver level until IAM permissions are corrected.

**Risk: Existing cluster behavior change on upgrade.**
Clusters without `storageKMSKeyARN` must not experience any behavior change after upgrading
to a version containing this feature.
*Mitigation:* The field is optional with `omitempty`. The condition is registered as
`Unknown` when the field is absent. The HCCO clears `DriverConfig` when the field is
empty, which is a no-op for clusters that never set it.

### Drawbacks

- Adds one more field to `AWSPlatformSpec`, which is already large. A nested
  `AWSStorageConfig` struct was considered but rejected (see Alternatives) as premature
  abstraction for a single field.
- The active KMS probe adds latency to the HCCO reconcile loop. This follows the
  established `ValidAWSKMSConfig` precedent and is bounded by the HCCO reconcile interval.

## Alternatives (Not Implemented)

**Nested `AWSStorageConfig` struct under `AWSPlatformSpec`:**
A separate nested struct was considered to group all storage-related AWS fields.
Rejected because there is currently only one storage field — nesting adds indirection
with no benefit. A nested struct can be introduced in a follow-on enhancement if
additional storage fields are needed.

**Run validation in CPO (where `ValidAWSKMSConfig` lives):**
Validation could be placed alongside the existing etcd KMS validation in the CPO.
Rejected because CPO uses a different role (`AWSKMSRoleARN`) for etcd encryption, and
storage concerns belong with the storage reconciliation owner (HCCO). Splitting would
increase coordination surface and mix concerns across operators.

**Propagation-confirmation-only validation (no KMS probe):**
Verify only that the ARN was written to `ClusterCSIDriver`, without calling KMS `Encrypt`.
Rejected because this approach cannot detect IAM permission issues until the first volume
creation failure, which is too late for operators to receive actionable feedback.


## Open Questions

None at this time.

## Test Plan

### Envtest (CEL Validation)

YAML-driven envtest cases are mandatory for all CEL validation rules per HyperShift
project convention. Test cases cover `onCreate` and `onUpdate` scenarios across
multiple Kubernetes API server versions to verify ratcheting compatibility:

- Valid KMS key ARN accepted (e.g., `arn:aws:kms:us-east-1:123456789012:key/mrk-abc123`)
- Valid alias ARN accepted (e.g., `arn:aws:kms:us-east-1:123456789012:alias/my-key`)
- Invalid format rejected: missing `arn:` prefix
- Invalid format rejected: wrong partition (e.g., `arn:gcp:kms:...`)
- Invalid format rejected: wrong service (e.g., `arn:aws:s3:...`)
- Invalid format rejected: malformed key ID
- Invalid format rejected: value exceeding `MaxLength=2048`
- Regression: cluster created without `storageKMSKeyARN` is unaffected

### Unit Tests

Unit tests will cover the propagation chain (HC controller mirroring, HCCO storage
reconciliation, condition bubble-up), KMS validation logic (all condition states), and
CLI flag wiring. Specific test cases will be determined during implementation.

### E2E Tests

E2E tests will be added to the HyperShift v2 E2E test suite (`test/e2e/v2/`), which
runs against a pre-existing hosted cluster on live AWS infrastructure. Tests follow the
established v2 lifecycle pattern: mutate the `HostedCluster` spec via the management
client (day-2 operation), verify propagation to the hosted cluster via the hosted
cluster client, and verify the actual AWS resource state via the AWS SDK. The test
reuses the existing CI KMS key (`alias/hypershift-ci`) already provisioned in the
CI AWS account.

Tests run in the `e2e-v2-aws` presubmit and `e2e-aws-ovn` periodic CI jobs, which use
the `hypershift` cluster profile with Boskos-managed AWS account leasing.

Test scenarios:

1. **Day-2 key configuration:**
   - Set `storageKMSKeyARN` on the pre-existing `HostedCluster` via `UpdateObject`
   - Wait for `ValidAWSStorageKMSConfig` condition to become `True/AsExpected`
   - Verify `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` is set in the hosted
     cluster via the hosted cluster client
   - Create a PVC using the default StorageClass, wait for it to bind
   - Verify the resulting EBS volume is encrypted with the specified KMS key via
     `ec2.DescribeVolumes` (following the existing `KMSRootVolumeTest` pattern)
   - Clean up PVC and restore `storageKMSKeyARN` to empty via `DeferCleanup`

2. **Day-2 key removal:**
   - Clear `storageKMSKeyARN` (set to empty)
   - Verify `ValidAWSStorageKMSConfig` transitions to `Unknown/StatusUnknown`
   - Verify `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` is cleared

3. **Regression (no key configured):**
   - On a hosted cluster where `storageKMSKeyARN` was never set, verify
     `ValidAWSStorageKMSConfig` is `Unknown/StatusUnknown` and PVCs use
     AWS-managed encryption

## Graduation Criteria

### Dev Preview -> Tech Preview

Not applicable. This feature targets GA directly — no Dev Preview or Tech Preview
phase is planned. The API surface is a single optional field with CEL validation,
and the propagation chain uses established HyperShift patterns.

### Tech Preview -> GA

Not applicable. See above.

### GA

- Unit test coverage for all propagation and validation paths.
- Envtest coverage for CEL validation across multiple Kubernetes versions.
- E2E test coverage for the full lifecycle (creation, key rotation, key removal).
- `ValidAWSStorageKMSConfig` condition registered in `ExpectedHCConditions`.
- E2E tests passing in CI for at least one release cycle without flakes.
- Upgrade testing completed (field present on upgrade, absent clusters unaffected).
- User documentation merged in `openshift-docs` covering the `--storage-volumes-kms-key`
  flag, day-2 key rotation behavior, and key lifecycle warnings.
- No open blocking bugs.

### Removing a deprecated feature

Not applicable. This enhancement adds new capability; nothing is deprecated.

## Upgrade / Downgrade Strategy

**Upgrade:** The new field is optional with `omitempty`. Existing `HostedCluster` objects
gain the field on upgrade, defaulting to empty. No action is required from customers.
The `ValidAWSStorageKMSConfig` condition is added to `ExpectedHCConditions` with
`ConditionUnknown` for clusters without the field, preserving existing condition
behavior.

**Failed upgrade rollback:** Control plane downgrades are not supported in HyperShift.
If an N→N+1 upgrade fails mid-way, the `storageKMSKeyARN` field is not yet active
and has no effect on storage behavior — existing clusters continue using their current
encryption settings. The CPO reconciliation is idempotent and will complete the upgrade
once the underlying failure is resolved. If `storageKMSKeyARN` was already configured
on a successfully upgraded cluster, the `ClusterCSIDriver` in the hosted cluster retains
its last-written `DriverConfig` throughout any subsequent upgrade attempts.

## Version Skew Strategy

All components in the propagation chain (HC controller, HCCO, CSO) run on the management
cluster in the HCP namespace and are versioned together with the HyperShift release.
There is no multi-version skew within the propagation chain.

The `ClusterCSIDriver` CRD in the guest cluster already includes
`spec.driverConfig.aws.kmsKeyARN` from the AWS EBS CSI driver. HyperShift writes this
field only when `storageKMSKeyARN` is set on the `HostedCluster`, so guest clusters
running older OCP versions that do not recognize the field will ignore it gracefully
(Kubernetes unknown-field pruning applies at admission).

## Operational Aspects of API Extensions

**SLIs for `ValidAWSStorageKMSConfig`:** The condition transition from `Unknown` to `True`
or `False` after cluster creation is the primary health indicator. The condition reaches
its terminal state within one HCCO reconcile interval.

**Impact on existing SLIs:** The HCCO reconcile loop gains one KMS API call per reconcile
when `storageKMSKeyARN` is set. This follows the same accepted pattern as
`ValidAWSKMSConfig` etcd validation in the CPO.

**Failure modes and cluster health impact:**
- If the KMS probe fails (condition `False`), the `ClusterCSIDriver` field is still
  written (the HCCO does not gate reconciliation on condition success). PVC provisioning
  may fail at the CSI driver level if the key is inaccessible.
- If the KMS API is temporarily unavailable, the condition remains at its last value
  until the next reconcile that successfully calls KMS.
- No impact on existing workloads or control plane availability.

## Support Procedures

**Detecting failures:**
- Inspect `HostedCluster.status.conditions` for `ValidAWSStorageKMSConfig`.
- A `False` condition includes the failing ARN, AWS error code, and remediation step
  in `message`.

**Diagnosing IAM issues (`False/InvalidIAMRole`):**
1. Verify the `StorageARN` role in `HostedCluster.spec.platform.aws.rolesRef.storageARN`.
2. Confirm the role's IAM policy includes `kms:Encrypt` and `kms:Decrypt` for the key ARN.
3. Confirm the key policy allows the `StorageARN` role principal.

**Diagnosing KMS access issues (`False/AWSError`):**
1. Confirm the KMS key is enabled in the AWS console / CLI.
2. Confirm the key exists in the correct AWS region (matches the cluster region).
3. For alias ARNs: confirm the alias points to an enabled key.

**Disabling the feature:**
Clear `spec.platform.aws.storageKMSKeyARN` on the `HostedCluster`. The HCCO will clear
`ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` on the next reconcile and the CSO
will revert the default StorageClass to AWS-managed encryption. Existing PVs are
unaffected.

**Graceful failure:** If the HCCO KMS probe errors repeatedly, the condition remains
`False` but control plane and existing workloads are not disrupted. New PVC provisioning
may fail at the CSI driver level if the key is inaccessible.

## Infrastructure Needed

No new subprojects, repositories, or testing infrastructure are required. The E2E tests
run in the existing HyperShift v2 E2E CI jobs (`e2e-v2-aws`, `e2e-aws-ovn`), which
already provision live AWS infrastructure with KMS access via the `alias/hypershift-ci`
key. The `StorageARN` role in the CI account may need `kms:Encrypt` and `kms:Decrypt`
permissions added for this key.
