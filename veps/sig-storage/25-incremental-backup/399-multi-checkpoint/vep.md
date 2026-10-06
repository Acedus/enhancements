# VEP #399: Unbounded incremental backup history

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

## Overview

[VEP 25](../vep.md) lets a backup be incremental only from the checkpoint the previous backup created. This VEP lets a backup be incremental from any earlier backup.

It does so without keeping a libvirt checkpoint per base. A VM keeps exactly one checkpoint, and KubeVirt keeps a per-disk *generation map* on the VM state PVC recording, for each 64 KiB extent, the backup in which it last changed. "Changed since backup `G`" is the map entries above `G` unioned with the live checkpoint's bitmap. Every backup is a libvirt pull backup underneath, because that is the only way to obtain the bitmap the map is built from. Push mode becomes a KubeVirt writer on top of it.

Consumers receive an opaque token per backup and pass it back to name a base. The cost of history is a fixed-size map per disk, independent of depth and of the number of trackers.

## Motivation

A `VirtualMachineBackupTracker` holds one checkpoint, so the only possible base is the latest backup. When a vendor loses that backup (failed upload, corrupted object store), the next backup must be full. On a multi-terabyte VM that is the expensive way to recover. Vendors also want a long-lived base, such as a monthly full, to stay usable without paying for every intermediate point.

The obvious fix, retaining N checkpoints per tracker, has two problems.

**It depends on libvirt behaviour that libvirt does not promise.** Retained checkpoints only work as independent bases if each one owns an always-recording bitmap and deleting one leaves the others untouched. Both hold in today's QEMU driver, and neither is documented. libvirt maintainers confirmed (2026-10-05) that libvirt may change how it arranges bitmaps across checkpoints, for example by disabling or chaining older ones. This design relies only on documented behaviour for every decision:

