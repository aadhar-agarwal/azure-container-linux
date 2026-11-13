---
title: Signed EROFS dm-verity availability review
type: project
updated: 2026-11-13
---

# Signed EROFS dm-verity availability review

Source: [`erofs-signed-dmverity.patch` at `aclmain@30df5483`][patch].
Locations below are patch lines unless otherwise identified. This revision
incorporates independent cross-review and focused synthetic checks; it is not
live AKS/kernel qualification or evidence of a production outage.

## Verdict

**Signed mount failures can block AKS node provisioning even with IPE in audit
mode.** If the affected image is `pause`, new pod sandboxes cannot start.
IPE audit does not relax containerd or kernel signature checks.

**The clearest recovery defect is stale cached signatures:** a corrected
signature can be accepted during re-pull without replacing the signature used
by existing signed snapshots.

Strict verification is appropriate when required. The rollout concern is making
signing an availability dependency merely by enabling IPE audit, without a tested
renewal and recovery path.

## AKS impact

The [sandbox path][sandbox] uses the patched `customopts.WithNewSnapshot` for
`pause`. [Task creation][task-create] activates mounts before starting the shim.
An image can therefore pass unpack checks but fail kernel signature verification
at mount time, before the container executes.

| Failure | Availability boundary |
|---|---|
| Signed `pause` cannot mount | New sandboxes using that image fail, including system pods. Provisioning, scale-out, and node replacement can stall. |
| Another signed system image fails | The component cannot start; wider network/storage/bootstrap effects depend on its role. |
| Signed application image fails | Ordinarily affects new/restarted containers using that image. |
| Configured EROFS snapshotter cannot initialize | Can also prevent CRI image-service initialization, affecting unsigned workloads. |
| Signature/referrer changes while containers are running | Existing mounted containers generally continue. Subsequent pulls, fresh mounts, or reboot expose failures. |

A pause failure does **not** automatically set `RuntimeReady=false` or make every
existing node `NotReady`: the [CRI status check][status] does not test pause
mounting. A ready node can still fail new pods. This is not an inherent host crash.
Actual data corruption is different: dm-verity checks reads and can produce
[I/O failures after startup][verity].

## Concerns

### 1. issue (blocking for renewal/recovery): Corrected signatures remain unused

An image is materialized with root H and signature S1. The publisher replaces
its bundle with the same H and a usable S2. Re-pull accepts the replacement,
but the snapshot key excludes the signature digest and cache validation ignores
the change. Application is skipped, preserving S1 in both snapshot labels and
the local `.sig` file.

If S1 is unusable for a fresh mapper, mounts keep failing despite the corrective
pull. A fresh snapshot cache can work while an older cache fails. An active
mapper can mask this until teardown/reboot. Restoring trust in S1 is a different
recovery path and may work without replacing the snapshot.

**Location:** key L1484-L1495; validation L1596-L1620; accepted replacement
L2056-L2077; snapshot labels used for mounting L2845-L2857.
**Remedy:** version signed snapshot keys/chains by signature identity, or design
an equivalent safe renewal mechanism. Never overwrite or evict snapshots
backing running containers.

### 2. issue (blocking for publisher compatibility): Referrer updates can break pulls

Referrers attach to a manifest digest, not a tag. Pinning the image digest does
not freeze its attached referrers. A new image digest is different from
re-signing or rerunning publication for the same digest.

| Publisher change | Current behavior |
|---|---|
| Add a second distinct matching dm-verity bundle | Pull fails before consulting the retained bundle, even when one candidate was usable. |
| Replace with one bundle having the same roots | Accepted, but encounters the stale-signature behavior above. |
| Replace with one bundle having different roots | Previously observing nodes reject it; fresh nodes can accept the sole current bundle. |
| Remove all referrers | Previously observing nodes preserve their signed selection. This is not a universal rollback. |

Only distinct matching bundle digests trigger the duplicate rejection. Repeated
identical descriptors and unrelated artifacts such as SBOMs do not.

**Location:** discovery L1789-L1815; replacement L2056-L2077; retained selection
L1987-L2020 and L2050-L2053.
**Remedy:** maintain one canonical, idempotently published bundle while current
clients reject duplicates. A replacement protocol needs a compatible client
rollout first. Pushing a new bundle before deleting the old creates a pull-failure
window; deleting first creates an unsigned-discovery window on fresh nodes.

**"Pick newest" alone is insufficient.** [OCI referrers][referrers-spec] have no
guaranteed chronological ordering or mandatory trusted timestamp. A deterministic
choice can still select a signature the node cannot trust. Selection must account
for trust and conflicting roots, and must be paired with signature-versioned
materialization. Do not silently accept a changed root merely because it is newer.

### 3. issue (rollout-policy): Audit is not optional signature enforcement

Missing trust, invalid signatures, or unreadable signature files can block
startup. This is deliberate fail-closed behavior, not itself an implementation
bug. `dmverity_mode = "auto"` does not retry signed failures unsigned.

Disabling referrers is not a hot recovery switch: existing signed parents reject
new mounts. Combining referrers with `dmverity_mode = "off"` prevents snapshotter
initialization. Existing running mounts are not automatically torn down.

**Location:** mount failures L2574-L2586 and L2615-L2648; configuration
L2793-L2806; disabled signed-parent rejection L2845-L2857.
**Remedy:** separate optional/required signing policy from IPE execution mode.
Qualify migration to separate ordinary snapshots and container recreation.

### 4. issue (availability and coverage): Discovery failures have mixed outcomes

