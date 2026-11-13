---
title: Signed EROFS dm-verity availability review
type: project
updated: 2026-11-13
---

# Signed container checks: what can break and what to fix first

**Bottom line:** signing failures can stop new containers, even in IPE audit
mode. If the affected container is `pause`, new pods cannot start and AKS node
provisioning can stall. The first code fix should make corrected signatures
actually replace the old signatures used by cached images.

These risks apply when the signed-container path is selected. The reviewed ACL
loader does not enable it by default; enabling IPE audit can select it.

Reviewed and rechecked against [`aclmain@30df5483`][patch]. The code is unchanged
since the independent cross-review. Findings below distinguish confirmed bugs,
intentional restrictions, and rollout decisions. This is not a report of a
verified production outage.

## Enough context to read this review

| Term | Plain-language meaning |
|---|---|
| Container image | The files needed to run an application. |
| Containerd | Software on the node that downloads ("pulls") images and starts containers. |
| Signature and certificate | A digital approval stamp, and information the machine uses to recognize a trusted signer. |
| Signing bundle / referrer | A separate attachment containing the signature and information needed to build and verify the protected container filesystem. |
| Local snapshot | Containerd's saved, unpacked copy of an image, reused when starting containers. |
| `pause` | A small helper container Kubernetes needs to start normal pods. Its failure has a wider impact than one application failing. |

For images using this signing path, Linux checks the signature when opening
the container filesystem. **IPE audit controls a different, later execution
check. It does not turn a failed signature check into a warning.**

## Recommended fix order

The order below is a recommendation, not an approved implementation plan:

| Order | What to address | Validity and priority |
|---|---|---|
| **1** | Corrected signatures are ignored by cached images | **Confirmed recovery defect.** Re-pulling can succeed without fixing the failing start. |
| **2** | Publishing another signing bundle breaks pulls | **Confirmed intentional restriction.** Define safe publishing and replacement before allowing retries or signing-key changes to create different bundles. |
| **3** | Audit behavior and recovery are not clearly separated | **Confirmed design/rollout gap, not a broken verifier.** Decide whether signing is optional and provide a tested transition. |
| **4** | Signing-discovery failures have inconsistent outcomes | **Confirmed integration behavior, partly inherited.** Distinguish an unavailable check from a confirmed absence of signing information. |
| **5** | A changing image tag can reject an otherwise cached image | **Source-confirmed startup regression.** Usually retryable; lower priority than shared pause or signature-recovery failures. |

## 1. A fixed signature can be ignored

**issue (blocking for recovery): confirmed defect**

A node saves an image with signature A. That signature cannot be trusted.
The publisher supplies a valid signature B for the same filesystem. The node
downloads the update, but containerd reuses its saved filesystem **with
signature A still attached**.

The code decides whether to reuse the saved copy using the filesystem
fingerprint, called its *root hash*. It deliberately ignores a changed
signature. Updating the downloaded bundle does not update the saved `.sig` file.

**Why it matters:** a fresh node can work while an older node keeps failing,
even after a successful re-pull. A currently mounted copy can hide the problem
until it is mounted again or the node reboots. Restoring trust in A is a
different recovery path that may also work.

**Fix first:** give different signatures separate saved-snapshot identities
and chains, or implement an equivalent safe renewal mechanism. Do not overwrite
or delete a copy that a running container is using.

**Evidence:** patch L1484-L1495, L1596-L1620, L2056-L2077, L2845-L2857.
An isolated test confirmed that cache validation accepts the old signature
when the root hash is unchanged. The Linux mount failure was not reproduced.

## 2. Two signing attachments can block the same image

**issue (blocking for publisher compatibility): valid concern, intentional behavior**

This means two different attachments for **one exact image**, not two container
versions:

```text
Same container image
  +-- Bundle A: original signing information
  +-- Bundle B: different signing information, perhaps after re-signing
```

The patch refuses to choose between them, so new pulls fail. This could happen
if a publishing rerun uploads a different bundle without removing the old one.
We have not verified that the publisher currently does this.

A genuinely new image version normally has a different identity and its own
bundle. Repeating an identical bundle, or attaching unrelated metadata, does
not trigger this duplicate-bundle rejection.

There are also replacement traps:

