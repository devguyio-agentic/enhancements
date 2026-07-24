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
`spec.operatorConfiguration.csiDriverConfig.aws` on the HyperShift `HostedCluster`
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

### User Stories

- As a ROSA HCP cluster administrator, I want to specify a KMS key ARN when creating
  a cluster so that all PVCs provisioned by the default StorageClass are encrypted with
  my organization's key instead of the default AWS-managed key.

- As a ROSA HCP cluster administrator, I want to update the KMS encryption
  configuration on a running cluster by editing the `ClusterCSIDriver` resource
  directly in the guest cluster, so that I can rotate keys or change encryption
  settings without recreating the cluster.

- As a self-managed HyperShift operator or HyperShift developer on AWS, I want to
  specify a KMS key for the default StorageClass at cluster creation time via the
  `hcp` or `hypershift` CLI so that I can enforce encryption standards from day 1.

### Goals

- Add an optional `kmsKeyARN` field under
  `spec.operatorConfiguration.csiDriverConfig.aws` as a string field accepting
  KMS key ARNs and alias ARNs.
- Propagate the field at cluster creation from `HostedCluster` →
  `HostedControlPlane` → `ClusterCSIDriver` in the guest cluster (write-once,
  not continuously reconciled).
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
- **Per-StorageClass KMS key granularity.** This targets only the default
  StorageClass. Users can still create custom StorageClasses with their own keys.
  Per-StorageClass granularity would require CSI operator changes, out of scope.
- **Day-2 key management via the HostedCluster API.** Day-2 key rotation and removal
  are performed directly on `ClusterCSIDriver` in the guest cluster. The HC field
  captures day-1 intent only.
- **Re-encrypting existing PVs.** The KMS key applies to newly created PVCs only.
  Existing volumes retain their original encryption.
- **NodePool root volume encryption.** Root volume encryption is configured separately
  on `NodePool.spec.platform.aws.rootVolume.encryptionKey` and is not affected by this
  enhancement.
- **Etcd encryption.** Etcd KMS key management is handled by the existing
  `ValidAWSKMSConfig` mechanism using a dedicated `AWSKMSRoleARN` role. This
  enhancement does not affect that code path.

## Proposal

When a customer sets `spec.operatorConfiguration.csiDriverConfig.aws.kmsKeyARN` on a
`HostedCluster` at creation time, the following propagation chain executes:

1. The HC controller mirrors the `operatorConfiguration` (including the new `aws`
   subfield) to `HostedControlPlane`.
2. The HCCO storage reconciliation writes
   `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` to the guest cluster via its
   guest cluster client. This is a **write-once operation**: if the `ClusterCSIDriver`
   resource already exists (has a `ResourceVersion`), the HCCO skips writing
   `DriverConfig`, preserving in-cluster modifications (like
   `ReconcileDefaultIngressController`, which returns early if the resource exists).
3. The CSO reads the `ClusterCSIDriver` value and configures the default StorageClass
   with `parameters.kmsKeyId`. New EBS volumes provisioned via the StorageClass carry
   the KMS encryption.

