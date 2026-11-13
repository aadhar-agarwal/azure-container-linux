---
title: Signed EROFS dm-verity availability review
type: project
updated: 2026-11-13
---

# Signed EROFS dm-verity availability review

Source: [`erofs-signed-dmverity.patch` at `aclmain@30df5483`][patch].
Locations below are patch lines. Findings are source-confirmed, not live reproductions.

## Verdict

**Yes: a signed layer that cannot create or reuse a verified dm-verity mapping
cannot mount or start the container.** IPE audit mode does not relax containerd
or kernel signature checks. This is a workload availability risk, not an inherent
host crash.

Strict verification is correct when required. Making optional signing a hard
dependency is the concern for a non-disruptive audit rollout.

## Concerns

### 1. issue (blocking for audit rollout): No signed-path fallback

Missing trust, invalid signatures, or unreadable signature files can block
startup: `setupSignedDmVerityDevice` errors propagate through the mount handler.
`dmverity_mode = "auto"` is not best-effort for signed layers; `"off"` is
incompatible with referrer enablement.

**Location:** patch L2574-L2586, L2615-L2648, L2793-L2795.
**Remedy:** keep verification strict; put operator-owned optional/required policy
above materialization and snapshot selection.

### 2. issue (blocking): Discovery can break otherwise usable image pulls

Referrer API errors abort the pull even before any signed bundle is selected.
Duplicate bundles and incomplete layer coverage also fail. The original OCI
image can remain usable. Successful discovery finding no bundle permits ordinary
unpack when no signed observation was previously retained.

**Location:** patch L1771-L1930.
**Remedy:** report signing-path failures and use ordinary OCI handling only when
operator policy explicitly permits it.

### 3. issue (blocking): Missing support can disable the selected snapshotter

Referrer enablement requires dm-verity support at initialization, even with
`"auto"`. Failure affects workloads selecting EROFS, not just signed images.

**Location:** patch L2798-L2806. The [profile selector][selector] has an audit
fallback for configuration failures, but does not exercise initialization or
signed mapper creation.
**Remedy:** preflight runtime prerequisites and test optional-mode base fallback.

### 4. issue (blocking for affected users): Unsupported existing paths

Local CRI pulls are rejected with referrer-capable snapshotters, even for
unsigned images. Signed image volumes are also rejected. These intentional
guards are compatibility blockers.

**Location:** patch L1159-L1166 and L1025-L1050.
**Remedy:** preserve optional-mode baseline behavior; surface required-mode
incompatibilities before activation.

## Proposed rollout behavior

Recommendation, not an implemented or approved design:

| Operator policy | Signing-path failure |
|---|---|
| Off | Preserve baseline image handling. |
| Optional / audit | Warn and record degradation; use a separate ordinary snapshot rebuilt from the original OCI layers. |
| Required | Fail the affected pull or mount; never silently downgrade. |

Never mount the failed signed filesystem raw, ignore corruption, or give an
unsigned snapshot a signed identity. Preserve OCI digest checks and unrelated
storage failures; clean up partial signed state. A generic mount retry is not enough.

The [loader][loader] leaves IPE inactive by default. Audit activation installs
the EROFS profile, enabling these risks.

Gate rollout on discovery failures, bad signatures, unknown keys, missing modules,
duplicate/partial bundles, local pulls, image volumes, and cached-snapshot rollback.

[patch]: https://github.com/microsoft/azure-container-linux/blob/30df5483fa40dc320a852546a2973effce5d4f99/acl/SPECS/containerd2/erofs-signed-dmverity.patch
[selector]: https://github.com/microsoft/azure-container-linux/blob/30df5483fa40dc320a852546a2973effce5d4f99/build_library/rpm/additional_files/containerd2/containerd-acl-select-profile#L235-L273
[loader]: https://github.com/microsoft/azure-container-linux/blob/30df5483fa40dc320a852546a2973effce5d4f99/build_library/rpm/additional_files/dracut-acl-ipe-load/acl-ipe-load.sh#L78-L125