| Change to the same image's attachments | Result today |
|---|---|
| Add a second distinct signing bundle | Pull fails, even if a previously used bundle is still available. |
| Replace the only bundle, keeping the same filesystem fingerprint | Accepted, but issue 1 can keep the old signature in use. |
| Replace it with a different filesystem fingerprint | Nodes remembering the old bundle reject it; fresh nodes can accept it. |
| Remove the bundles entirely | Nodes remembering a signed bundle keep using it. Removal is not a universal rollback. |

**Next fix:** keep publishing repeatable without creating extra bundles while
current clients require one. Then design replacement together with issue 1.
Adding the new bundle before deleting the old creates a failure window;
deleting first can make fresh nodes treat the image as unsigned.

**Do not just select "newest."** Registry order is not a reliable clock, a
timestamp does not prove trust, and a newer signature may be untrusted on
older nodes. A safe rule must handle trusted signatures, conflicting filesystem
fingerprints, and the saved copies already on nodes.

**Evidence:** patch L1789-L1815, L1987-L2020, L2050-L2077; [OCI referrers][referrers-spec].

## 3. Audit mode does not provide a safe escape hatch

**issue (rollout-policy): valid risk, not a signature-verification bug**

If a node cannot recognize a signer, it refuses to open the signed filesystem.
That is correct when signing is required. The surprise is that enabling IPE
audit also enables this runtime path, although audit sounds non-disruptive.

Simply turning the feature off is not enough either. Existing saved copies
marked as signed reject new mounts when referrer support is disabled.
Setting `dmverity_mode = "off"` while referrers remain enabled prevents the
EROFS filesystem plugin from initializing. These switches do not automatically
tear down already-running containers.

**Decision needed:** separate the signing policy from IPE audit mode:

| Explicit operator choice | Proposed behavior on a signing-path failure |
|---|---|
| Signing required | Refuse to start; never silently weaken verification. |
| Signing optional | Report the failure and use a separate ordinary copy rebuilt from the original image, if that downgrade is authorized. |

This fallback is **not implemented today**. Test how existing containers and
saved images move to the ordinary path during rollback. Never bypass checks on
the failed signed filesystem itself or reuse its signed identity for unsigned data.

**Evidence:** patch L2574-L2586, L2615-L2648, L2793-L2806, L2845-L2857;
[boot-time activation][loader].

## 4. Checking for signing information can itself fail

**issue (availability and coverage): confirmed, but not every registry error is fatal**

Containerd makes extra requests to the image registry (the server hosting
images) to find signing bundles. The original image may be available while
those requests fail.

Some failures abort the pull, including for unsigned images. Others become
"no bundle found," allowing ordinary unsigned handling on a node that has
not previously remembered a signed bundle. That existing signed record is not
cleared by an empty lookup.

**Fix after agreeing on policy:** distinguish "the check was unavailable" from
"the check succeeded and found nothing." Record degradation visibly. Do not
blindly treat every error as either fatal or permission to run unsigned.
Part of this behavior comes from upstream containerd's registry client,
not new code in this patch.

**Evidence:** patch L1781-L1788, L1922-L1929, L1982-L2020; [registry client][fetcher].
The response combinations in the appendix were reproduced.

## 5. A moving image name can cause a retryable startup failure

**issue (non-blocking): source-confirmed regression**

Kubernetes resolves `app:stable` to image A. Before the container is created,
another pull moves that name to image B. The new helper looks up the name again
and rejects the mismatch, even if the exact image A remains saved locally.
Removing the name can produce a similar failure.

This affects unsigned images too when the new snapshotter capability is active.
It is a narrower startup race, not evidence that a node is permanently broken.

**Fix:** preserve the exact image already requested. Only fall back to another
saved reference when it unambiguously identifies the same image and signed
filesystem; do not silently select a different signed version.

**Evidence:** `dmverityContainerImage`, patch L790-L908. Source path confirmed;
no live Kubernetes race reproduction.

## What this does and does not mean for AKS

| Situation | Expected scope |
|---|---|
| `pause` cannot open its filesystem | New pod sandboxes using it fail, including essential system pods. Fresh-node provisioning and node replacement can stall. |
| One application image fails | Usually that application's new/restarted containers, not every workload. |
| Containers are already running | They generally continue through signing-metadata changes. A new mount or reboot can reveal the problem later. |
| Existing node reports `Ready` | This does not prove it can start pause. The runtime status check does not attempt a pause mount. |

