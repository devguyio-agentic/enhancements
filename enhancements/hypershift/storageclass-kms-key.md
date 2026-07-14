---
title: storageclass-kms-key
authors:
  - "@devguyio"
reviewers:
  - "@csrwng, for HyperShift architecture and API design"
  - "@muraee, for HyperShift AWS platform and propagation chain"
  - "@celebdor, for HyperShift platform and storage integration"
  - "@jsafrane, for storage operator and ClusterCSIDriver"
  - "@joshbranham, for managed services and ROSA integration"
  - "@JoelSpeed, for API conventions and review"
  - "@everettraven, for API conventions and review"
approvers:
  - "@csrwng"
  - "@enxebre"
api-approvers:
  - "@enxebre"
creation-date: 2026-06-09
last-updated: 2026-07-13
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

This enhancement adds an optional `kmsKeyARN` field under
`spec.operatorConfiguration.aws.csiDriverConfig` on the HyperShift `HostedCluster`
API. When set at cluster creation time, the Hosted Cluster Config Operator (HCCO)
propagates the key ARN to the `ClusterCSIDriver` resource in the guest cluster,
causing the cluster-storage-operator to configure the default StorageClass to encrypt
new EBS volumes with the customer-specified AWS KMS key. This closes a parity gap
between ROSA classic and ROSA Hosted Control Planes (HCP) for storage encryption.

The field is a **day-1 knob**: it configures the initial default StorageClass
encryption at cluster creation. Day-2 changes to storage encryption are made
directly on the `ClusterCSIDriver` resource in the guest cluster by the cluster
administrator, following the same pattern as the default ingress controller
configuration.

## Motivation