The cluster-storage-operator already reads
`ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` and configures the default
StorageClass accordingly. This was established in
[openshift/enhancements#1163](https://github.com/openshift/enhancements/pull/1163).
No CSO changes are needed.

### Workflow Description

#### Actors

- **Cluster administrator:** A human operator who creates or manages `HostedCluster`
  resources (ROSA HCP customer or self-managed HyperShift operator).
- **HC controller:** The HyperShift HostedCluster controller running on the management
  cluster.
- **HCCO:** The Hosted Cluster Config Operator running in the HCP namespace on the
  management cluster.
- **CSO:** The cluster-storage-operator, running in the HCP namespace on the
  management cluster.

#### Day-1: Cluster Creation with KMS Key

1. The cluster administrator runs:
   ```bash
   hcp create cluster aws \
     --storage-volumes-kms-key arn:aws:kms:us-east-1:123456789012:key/mrk-abc123 \
     ...
   ```
   or sets `spec.operatorConfiguration.csiDriverConfig.aws.kmsKeyARN` directly on the
   `HostedCluster` manifest.
2. The HC controller creates the `HostedControlPlane` with `operatorConfiguration`
   mirrored.
3. The HCCO writes `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` in the guest
   cluster (write-once: only on initial creation).
4. The CSO configures the default StorageClass with `parameters.kmsKeyId` set to the
   ARN.
5. New PVCs created from the default StorageClass produce EBS volumes encrypted with
   the customer's key.

If an invalid KMS key is configured, errors surface naturally during PVC
provisioning through the CSI driver. This is the same behavior as standalone OCP.

#### Day-2: Key Rotation (In-Cluster)

Day-2 key rotation is performed by the cluster administrator directly on the
`ClusterCSIDriver` resource in the guest cluster:

```bash
oc patch clustercsidriver ebs.csi.aws.com --type merge \
  -p '{"spec":{"driverConfig":{"driverType":"AWS","aws":{"kmsKeyARN":"arn:aws:kms:us-east-1:123456789012:key/new-key"}}}}'
```

The CSO picks up the change and updates the default StorageClass. New PVCs use the
new key. Existing PVs retain encryption with the original key. The HCCO does not overwrite in-cluster `ClusterCSIDriver` after initial creation.

#### Day-2: Disabling KMS Encryption (In-Cluster)

The cluster administrator clears `DriverConfig` on the `ClusterCSIDriver` directly in
the guest cluster. The CSO reverts the default StorageClass to AWS-managed encryption.

### API Extensions

#### New field: `kmsKeyARN` on `AWSCSIDriverConfig`

A new field is added under `spec.operatorConfiguration.csiDriverConfig.aws` on both
`HostedCluster` and `HostedControlPlane`. This follows the ingress operator pattern
where platform-specific configuration is nested inside the operator's own config
(`ingressOperator.endpointPublishingStrategy.loadBalancer.providerParameters.aws`).
The `CSIDriverOperatorConfig` struct naturally extends to `azure`, `gcp` in the future.

New types:

```go
// CSIDriverOperatorConfig specifies configuration for CSI driver operators
// in the hosted cluster.
type CSIDriverOperatorConfig struct {
	// aws specifies configuration for the AWS EBS CSI driver operator.
	// +optional
	AWS *AWSCSIDriverConfig `json:"aws,omitempty"`
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
	// aws-iso-b, aws-iso-e, or aws-iso-f).
	//
	// This field is applied at cluster creation time only. Day-2 changes to
	// storage encryption should be made directly on the ClusterCSIDriver
	// resource in the guest cluster.
	//
	// The StorageARN role in AWSRolesRef must have kms:Decrypt,
	// kms:GenerateDataKeyWithoutPlaintext, and kms:CreateGrant
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

	// csiDriverConfig specifies configuration for CSI driver operators in the hosted cluster.
	// +optional
	CSIDriverConfig *CSIDriverOperatorConfig `json:"csiDriverConfig,omitempty"`
}
```

The CEL validation regex aligns with the downstream
`ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` field in
[openshift/api](https://github.com/openshift/api/blob/master/operator/v1/types_csi_cluster_driver.go),
including the full set of AWS partitions.

#### KMS Error Visibility

If an invalid KMS key is configured, errors surface naturally during PVC
provisioning through the CSI driver. This is the same behavior as standalone OCP.
Proactive KMS key validation is planned as a separate feature (OCPSTRAT-3501).

#### CLI flag: `--storage-volumes-kms-key`

A new flag is added to the AWS cluster creation command, consistent with the existing
`--root-volume-kms-key` naming convention:

```text
--storage-volumes-kms-key string
    AWS KMS key ARN (arn:...:key/...) or alias ARN (arn:...:alias/...) used
    to encrypt PVCs created by the default StorageClass at cluster creation.
    If omitted, PVCs use AWS-managed encryption. The StorageARN role must have
    kms:Decrypt, kms:GenerateDataKeyWithoutPlaintext, and kms:CreateGrant
    permissions on the specified key. Day-2 key changes should be made directly
    on the ClusterCSIDriver resource in the guest cluster.
```

The flag is bound through the shared options mechanism so it is exposed in both the
HCP CLI (`hcp create cluster aws`) and the developer CLI
(`hypershift create cluster aws`) automatically.

### Topology Considerations

#### Hypershift / Hosted Control Planes

All control plane components (HC controller, HCCO, CSO, CSI driver operators) run on
the management cluster in the HCP namespace. The HCCO uses a dual-client architecture:
a management cluster client for reading `HostedControlPlane` spec, and a guest cluster
client (via an injected guest kubeconfig) for writing `ClusterCSIDriver` and cloud
credential secrets. CSI DaemonSets run on guest cluster worker nodes.

The `StorageARN` role is an IRSA-style (IAM Roles for Service Accounts) role that the
HCCO already provisions for storage credential management (`ebs-cloud-credentials`).
No new roles or permissions are introduced beyond what the feature already requires.

#### Standalone Clusters

Not applicable. This enhancement targets only `HostedCluster` resources managed by
HyperShift. The `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` field already
exists in standalone OCP.

#### Single-node Deployments or MicroShift

Not applicable.

#### OpenShift Kubernetes Engine (OKE)

Not applicable. Storage KMS configuration in standard OpenShift clusters is handled
via `ClusterCSIDriver` directly.

### Implementation Details/Notes/Constraints

#### IAM Permissions

The StorageARN role requires the following KMS permissions:
- `kms:Decrypt` (for EBS volume attachment)
- `kms:GenerateDataKeyWithoutPlaintext` (for EBS volume creation)
- `kms:CreateGrant` (for EBS service principal access)

ROSA HCP clusters require these permissions in the
`ROSAAmazonEBSCSIDriverOperatorPolicy` AWS managed policy. Self-managed
HyperShift clusters require equivalent permissions on the StorageARN role.

#### Day-1-Only Reconciliation

The HCCO writes `kmsKeyARN` to
`ClusterCSIDriver` only on initial resource creation. If the `ClusterCSIDriver` already
exists (has a `ResourceVersion`), the reconcile function returns early without
modifying `DriverConfig`. This follows the ingress controller pattern
(`ReconcileDefaultIngressController` returns early when the resource exists).

### Risks and Mitigations

#### Key Disabled After Volumes Encrypted

If the customer disables or deletes the KMS key in AWS after volumes are encrypted,
those volumes become inaccessible. HyperShift cannot prevent this.
*Mitigation:* The CSI driver will fail to attach/use the volume, surfacing the AWS
error during PVC provisioning. Document the key lifecycle responsibility.

#### IAM Role Misconfiguration

If the `StorageARN` role lacks the required KMS permissions on the key, PVC
provisioning fails at the CSI driver level with the AWS error.
*Mitigation:* The HCCO still writes the ARN to `ClusterCSIDriver`. IAM permission
errors surface during PVC provisioning. Document required permissions.

#### Existing Cluster Behavior on Upgrade

Clusters without `kmsKeyARN` must not experience any behavior change after upgrading
to a version containing this feature.
*Mitigation:* The field is optional with `omitempty`. The write-once pattern
(see Implementation Details) prevents any behavioral change for existing clusters.

#### Disrupting ClusterCSIDriver Editability

Cluster administrators and GitOps workflows may already rely on `ClusterCSIDriver`
being directly editable in the guest cluster. Continuously overwriting it from the
HC spec would be a disruptive change.
*Mitigation:* The write-once pattern (see Implementation Details) preserves this.

### Drawbacks

- Introduces a new operator entry in `OperatorConfiguration` (`csiDriverConfig`).
  Platform branching is inside the operator config, following the ingress operator
  pattern.

## Alternatives (Not Implemented)

#### Field on AWSPlatformSpec

Placing `storageKMSKeyARN` directly on the existing `AWSPlatformSpec` struct was the
initial design. Rejected because `AWSPlatformSpec` holds platform infrastructure
configuration (region, VPC, IAM roles), while this field configures an operator
(the CSI driver). The `operatorConfiguration` struct is where operator configs belong. OCP APIs are hard to modify after creation;
getting the nesting right before GA avoids a future deprecation cycle.

#### Continuous Reconciliation of ClusterCSIDriver

Continuously reconciling `ClusterCSIDriver.DriverConfig` from the HC spec (like
OAuth configuration) was considered. Rejected because `ClusterCSIDriver` is already
editable by cluster administrators in the guest cluster, and breaking that UX would
be disruptive. The ingress controller uses the same day-1-setup / day-2-admin-control model.

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
reconciliation with write-once semantics), and CLI flag wiring. Specific test cases:

- HC controller mirrors `operatorConfiguration.csiDriverConfig.aws` from HC to HCP
- HCCO writes `ClusterCSIDriver.DriverConfig.AWS.KMSKeyARN` on initial creation
- HCCO skips `DriverConfig` when `ClusterCSIDriver` already exists (write-once)
- CLI: `--storage-volumes-kms-key` flag parsed and wired to HC spec

### E2E Tests

E2E tests will be added to the HyperShift E2E test suite, which runs against a
pre-existing hosted cluster on live AWS infrastructure. The test
reuses the existing CI KMS key (`alias/hypershift-ci`) already provisioned in the
CI AWS account.

Tests run in the `e2e-aws` presubmit and `e2e-aws-ovn` periodic CI jobs, which use
the `hypershift` cluster profile with Boskos-managed AWS account leasing.

Test scenarios:

1. **Day-1 key configuration:**
   - Create a `HostedCluster` with `operatorConfiguration.csiDriverConfig.aws.kmsKeyARN`
     set
   - Verify `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` is set in the hosted
     cluster
   - Create a PVC using the default StorageClass, wait for it to bind
   - Verify the resulting EBS volume is encrypted with the specified KMS key via
     `ec2.DescribeVolumes` (following the existing `KMSRootVolumeTest` pattern)

2. **Write-once semantics:**
   - Verify that modifying `ClusterCSIDriver.spec.driverConfig` directly in the guest
     cluster persists across HCCO reconcile cycles (the HCCO does not revert it)

3. **Regression (no key configured):**
   - On a hosted cluster where `kmsKeyARN` was never set, verify PVCs use
     AWS-managed encryption

## Graduation Criteria

### Dev Preview -> Tech Preview

This feature ships directly to GA. No Dev Preview or Tech Preview phase.

### Tech Preview -> GA

See above.

### GA

- Unit test coverage for all propagation paths.
- Envtest coverage for CEL validation across multiple Kubernetes versions.
- E2E test coverage for the full lifecycle (creation, write-once verification).
- E2E tests passing in CI for at least one release cycle without flakes.
- Upgrade testing completed (field present on upgrade, absent clusters unaffected).
- User documentation merged in `openshift-docs` covering the
  `--storage-volumes-kms-key` flag, day-1 encryption setup, and day-2 in-cluster
  key management via `ClusterCSIDriver`.
- No open blocking bugs.

### Removing a deprecated feature

Not applicable. This enhancement adds new capability; nothing is deprecated.

## Upgrade / Downgrade Strategy

#### Upgrade

The new field is optional with `omitempty`. Existing `HostedCluster`
objects gain the field on upgrade, defaulting to empty. No action is required from
customers. The write-once pattern means existing `ClusterCSIDriver` resources are not
modified on upgrade.

#### Failed Upgrade Rollback

Control plane downgrades are not supported in
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

#### SLIs

The primary indicator is successful PVC provisioning with the configured KMS key.
If the key is invalid, PVC provisioning failures surface the issue.

#### Impact on Existing SLIs

No additional AWS API calls are introduced by this feature. The HCCO writes the
field once at cluster creation.

#### Failure Modes

- If the configured KMS key is invalid or the IAM role lacks permissions, PVC
  provisioning fails at the CSI driver level with the AWS error. The
  `ClusterCSIDriver` field is still written by the HCCO.
- No impact on existing workloads or control plane availability.

## Support Procedures

#### Detecting Failures

- If PVCs from the default StorageClass fail to provision, check the PVC events
  for AWS error codes related to KMS key access.
- Verify `ClusterCSIDriver.spec.driverConfig.aws.kmsKeyARN` was written correctly
  by inspecting the guest cluster resource.

#### Diagnosing IAM Permission Errors

1. Verify the `StorageARN` role in `HostedCluster.spec.platform.aws.rolesRef.storageARN`.
2. Confirm the role's IAM policy includes `kms:Decrypt`,
   `kms:GenerateDataKeyWithoutPlaintext`, and `kms:CreateGrant`
   for the key ARN.
3. Confirm the key policy allows the `StorageARN` role principal.

#### Diagnosing KMS Key Errors

1. Confirm the KMS key is enabled in the AWS console / CLI.
2. Confirm the key exists in the correct AWS region (matches the cluster region).
3. For alias ARNs: confirm the alias points to an enabled key.

#### Day-2 Storage Encryption Changes

Day-2 key rotation, key removal, or other `ClusterCSIDriver` changes are made
directly in the guest cluster by the cluster administrator. The HCCO does not
manage `ClusterCSIDriver.DriverConfig` after initial creation.

#### Graceful Failure

If the configured KMS key is invalid, control plane provisioning continues
normally. PVC provisioning may fail at the CSI driver level if the key is invalid
or permissions are insufficient. This is the same behavior as standalone OCP.

## Infrastructure Needed

No new subprojects, repositories, or testing infrastructure are required. The E2E
tests run in the existing HyperShift E2E CI jobs (`e2e-aws`, `e2e-aws-ovn`),
which already provision live AWS infrastructure with KMS access via the
`alias/hypershift-ci` key. The `StorageARN` role in the CI account must have
`kms:Decrypt`, `kms:GenerateDataKeyWithoutPlaintext`, and `kms:CreateGrant`
permissions added for this key.

Changes are required in `openshift/hypershift` only: API field, CLI flag, HCCO
write-once propagation. Proactive KMS key validation is planned as a separate
feature (OCPSTRAT-3501).