These are not inherent host crashes. Actual data corruption is a different case:
dm-verity checks reads and can cause [I/O failures after startup][verity].
See the [sandbox startup][sandbox], [task creation][task-create], and
[runtime status][status] paths.

## Other checked items: do not confuse these with new bugs

| Item | Assessment |
|---|---|
| Kernel feature unavailable | Referrer-enabled EROFS can fail initialization. The service attempts module loading, but this does not prove signer trust. |
| Incompatible local-pull configuration | ACL already has an [audit-mode base-profile fallback][selector]. Do not claim this is wholly unguarded. |
| Signed image volumes or native EROFS image layers | Explicitly unsupported paths, not accidental verification failures. |
| Preloaded images that never discovered a bundle | Can remain unsigned. Successful warm-cache tests do not qualify a fresh signed pull. |
| Concurrent first-mount race | Inherited from unsigned dm-verity handling; not introduced by this patch. |
| Non-Linux builds | New unconditional dependency on Linux keyring code breaks portability. Lower priority for this Linux-only rollout. |

## Verification and release gates

The source and earlier focused test results were rechecked. The core signature
recovery finding was independently identified by all four requested reviewers.
Other claims were narrowed or reclassified rather than counting every rejection
as a bug.

Before wider rollout, test the real pause image with missing trust, corrected
signatures, overlapping old/new signing certificates, duplicate bundles,
registry errors, offline preloads, fresh and existing saved copies, reboot,
and rollback during node replacement.

**Still unverified:** live AKS/kernel mounts, actual deployed trust, production
registry failure rates, and publisher rerun behavior. No full production RPM
qualification was performed. Fixing SELinux mount labels does not fix these
signature-renewal or registry-selection paths.

<details>
<summary>Technical appendix: reproduced registry behavior</summary>

The registry client can try both the referrers API and a fallback tag lookup.
These results are for the reviewed source, not measurements of MCR:

| API / fallback response | Result |
|---|---|
| 404/404, 405/404, or 500/404 | Empty successful discovery |
| 403/403 or 500/500 | Error; patch aborts pull |
| Malformed JSON with API 200 | Error; patch aborts pull |
| Valid empty index with API 200 | Empty successful discovery |

The signature-cache test used unmodified `ValidateDmveritySnapshot` and showed
that different signature identities with the same root hash are accepted without
updating the old identity. It did not exercise Linux signature verification.

Current configuration details: the loader defaults IPE off, activates the EROFS
profile after successful audit activation, and does not support the
`ipe=enforcing` runtime tag. `dmverity_mode = "auto"` is not an unsigned retry
for failed signed mounts.

</details>

[patch]: https://github.com/microsoft/azure-container-linux/blob/30df5483fa40dc320a852546a2973effce5d4f99/acl/SPECS/containerd2/erofs-signed-dmverity.patch
[selector]: https://github.com/microsoft/azure-container-linux/blob/30df5483fa40dc320a852546a2973effce5d4f99/build_library/rpm/additional_files/containerd2/containerd-acl-select-profile#L235-L273
[loader]: https://github.com/microsoft/azure-container-linux/blob/30df5483fa40dc320a852546a2973effce5d4f99/build_library/rpm/additional_files/dracut-acl-ipe-load/acl-ipe-load.sh#L78-L125
[sandbox]: https://github.com/containerd/containerd/blob/v2.3.7/internal/cri/server/podsandbox/sandbox_run.go#L195-L255
[task-create]: https://github.com/containerd/containerd/blob/v2.3.7/core/runtime/v2/task_manager.go#L188-L211
[status]: https://github.com/containerd/containerd/blob/v2.3.7/internal/cri/server/status.go#L37-L55
[fetcher]: https://github.com/containerd/containerd/blob/v2.3.7/core/remotes/docker/referrers.go#L35-L166
[referrers-spec]: https://github.com/opencontainers/distribution-spec/blob/v1.1.1/spec.md#listing-referrers
[verity]: https://docs.kernel.org/admin-guide/device-mapper/verity.html#theory-of-operation
