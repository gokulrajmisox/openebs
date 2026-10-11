---
oep-number: OEP 677
title: PVC-level overrides for selected ZFS volume properties
authors:
  - "@gokulrajmisox"
owners:
  - "@tiagolobocastro"
editor: TBD
creation-date: 2026-10-11
last-updated: 2026-10-11
status: provisional
see-also:
  - OEP 4146
---

# PVC-level overrides for selected ZFS volume properties

> This is a provisional OEP submitted for discussion. No API choice in this
> document is maintainer-approved.

### Decision status

- **Verified:** the current driver reads ZFS parameters from the StorageClass,
  builds a `ZFSVolume`, and emits ZFS commands through the existing controller
  and node-side paths.
- **Provisional proposal:** use `VolumeAttributesClass` as the primary API,
  allow only selected mutable properties, and give an approved attribute value
  precedence over a StorageClass value.
- **Unresolved:** API selection, opt-in policy, exact allowlist, failure
  behavior, runtime modification, supported Kubernetes/sidecar versions, and
  clone/snapshot/restore semantics.

The words **proposed**, **initial**, and **should** in this document describe
design suggestions for maintainer review. They do not describe implemented or
approved behavior.

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Current implementation](#current-implementation)
  - [Proposed user-facing API](#proposed-user-facing-api-provisional)
  - [Property allowlist and validation](#property-allowlist-and-validation)
  - [Precedence and lifecycle](#precedence-and-lifecycle)
  - [Implementation details and constraints](#implementation-details-and-constraints)
  - [Risks and mitigations](#risks-and-mitigations)
- [Graduation Criteria](#graduation-criteria)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)
- [Testing](#testing)
- [Unresolved Questions](#unresolved-questions)
- [References](#references)

## Summary

Issue #677 requests a way to use one ZFS LocalPV StorageClass for multiple
workloads while allowing selected ZFS properties to vary per PVC. The current
driver receives ZFS parameters from the StorageClass and copies them into a
`ZFSVolume` custom resource. The proposed enhancement would add a Kubernetes-
native, administrator-controlled mechanism for selecting a limited set of PVC
volume attributes without allowing a PVC to change storage placement, capacity,
encryption, or other structural settings.

This draft proposes **VolumeAttributesClass (VAC)** as the primary API direction,
subject to maintainer confirmation. PVC annotations remain a possible fallback
if the supported CSI sidecars and Kubernetes versions make VAC impractical for
this driver. The feature must be opt-in, allowlisted, validated, and backward
compatible with existing StorageClass parameters.

## Motivation

A StorageClass currently combines the default storage policy and workload-
specific ZFS tuning. Supporting every combination through separate
StorageClasses can cause StorageClass proliferation. A database workload may
need a different `recordsize`, compression algorithm, or synchronous-write hint
than another workload using the same pool.

The issue proposes per-PVC overrides and gives annotations as one possible API.
The discussion also raised StorageClass opt-in syntax, a global controller
allowlist, and later Kubernetes VolumeAttributesClass. None of those choices
has been approved in the issue discussion as of this draft.

### Goals

1. Allow an administrator to define a reusable set of approved, mutable ZFS
   properties and select it for an individual PVC.
2. Preserve the existing StorageClass parameter path and behavior when no
   override mechanism is configured.
3. Keep placement, capacity, reservation, encryption, volume type, and other
   structural settings under StorageClass or controller policy.
4. Validate property names and values before creating or updating a `ZFSVolume`.
5. Make dataset-versus-ZVOL behavior explicit.
6. Define behavior for ordinary provisioning, clones, snapshots, and restores.
7. Provide unit and integration coverage for precedence, rejection, and actual
   ZFS property application.

### Non-Goals

1. Allowing arbitrary ZFS property names or arbitrary `userProperties` from a
   PVC or VAC.
2. Allowing PVC users to select a ZFS pool, node, capacity, reservation,
   filesystem type, volume type, encryption mode, or encryption key settings.
3. Changing existing StorageClass semantics.
4. Changing source-volume property inheritance for clones unless a separate
   approved design requires it.
5. Introducing a generic mechanism to mutate every ZFS property after
   provisioning.
6. Recompressing or rewriting data that was written before a property change.

## Proposal

### Current implementation

The current ordinary-volume path is:

```text
StorageClass parameters
  -> CSI CreateVolumeRequest.Parameters
  -> pkg/driver/controller.go: CreateZFSVolume
  -> pkg/builder/volbuilder/build.go
  -> ZFSVolume.Spec
  -> pkg/mgmt/volume/volume.go: syncZV
  -> pkg/zfs/zfs_util.go
  -> zfs create / zfs set
```

`CreateZFSVolume` normalizes parameter keys to lowercase, reads the ZFS
parameters, and passes them to the volume builder. The CSI external provisioner
also supplies PVC name and namespace as CSI parameters, but the current
controller does not fetch the PVC object or inspect PVC annotations.

The existing `ZFSVolume` API already represents `recordsize`, `compression`,
`dedup`, `atime`, and `logbias`. `PropertyChanged` and `SetVolumeProp` already
support updates for those fields, with `recordsize` and `atime` gated to
filesystem datasets. `volblocksize` is documented as non-editable after
provisioning.

### Proposed user-facing API (provisional)

The provisional preferred design is to use the Kubernetes
`VolumeAttributesClass` resource for mutable attributes. This separates the
StorageClass, which selects the storage policy, from a reusable class of volume
attributes. A PVC would select a VAC using the Kubernetes PVC field supported
by the target Kubernetes and CSI sidecar versions.

This is technically feasible, but it is not a drop-in extension of the current
driver. Kubernetes documents VAC as applying to CSI drivers that implement the
`ModifyVolume` API. The current driver does not advertise
`RPC_MODIFY_VOLUME`, and `ControllerModifyVolume` currently returns
`Unimplemented`. A VAC implementation would therefore require CSI controller
and sidecar lifecycle work in addition to parsing the class parameters.

Illustrative **provisional** example; exact parameter keys, supported versions,
and lifecycle behavior require maintainer and Kubernetes compatibility review:

```yaml
apiVersion: storage.k8s.io/v1
kind: VolumeAttributesClass
metadata:
  name: postgres-zfs
driverName: zfs.csi.openebs.io
parameters:
  compression: zstd
  recordsize: 16k
  logbias: latency
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  storageClassName: local-zfs
  volumeAttributesClassName: postgres-zfs
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

The StorageClass remains the source of defaults and policy in this
**provisional** model:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-zfs
provisioner: zfs.csi.openebs.io
parameters:
  poolname: pool-a
  fstype: zfs
  compression: lz4
  recordsize: 128k
  dedup: off
  logbias: latency
```

The proposed, not yet approved, merge is:

```text
approved VAC value > StorageClass value > ZFS default
```

This precedence is a proposal, not an existing project rule. If maintainers
select annotations instead, the following is an equivalent **provisional
illustrative form**, not an approved API:

```yaml
metadata:
  annotations:
    zfs.csi.openebs.io/compression: zstd
    zfs.csi.openebs.io/recordsize: 16k
```

Annotations should only be used after an explicit decision about opt-in,
RBAC, validation, and whether the external-provisioner request contains enough
information to read the PVC safely.

### Property allowlist and validation

The initial allowlist should be limited to properties already represented and
handled as mutable by the driver:

- `compression`: dataset and ZVOL; validate against the existing API pattern.
- `dedup`: dataset and ZVOL; allow only `on` or `off`.
- `logbias`: dataset and ZVOL; allow only `latency` or `throughput`.
- `recordsize`: dataset only; validate the documented ZFS size constraints.
- `atime`: dataset only; allow only `on` or `off`.

The first implementation should not allow:

- `volblocksize`;
- `encryption`, `keyformat`, or `keylocation`;
- `poolname`, `poolpattern`, or node selection;
- `capacity`, `thinprovision`, `quotatype`, or reservation-related values;
- `fstype` or volume type;
- arbitrary ZFS properties or arbitrary user properties.

Validation should happen before the merged values are written to the
`ZFSVolume` resource. The implementation should use one shared validation path
for StorageClass and VAC/annotation values, rather than allowing the new input
path to bypass existing CRD constraints.

An invalid or disallowed attribute should fail the provisioning request with a
clear `InvalidArgument`-style error. Silently ignoring a user-specified value
would make the resulting storage policy difficult to audit. The exact CSI
status code and whether an invalid unused VAC parameter is rejected require
maintainer confirmation.

### Precedence and lifecycle

For ordinary provisioning:

1. Start with the existing StorageClass parameter map.
2. Read the selected VAC parameters, if VAC is approved and selected.
3. Reject names outside the allowlist.
4. Validate the override values and volume-type applicability.
5. Merge approved values over StorageClass defaults.
6. Build the existing `ZFSVolume.Spec` without changing the ZFS command path.

For an existing provisioned volume, VAC behavior depends on CSI sidecar and
controller support for mutable volume attributes. The provisional design must
choose one of these scopes:

- support the Kubernetes VAC lifecycle, which requires the CSI `ModifyVolume`
  contract, and reconcile updates through the existing `PropertyChanged` and
  `SetVolumeProp` path; or
- defer VAC until that lifecycle is implemented and choose a separately
  approved creation-time API.

The OEP must not claim runtime modification until that lifecycle is implemented
and tested.

For clones and snapshot restores, the safe default is to preserve current
behavior: the source `ZFSVolume` or snapshot specification determines inherited
properties. A selected VAC must not silently change encryption inheritance or
creation-only properties. If VAC values are permitted for a new clone or
restore, the proposal must define which values override inherited source
values and how ZFS clone restrictions are handled.

### Implementation details and constraints

A likely implementation boundary is a pure merge/validation helper in the CSI
controller layer. It should accept the StorageClass parameters, the selected
mutable attributes, and the volume type, and return the effective parameter
map. This allows table-driven unit tests without requiring a Kubernetes API
server or ZFS installation.

The controller would then continue using the existing builder methods:

- `WithRecordSize`
- `WithCompression`
- `WithDedup`
- `WithATime`
- `WithLogBias`

No new shell command construction is needed. The existing ZFS builders already
emit the effective fields for dataset creation, ZVOL creation, cloning, and
property updates.

If PVC annotations are selected instead of VAC, the controller will need to
fetch the PVC using the name and namespace already present in the CSI request.
That requires reviewing controller client initialization, namespace behavior,
cache consistency, and RBAC. The OEP should not assume that annotation lookup
is free or reliable during provisioning.

The deployment must advertise `RPC_MODIFY_VOLUME` only if the selected design
actually implements the CSI modify-volume contract. CSI
external-provisioner and external-resizer versions, Kubernetes versions, and
the request fields used to carry VAC parameters must be documented and tested.
These are provisional implementation requirements, not current driver
capabilities.

### Risks and mitigations

#### Tenant and administrator policy bypass

A PVC-controlled input could change pool placement, capacity, encryption, or
resource consumption if the allowlist is incomplete. Mitigate this with a
fixed allowlist, strict validation, per-driver identity checks for VAC, and
explicit rejection of unsupported parameters.

#### Resource exhaustion

Enabling deduplication can consume significant CPU and memory and generate
additional I/O. Changing `logbias` can alter synchronous-write behavior. These
properties should be opt-in by administrators and documented as workload and
pool policy decisions.

#### Data and semantic surprises

Changing compression affects newly written data; it does not rewrite existing
data. Changing recordsize affects files created afterward. A property change
may therefore produce mixed behavior within one dataset. Tests and
 documentation must state this clearly.

#### Clone and encryption hazards

ZFS clones inherit encryption settings from the source snapshot, and encryption
properties cannot simply be set on a clone. The implementation must not pass
creation-time encryption overrides into clone commands.

#### Backward compatibility

When no VAC or override configuration is selected, existing StorageClass
parameters must produce the same `ZFSVolume.Spec` and ZFS command arguments.
Existing clones, snapshot restores, and encrypted volumes must retain their
current behavior. New validation must not reject values that the current
supported path intentionally accepts unless that behavior is explicitly
versioned and documented.

#### Availability and API dependency

VAC support depends on Kubernetes and CSI sidecar versions. PVC annotation
lookup would add a Kubernetes API/RBAC dependency to provisioning. The chosen
implementation must define supported versions and failure behavior.

## Graduation Criteria

The feature is ready to move beyond provisional status only when maintainers
have approved:

1. VAC versus annotation API direction.
2. The allowlist and value validation rules.
3. Opt-in and enforcement semantics.
4. Precedence over StorageClass values.
5. Ordinary-volume, clone, snapshot, and restore behavior.
6. Supported Kubernetes and CSI sidecar versions.
7. Security and RBAC review.

Implementation criteria include passing focused unit tests, relevant package
tests, integration tests that inspect actual ZFS properties, and CSI tests for
backward compatibility.

## Implementation History

- 2025-10-20: Issue #677 opened.
- 2025-10-27 through 2026-01-23: Issue discussion considered annotations,
  global and StorageClass opt-in, and special `?` syntax.
- 2026-06-23: Issue discussion referenced Kubernetes VolumeAttributesClass.
- 2026-10-10: Local provisional OEP draft created for review.
- 2026-10-11: Provisional OEP submitted in a focused upstream PR. It has no
  maintainer approval yet.

## Drawbacks

VAC introduces another Kubernetes object and depends on Kubernetes support for
mutable volume attributes. An annotation design would be simpler for users but
would be driver-specific, harder to govern consistently, and would require the
controller to fetch PVC objects. Supporting both mechanisms would increase
complexity and create ambiguous precedence.

The allowlist intentionally limits flexibility. Some advanced ZFS workloads
may still need separate StorageClasses or a future, separately reviewed
attribute.

## Alternatives

### PVC annotations (unapproved alternative)

Annotations match the original issue example and can be read from the PVC
identified by the CSI request. They are easy to demonstrate but are
provider-specific, require API lookup and RBAC, and do not provide a standard
lifecycle for mutable volume attributes.

### StorageClass opt-in syntax (unapproved alternative)

A StorageClass could mark a default as overridable, for example
`compression: lz4?`, or mark the key, for example `?compression: lz4`. This
preserves the current request path but introduces special syntax and does not
provide a reusable attribute object. It also requires defining how the PVC
supplies the replacement value.

### Global controller allowlist (unapproved alternative)

A controller flag or ConfigMap could define which properties are overridable.
This centralizes policy but makes per-StorageClass enforcement difficult and
adds operational configuration and reload semantics.

### Separate StorageClasses (current behavior)

This is the current supported solution. It is explicit and easy to enforce,
but it causes StorageClass proliferation when only one or two ZFS properties
differ between workloads.

### No change

Maintaining the current behavior avoids compatibility and security risk while
leaving the issue's workload-specific configuration problem unresolved.

## Infrastructure Needed

The canonical OpenEBS OEP process is documented in the `openebs/openebs`
repository. The official template is `designs/oep-template.md`, and OEPs are
stored under `designs/xxx`. The tracking issue supplies the OEP number, so this
proposal uses `OEP 677`.

This document is submitted under the canonical `openebs/openebs` designs
directory. The owner entry uses `@tiagolobocastro` as a proposed local-PV owner
based on the existing local-PV VolumeAttributesClass OEP; maintainers should
correct that metadata if another owner is appropriate.

## Testing

### Unit tests

Add table-driven tests for the merge and validation helper covering:

- no override preserves StorageClass values;
- approved override replaces a StorageClass value;
- omitted override falls back to StorageClass, then ZFS default;
- disabled or unapproved override is rejected;
- unknown property names are rejected;
- invalid compression, dedup, logbias, atime, and recordsize values are rejected;
- dataset-only properties are rejected or ignored for ZVOLs according to the
  approved contract;
- structural properties such as pool and volume size cannot be overridden;
- case handling is explicit and compatible with existing StorageClass behavior;
- clone and restore inputs preserve their approved inheritance behavior.

Retain the existing ZFS command-builder tests in `pkg/zfs/volume_test.go` and
add only focused cases if the effective parameter path exposes a regression.

### Integration tests

Extend the existing Ginkgo provisioning tests to verify both the `ZFSVolume`
resource and the actual ZFS property obtained with `zfs get`:

1. PVC without an override receives the StorageClass value.
2. PVC with an approved override receives the override.
3. Two PVCs using one StorageClass can receive different approved values.
4. Disallowed placement, capacity, encryption, and volume-type overrides do
   not take effect.
5. Invalid input fails provisioning clearly and does not leave a ready volume.
6. Dataset and ZVOL behavior is covered.
7. Clone, snapshot, and restore behavior matches the approved inheritance rules.
8. Existing StorageClass-only and encrypted-volume tests continue to pass.

The existing helpers `VerifyStorageClassParams`, `GetVolumeProperty`, and
`VerifyZFSVolumePropEdit` provide the repository convention for property checks.

## Unresolved Questions

These questions require maintainer or Kubernetes compatibility feedback before
implementation:

1. Should this feature use VolumeAttributesClass, annotations, or another API?
2. What Kubernetes and CSI sidecar versions must be supported?
3. Is the feature enabled by VAC selection alone, by StorageClass opt-in, or by
   an additional controller policy?
4. Does a VAC value override a StorageClass value, and how are omitted values
   represented?
5. Which exact properties are approved for the first version?
6. Should invalid or unsupported attributes reject provisioning or be ignored?
7. Are VAC parameters creation-only, or must runtime modification be supported?
8. Do VAC values apply to ordinary volumes only, or also clones and restores?
9. If annotations are retained, what controller RBAC and API-cache behavior is
   acceptable?
10. Should a separate OEP be created in `openebs/openebs`, or should the design
    be maintained alongside `zfs-localpv` documentation?

## References

1. [OpenEBS OEP template](https://github.com/openebs/openebs/blob/develop/designs/oep-template.md)
2. [OpenEBS OEP process](https://github.com/openebs/openebs/blob/develop/designs/oep-process.md)
3. [OpenEBS designs README](https://github.com/openebs/openebs/blob/develop/designs/README.md)
4. [Issue #677](https://github.com/openebs/zfs-localpv/issues/677)
5. [Existing OpenEBS LVM VolumeAttributesClass OEP](https://github.com/openebs/openebs/blob/develop/designs/local-pv/lvm/volume-attributes-class.md)
6. [Kubernetes VolumeAttributesClass documentation](https://kubernetes.io/docs/concepts/storage/volume-attributes-classes/)
7. [CSI external-provisioner documentation](https://kubernetes-csi.github.io/docs/external-provisioner.html)
8. [ZFS LocalPV StorageClass documentation](docs/storageclasses.md)
9. [`CreateZFSVolume`](pkg/driver/controller.go)
10. [`ZFSVolume` API](pkg/apis/openebs.io/zfs/v1/zfsvolume.go)
11. [ZFS command builders](pkg/zfs/zfs_util.go)
12. [Provisioning integration tests](tests/provision_test.go)
