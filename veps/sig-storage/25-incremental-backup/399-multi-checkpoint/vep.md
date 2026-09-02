# VEP #399: Multi-checkpoint incremental backup

## VEP Status Metadata

### Target releases

- This VEP targets alpha for version: v1.10
- This VEP targets beta for version:
- This VEP targets GA for version:

### Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] : https://github.com/kubevirt/enhancements/issues/399
- [x] (R) Alpha target version is explicitly mentioned and approved
- [ ] (R) Beta target version is explicitly mentioned and approved
- [ ] (R) GA target version is explicitly mentioned and approved

## Table of contents

- [Overview](#overview)
- [Motivation](#motivation)
- [Goals](#goals)
- [Non Goals](#non-goals)
- [Definition of Users](#definition-of-users)
- [User Stories](#user-stories)
- [Repos](#repos)
- [Design](#design)
  - [Checkpoint semantics](#checkpoint-semantics)
  - [Checkpoint retention](#checkpoint-retention)
  - [Backing up from an older checkpoint](#backing-up-from-an-older-checkpoint)
  - [Redefinition after restart](#redefinition-after-restart)
  - [Cleanup](#cleanup)
- [API Examples](#api-examples)
  - [VirtualMachineBackupTracker CR](#virtualmachinebackuptracker-cr-changes)
  - [VirtualMachineBackup CR](#virtualmachinebackup-cr-changes)
- [Edge cases](#edge-cases)
- [Alternatives](#alternatives)
- [Scalability](#scalability)
  - [Bitmap count on the backend storage PVC](#bitmap-count-on-the-backend-storage-pvc)
- [Update/Rollback Compatibility](#updaterollback-compatibility)
- [Functional Testing Approach](#functional-testing-approach)
- [Open questions](#open-questions)
- [Known limitations](#known-limitations)
- [Implementation History](#implementation-history)
- [Graduation Requirements](#graduation-requirements)
  - [Alpha](#alpha)
  - [Beta](#beta)
    - [On-By-Default Readiness](#on-by-default-readiness)
  - [GA](#ga)

## Overview

Extends [VEP 25](../vep.md) incremental backup from a single retained checkpoint per `VirtualMachineBackupTracker` to several, and lets a `VirtualMachineBackup` name which retained checkpoint to use as its base.

## Motivation

A `VirtualMachineBackupTracker` retains one checkpoint, so the only possible incremental base is the one the most recent backup created.

When a vendor loses that backup for reasons outside KubeVirt (e.g., a failed upload, a corrupted object store, any fault on their side) the only base KubeVirt still offers is the one whose data is gone, and the next backup has to be full. On a multi-terabyte VM that is an expensive way to recover from someone else's bad day. Retaining a few checkpoints, and letting a backup pick among them, turns that full backup back into a delta.

## Goals

- Count-based retention of several checkpoints per `VirtualMachineBackupTracker`.
- Ability to base a `VirtualMachineBackup` on any retained checkpoint.
- Bitmap reclaim bounded by the retention count.

## Non Goals

- Checkpoint sharing across `VirtualMachineBackupTracker`s. Each chain is exclusive to its `VirtualMachineBackupTracker` by convention; nothing in this proposal enforces it at admission (see [Open questions](#open-questions)).
- Parallel backup chains within a single `VirtualMachineBackupTracker`.
- Age or TTL-based checkpoint retention.
- Per-disk checkpoint selection.
- Chain maintenance while the VM is stopped (see [VEP 401](../401-offline-incremental-backup/offline-incremental-backup.md)).

## Definition of Users

- Backup vendors: the primary consumer, integrating against the backup API.

## User Stories

- As a backup vendor, I want to retry an incremental backup from an older checkpoint when my latest backup data is lost, instead of falling back to full.
- As a backup vendor, I want to discard a chain I no longer trust and have its bitmaps reclaimed, without recreating my `VirtualMachineBackupTracker`.

## Repos

[KubeVirt](https://github.com/kubevirt/kubevirt)

## Design

### Checkpoint semantics

Checkpoint bitmaps are self-contained and mutually independent. Every bitmap therefore records from its own checkpoint's creation up to the present. For checkpoints `C1`, `C2` and `C3` taken in that order, $b(C3) \subseteq b(C2) \subseteq b(C1)$, and the delta since `C1` is `b(C1)` alone. Three consequences follow:

1. A backup from `C` needs only `b(C)`.
2. Deleting a checkpoint deletes its own bitmaps and nothing else.
3. The parent relationship of checkpoints is metadata only (nothing in the backup path reads it).

This independence is what keeps vendors out of each other's way, each `VirtualMachineBackupTracker` already owns its chain outright and pruning in one cannot invalidate another's bases.

Checkpoint names are already prefixed with the owning `VirtualMachineBackupTracker`'s name (`<tracker>-2006-01-02_15-04-05`), so checkpoints from different `VirtualMachineBackupTracker`s cannot collide and every bitmap on disk names its owner. Reclaim (see [Cleanup](#cleanup)) does not depend on that: it decides membership by comparing against the desired set, not by parsing names.

### Checkpoint retention

`VirtualMachineBackupTracker` gains `spec.retainCheckpoints` and an ordered `status.checkpoints` list alongside the existing `latestCheckpoint`.

`spec.retainCheckpoints` is the desired chain length, and the controller reconciles toward it on every sync rather than only when a backup completes. A completed backup appends its checkpoint, and anything past the count is pruned oldest first.

Pruning is deferred while a non-terminal `VirtualMachineBackup` references the same `VirtualMachineBackupTracker`, since removing a bitmap from under a running job is unsafe. The deferral is scoped to that `VirtualMachineBackupTracker` specifically, meaning a backup on one does not hold up pruning on another, because the bitmaps are disjoint. `status.checkpoints` is therefore a steady-state bound rather than an instantaneous one, which matters mostly in pull mode where a backup lives until its client deletes it or its TTL expires (see [Open questions](#open-questions)).

### Backing up from an older checkpoint

`VirtualMachineBackup` gains `spec.fromCheckpoint`, naming the checkpoint to use as the incremental base, unset means `latestCheckpoint`.

Resolution is per disk, a disk is incremental when the named checkpoint's bitmap is present and usable on it (as established in [VEP 25](../vep.md)), and full otherwise. Each checkpoint is judged on its own bitmap, so a disk missing `b(C1)` but still holding `b(C2)` is full for a `C1`-based backup and incremental for a `C2`-based one (i.e., partially covered chain degrades disk by disk instead of failing the backup).

### Redefinition after restart

Libvirt checkpoint metadata is transient, so it has to be redefined after a VM restart. [VEP 25](../vep.md) already detects this by recording the virt-launcher pod UID the checkpoints were last defined against in a `VirtualMachineBackupTracker`'s `status.lastTrackedPodUID`; this VEP changes only what gets redefined, the whole chain rather than a single checkpoint.

Checkpoints are redefined oldest to newest, each naming its predecessor in the retained chain as parent. The oldest carries no `<parent>` element, since pruning has already removed whatever it used to point at and libvirt rejects a redefinition naming a parent it has not seen. Libvirt therefore ends up with the retained chain rather than the one that existed before the restart, which is harmless because nothing in the backup path reads the parent relationship.

Missing or inconsistent bitmaps neither fail the pass nor truncate the chain; their consequence is deferred to backup start, per disk. Each checkpoint is redefined against exactly the disks that still carry its bitmap, since libvirt otherwise records metadata claiming bitmaps that are not there. One whose bitmap is gone from every disk has nothing left to redefine and is dropped from `status.checkpoints`, since it could only ever produce an all-`Full` backup and keeping the name offers a vendor a base that is not one.

A checkpoint libvirt permanently rejects (e.g., malformed XML, unknown disk) is dropped as well. A transient failure (launcher not reachable yet, a conflicting job) requeues with backoff and drops nothing, since dropping a checkpoint on an error that will clear destroys a base the vendor may still be holding data against.

`lastTrackedPodUID` advances only once the entire chain has been processed, so an interrupted pass is retried from the start; redefining an already-defined checkpoint is idempotent. Holding it back until the end is also what keeps backups off a half-redefined chain, with no new gate needed: [VEP 25](../vep.md) already blocks the VMI's changed block tracking state from leaving `Initializing` while any `VirtualMachineBackupTracker`'s `lastTrackedPodUID` is stale, and refuses a `VirtualMachineBackup` until it reaches `Enabled`. That gate simply widens from one checkpoint to the whole chain.

### Cleanup

Bitmaps live in the QCOW2 overlays on the backend storage PVC. The desired set for a VM is the union of every `VirtualMachineBackupTracker`'s `status.checkpoints` and the `status.checkpointName` of every non-terminal `VirtualMachineBackup` against that VM, so a checkpoint a backup has created but has not yet had appended to its `VirtualMachineBackupTracker` is not reclaimed out from under it. Anything else on disk (under the CBT path) is garbage, whatever produced it (e.g., a pruned checkpoint, a backup that failed after `BackupBegin`, a virt-launcher that terminated mid-job, a `VirtualMachineBackupTracker` deleted while the VM was down).

Removal goes through a new `DeleteCheckpoint` RPC to virt-launcher. Idempotency is a property of the sequence it runs rather than of the libvirt API, which errors both on a checkpoint it does not know and, at delete time, on a listed disk whose bitmap is missing (one such disk aborts the whole transaction). So the RPC first queries the disks for the checkpoint's bitmaps; where libvirt has lost the metadata it redefines the checkpoint against exactly the disks that still carry one, then deletes it. Bitmaps flagged `inconsistent` are removed directly via QMP, and a checkpoint no disk carries anymore reduces to dropping whatever metadata libvirt still holds.

To keep this level-triggered, the controller drives toward the desired set at reconciliation time while the VMI is running, on both the `VirtualMachineBackupTracker` and the `VirtualMachineBackup` path, and sweeps at VMI start by comparing the bitmaps present on each disk against the desired set. The sweep is what covers what the running path cannot: phantom checkpoints, orphaned bitmaps, and anything produced or abandoned while the VM was down.

`forceFullBackup` on a `VirtualMachineBackupTracker` source sets the desired set, on completion, to just the checkpoint that backup created, and the retained bases are then reclaimed like any other garbage. It is the escape hatch for a chain a vendor no longer trusts, so it discards the retained bases rather than appending to them. A forced backup that fails before creating its checkpoint leaves `status.checkpoints` unchanged, so the chain it was meant to replace survives and the vendor can retry.

A finalizer holds `VirtualMachineBackupTracker` deletion while a non-terminal `VirtualMachineBackup` references it, then reclaims its bitmaps if the VMI is running, leaves them to the next sweep if the VM is stopped, and clears the list outright once the VM and VMI are gone, before removing itself.

## API Examples

### VirtualMachineBackupTracker CR (changes)

**Spec:**
- `retainCheckpoints`: how many checkpoints to retain. Default `1`. Mutable.

**Status:**
- `checkpoints`: the retained checkpoints, oldest first, at most `retainCheckpoints` in steady-state.

```yaml
apiVersion: backup.kubevirt.io/v1alpha1
kind: VirtualMachineBackupTracker
metadata:
  name: my-vm-tracker
  namespace: default
  generation: 2
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: my-vm
  retainCheckpoints: 3
status:
  observedGeneration: 2
  checkpoints:
    - name: my-vm-tracker-2026-03-01_12-00-00
      creationTime: "2026-03-01T12:00:00Z"
    - name: my-vm-tracker-2026-03-02_12-00-00
      creationTime: "2026-03-02T12:00:00Z"
    - name: my-vm-tracker-2026-03-03_12-00-00
      creationTime: "2026-03-03T12:00:00Z"
  latestCheckpoint:
    name: my-vm-tracker-2026-03-03_12-00-00
    creationTime: "2026-03-03T12:00:00Z"
  lastTrackedPodUID: 7d3f1c1e-0b7a-4a1e-9a2b-5f0c1d2e3a4b
```

### VirtualMachineBackup CR (changes)

**Spec:**
- `fromCheckpoint`: retained checkpoint to use as the incremental base. Defaults to the `VirtualMachineBackupTracker`'s `latestCheckpoint`.

**Status:**
- `fromCheckpoint`: the base this `VirtualMachineBackup` actually ran against, reported even when the spec field was unset.

A backup based on the oldest retained checkpoint rather than the latest, where one disk lost its bitmap and fell back to full on its own:

```yaml
apiVersion: backup.kubevirt.io/v1alpha1
kind: VirtualMachineBackup
metadata:
  name: my-vm-differential
  namespace: default
spec:
  source:
    apiGroup: backup.kubevirt.io
    kind: VirtualMachineBackupTracker
    name: my-vm-tracker
  mode: Push
  pvcName: backup-target-pvc
  fromCheckpoint: my-vm-tracker-2026-03-01_12-00-00
status:
  checkpointName: my-vm-tracker-2026-03-04_12-00-00
  fromCheckpoint: my-vm-tracker-2026-03-01_12-00-00
  includedVolumes:
    - volumeName: rootdisk
      type: Incremental
    - volumeName: datadisk
      type: Full
```

## Edge cases

**Bitmap state**

| Scenario | Handling |
|---|---|
| Bitmap for the requested base absent on one disk | That disk is `Full`, the others stay `Incremental` |
| Bitmap present but flagged `inconsistent` | Counts as absent, per [VEP 25](../vep.md#vm-crash) VM crash semantics|
| Disk hot-plugged after the requested base was created | `Full` for that base, `Incremental` for any base created after the hotplug |
| Disk unplugged and replugged, or volume live-migrated | All its bitmaps are gone, so every base is `Full` until the next checkpoint |
| Unclean shutdown marks bitmaps `inconsistent` | All of a disk's bitmaps are affected together, so retention does not protect against a crash |
| A newer base is present but the requested one is not | Still `Full`; a newer delta cannot be applied to the vendor's older backup |

**Retention and pruning**

| Scenario | Handling |
|---|---|
| Retained checkpoint whose bitmap is gone from every disk | It holds no storage and can only produce an all-`Full` backup. Still listed and selectable while the VM runs; the next redefinition pass drops it from `status.checkpoints` |
| Prune requested while a `VirtualMachineBackup` on the same `VirtualMachineBackupTracker` is running | Deferred until that backup is terminal |
| Prune requested while a `VirtualMachineBackup` on a *different* `VirtualMachineBackupTracker` is running | Proceeds; the bitmaps are disjoint |
| `retainCheckpoints` lowered | Pruned to the new value on the next reconciliation |
| `retainCheckpoints` raised | Applies from the next backup; already-pruned checkpoints are unrecoverable |
| `forceFullBackup` on a `VirtualMachineBackupTracker` source | Every disk `Full`, chain purged, only the new checkpoint retained |

**Lifecycle**

| Scenario | Handling |
|---|---|
| VM restart | Chain redefined oldest to newest once `lastTrackedPodUID` differs from the current launcher pod |
| Live migration | Bitmaps transfer with the disks; the new pod UID triggers redefinition on the destination |
| Node failure, or virt-launcher killed | An unclean shutdown, so the VM's bitmaps may return `inconsistent` and each affected disk degrades to `Full`. Any in-flight `VirtualMachineBackup` fails and its checkpoint is reclaimed with the rest |
| Libvirt permanently rejects one checkpoint at redefinition | Dropped from `status.checkpoints`; its neighbours are unaffected |
| Redefinition fails transiently | The pass requeues with backoff and drops nothing; backups on that `VirtualMachineBackupTracker` are requeued until it completes |
| `VirtualMachineBackup` fails, is canceled, or hits TTL after `BackupBegin` | Its checkpoint is no longer in the desired set, so the [cleanup path](#cleanup) reclaims it: promptly while the VMI runs, otherwise at the next VMI start |
| Guest-initiated shutdown during a backup | Same path. The orphan survives in the QCOW2 overlay and the sweep reclaims it at the next VMI start |
| `VirtualMachineBackupTracker` deleted while the VM is stopped | Finalizer released immediately; its checkpoints leave the desired set, so the same sweep reclaims them at the next boot |

## Alternatives

A separate checkpoint CRD: a checkpoint has no lifecycle independent of its chain, ordering would have to be reconstructed from object metadata, and it multiplies API objects per VM per backup without adding expressiveness.

Falling back to the nearest newer usable base instead of full: a disk missing the requested base but holding a later one could produce a smaller delta, but the vendor asked for the base whose data they hold. Applying a later delta to it drops the writes in between, and the result looks valid.

## Scalability

### Bitmap count on the backend storage PVC

[VEP 25](../vep.md#scalability) sizes the backend storage PVC with a flat [`CBTBackendStateOverhead`](https://github.com/kubevirt/kubevirt/blob/92dff93d6f/pkg/storage/cbt/cbt.go#L41-L57) of 512 MiB, derived for 5 disks x 500 GiB with 256 KiB clusters, 64 KiB bitmap granularity and 10 bitmaps per disk. Retention turns that last term into a user-controlled value. This proposal does not bound it; whether that is an admission cap on `retainCheckpoints`, an overhead that scales with retention and geometry instead of a flat constant, or both, is left open (see [Open questions](#open-questions)).

What lands on a disk is the sum of `retainCheckpoints` across every `VirtualMachineBackupTracker` referencing the VM, not any single one's value, for example, three vendors retaining 10 each puts 30 bitmaps on every disk (see [Known limitations](#known-limitations)).

For the reference geometry each retained checkpoint costs roughly ~1.25 MiB of overlay per disk, `ceil(V / G / 8 / S) x S` of bitmap data plus one cluster for the bitmap table, so about ~6.25 MiB per checkpoint across 5 disks. QEMU additionally holds each bitmap in memory at `V / G / 8`, about 1 MiB per bitmap per disk, and every bitmap transfers with its disk during live migration.

## Update/Rollback Compatibility

All new fields are optional with backward-compatible defaults, and the feature is guarded by a new `MultiCheckpointIncrementalBackup` feature gate.

**Upgrade**: existing `VirtualMachineBackupTracker`s keep working. `status.checkpoints` is populated from `latestCheckpoint` on first reconciliation and `retainCheckpoints` defaults to 1, so an upgraded `VirtualMachineBackupTracker` behaves exactly as before until a vendor opts into a larger value. Raising `retainCheckpoints` takes effect from the next backup onward and cannot recover checkpoints already pruned.

**Rollback**: an older controller ignores `status.checkpoints` and uses `latestCheckpoint`, and ignores `fromCheckpoint` on in-flight backups. Deltas from `latestCheckpoint` stay correct, because that bitmap is self-contained and does not depend on the checkpoints the old controller stopped tracking. The other retained checkpoints' bitmaps remain in the QCOW2 overlays, and an older controller has neither the desired-set reclaim nor the sweep to remove them, so the cost of a rollback is wasted space rather than a broken chain.

## Functional Testing Approach

- After N backups with `retainCheckpoints: K`, the `VirtualMachineBackupTracker` lists `min(N, K)` checkpoints and each disk carries the matching number of bitmaps.
- A backup from each retained checkpoint produces data consistent with a full backup taken at that point.
- Pruning a middle checkpoint leaves the remaining bases usable, confirming that bitmaps are independent.
- Two `VirtualMachineBackupTracker`s on one VM: pruning and `forceFullBackup` on one leave the other's bases intact, and a backup on one does not defer pruning on the other.
- Lowering `retainCheckpoints` on a running VM prunes to the new value and advances `status.observedGeneration`.
- Unplugging and replugging a disk of a running VM destroys its bitmap; a subsequent `fromCheckpoint` backup reports that disk as `Full` and the others as `Incremental`.
- Chain redefinition after VM restart and after live migration, at the maximum retention count, with incremental backups resuming from the oldest retained checkpoint.
- A `VirtualMachineBackup` created while redefinition is still in progress is held by the existing changed block tracking gate until the whole chain is redefined, then runs incrementally.
- A checkpoint whose bitmap is gone from every disk is dropped from `status.checkpoints` at the next redefinition pass, and the checkpoints that kept theirs are not.
- A checkpoint created by an in-flight `VirtualMachineBackup` but not yet appended to its `VirtualMachineBackupTracker` survives a concurrent reclaim on that VM.
- Redefinition with a partially covered chain excludes the affected disk from that checkpoint only and does not truncate.
- Killing virt-launcher mid-backup leaves an orphan checkpoint that the sweep reclaims at the next VMI start, and the surviving bases still produce correct deltas.
- Pruning under a continuous pull-mode cadence returns `status.checkpoints` to `retainCheckpoints` between backups.
- `forceFullBackup` leaves exactly one checkpoint and one bitmap per disk.
- `VirtualMachineBackupTracker` deletion removes all bitmaps when the VM is running, defers to the sweep when it is stopped, and clears checkpoints when the VM is gone.

## Open questions

- **Bounding `retainCheckpoints`.** Nothing caps it, and what lands on a disk is the sum across every `VirtualMachineBackupTracker` referencing the VM (see [Scalability](#scalability)). An admission cap is the blunt fix; making `CBTBackendStateOverhead` a function of retention, disk count and disk size instead of a flat constant is the accurate one, and only the latter also fixes the VM that exceeds the reference geometry with `retainCheckpoints: 1`.
- **Validating `fromCheckpoint` at admission.** Rejecting a name absent from `status.checkpoints` catches vendor typos before they cost a full backup, but `status.checkpoints` is a cache: the checkpoint can be pruned between admission and backup start regardless, so the backup path has to handle the missing case either way. The level-triggered reading argues for accepting the field and resolving it at start; usability argues for both.
- **Narrowing the prune deferral.** Pruning is currently deferred for a whole `VirtualMachineBackupTracker` while any non-terminal `VirtualMachineBackup` references it. Only two checkpoints actually need pinning: the one that backup reads and the one it created. Deferring per checkpoint instead of per `VirtualMachineBackupTracker` would keep the chain at `retainCheckpoints` even under a pull-mode client that holds a backup open to its TTL, which is the case where the current rule lets it grow (see [Known limitations](#known-limitations)).
- **Fail-closed behaviour for the sweep.** When the sweep cannot enumerate the bitmaps on a disk, reclaiming on incomplete information can destroy a base a vendor holds data against, while skipping the disk leaks bitmaps until the next boot. Skipping is the safer default, but it needs to be observable rather than silent.
- **Enforcing chain exclusivity.** Checkpoint names carry their owner's prefix, so a `VirtualMachineBackup` naming a `fromCheckpoint` belonging to a different `VirtualMachineBackupTracker` than its `spec.source` is detectable at admission. This proposal leaves exclusivity a convention (see [Non Goals](#non-goals)); whether to enforce it depends on whether cross-`VirtualMachineBackupTracker` bases are worth keeping as an escape hatch.
- **Naming of `forceFullBackup`.** The field now means "discard the retained chain and start over", which is a `VirtualMachineBackupTracker`-level action expressed on the `VirtualMachineBackup`. Something like `resetChain` on the `VirtualMachineBackupTracker` would say what it does declaratively, at the cost of a second way to spell an existing [VEP 25](../vep.md) field.
- **Reporting per-checkpoint usability.** `status.checkpoints` lists names, not which disks still carry each bitmap, so a vendor cannot distinguish a base that will produce a delta from one that will degrade to full until the backup runs. Surfacing it turns the status into a report of live disk state, which is expensive to keep accurate and stale the moment it is written.

## Known limitations

- **Per-VM retention**: `retainCheckpoints` caps one `VirtualMachineBackupTracker`'s chain, but the bitmaps on a disk are the sum across every `VirtualMachineBackupTracker` referencing the VM. `CBTBackendStateOverhead` is a flat constant that scales with neither disk size nor disk count, so a VM with many of them, many disks, or disks larger than the reference geometry can exceed what its backend storage was sized for.
- **Crash resilience**: an unclean shutdown marks a disk's bitmaps `inconsistent` together, so every retained base for that disk degrades to full at once. On a VM where every disk is affected, no base retains a usable bitmap anywhere and the next redefinition pass drops the whole chain from `status.checkpoints`.
- **Deferred pruning**: a chain can exceed `retainCheckpoints` while a backup is in flight, so the storage bound is a steady-state one. It is observable by comparing `status.checkpoints` against `spec.retainCheckpoints`.
- **Offline chain maintenance**: pruning, `VirtualMachineBackupTracker` deletion cleanup and orphan reclaim all defer to the next boot while the VM is stopped, pending [VEP 401](../401-offline-incremental-backup/offline-incremental-backup.md).

## Implementation History

TBD.

## Graduation Requirements

### Alpha

- [ ] `IncrementalBackup` graduated to beta
- [ ] Count-based retention reconciled toward `spec.retainCheckpoints`, with the retained chain and `observedGeneration` reported on the `VirtualMachineBackupTracker` status
- [ ] `fromCheckpoint` selecting any retained checkpoint as the incremental base, resolved by the controller at backup start and reported on `status.fromCheckpoint`
- [ ] `forceFullBackup` purging the retained chain
- [ ] Chain redefinition after VM restart and live migration
- [ ] Bitmap reclaim via `DeleteCheckpoint`, driven from the retained set, with a sweep at VMI start and a `VirtualMachineBackupTracker` finalizer

### Beta

TBD.

#### On-By-Default Readiness

TBD.

### GA

TBD.