| Behaviour relied on | Status |
| --- | --- |
| `virDomainBackupBegin` with `checkpointXML` creates the checkpoint at the same point in time as the backup | documented |
| A pull export advertises the blocks changed since the `incremental` checkpoint, named by `exportbitmap` | documented ([formatbackup](https://libvirt.org/formatbackup.html)) |
| `REDEFINE` restores a checkpoint from saved XML; `REDEFINE_VALIDATE` fails if its disk state is unusable | documented, since 6.10.0 |
| Bitmaps of recording checkpoints migrate with the domain | implementation; the fallback is a full backup of the affected disks |

With one checkpoint alive at a time, how libvirt arranges bitmaps *across* checkpoints never matters.

**Its cost scales with the wrong things.** Each retained base is a persistent bitmap per disk: resident in QEMU, updated on every guest write, and migrated. The count is the sum of the retention limits of every tracker on the VM. Overflowing the state PVC surfaces at migration switchover, not at backup time. Depth is bounded by storage, not by policy.

VMware CBT and Hyper-V RCT solve the same problem the same way: a per-disk change file holding a per-block sequence number, queried with a sequence the caller kept.

## Goals

- Use any earlier backup of a VM as an incremental base, with depth bounded by policy rather than storage.
- One libvirt checkpoint and one dirty bitmap per disk, regardless of history depth or tracker count.
- Every backup decision made from documented libvirt API, with no QMP passthrough on any backup path.
- Per-volume reporting of why a volume was backed up in full, and an option to fail instead.
- Differential backups (repeated backups against a fixed base) without extending history.
- An explicit way to declare the tracked history invalid.

## Non Goals

- A different base per disk within one backup. The base applies to the whole VM, and individual disks degrade to full on their own.
- Several independent histories per VM. All trackers of a VM share one history.
- Time-based expiry of history.
- Detecting writes made outside KubeVirt's view (for example `guestfs` or an in-place re-import). Those must be declared with `resetChangeTracking`.
- Maintaining history while the VM is stopped, beyond what [VEP 401](../401-offline-incremental-backup/offline-incremental-backup.md) does.

## Definition of Users

- Backup vendors integrating against the backup API.
- Cluster operators who modify VM disks out of band.

## User Stories

- As a backup vendor, when my latest backup is lost, I want to back up incrementally against an older one I still hold instead of taking a full.
- As a backup vendor, I want a monthly base to stay usable for a month without the platform paying for every daily point in between.
- As a backup vendor, I want one opaque token per backup that I store with my data and hand back later.
- As a backup vendor, I want to know per volume why a backup was full, or to have the backup fail instead.
- As an operator, after writing to a VM's disks out of band, I want to declare its change tracking invalid so no vendor builds on it.

## Repos

[KubeVirt](https://github.com/kubevirt/kubevirt)

## Design

### Model

- A **generation** is a per-VM counter. Backup `N` closes generation `N`: the interval ending at the checkpoint `C_N` it creates.
- The **live checkpoint** `L = C_N` is the only libvirt checkpoint in steady state. Its bitmap `b(L)` records every write since `C_N`.
- The **map** `M_d` stores, per extent of disk `d`, the generation in which it last changed, covering writes before `C_N` only.
- `validFrom_d` is the oldest generation from which disk `d`'s history is complete.
- The **lineage** `mapID` is a UUID naming one unbroken history. Generation numbers are never reused within a lineage, see [Lineage integrity](#lineage-integrity).

For a requested base `G`:

$$
\mathrm{changed}_d(G) =
\begin{cases}
\text{all extents of } d & \text{if } G < \mathrm{validFrom}_d \\
\lbrace e : M_d[e] > G \rbrace \cup b(L)_d & \text{otherwise}
\end{cases}
$$

The result is exact. libvirt creates `C_{N+1}` and freezes a copy of `b(L)` in one QMP transaction, so every write is recorded either in the frozen copy (stamped into the map as `N+1`) or in `b(C_{N+1})`, never both and never neither. For the latest base, `G = N`, the map contributes nothing and the result is today's VEP 25 incremental.

### State on the VM state PVC

Everything lives in the existing `cbt` subPath of the backend state PVC, next to the QCOW2 overlays:

```
cbt/
  <volume>.qcow2      existing overlay, holds b(L)
  manifest.json       commit record
  <volume>.map        generation map, preallocated
  <volume>.journal    frozen bitmap of the generation being closed, preallocated
  journal.json        journal commit record, present only while a rotation is pending
```

#### Where the format is defined

The types below live in a new internal package, `pkg/storage/cbt/history`, not in `kubevirt.io/api`. That module is the contract for objects the apiserver serves to external clients, and these files never pass through the apiserver. Every component that reads or writes them is built from this repository: virt-launcher, the export server, and the offline export pod.

KubeVirt already keeps internal state that has to survive version skew:

- `KubeVirtMetadata` (`pkg/virt-launcher/virtwrap/api/schema.go`) lives in the libvirt domain XML and passes between launchers of different versions on every migration.
- virt-handler's mount records (`pkg/virt-handler/{container,hotplug}-disk/mount.go`) still carry `usesSafePaths` so that a downgrade keeps working.
- The backend-storage `migrated` marker is written by the launcher's libvirt hook and read by a recovery Job.

What is new is a binary, crash-consistent format with several readers. The package therefore owns the rules, and no component touches the files directly:

- Every file carries a version. Readers accept every version written by launchers within the supported upgrade skew. Writers write the oldest version that every supported reader accepts.
- Fields are only ever added, and a field's meaning never changes.
- A reader that meets an unknown version refuses the file, and the affected disks run `latestOnly`.
- Encoding, decoding, fsync and rename all happen inside the package.

#### Manifest

`manifest.json` is the single commit point. Each update writes a temporary file, fsyncs it, renames it over the old one and fsyncs the directory. The same type is encoded as XML into `KubeVirtMetadata` to carry it across RWO migration, see [Live migration](#live-migration).

```go
type Manifest struct {
	Version        uint16    `json:"version" xml:"version,attr"`
	MapID          types.UID `json:"mapID" xml:"mapID"`
	StateVolumeUID types.UID `json:"stateVolumeUID" xml:"stateVolumeUID"`
	// Generation is N, the last closed generation.
	Generation    uint64      `json:"generation" xml:"generation"`
	Live          *Checkpoint `json:"live,omitempty" xml:"live,omitempty"`
	PendingCreate *Checkpoint `json:"pendingCreate,omitempty" xml:"pendingCreate,omitempty"`
	PendingDelete *Checkpoint `json:"pendingDelete,omitempty" xml:"pendingDelete,omitempty"`
	Disks         []Disk      `json:"disks" xml:"disk"`
}

type Checkpoint struct {
	Name string `json:"name" xml:"name"`
	XML  string `json:"xml" xml:"xml"`
}

type HistoryMode string

const (
	HistoryFull       HistoryMode = "Full"
	HistoryLatestOnly HistoryMode = "LatestOnly"
)

type Disk struct {
	Volume    string      `json:"volume" xml:"volume"`
	SizeBytes uint64      `json:"sizeBytes" xml:"sizeBytes"`
	ValidFrom uint64      `json:"validFrom" xml:"validFrom"`
	History   HistoryMode `json:"history" xml:"history"`
}
```

- `Live` and `PendingDelete` hold the XML that `virDomainCheckpointGetXMLDesc` returns with `NO_DOMAIN`. Without `<domain>`, a redefinition cannot fail on a domain UUID mismatch, which libvirt reports as `VIR_ERR_INVALID_ARG` rather than as the inconsistency the recovery path handles.
- `PendingCreate` holds the XML passed to `BackupBegin`, because libvirt has not returned any XML for that checkpoint yet. Recovery redefines the checkpoint from it in order to delete it.
- `PendingCreate` and `PendingDelete` exist at all because libvirt forgets checkpoints when the domain stops. Without them, an interrupted rotation would leave a bitmap that nothing names and nothing can delete without QMP.

#### Map and journal files

Both files start with the same 4 KiB header, little-endian and zero-padded:

```go
const (
	MapMagic     = "KVCBTMAP"
	JournalMagic = "KVCBTJNL"
	Granularity  = 64 << 10
	HeaderSize   = 4 << 10
)

type FileHeader struct {
	Magic       [8]byte  // 0
	Version     uint16   // 8
	Flags       uint16   // 10
	MapID       [16]byte // 12
	SizeBytes   uint64   // 28
	Granularity uint32   // 36
	// Base is the window floor of a map, or the generation a journal closes.
	Base       uint64 // 40
	EntryCount uint64 // 48, ⌈SizeBytes / Granularity⌉
	CRC32C     uint32 // 56, over bytes 0-55
}
```

- **Map body:** `EntryCount` `uint16` entries. Each entry holds `generation − Base`, and `0` means "at or below `Base`", see [The generation window](#the-generation-window).
- **Journal body:** ⌈`EntryCount` / 8⌉ bytes, one bit per extent, least significant bit first, set where the frozen bitmap reported a change.

`journal.json` is written last, and its presence marks the journal complete:

```go
type JournalCommit struct {
	Version    uint16     `json:"version"`
	MapID      types.UID  `json:"mapID"`
	Generation uint64     `json:"generation"`
	Closes     string     `json:"closes"`
	Live       Checkpoint `json:"live"`
	Volumes    []string   `json:"volumes"`
}
```

- `Live` is the new checkpoint. If the domain restarts after the journal commits but before the manifest does, libvirt has forgotten that checkpoint, and this is the only record of it.
- Replay refuses a journal whose `MapID` or `Closes` does not match the manifest.
- Applying a journal writes a fixed value into fixed entries, so replay is idempotent. A journal whose `Generation` is at or below the manifest's is stale and is removed.

The map and journal files are preallocated with `fallocate` when a disk first gets them. A rotation therefore never allocates space and cannot hit ENOSPC partway through. If preallocation fails, the disk runs `LatestOnly`: it has no map and `ValidFrom` follows `N`, so only the latest base is incremental. That is exactly VEP 25 behaviour, reported with reason `HistoryUnavailable` for older bases. History degrades; the backup does not fail.

libvirt does not expose bitmap granularity. KubeVirt overlays use clusters of at least 64 KiB, and QEMU's default bitmap granularity is the cluster size capped at 64 KiB, so in practice the two match. Extents read from the export are rounded outward to `Granularity` regardless, so a different granularity costs precision and never correctness.

#### Ownership

virt-launcher owns the directory. Its backup-start RPC and its job-completion handler run on different goroutines, and both serialize on one lock. Two other components read or write it:

| Component | Access |
| --- | --- |
| Backup export server | Mounts the `cbt` subPath read-only, with the launcher's SELinux level, to read the manifest and maps while a pull export is live. |
| Offline export pod ([VEP 401](../401-offline-incremental-backup/offline-incremental-backup.md)) | Mounts the state PVC while the VM is stopped and performs the same rotation with `qemu-img bitmap`. VEP 401 already keeps it and the VMI from holding the PVC at the same time. |

Both upgrade with the control plane, while a running VM keeps its virt-launcher until it restarts or migrates. That skew is what the versioning rules above exist for.

### Backup flow

| Step | Where | Action |
| --- | --- | --- |
| 1. Resolve | Start RPC | Replay a pending journal and finish a pending delete. Refuse the backup if that fails. Make sure `L` is defined in libvirt, see [The live checkpoint](#the-live-checkpoint). Resolve each disk to incremental or full from `G`. |
| 2. Begin | Start RPC | Commit `pendingCreate = C_{N+1}`. Call `BackupBegin` in pull mode with `incremental=L` for every disk whose `b(L)` is healthy, even if the consumer asked for a full backup. Disks without a healthy `b(L)` get `backupmode="full"`. `checkpointXML` creates `C_{N+1}` on all CBT disks. |
| 3. Journal | Start RPC | Read each disk's exported bitmap over the launcher's own NBD connection to the export socket. Write the journal and commit `journal.json`. |
| 4. Serve | — | Only now does the launcher report the backup as started. The controller creates the pull export, or the launcher starts the push writer, only from that state. |
| 5. Finalize | Job-completed event | Apply the journal to the maps and fsync them. Commit the manifest: generation `N+1`, `live = C_{N+1}`, `pendingDelete = L`. Delete `L` from libvirt, remove the journal, and clear `pendingDelete`. |
| 6. Report | Job-completed event | Report the backup completed, with its token if it succeeded. |

Rules that follow from this order:

- **Step 4 waits for step 3** because the frozen bitmap is only reachable while the export exists. If a consumer could finish and tear down the export before the journal committed, the generation would be lost.
- **A failed backup still closes its generation** once step 3 has committed. The interval is real, and the map must record it for older tokens to stay exact. Older tokens remain valid and resolve to a larger delta.
- **The token is published only after step 5 commits.** A finalize that cannot commit fails the backup even if all data moved, because a backup whose checkpoint is not durably recorded cannot be built on. Preallocation leaves an I/O error as the only realistic cause.
- **`Completed` is reported only after step 5 commits or definitively fails.** Live migration is gated on it (`IsBackupInProgress`), so the maps never move or are shared while a rotation is in flight. A finalize failure still reports completion, so a VM never becomes undrainable. The leftover journal is recovered at the next start.
- **The maps are immutable while an export is live.** Step 1 runs before the export exists and step 5 after it is gone, so every answer an export gives is consistent with its point in time.

The first backup of a VM, or one where no disk has a healthy `b(L)`, is a libvirt full backup with `checkpointXML`. Its disks get `validFrom = N+1`.

### Crash recovery

| State found | Recovery |
| --- | --- |
| `pendingCreate` set, no journal | Redefine and delete `C_{N+1}`, then clear `pendingCreate`. `b(L)` kept recording, so the manifest at `N` is still exact. |
| Journal generation above the manifest's | Replay the journal and commit the manifest. |
| `pendingDelete` set | Redefine the checkpoint from its saved XML, delete it, and clear the field. |
| Stale journal | Remove it. |
| Bitmaps inconsistent after QEMU died uncleanly | Redefinition fails validation. The affected disks reset to `validFrom = N+1`, see below. |

virt-launcher-monitor shuts QEMU down gracefully when virt-launcher exits, so a launcher crash usually leaves the bitmaps consistent and the journal is what recovers the rotation. Node loss or a killed QEMU leaves the bitmaps inconsistent, and the map cannot recover what `b(L)` lost.

### The live checkpoint

`L` is redefined only when libvirt does not know it: after a VM start, or on a migration target. One `CreateCheckpointXML` with `REDEFINE | REDEFINE_VALIDATE` and the saved XML either succeeds, meaning every listed disk's bitmap is usable, or fails with `VIR_ERR_CHECKPOINT_INCONSISTENT`.

On failure, virt-launcher finds the bad disks by redefining against each disk alone, which is the probe the per-volume backup type work already uses. It resets those disks to `validFrom = N+1`, redefines `L` with the remaining disks, and saves that XML. libvirt deletes a checkpoint in one transaction that fails if any present disk lacks the bitmap, so the saved XML must always list exactly the disks that carry it. Unplugged disks are tolerated by both redefinition and deletion.

This replaces the `query-named-block-nodes` QMP call, the `RedefineCheckpoint` RPC, `checkpointRedefinitionRequired`, and the virt-handler pass that sets it. Redefinition becomes part of step 1.

### Serving changes

#### Pull

The export server mounts the `cbt` subPath read-only. For each incremental volume it answers a map request with `{M_d > G}` unioned with the exported bitmap. For a volume resolved to full it reports every extent. It receives `G`, the expected generation `N`, and each volume's resolution from the backup. It refuses to serve if the manifest's `mapID` or generation does not match, which catches a stale or replaced state volume.

#### Push

Push runs the same pull backup, and a writer in virt-launcher copies `changed_d(G)` from the export socket to a QCOW2 on the target PVC. The target PVC is already hotplugged into the launcher for push output and pull scratch. When the writer finishes, the launcher aborts the job, the only way a pull job ends, and reports the writer's result.

Push cannot stay on libvirt's push job, for two reasons:

- libvirt exposes the frozen bitmap the journal needs only through a pull export.
- A base older than `L` needs a point-in-time read of extents outside `b(L)`. libvirt's push job copies only `b(L)` and offers no such read.

Requirements on the writer:

- An extent that is changed but reads as zero is written as an explicit zero cluster. An unallocated cluster in an incremental image means "unchanged".
- A full backup copies the extents that `base:allocation` reports as allocated.
- Scratch and output share the target PVC and get distinct file names; today they collide. Auto-provisioned target PVCs ([#417](https://github.com/kubevirt/enhancements/pull/417)) must be sized for scratch as well.
- In-flight buffers are bounded and counted in the launcher's CBT memory overhead.
- The output carries no backing-file reference. libvirt's push output named the overlay path inside the launcher, which no consumer can resolve.

### Selecting a base

`VirtualMachineBackup.spec.fromCheckpoint` takes a token `<mapID>/<generation>`. When unset, it defaults to the tracker's latest token. Tokens describe themselves, so the tracker publishes only the resolvable range, not a list that grows with depth.

On create, the controller validates the token's syntax and `mapID`. virt-launcher resolves it at step 1, so a token that no longer resolves gives a full backup with a reason rather than a rejected object. On upgrade, a tracker's `latestCheckpoint.name` that equals `L` resolves to generation `N`, so the first backup after upgrade stays incremental.

Each volume reports its type and, when full, one reason:

| Reason | Cause |
| --- | --- |
| `NoBase` | No base was requested: a first backup or `forceFullBackup`. |
| `NewSinceCheckpoint` | The volume was added after `G`. |
| `HistoryUnavailable` | The volume's history before `G` is gone: lost bitmap, map not carried across an RWO migration, or `latestOnly`. |
| `HistoryExpired` | `G` is below the volume's window, see [The generation window](#the-generation-window). |
| `TokenMismatch` | The token names another lineage. |

Whether a lost bitmap was missing or inconsistent is logged and raised as a VMI event, not reported in the API, because telling them apart needs bitmap inspection and both lead to the same consumer action.

Three spec fields complete the API:

- **`requireIncremental`**: if a volume whose history covers `G` would resolve to full, the backup fails in step 1. No data moves and no generation is consumed. `NewSinceCheckpoint` and `NoBase` never trigger it.
- **`createCheckpoint: false`**: run against the base, with no `checkpointXML`, no journal and no generation. `L` keeps accumulating, so repeated runs against one base return growing supersets.
- **`resetChangeTracking`**: every volume is full. If the backup **succeeds**, finalize mints a new `mapID` with every disk at `validFrom = N+1`. A failed backup leaves the lineage intact, so there is never a moment with no history and no new base. Unlike `forceFullBackup`, which leaves all tokens valid, this invalidates every token on every tracker of the VM.

### Lineage integrity

A token is only meaningful if a `(mapID, generation)` pair always names the same point in time. That breaks if a manifest ever goes backwards, for example when a snapshot restore brings back an older state PVC with the same `mapID`. Once the counter climbed past the restored value, a stored token would resolve against a different timeline and return a plausible, wrong delta.

The manifest therefore records the UID of the state PVC it was written on. virt-launcher has no access to PVCs, so the controllers that already track the state PVC publish its UID on the VMI, see [VirtualMachineInstance](#virtualmachineinstance):

- the VMI controller sets `status.changedBlockTracking.stateVolumeUID`;
- the migration controller sets `persistentStatePVCUID` next to the `persistentStatePVCName` it already sets on both the source and target migration state.

A new `mapID` is minted, with every disk at `validFrom = N+1`, whenever:

- no manifest exists;
- `StateVolumeUID` differs from `status.changedBlockTracking.stateVolumeUID`, as after any restore (in place or to a new VM) or a replaced state PVC;
- `resetChangeTracking` succeeds.

The only sanctioned change of state volume is RWO live migration. The manifest arrives in the domain metadata, and the target launcher rebinds it only if its `StateVolumeUID` equals the source's `persistentStatePVCUID`. It writes the rebound manifest to the target PVC as soon as the migration completes. A snapshot taken after that point therefore carries the target UID, and restoring it still mints a new lineage.

The controller never seeds generations from tracker status, because tracker status can lag the manifest. Cloning a VM that has backend storage is already rejected, so clones cannot share a lineage.

### Live migration

Migration never overlaps a rotation, because migration waits for `Completed`, which waits for finalize.

- **RWX state PVC.** Source and target share the directory. The source makes no writes once migration starts. The target defers recovery work until migration completes, so a failed migration leaves the directory as the source last committed it.
- **RWO state PVC.** The target gets a new PVC. libvirt has no channel to carry the maps, so virt-launcher mirrors the manifest into the KubeVirt domain metadata on every commit, and the manifest travels with the migrated domain XML. On the target, `b(L)` has migrated with its disk and the manifest names `L`, so the latest base stays incremental. The maps are lost, so the target creates fresh ones with `validFrom = N`, and older bases report `HistoryUnavailable` until beta carries the maps. A pending journal is lost too, but the manifest's `pendingCreate` and `pendingDelete` still let the target remove the stray bitmaps.

Beta transfers the maps as a one-shot copy between source and target virt-handler before switchover. They are immutable during migration, so this is a copy, not a synchronization.

Storage live migration and decentralized migration of CBT disks fail today for an unrelated reason: libvirt rejects the `<slices>` element KubeVirt adds to the overlay's data store. That needs fixing first. Once it is, a byte-identical mirrored volume keeps its history.

### Lifecycle

| Event | Outcome |
| --- | --- |
| Clean restart | Redefine `L`, continue. |
| VM crash | Disks with inconsistent bitmaps reset to `validFrom = N+1`. Other disks continue. |
| RWX migration | Nothing lost. |
| RWO migration | Latest base incremental. Older bases `HistoryUnavailable` until beta. |
| Disk hotplug | `validFrom = N+1`, so older bases report `NewSinceCheckpoint`. |
| Unplug and replug | The bitmap is gone, so the disk resets. |
| Online disk growth | The grown tail counts as changed for every base until the next rotation extends the map. |
| Restore, with or without the state PVC | New lineage. |
| State PVC lost | New lineage. |
| CBT disabled and re-enabled | Overlays and maps are deleted, giving a new lineage. |
| Tracker deleted | Nothing to reclaim. |
| Write outside KubeVirt's view | Undetectable, so it must be declared with `resetChangeTracking`. |

## API Examples

### VirtualMachineBackupTracker

No retention field: nothing needs bounding.

```yaml
status:
  latestCheckpoint:
    name: my-vm-daily-1042-2026-03-03_12-00-00
    creationTime: "2026-03-03T12:00:00Z"
    token: 7b3e4c21-9f0a-4d53-8e71-2a6c5f8b1d04/1042
  changeTracking:
    mapID: 7b3e4c21-9f0a-4d53-8e71-2a6c5f8b1d04
    oldestGeneration: 1
    currentGeneration: 1058
```

- `latestCheckpoint.token` is the last backup **this tracker** completed successfully.
- `changeTracking` is per VM and reads the same on every tracker. Generations advance on every backup that closes one, including failed backups and backups taken through other trackers, so the two numbers differ.

`changeTracking` only answers whether a stored token still resolves: same `mapID`, and a generation at or above `oldestGeneration`. A consumer must never build a token from it, because that would name a point it holds no data for.

### VirtualMachineBackup

```yaml
spec:
  source:
    apiGroup: backup.kubevirt.io
    kind: VirtualMachineBackupTracker
    name: my-vm-tracker
  mode: Push
  pvcName: backup-target-pvc
  fromCheckpoint: 7b3e4c21-9f0a-4d53-8e71-2a6c5f8b1d04/1002
  requireIncremental: false   # default
  createCheckpoint: true      # default
  resetChangeTracking: false  # default
status:
  fromCheckpoint: 7b3e4c21-9f0a-4d53-8e71-2a6c5f8b1d04/1002
  checkpointName: my-vm-daily-1043-2026-03-04_12-00-00
  checkpointToken: 7b3e4c21-9f0a-4d53-8e71-2a6c5f8b1d04/1043
  includedVolumes:
    - volumeName: rootdisk
      type: Incremental
    - volumeName: datadisk
      type: Full
      reason: HistoryUnavailable
```

- `status.fromCheckpoint` is the base actually used, reported even when the spec field is unset.
- `status.checkpointToken` appears only after the rotation commits, so a published token always resolves. Both it and `checkpointName` are unset when `createCheckpoint` is false.

### Type changes in `backup/v1alpha1`

Additions only. `BackupVolumeInfo.Type` comes from the per-volume backup type work in VEP 25.

```go
// BackupToken names a point in a VM's change tracking history, "<mapID>/<generation>".
// +kubebuilder:validation:Pattern=`^[0-9a-f-]{36}/[0-9]+$`
type BackupToken string

type BackupCheckpoint struct {
	Name         string       `json:"name,omitempty"`
	CreationTime *metav1.Time `json:"creationTime,omitempty"`
	// +optional
	Token *BackupToken `json:"token,omitempty"`
}

type ChangeTracking struct {
	MapID             types.UID `json:"mapID"`
	OldestGeneration  int64     `json:"oldestGeneration"`
	CurrentGeneration int64     `json:"currentGeneration"`
}

type VirtualMachineBackupTrackerStatus struct {
	LatestCheckpoint *BackupCheckpoint `json:"latestCheckpoint,omitempty"`
	// +optional
	ChangeTracking *ChangeTracking `json:"changeTracking,omitempty"`
}

type VirtualMachineBackupSpec struct {
	// ...existing fields
	// +optional
	FromCheckpoint *BackupToken `json:"fromCheckpoint,omitempty"`
	// +optional
	RequireIncremental bool `json:"requireIncremental,omitempty"`
	// +optional
	// +kubebuilder:default=true
	CreateCheckpoint *bool `json:"createCheckpoint,omitempty"`
	// +optional
	ResetChangeTracking bool `json:"resetChangeTracking,omitempty"`
}

type VirtualMachineBackupStatus struct {
	// ...existing fields
	// +optional
	FromCheckpoint *BackupToken `json:"fromCheckpoint,omitempty"`
	// +optional
	CheckpointToken *BackupToken `json:"checkpointToken,omitempty"`
}

// +kubebuilder:validation:Enum=NoBase;NewSinceCheckpoint;HistoryUnavailable;HistoryExpired;TokenMismatch
type FullBackupReason string

type BackupVolumeInfo struct {
	// ...existing fields
	Type BackupType `json:"type,omitempty"`
	// +optional
	Reason FullBackupReason `json:"reason,omitempty"`
}

type BackupOptions struct {
	// ...existing fields, Incremental removed
	FromCheckpoint      *BackupToken `json:"fromCheckpoint,omitempty"`
	RequireIncremental  bool         `json:"requireIncremental,omitempty"`
	CreateCheckpoint    bool         `json:"createCheckpoint,omitempty"`
	ResetChangeTracking bool         `json:"resetChangeTracking,omitempty"`
}
```

`BackupOptions` crosses only from the controller to virt-handler. Removing `Incremental` is a breaking change on an internal subresource, so during the upgrade window the controller sends both fields, and the launcher prefers `FromCheckpoint`.

### VirtualMachineInstance

```go
type ChangedBlockTrackingStatus struct {
	// ...existing fields
	// StateVolumeUID is the UID of the backend state PVC, set by the VMI controller.
	// +optional
	StateVolumeUID types.UID `json:"stateVolumeUID,omitempty"`
	// ChangeTracking is the VM's lineage and resolvable range, reported by virt-launcher.
	// +optional
	ChangeTracking *VirtualMachineInstanceChangeTracking `json:"changeTracking,omitempty"`
}

type VirtualMachineInstanceChangeTracking struct {
	MapID             types.UID `json:"mapID"`
	OldestGeneration  int64     `json:"oldestGeneration"`
	CurrentGeneration int64     `json:"currentGeneration"`
}

type VirtualMachineInstanceCommonMigrationState struct {
	// ...existing fields
	// +optional
	PersistentStatePVCUID *types.UID `json:"persistentStatePVCUID,omitempty"`
}

type VirtualMachineInstanceBackupVolumeInfo struct {
	VolumeName string `json:"volumeName"`
	// +optional
	Type string `json:"type,omitempty"`
	// +optional
	Reason string `json:"reason,omitempty"`
}

type VirtualMachineInstanceBackupStatus struct {
	// ...existing fields
	// +optional
	CheckpointToken string `json:"checkpointToken,omitempty"`
}
```

The two API groups do not import each other, so the core group mirrors the backup group's types, as `VirtualMachineInstanceBackupVolumeInfo` already mirrors `BackupVolumeInfo`. The launcher reports the change tracking state in the backup metadata, virt-handler syncs it into the VMI status as it already does for backup status, and the backup controller copies it to every tracker of the VM.

The domain metadata gains the manifest:

```go
type KubeVirtMetadata struct {
	// ...existing fields
	ChangeTracking *history.Manifest `xml:"changeTracking,omitempty"`
}
```

## Alternatives

| Alternative | Why not |
| --- | --- |
| Retain N checkpoints per tracker | Rests on undocumented libvirt behaviour, costs a bitmap per base per disk, fails at migration rather than at backup when oversized, and cannot be unbounded. See [Motivation](#motivation). |
| Store the map as bitmap planes in the overlay | Disabled bitmaps do not migrate. Enabled ones are walked on every guest write, and populating them needs QMP. |
| Store the map in a CR | A 500 GiB disk needs about 15.6 MiB of map, ten times etcd's default request limit. |
| Store the map as a hidden QEMU blockdev | Attaching a non-guest blockdev needs `qemu:commandline`, which taints the domain. |
| Keep libvirt push for the latest base and use the writer only for older ones | Two data paths. Deriving the journal from the push target's allocation relies on QEMU's zero handling and cluster layout. Kept as a fallback if push throughput regresses. |
| Read the frozen bitmap through a second export after a push job | Two jobs per backup, and it still cannot serve an older base. |
| Fall back to the nearest newer base | The consumer holds data for the base it named. A later delta applied to it drops writes, and the result looks valid. |
| Upstream: libvirt imports an external bitmap into a backup | Would let libvirt push serve older bases directly. Worth pursuing, but no API exists today. |

## Scalability

### Storage footprint

Per disk of size `V`, with a 64 KiB granularity:

| Item | Size | 500 GiB disk |
| --- | --- | --- |
| Overlay metadata (VEP 25) | unchanged | ≈17 MiB |
| `b(L)`, plus `b(C_{N+1})` during a backup | ≈V / 64 KiB / 8 each | 1.25 MiB, 2.5 MiB during a backup |
| Map, preallocated | 2 × ⌈V / 64 KiB⌉ | 15.6 MiB |
| Journal, preallocated | ⌈V / 64 KiB⌉ / 8 | 1 MiB |

None of this changes with history depth or tracker count. A VM holds at most three checkpoints at any instant: `L`, one pending delete and one pending create. Steady state is one, and two while a backup runs.

### Backend state sizing

[`CBTBackendStateOverhead`](https://github.com/kubevirt/kubevirt/blob/92dff93d6f/pkg/storage/cbt/cbt.go#L41-L57) is a flat 512 MiB sized for five 500 GiB disks, about 180 MiB of which this design uses at peak. Five 2 TiB disks need about 320 MiB of map alone. Because maps are preallocated, the need is deterministic, and the overhead becomes a function of disk count and size. It is shared with TPM and EFI NVRAM state. A VM whose existing PVC is too small runs `latestOnly` on the disks that do not fit.

### Write path

Guest writes update one bitmap per disk, or two during a backup, regardless of depth. The map is written only at finalize, in proportion to what changed in that generation.

### The generation window

Map entries are 16-bit offsets from a per-map `base`. History covers at least the last 32768 generations: 3.7 years of hourly backups, or 113 days at five-minute intervals.

When a map's span nears 65535, it is rebased. `base` advances by 32768 and every entry drops by the same amount, floored at zero. The rebase is lossless for every base still queryable, since anything at or below the new `base` reports `HistoryExpired` anyway. It runs per disk as write, fsync and rename, during step 1 or at VM start, with thousands of generations of slack before the hard limit.

32-bit entries would remove the rebase at twice the map size, and the space falls exactly where state sizing is tight.

## Update/Rollback Compatibility

All new fields are optional, and the feature is gated by `IncrementalBackupHistory`.

- **Upgrade.** A running VM keeps its virt-launcher, and with it VEP 25 behaviour, until it restarts or migrates. On the first new launcher, the tracker's latest checkpoint becomes `L`, and a manifest is created with a new `mapID`, `generation = 1` and `validFrom = 1`. History accumulates from there. Any extra bitmaps from earlier retained-checkpoint builds are deleted at the first rotation.
- **Rollback.** An older controller asks for an incremental from the tracker's `latestCheckpoint.name`. That succeeds if it equals `L` and is full otherwise. The manifest and maps are ignored files. An older controller ignores `resetChangeTracking`, so a consumer relying on it must check `status` across a rollback.
- **Push output** carries no backing-file reference. See [Push](#push).

## Functional Testing Approach

- After N backups each disk holds one bitmap, and the tracker reports generation N.
- A backup from a base k generations old matches a full backup taken at that base, for k = 1, a mid value, and the window floor. Covers pull and push.
- A failed backup advances the generation, and an older token afterwards covers both intervals.
- Killing virt-launcher at each step of the rotation leaves state that the next start recovers to an exact result.
- A pull export is not reported ready before the journal commits, and serves a consistent map for its whole life.
- Migration waits for finalize. After an RWX migration all bases stay incremental. After an RWO migration the latest base is incremental and older ones report `HistoryUnavailable`.
- A restore with the state PVC mints a new lineage, and old tokens report `TokenMismatch`.
- A crash leaves only the affected disks full, with `HistoryUnavailable`.
- Hotplug reports `NewSinceCheckpoint`. Disk growth reports the tail as changed until the next rotation.
- `requireIncremental`, `createCheckpoint: false`, `resetChangeTracking` and `forceFullBackup` behave as specified.
- Changed extents that read as zero restore correctly from a push chain.
- A map header written at the rebase threshold rebases without changing any answer above the new floor.
- No backup operation taints the domain, across restart, migration and crash recovery.
- Files written by the oldest launcher within the supported skew are read correctly by the current export server, offline export pod and launcher; this runs as a unit test over checked-in fixtures.

## Implementation History

TBD.

## Graduation Requirements

### Alpha

- [ ] `IncrementalBackup` graduated to beta
- [ ] `pkg/storage/cbt/history`: manifest, preallocated maps and journal, with the commit, recovery and versioning rules above
- [ ] One libvirt checkpoint per VM, rotated by every backup that closes a generation
- [ ] Tokens, `fromCheckpoint`, per-volume `type` and `reason`, and the tracker's `changeTracking`
- [ ] `requireIncremental`, `createCheckpoint` and `resetChangeTracking`
- [ ] Redefinition by `REDEFINE_VALIDATE` in the start path; no QMP on any backup path
- [ ] Push served by the launcher writer on top of a pull export
- [ ] Lineage bound to the state PVC UID published on the VMI; restores mint a new lineage
- [ ] Manifest carried in domain metadata across RWO migration
- [ ] `CBTBackendStateOverhead` as a function of disk geometry, with `latestOnly` for VMs that do not fit

### Beta

- [ ] Map transfer between source and target virt-handler for RWO migration
- [ ] Push throughput within an agreed margin of VEP 25's libvirt push
- [ ] Storage and decentralized migration of CBT disks working
- [ ] [VEP 401](../401-offline-incremental-backup/offline-incremental-backup.md) performing the rotation offline

#### On-By-Default Readiness

TBD.

### GA

TBD.