A returned `FetchReferrers` error aborts the pull, including for otherwise usable
unsigned images. Malformed/incomplete bundles also fail. However, the upstream
[Docker fetcher][fetcher] can turn an API error followed by a fallback-tag 404
into an empty result.

Focused synthetic results against the reviewed source:

| Referrers API / fallback response | Result |
|---|---|
| 404/404, 405/404, or 500/404 | Empty successful discovery |
| 403/403 or 500/500 | Error; patch aborts pull |
| Malformed JSON with API 200 | Error; patch aborts pull |
| Valid empty index with API 200 | Empty successful discovery |

Empty discovery permits ordinary unsigned handling when no signed observation
was previously retained; it does not clear an existing signed observation.
Neither outcome's production frequency was measured.

**Location:** patch L1781-L1788, L1922-L1929, L1982-L2020; upstream fetcher
`core/remotes/docker/referrers.go`.
**Remedy:** distinguish unknown discovery state from confirmed absence. Make
degradation policy explicit rather than treating every error as either fatal
or unsigned.

### 5. issue (non-blocking): Mutable-tag resolution can reject a cached image ID

The new CRI resolver can receive a resolved image ID plus `UserSpecifiedImage`.
If another pull moves that tag, or the tag disappears before container creation,
the helper rejects the request despite the old immutable image remaining cached.
This path applies to unsigned images on a referrer-capable snapshotter too.
It is generally a retryable startup failure, not a permanent node failure.

**Location:** `dmverityContainerImage`, patch L790-L908.
**Remedy:** preserve the requested immutable identity; use an ID-based fallback
only when the retained manifest/materialization is unambiguous.

## Compatibility boundaries

- The [loader][loader] defaults IPE off. Successful audit activation installs the
  EROFS profile. The `ipe=enforcing` tag is not supported by this loader.
- ACL already [falls back to its base profile][selector] in audit mode for
  incompatible local-pull configuration. The service attempts module loading.
  Neither guard tests actual kernel signer trust or a signed pause mount.
- Signed image volumes and signed native EROFS layers are explicitly unsupported
  (patch L1025-L1050 and L2300-L2304).
- Preloaded images without a signed observation can remain unsigned without
  discovery. Warm-cache success does not prove a fresh signed pull/mount works.
- The shared-mapper first-mount race is inherited from upstream unsigned
  dm-verity handling, not a new patch-introduced defect.
- The unconditional `verity` import also breaks non-Linux `pkg/snapshotters`
  builds through its Linux-only keyring dependency. This is a portability issue,
  not an ACL Linux runtime failure.

## Proposed rollout behavior

Recommendation, not an implemented or approved design. IPE audit alone must not
be treated as authorization to downgrade a required signed mount.

| Operator policy | Signing-path failure |
|---|---|
| Off | Preserve baseline image handling, with a tested transition for existing signed snapshots. |
| Explicitly optional signing | Warn and record degradation; use a separate ordinary snapshot rebuilt from the original OCI layers. |
| Required | Fail the affected pull or mount; never silently downgrade. |

Never mount the failed signed filesystem raw, ignore corruption, or give an
unsigned snapshot a signed identity. Preserve OCI digest checks and unrelated
storage failures; clean up partial signed state. A generic mount retry is not enough.

Keep activation controlled until renewal and recovery are qualified. Validate
the real pause image against the actual kernel trust available on each supported
image-definition path, not just certificate presence in a publishing artifact.

Gate rollout on corrected signatures, old/new trust overlap, duplicate and
root-changing bundles, registry failures, offline preloads, fresh versus existing
snapshots, active mappers, reboot, and rollback during node replacement.

## Evidence limits

The registry cases above were reproduced, and an isolated test of unmodified
`ValidateDmveritySnapshot` confirmed that same-root/different-signature cache
validation succeeds without updating the old identity. These demonstrate code
behavior, not Linux signature verification.

No live AKS/kernel mount test or full production RPM qualification was performed.
Actual deployed keyrings, registry failure frequency, and publisher rerun
behavior remain to be verified. SELinux mount-label fixes do not alter the
signature verification or referrer replacement paths reviewed here.

[patch]: https://github.com/microsoft/azure-container-linux/blob/30df5483fa40dc320a852546a2973effce5d4f99/acl/SPECS/containerd2/erofs-signed-dmverity.patch
[selector]: https://github.com/microsoft/azure-container-linux/blob/30df5483fa40dc320a852546a2973effce5d4f99/build_library/rpm/additional_files/containerd2/containerd-acl-select-profile#L235-L273
[loader]: https://github.com/microsoft/azure-container-linux/blob/30df5483fa40dc320a852546a2973effce5d4f99/build_library/rpm/additional_files/dracut-acl-ipe-load/acl-ipe-load.sh#L78-L125
[sandbox]: https://github.com/containerd/containerd/blob/v2.3.7/internal/cri/server/podsandbox/sandbox_run.go#L195-L255
[task-create]: https://github.com/containerd/containerd/blob/v2.3.7/core/runtime/v2/task_manager.go#L188-L211
[status]: https://github.com/containerd/containerd/blob/v2.3.7/internal/cri/server/status.go#L37-L55
[fetcher]: https://github.com/containerd/containerd/blob/v2.3.7/core/remotes/docker/referrers.go#L35-L166
[referrers-spec]: https://github.com/opencontainers/distribution-spec/blob/v1.1.1/spec.md#listing-referrers
[verity]: https://docs.kernel.org/admin-guide/device-mapper/verity.html#theory-of-operation