ROSA classic clusters allow customers to configure KMS encryption for volumes created
by the default StorageClass via `rosa create cluster --kms-key-arn`. ROSA HCP clusters
cannot currently expose this capability, even though the underlying CSI operator
already supports it via `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN`
(established in [openshift/enhancements#1163](https://github.com/openshift/enhancements/pull/1163)).
HyperShift simply does not expose a creation-time knob in the `HostedCluster` API or
propagate it to the guest cluster.

Self-managed HyperShift on AWS has the same gap: operators cannot configure default
StorageClass encryption at cluster creation time.

**IAM policy readiness:** Self-managed HyperShift clusters use inline IAM policies
(defined in `cmd/infra/aws/iam.go`) that already include the required KMS actions —
encryption works out of the box. ROSA HCP clusters use the AWS-managed
`ROSAAmazonEBSCSIDriverOperatorPolicy`, which currently has zero KMS actions. A
managed policy change request to add `kms:Decrypt`,
`kms:GenerateDataKeyWithoutPlaintext`, and `kms:CreateGrant` (conditioned with
`kms:ViaService: ec2.*.amazonaws.com`) is being submitted to AWS. ROSA HCP GA for
this feature depends on AWS approving that managed policy update.

### User Stories

- As a ROSA HCP cluster administrator, I want to specify a KMS key ARN when creating
  a cluster so that all PVCs provisioned by the default StorageClass are encrypted with
  my organization's key, not AWS-managed encryption, so that I can meet my organization's
  data-at-rest encryption requirements.

- As a ROSA HCP cluster administrator, I want to update the KMS encryption
  configuration on a running cluster by editing the `ClusterCSIDriver` resource
  directly in the guest cluster, so that I can rotate keys or change encryption
  settings without recreating the cluster.

- As a self-managed HyperShift operator or HyperShift developer on AWS, I want to
  specify a KMS key for the default StorageClass at cluster creation time via the
  `hcp` or `hypershift` CLI so that I can enforce encryption standards from day 1.

- As a cluster operations engineer, I want a dedicated status condition on the
  `HostedCluster` that reports whether the initially configured KMS key is valid and
  accessible, so that I can detect and remediate KMS permission issues during cluster
  provisioning.

### Goals

- Add an optional `kmsKeyARN` field under
  `spec.operatorConfiguration.aws.csiDriverConfig` as a string field accepting
  KMS key ARNs and alias ARNs.
- Propagate the field at cluster creation from `HostedCluster` →
  `HostedControlPlane` → `ClusterCSIDriver` in the guest cluster (write-once,
  not continuously reconciled).
- Report key validity via a dedicated `ValidAWSStorageKMSConfig` condition on the
  `HostedCluster` status at creation time.
- Expose `--storage-volumes-kms-key` in the `hcp create cluster aws` command and the
  `hypershift create cluster aws` developer CLI.
- Apply identically to ROSA HCP and self-managed HyperShift on AWS.
- Preserve backward compatibility: clusters without the field continue to use
  AWS-managed encryption with no behavioral change.
- Preserve existing `ClusterCSIDriver` editability: cluster administrators can
  continue to modify `ClusterCSIDriver` directly in the guest cluster for day-2
  configuration changes, and the HCCO will not overwrite those changes.

### Non-Goals

- **ROSA CLI (`rosa create cluster`), Terraform, CAPI, and Hybrid Cloud Console
  integration** are out of scope for this enhancement. These are downstream concerns
  tracked separately by the ROSA product team, which consumes the upstream
  `HostedCluster` API to wire it into their clients.
- **Per-StorageClass KMS key granularity.** This enhancement targets only the default
  StorageClass managed by the cluster-storage-operator. Users can still create their
  own StorageClasses with their own KMS keys directly in the guest cluster; this
  enhancement affects only the operator-managed default StorageClass. The CSI
  operator's `ClusterCSIDriver.DriverConfig` applies to the driver globally;
  per-StorageClass granularity would require CSI operator changes, which are out of
  scope.
- **Day-2 key management via the HostedCluster API.** Day-2 key rotation and removal
  are performed directly on the `ClusterCSIDriver` resource in the guest cluster by
  the cluster administrator. The `HostedCluster` field captures day-1 intent only.
  This follows the same pattern as the default ingress controller: HyperShift sets it
  up at creation, day-2 is up to the cluster admin.
- **Re-encrypting existing PVs.** The KMS key applies to newly created PVCs only.
  Existing volumes retain their original encryption.
- **NodePool root volume encryption.** Root volume encryption is configured separately
  on `NodePool.spec.platform.aws.rootVolume.encryptionKey` and is not affected by this
  enhancement.
- **Etcd encryption.** Etcd KMS key management is handled by the existing
  `ValidAWSKMSConfig` mechanism using a dedicated `AWSKMSRoleARN` role. This
  enhancement does not affect that code path.

## Proposal

When a customer sets `spec.operatorConfiguration.aws.csiDriverConfig.kmsKeyARN` on a
`HostedCluster` at creation time, the following propagation chain executes:

1. The HC controller mirrors the `operatorConfiguration` (including the new `aws`
   subfield) to `HostedControlPlane`, following the existing mirroring pattern.
2. The HCCO storage reconciliation writes
   `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` to the guest cluster via its
   guest cluster client. This is a **write-once operation**: if the `ClusterCSIDriver`
   resource already exists (has a `ResourceVersion`), the HCCO skips writing
   `DriverConfig`, preserving any in-cluster modifications made by the cluster
   administrator. This follows the established ingress controller pattern
   (`ReconcileDefaultIngressController` returns early if the resource exists).
3. The **aws-ebs-csi-driver-operator** (in `openshift/csi-operator`) validates the
   KMS key by calling `kms:Encrypt` with a test payload using the
   `ebs-cloud-credentials` (StorageARN role). This operator already uses the AWS SDK
   for volume tagging and has IRSA credential plumbing. The validation runs
   continuously in the operator's reconcile loop, reporting a `KMSKeyValid` condition
   on `ClusterCSIDriver.status.conditions`.
4. The HCCO reads the `KMSKeyValid` condition from `ClusterCSIDriver` in the guest
   cluster and maps it to `ValidAWSStorageKMSConfig` on `HostedControlPlane` status.
   The HC controller bubbles it to `HostedCluster` status.
5. The CSO reads the `ClusterCSIDriver` value and configures the default StorageClass
   with `parameters.kmsKeyId`. New EBS volumes provisioned via the StorageClass carry
   the KMS encryption.

The cluster-storage-operator already reads
`ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` and configures the default
StorageClass accordingly — this was established in
[openshift/enhancements#1163](https://github.com/openshift/enhancements/pull/1163).
No CSO changes are needed.

### Workflow Description

**Actors:**
- **Cluster administrator:** A human operator who creates or manages `HostedCluster`
  resources (ROSA HCP customer or self-managed HyperShift operator).
- **HC controller:** The HyperShift HostedCluster controller running on the management
  cluster.
- **HCCO:** The Hosted Cluster Config Operator running in the HCP namespace on the
  management cluster.
- **CSO:** The cluster-storage-operator, running in the HCP namespace on the
  management cluster.

**Day-1 (cluster creation with KMS key):**

1. The cluster administrator runs:
   ```bash
   hcp create cluster aws \
     --storage-volumes-kms-key arn:aws:kms:us-east-1:123456789012:key/mrk-abc123 \
     ...
   ```
   or sets `spec.operatorConfiguration.aws.csiDriverConfig.kmsKeyARN` directly on the
   `HostedCluster` manifest.
2. The HC controller creates the `HostedControlPlane` with `operatorConfiguration`
   mirrored.
3. The HCCO writes `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` in the guest
   cluster (write-once: only on initial creation).
4. The aws-ebs-csi-driver-operator validates the key:
   - If the `ebs-cloud-credentials` role can assume and KMS `Encrypt` succeeds →
     `KMSKeyValid` condition `True/AsExpected` on ClusterCSIDriver.
   - If role assumption fails → `KMSKeyValid` condition `False/InvalidIAMRole`.
   - If KMS call fails (key disabled, deleted, or wrong region) → `KMSKeyValid`
     condition `False/AWSError`.
5. The HCCO reads `KMSKeyValid` from ClusterCSIDriver and maps it to
   `ValidAWSStorageKMSConfig` on HCP status. The HC controller bubbles it to HC.
6. The CSO configures the default StorageClass with `parameters.kmsKeyId` set to the
   ARN.
7. New PVCs created from the default StorageClass produce EBS volumes encrypted with
   the customer's key.

**Day-2 (key rotation — in-cluster operation):**

Day-2 key rotation is performed by the cluster administrator directly on the
`ClusterCSIDriver` resource in the guest cluster:

```bash
oc patch clustercsidriver ebs.csi.aws.com --type merge \
  -p '{"spec":{"driverConfig":{"driverType":"AWS","aws":{"kmsKeyARN":"arn:aws:kms:us-east-1:123456789012:key/new-key"}}}}'
```

The CSO picks up the change and updates the default StorageClass. New PVCs use the
new key. Existing PVs retain encryption with the original key. The HCCO does not
overwrite the in-cluster `ClusterCSIDriver` — the cluster administrator owns day-2
configuration.

This is the same pattern as the default ingress controller: HyperShift configures it
at creation time, but does not continuously reconcile it, preserving cluster admin
changes.

**Day-2 (disabling KMS encryption — in-cluster operation):**

The cluster administrator clears `DriverConfig` on the `ClusterCSIDriver` directly in
the guest cluster. The CSO reverts the default StorageClass to AWS-managed encryption.
The `ValidAWSStorageKMSConfig` condition on the `HostedCluster` is not affected —
it reflects the day-1 validation result only.

### API Extensions

#### New field: `kmsKeyARN` on `AWSCSIDriverConfig`

A new field is added under `spec.operatorConfiguration.aws.csiDriverConfig` on both
`HostedCluster` and `HostedControlPlane`. This introduces a pattern for
platform-specific operator configuration: platform-agnostic configs (CVO, CNO,
Ingress) remain at the top level of `operatorConfiguration`, while platform-specific
configs go under `operatorConfiguration.<platform>`.

New types:

```go
// AWSOperatorConfiguration specifies AWS-specific configuration for operators
// in the hosted cluster.
type AWSOperatorConfiguration struct {
	// csiDriverConfig specifies configuration for the AWS EBS CSI driver operator.
	// +optional
	CSIDriverConfig *AWSCSIDriverConfig `json:"csiDriverConfig,omitempty"`
}

// AWSCSIDriverConfig specifies configuration for the AWS EBS CSI driver.
type AWSCSIDriverConfig struct {
	// kmsKeyARN sets the cluster default storage class to encrypt volumes with
	// a user-defined KMS key, rather than the default KMS key used by AWS.
	// The value may be either the ARN or Alias ARN of a KMS key.
	//
	// The ARN must follow the format:
	//   arn:<partition>:kms:<region>:<account-id>:(key|alias)/<key-id-or-alias>
	// where <partition> is the AWS partition (aws, aws-cn, aws-us-gov, aws-iso,
	// aws-iso-b, aws-iso-e, aws-iso-f, or aws-eusc).
	//
	// This field is applied at cluster creation time only. Day-2 changes to
	// storage encryption should be made directly on the ClusterCSIDriver
	// resource in the guest cluster.
	//
	// The StorageARN role in AWSRolesRef must have kms:Encrypt (for validation),
	// kms:Decrypt, kms:GenerateDataKeyWithoutPlaintext, and kms:CreateGrant
	// permissions on the specified key.
	//
	// +optional
	// +kubebuilder:validation:MaxLength=2048
	// +openshift:validation:FeatureGateAwareXValidation:featureGate="",rule="matches(self, '^arn:(aws|aws-cn|aws-us-gov|aws-iso|aws-iso-b|aws-iso-e|aws-iso-f):kms:[a-z0-9-]+:[0-9]{12}:(key|alias)/.*$')",message="kmsKeyARN must be a valid AWS KMS key ARN in the format: arn:<partition>:kms:<region>:<account-id>:(key|alias)/<key-id-or-alias>"
	KMSKeyARN string `json:"kmsKeyARN,omitempty"`
}
```

Added to `OperatorConfiguration`:

```go
type OperatorConfiguration struct {
	// ... existing fields (clusterVersionOperator, clusterNetworkOperator, ingressOperator)

	// aws specifies AWS-specific operator configuration for the hosted cluster.
	// +optional
	AWS *AWSOperatorConfiguration `json:"aws,omitempty"`
}
```

The CEL validation regex aligns with the downstream
`ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` field in
[openshift/api](https://github.com/openshift/api/blob/master/operator/v1/types_csi_cluster_driver.go),
including the full set of AWS partitions.

#### New condition type: `ValidAWSStorageKMSConfig`

A new condition type is added to the HyperShift API, following the established pattern
of `ValidAWSKMSConfig` (etcd encryption):

| Status | Reason | Trigger | Example Message |
|--------|--------|---------|-----------------|
| `Unknown` | `StatusUnknown` | Field not configured | `"Storage KMS is not configured"` |
| `True` | `AsExpected` | Key validated successfully | `"Storage KMS key is valid and accessible"` |
| `False` | `AWSError` | KMS API call failed | `"KMS key arn:aws:kms:us-east-1:123456789012:key/abc-def is not accessible: KMSKeyDisabled. Verify the key is enabled and the StorageARN role has the required KMS permissions."` |
| `False` | `InvalidIAMRole` | Role assumption failed | `"Failed to assume StorageARN role for KMS key arn:aws:kms:us-east-1:123456789012:key/abc-def: AccessDenied. Verify the StorageARN role in AWSRolesRef has kms:Decrypt, kms:GenerateDataKeyWithoutPlaintext, and kms:CreateGrant permissions on this key."` |

The condition is registered in `ExpectedHCConditions` with `ConditionUnknown` for
clusters where `kmsKeyARN` is not configured. Because the validation runs
continuously in the aws-ebs-csi-driver-operator's reconcile loop (not just at
cluster creation), the condition reflects the **current** accessibility of the
KMS key — if a key becomes disabled or IAM permissions are revoked, the condition
transitions to `False` on the next reconcile.

Error messages include the failing ARN, the AWS error code, and a remediation step so
administrators can diagnose the issue from the condition alone without consulting
operator logs.

#### CLI flag: `--storage-volumes-kms-key`

A new flag is added to the AWS cluster creation command, consistent with the existing
`--root-volume-kms-key` naming convention:

```text
--storage-volumes-kms-key string
    AWS KMS key ARN (arn:...:key/...) or alias ARN (arn:...:alias/...) used
    to encrypt PVCs created by the default StorageClass at cluster creation.
    If omitted, PVCs use AWS-managed encryption. The StorageARN role must have
    kms:Encrypt, kms:Decrypt, kms:GenerateDataKeyWithoutPlaintext, and kms:CreateGrant
    permissions on the specified key. Day-2 key changes should be made directly
    on the ClusterCSIDriver resource in the guest cluster.
```

The flag is bound through the shared options mechanism so it is exposed in both the
HCP CLI (`hcp create cluster aws`) and the developer CLI
(`hypershift create cluster aws`) automatically.

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

**Day-1-only / set-and-forget reconciliation:** The HCCO writes `kmsKeyARN` to
`ClusterCSIDriver` only on initial resource creation. If the `ClusterCSIDriver` already
exists (has a `ResourceVersion`), the reconcile function returns early without
modifying `DriverConfig`. This follows the established pattern from the default ingress
controller (`ReconcileDefaultIngressController` in the HCCO), where day-1 setup is
provided via the HC spec but day-2 changes are made directly by the cluster
administrator on the in-cluster resource.

**KMS key validation in the storage stack:** Validation is performed by the
aws-ebs-csi-driver-operator (in `openshift/csi-operator`), not by the HCCO or CPO.
The operator already uses the AWS SDK v2 for volume tagging
(`pkg/driver/aws-ebs/aws_ebs_tags_controller.go`), with IRSA credential plumbing via
the `ebs-cloud-credentials` secret and `stscreds.NewWebIdentityRoleProvider`. A new
`EBSKMSKeyValidationController` follows the same pattern: it reads `kmsKeyARN` from
`ClusterCSIDriver.spec.driverConfig.aws`, calls `kms:Encrypt` with a test payload,
and reports a `KMSKeyValid` condition on `ClusterCSIDriver.status.conditions`. This
controller runs continuously in the operator's reconcile loop, providing up-to-date
key accessibility status — if a key is disabled or IAM permissions are revoked after
cluster creation, the condition transitions to `False`.

**Condition propagation chain:** The `KMSKeyValid` condition set by the
aws-ebs-csi-driver-operator on `ClusterCSIDriver.status` propagates through:
1. The CSO's `CSIDriverOperatorCRController.syncConditions()` copies it to the
   Storage operator CR as `AWSEBSCSIDriverOperatorCRKMSKeyValid`
2. The CSO's `ClusterOperatorStatusController` aggregates it into the
   `ClusterOperator "storage"` resource (contributing to the Degraded signal)
3. For explicit HyperShift visibility, the HCCO reads the `KMSKeyValid` condition
   from `ClusterCSIDriver` in the guest cluster and maps it to
   `ValidAWSStorageKMSConfig` on `HostedControlPlane` status
4. The HC controller copies `ValidAWSStorageKMSConfig` from HCP to HostedCluster

This is a cross-repo effort requiring changes in `openshift/csi-operator` (validation
controller), `openshift/hypershift` (HCCO condition watch + HC propagation), and
potentially `openshift/cloud-credential-operator` (adding `kms:Encrypt` to the
CredentialsRequest for ROSA standalone clusters).

**Why the aws-ebs-csi-driver-operator and not HCCO or CPO:** The driver operator is
the component closest to the actual KMS usage — it already reads `kmsKeyARN` from
`ClusterCSIDriver`, injects it into the StorageClass, and has the AWS SDK + IRSA
credential plumbing for making AWS API calls. Placing validation here means the probe
validates what the operator actually consumes, using the same credentials that the CSI
driver will use at volume creation time. It also means standalone (non-HyperShift)
clusters get the same validation — the condition is on `ClusterCSIDriver`, which is a
standard OpenShift resource. The existing `ValidAWSKMSConfig` etcd validation in the
CPO is not a precedent for placement because it uses a different role
(`AWSKMSRoleARN`), validates a different key (etcd encryption), and runs in a
HyperShift-only component.

### Risks and Mitigations

**Risk: Key disabled after volumes are encrypted.**
If the customer disables or deletes the KMS key in AWS after volumes are encrypted,
those volumes become inaccessible. HyperShift cannot prevent this.
*Mitigation:* The `KMSKeyValid` condition on `ClusterCSIDriver` transitions to
`False/AWSError` on the next driver operator reconcile, and the
`ValidAWSStorageKMSConfig` condition on the `HostedCluster` follows. This provides
continuous monitoring, unlike a one-shot probe. Document the key lifecycle
responsibility.

**Risk: Cross-team dependency on csi-operator changes.**
The KMS validation controller must be implemented in `openshift/csi-operator` by
the storage team. The HyperShift-side changes (HCCO condition watch, HC propagation)
depend on the `KMSKeyValid` condition existing on `ClusterCSIDriver.status`.
*Mitigation:* The HyperShift API field and CLI flag can ship independently — the
validation condition is additive. Clusters function correctly without the validation
(encryption works, you just don't get the proactive condition). The csi-operator
change can land in a subsequent release if needed.

**Risk: IAM role misconfiguration.**
If the `StorageARN` role lacks the required KMS permissions on the key, the KMS
probe fails and the condition is `False/InvalidIAMRole`.
*Mitigation:* The condition message includes the failing ARN and a remediation hint.
The HCCO still writes the ARN to `ClusterCSIDriver` (validation does not gate
reconciliation, following the established `ValidAWSKMSConfig` pattern), so PVC
provisioning may fail at the CSI driver level until IAM permissions are corrected.

**Risk: Existing cluster behavior change on upgrade.**
Clusters without `kmsKeyARN` must not experience any behavior change after upgrading
to a version containing this feature.
*Mitigation:* The field is optional with `omitempty`. The condition is registered as
`Unknown` when the field is absent. The HCCO's write-once pattern means it will not
touch `ClusterCSIDriver.DriverConfig` for clusters that already have the resource —
no behavioral change.

**Risk: Disrupting existing ClusterCSIDriver editability.**
Cluster administrators and GitOps workflows may already rely on `ClusterCSIDriver`
being directly editable in the guest cluster. Continuously overwriting it from the
HC spec would be a disruptive change.
*Mitigation:* The write-once pattern explicitly preserves this. The HCCO writes
`DriverConfig` only on initial `ClusterCSIDriver` creation and never overwrites
admin changes. This is the same approach used for the default ingress controller.

### Drawbacks

- Introduces a new sub-pattern in `OperatorConfiguration`: the first
  platform-specific entry (`aws`). This establishes a precedent that future
  platform-specific operator configs (Azure, GCP) would follow.
- The active KMS probe adds latency to the initial HCCO reconcile. This follows the
  established `ValidAWSKMSConfig` precedent and runs only at cluster creation.

## Alternatives (Not Implemented)

**Field on `AWSPlatformSpec`:**
Placing `storageKMSKeyARN` directly on the existing `AWSPlatformSpec` struct was the
initial design. Rejected because `AWSPlatformSpec` holds platform infrastructure
configuration (region, VPC, IAM roles), while this field configures an operator
(the CSI driver). The `operatorConfiguration` struct is the established home for
operator configs. Additionally, OCP APIs are hard to modify after creation —
getting the nesting right before GA avoids a future deprecation cycle.

**Continuous reconciliation of `ClusterCSIDriver`:**
Continuously reconciling `ClusterCSIDriver.DriverConfig` from the HC spec (like
OAuth configuration) was considered. Rejected because `ClusterCSIDriver` is already
editable by cluster administrators in the guest cluster, and breaking that UX would
be disruptive. The ingress controller precedent shows that day-1 setup via HC spec
with day-2 admin control is an established and preferred pattern.

**Run validation in CPO (where `ValidAWSKMSConfig` lives):**
Validation could be placed alongside the existing etcd KMS validation in the CPO.
Rejected because CPO uses a different role (`AWSKMSRoleARN`) for etcd encryption, and
storage concerns belong with the storage reconciliation owner (HCCO). Splitting would
increase coordination surface and mix concerns across operators.

**Propagation-confirmation-only validation (no KMS probe):**
Verify only that the ARN was written to `ClusterCSIDriver`, without calling KMS
`Encrypt`. Rejected because this approach cannot detect IAM permission issues until
the first volume creation failure, which is too late for actionable feedback.

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
- Regression: cluster created without `kmsKeyARN` is unaffected

### Unit Tests

Unit tests will cover the propagation chain (HC controller mirroring, HCCO storage
reconciliation with write-once semantics, condition bubble-up), KMS validation logic
(all condition states), and CLI flag wiring. Specific test cases:

- HC controller mirrors `operatorConfiguration.aws.csiDriverConfig` from HC to HCP
- HCCO writes `ClusterCSIDriver.DriverConfig.AWS.KMSKeyARN` on initial creation
- HCCO skips `DriverConfig` when `ClusterCSIDriver` already exists (write-once)
- Condition states: no key → Unknown, valid key → True, invalid role → False,
  failed encrypt → False
- CLI: `--storage-volumes-kms-key` flag parsed and wired to HC spec

### E2E Tests

E2E tests will be added to the HyperShift v2 E2E test suite (`test/e2e/v2/`), which
runs against a pre-existing hosted cluster on live AWS infrastructure. The test
reuses the existing CI KMS key (`alias/hypershift-ci`) already provisioned in the
CI AWS account.

Tests run in the `e2e-v2-aws` presubmit and `e2e-aws-ovn` periodic CI jobs, which use
the `hypershift` cluster profile with Boskos-managed AWS account leasing.

Test scenarios:

1. **Day-1 key configuration:**
   - Create a `HostedCluster` with `operatorConfiguration.aws.csiDriverConfig.kmsKeyARN`
     set
   - Wait for `ValidAWSStorageKMSConfig` condition to become `True/AsExpected`
   - Verify `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` is set in the hosted
     cluster
   - Create a PVC using the default StorageClass, wait for it to bind
   - Verify the resulting EBS volume is encrypted with the specified KMS key via
     `ec2.DescribeVolumes` (following the existing `KMSRootVolumeTest` pattern)

2. **Write-once semantics:**
   - Verify that modifying `ClusterCSIDriver.spec.driverConfig` directly in the guest
     cluster persists across HCCO reconcile cycles (the HCCO does not revert it)

3. **Regression (no key configured):**
   - On a hosted cluster where `kmsKeyARN` was never set, verify
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
- E2E test coverage for the full lifecycle (creation, write-once verification).
- `ValidAWSStorageKMSConfig` condition registered in `ExpectedHCConditions`.
- E2E tests passing in CI for at least one release cycle without flakes.
- Upgrade testing completed (field present on upgrade, absent clusters unaffected).
- User documentation merged in `openshift-docs` covering the
  `--storage-volumes-kms-key` flag, day-1 encryption setup, and day-2 in-cluster
  key management via `ClusterCSIDriver`.
- No open blocking bugs.

### Removing a deprecated feature

Not applicable. This enhancement adds new capability; nothing is deprecated.

## Upgrade / Downgrade Strategy

**Upgrade:** The new field is optional with `omitempty`. Existing `HostedCluster`
objects gain the field on upgrade, defaulting to empty. No action is required from
customers. The `ValidAWSStorageKMSConfig` condition is added to
`ExpectedHCConditions` with `ConditionUnknown` for clusters without the field,
preserving existing condition behavior. The HCCO's write-once pattern means existing
`ClusterCSIDriver` resources are not modified on upgrade.

**Failed upgrade rollback:** Control plane downgrades are not supported in
HyperShift. If an N→N+1 upgrade fails mid-way, the `kmsKeyARN` field is not yet
active and has no effect on storage behavior. If `kmsKeyARN` was already configured
on a successfully upgraded cluster, the `ClusterCSIDriver` in the hosted cluster
retains its last-written `DriverConfig` throughout any subsequent upgrade attempts.

## Version Skew Strategy

All components in the propagation chain (HC controller, HCCO, CSO) run on the
management cluster in the HCP namespace and are versioned together with the
HyperShift release. There is no multi-version skew within the propagation chain.

The `ClusterCSIDriver` CRD in the guest cluster already includes
`spec.driverConfig.aws.kmsKeyARN` from the AWS EBS CSI driver. HyperShift writes
this field only at initial cluster creation, so guest clusters running older OCP
versions that do not recognize the field will ignore it gracefully (Kubernetes
unknown-field pruning applies at admission).

## Operational Aspects of API Extensions

**SLIs for `ValidAWSStorageKMSConfig`:** The condition transition from `Unknown` to
`True` or `False` after cluster creation is the primary health indicator. The
condition reaches its terminal state within one HCCO reconcile interval during
cluster provisioning.

**Impact on existing SLIs:** The aws-ebs-csi-driver-operator reconcile loop gains
one KMS API call per reconcile when `kmsKeyARN` is set on `ClusterCSIDriver`.
This follows the same accepted pattern as the volume tagging controller in the
same operator.

**Failure modes and cluster health impact:**
- If the KMS probe fails (condition `False`), the `ClusterCSIDriver` field is still
  written (the HCCO does not gate reconciliation on condition success). PVC
  provisioning may fail at the CSI driver level if the key is inaccessible.
- No impact on existing workloads or control plane availability.

## Support Procedures

**Detecting failures:**
- Inspect `HostedCluster.status.conditions` for `ValidAWSStorageKMSConfig`.
- A `False` condition includes the failing ARN, AWS error code, and remediation step
  in `message`.

**Diagnosing IAM issues (`False/InvalidIAMRole`):**
1. Verify the `StorageARN` role in `HostedCluster.spec.platform.aws.rolesRef.storageARN`.
2. Confirm the role's IAM policy includes `kms:Encrypt` (for validation),
   `kms:Decrypt`, `kms:GenerateDataKeyWithoutPlaintext`, and `kms:CreateGrant`
   for the key ARN.
3. Confirm the key policy allows the `StorageARN` role principal.

**Diagnosing KMS access issues (`False/AWSError`):**
1. Confirm the KMS key is enabled in the AWS console / CLI.
2. Confirm the key exists in the correct AWS region (matches the cluster region).
3. For alias ARNs: confirm the alias points to an enabled key.

**Day-2 storage encryption changes:**
Day-2 key rotation, key removal, or other `ClusterCSIDriver` changes are made
directly in the guest cluster by the cluster administrator. The HCCO does not
manage `ClusterCSIDriver.DriverConfig` after initial creation.

**Graceful failure:** If the HCCO KMS probe errors during cluster creation, the
condition remains `False` but control plane provisioning continues. New PVC
provisioning may fail at the CSI driver level if the key is inaccessible.

## Infrastructure Needed

No new subprojects, repositories, or testing infrastructure are required. The E2E
tests run in the existing HyperShift v2 E2E CI jobs (`e2e-v2-aws`, `e2e-aws-ovn`),
which already provision live AWS infrastructure with KMS access via the
`alias/hypershift-ci` key. The `StorageARN` role in the CI account may need
`kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKeyWithoutPlaintext`, and
`kms:CreateGrant` permissions added for this key.

Changes are required in the following repositories:
- `openshift/csi-operator` — new `EBSKMSKeyValidationController` in
  `pkg/driver/aws-ebs/`, addition of `aws-sdk-go-v2/service/kms` dependency
- `openshift/hypershift` — HCCO condition watch, HC condition propagation, API field,
  CLI flag, HCCO write-once propagation
