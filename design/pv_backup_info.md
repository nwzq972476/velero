# PersistentVolume backup information design

## Abstract
Create a new metadata file in the backup repository's backup name sub-directory to store the backup-including PVC and PV information. The information includes the way of backing up the PVC and PV data, snapshot information, and status. The needed snapshot status can also be recorded there, but the Velero-Native snapshot plugin doesn't provide a way to get the snapshot size from the API, so it's possible that not all snapshot size information is available.

This new additional metadata file is needed when:
* Get a summary of the backup's PVC and PV information, including how the data in them is backed up, or whether the data in them is skipped from backup.
* Find out how the PVC and PV should be restored in the restore process.
* Retrieve the PV's snapshot information for backup.

## Background
There is already a [PR](https://github.com/vmware-tanzu/velero/pull/6496) to track the skipped PVC in the backup. This design will depend on it and go further to get a summary of PVC and PV information, then persist into a metadata file in the backup repository.

In the restore process, the Velero server needs to decide how the PV resource should be restored according to how the PV is backed up. The current logic is to check whether it's backed up by Velero-native snapshot, by file-system backup, or having `DeletionPolicy` set as `Delete`.

The checks are made by the backup-generated PVBs or Snapshots. There is no generic way to find this information, and the CSI backup and Snapshot data movement backup are not covered.

Another thing that needs noticing is when describing the backup, there is no generic way to find the PV's snapshot information.

## Goals
- Create a new metadata file to store backup's PVCs and PVs information and volume data backing up method. The file can be used to let downstream consumers generate a summary.
- Create a generic way to let the Velero server know how the PV resources are backed up.
- Create a generic way to let the Velero server find the PV corresponding snapshot information.

## Non Goals
- Unify how to get snapshot size information for all PV backing-up methods, and all other currently not ready PVs' information.

## High-Level Design
Create _backup-name_-volumes-info.json metadata file in the backup's repository. This file will be encoded to contain all the PVC and PV information included in the backup. The information covers whether the PV or PVC's data is skipped during backup, how its data is backed up, and the backed-up detail information.

Please notice that the new metadata file includes all skipped volume information. This is used to address [the second phase needs of skipped volumes information](https://github.com/vmware-tanzu/velero/issues/5834#issuecomment-1526624211).

The `restoreItem` function can decode the _backup-name_-volumes-info.json file to determine how to handle the PV resource. 

## Detailed Design

### The VolumeInfo structure
_backup-name_-volumes-info.json file is a structure that contains an array of structure `VolumeInfo` and a VolumeInfo version parameter. The version is used to support multiple VolumeInfo version, and make the backward-compatible support possible. The current version is `1`.

The 1 version of `VolumeInfo` definition is:
``` golang
type VolumeInfoV1 struct {
    PVCName        string    // The PVC's name. The format should be <namespace-name>/<PVC-name>
    PVName         string    // The PV name.
    BackupMethod   string    // The way the volume data is backed up. The valid value includes `VeleroNativeSnapshot`, `PodVolumeBackup`, `CSISnapshot` and `Skipped`.
    SnapshotDataMovement bool // Whether the volume's snapshot data is moved to specified storage.

    SkippedReason   string    // The reason for the volume is skipped in the backup.
    StartTimestamp  *metav1.Time // Snapshot starts timestamp.

    CSISnapshotInfo          CSISnapshotInfo
    SnapshotDataMoveInfo     SnapshotDataMoveInfo
    NativeSnapshotInfo       VeleroNativeSnapshotInfo
    PVBInfo                  PodVolumeBackupInfo
}

// CSISnapshotInfo is used for displaying the CSI snapshot status
type CSISnapshotInfo struct {
    SnapshotHandle  string       // The actual snapshot ID. It can be the cloud provider's snapshot ID or the file-system uploader's snapshot.
    Size            int64        // The snapshot corresponding volume size. Some of the volume backup methods cannot retrieve the data by current design, for example, the Velero native snapshot.

    Driver          string  // The name of the CSI driver.
    VSCName         string // The name of the VolumeSnapshotContent. 
}

// SnapshotDataMoveInfo is used for displaying the snapshot data mover status.
type SnapshotDataMoveInfo struct {
    DataMover        string    // The data mover used by the backup. The valid values are `velero` and ``(equals to `velero`).
    UploaderType     string    // The type of the uploader that uploads the snapshot data. The valid values are `kopia` and `restic`. It's useful for file-system backup and snapshot data mover.
    RetainedSnapshot string    // The name or ID of the snapshot associated object(SAO).
}

// VeleroNativeSnapshotInfo is used for displaying the Velero native snapshot status.
type VeleroNativeSnapshotInfo struct {
    SnapshotHandle      string       // The actual snapshot ID. It can be the cloud provider's snapshot ID or the file-system uploader's snapshot.
    Size                int64        // The snapshot corresponding volume size. Some of the volume backup methods cannot retrieve the data by current design, for example, the Velero native snapshot.

    VolumeType string    // The cloud provider snapshot volume type.
    VolumeAZ   string    // The cloud provider snapshot volume's availability zones.
    IOPS       string    // The cloud provider snapshot volume's IOPS.
}

// PodVolumeBackupInfo is used for displaying the PodVolumeBackup snapshot status.
type PodVolumeBackupInfo struct {
    SnapshotHandle      string       // The actual snapshot ID. It can be the cloud provider's snapshot ID or the file-system uploader's snapshot.
    Size                int64        // The snapshot corresponding volume size. Some of the volume backup methods cannot retrieve the data by current design, for example, the Velero native snapshot.

    UploaderType  string    // The type of the uploader that uploads the data. The valid values are `kopia` and `restic`. It's useful for file-system backup and snapshot data mover.
    VolumeName    string // The PVC's corresponding volume name used by Pod
    PodName       string // The Pod name mounting this PVC. The format should be <namespace-name>/<pod-name>.
}
```

To make Velero support multiple versions of VolumeInfo, Velero needs to create a structure to include the version and the versioned volume information together. The information is persisted in the metadata file during backup creation. When reading the VolumeInfo metadata file, Velero reads the version information first by un-marshalling the metadata file by structure `VolumeInfoVersion`, then decide to use which version of the VolumeInfos according to the version value.

If there is need to introduce a non compatible change to the VolumeInfo, a new version of `VolumeInfos` and `VolumeInfo` are needed. For example, version 2 is created, then `VolumeInfosV2` and `VolumeInfoV2` structures are needed.

Only when there a non-backward-compatible change introduced in the `VolumeInfo` structure, the `version`'s version will be incremented.

``` golang
type VolumeInfoVersion struct {
    Version string 
}

type VolumeInfosV2 struct {
    Infos   []VolumeInfoV2
    Version string // VolumeInfo structure's version information.
}

type VolumeInfoV2 struct {
    ...
}
```

### How the VolumeInfo array is generated.
The function `persistBackup` has `backup *pkgbackup.Request` in parameters.
From it, the `VolumeSnapshots`, `PodVolumeBackups`, `CSISnapshots`, `itemOperationsList`, and `SkippedPVTracker` can be read. All of them will be iterated and merged into the `VolumeInfo` array, and then persisted into backup repository in function `persistBackup`.

Please notice that the change happened in async operations are not reflected in the new metadata file. The file only covers the volume changes happen in the Velero server process scope.

A new methods are added to BackupStore to download the VolumeInfo metadata file.
Uploading the metadata file is covered in the exiting `PutBackup` method.

``` golang
type BackupStore interface {
    ...
    GetVolumeInfos(name string) ([]*VolumeInfo, error)
    ...
}
```

### How the VolumeInfo array is used.

#### Generate the PVC backed-up information summary
The downstream tools can use this VolumeInfo array to format and display their volume information. This is in the scope of this feature.

#### Retrieve volume backed-up information for `velero backup describe` command
The `velero backup describe` can also use this VolumeInfo array structure to display the volume information. The snapshot data mover volume should use this structure at first, then the Velero native snapshot, CSI snapshot, and PodVolumeBackup can also use this structure. The detailed implementation is also not in this feature's scope.

#### Let restore know how to restore the PV
In the function `restoreItem`, it will determine whether to restore the PV resource by checking it in the Velero native snapshots list, PodVolumeBackup list, and its DeletionPolicy. This logic is still kept. The logic will be used when the new `VolumeInfo` metadata cannot be found to support backward compatibility.

``` golang
	if groupResource == kuberesource.PersistentVolumes {
		switch {
		case hasSnapshot(name, ctx.volumeSnapshots):
            ...
        case hasPodVolumeBackup(obj, ctx):
            ...
        case hasDeleteReclaimPolicy(obj.Object):
            ...
        default:
            ...
```

After introducing the VolumeInfo array, the following logic will be added.
``` golang
	if groupResource == kuberesource.PersistentVolumes {
        volumeInfo := GetVolumeInfo(pvName)
		switch volumeInfo.BackupMethod {
		case VeleroNativeSnapshot:
            ...
        case PodVolumeBackup:
            ...
        case CSISnapshot:
            ...
        case Skipped:
            // Check whether the Velero server should restore the PV depending on the DeletionPolicy setting.
        default:
            // Need to check whether the volume is backed up by the SnapshotDataMover.
            if volumeInfo.SnapshotDataMovement:
```

### How the VolumeInfo metadata file is deleted
_backup-name_-volumes-info.json file is deleted during backup deletion.

## Alternatives Considered
The restore process needs more information about how the PVs are backed up to determine whether this PV should be restored. The released branches also need a similar function, but backporting a new feature into previous releases may not be a good idea, so according to [Anshul Ahuja's suggestion](https://github.com/vmware-tanzu/velero/issues/6595#issuecomment-1731081580), adding more cases here to support checking PV backed-up by CSI plugin and CSI snapshot data mover: https://github.com/vmware-tanzu/velero/blob/5ff5073cc3f364bafcfbd26755e2a92af68ba180/pkg/restore/restore.go#L1206-L1324.

## Security Considerations
There should be no security impact introduced by this design.

## Compatibility
After this design is implemented, there should be no impact on the existing [skipped PVC summary feature](https://github.com/vmware-tanzu/velero/pull/6496).

To support older version backup, which doesn't have the VolumeInfo metadata file, the old logic, which is checking the Velero native snapshots list, PodVolumeBackup list, and PVC DeletionPolicy, is still kept, and supporting CSI snapshots and snapshot data mover logic will be added too.

## Implementation
This will be implemented in the Velero v1.13 development cycle.

## Open Issues
There are no open issues identified by now.
