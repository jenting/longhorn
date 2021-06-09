# Title

Final consistent backup volumes with volume snapshot backups from the remote backup store.

## Summary

Currently, Longhorn uses a blocking way for communication with the remote backup store, so there will be some potential voluntary or 
involuntary factors (ex: network latency) impacting the functions relying on remote backup stores like listing volume backups or even 
causing further [cascading problems](###related-issues) after the backup store operation.

This enhancement is to propose an asynchronous way to pull backup volumes and volume snapshot backups from the remote backup store (S3/NFS) 
then persistently saved via cluster custom resources.

This can resolve the problems above mentioned by asynchronously querying the list of backup volumes and volume snapshot backups from the remote 
backup store for final consistent available results. It's also scalable for the costly resources created by the original blocking query operations.

### Related Issues

- https://github.com/longhorn/longhorn/issues/1761
- https://github.com/longhorn/longhorn/issues/1955
- https://github.com/longhorn/longhorn/issues/2536
- https://github.com/longhorn/longhorn/issues/2543

## Motivation

### Goals

Decrease the query latency when listing volume backups _or_ volume snapshot backups in the circumstances like lots of volume backups, lots of 
volume snapshot backups, or the network latency between the Longhorn cluster and the remote backup store.

### Non-goals [optional]

Automatically adjust the remote backup store pull period.

## Proposal

Currently, we have a setting `backupstore-poll-interval` to periodically pull the remote backup store lastest volume backups within a setting controller.

We want to leverage the same concept but pull the backup volumes and volume snapshot backups from the remote backup store and saves them into the cluster custom resource (CR). Therefore, we'll:
1. Change the [longhorn/backupstore](https://github.com/longhorn/backupstore) list and inspect command behavior.
   - The `backup list` command includes listing all backup volumes and the volume snapshot backups and read these metadata.
     We'll change the `backup list` behavior to perform list only, but not read the metadata.
   - The `backup inspect` command supports read volume snapshot backup metadata only.
     We'll add a new `backup inspect-volume` subcommand to support read backup volume metadata.
2. Create a CRD _backuptargets.longhorn.io_ to save the backup target URL, credential secret, and poll interval.
3. Create a CRD _backupvolumes.longhorn.io_ to save to backup volume metadata.
4. Create a CRD _backups.longhorn.io_ to save the volume snapshot backup metadata.
5. At the existed `setting_controller`, which is responsible creating/deleting BackupTarget CR according to the settings `backup-target`, `backup-target-credential-secret`, and `backupstore-poll-interval`.
6. Create a new controller `backup_target_controller`, which is responsible to update the BackupTarget CR status and creating/deleting the BackupVolume CR metadata and spec.
7. Create a new controller `backup_volume_controller`, which is responsible to update BackupVolume CR status, delete backup volume from the remote backup store, and creating/deleting Backup CR metadata and spec.
8. Create a new controller `backup_controller`, which is responsible to update the Backup CR status, delete backup from the remote backup store.
9. The HTTP endpoints CRUD methods related to backup volume and backups will interact with BackupVolume CR and Backup CR instead interact with the remote backup store.

### User Stories

Before this enhancement, when the user's environment under the circumstances that the remote backup store has lots of backup volumes _or_ 
volume snapshot backups, and the latency between the longhorn manager to the remote backup store is high.
Then if the user clicks the `Backup` on the GUI, the user might hit list backup volumes _or_ list volume snapshot backups timeout issue 
(the default timeout is 1 min).

We choose to not create a new setting for the user to increase the list timeout value is because the browser has its timeout value also.
Let's say the list backups needs 5 minutes to finish. Even we allow the user to increase the longhorn manager list timeout value, we can't change 
the browser default timeout value. Furthermore, some browser doesn't allow the user to change default timeout value like Google Chrome.

After this enhancement, when the user's environment under the circumstances that the remote backup store has lots of backup volumes _or_ 
volume snapshot backups, and the latency between the longhorn manager to the remote backup store is high.
Then if the user clicks the `Backup` on the GUI, the user can eventually list backup volumes _or_ list volume snapshot backups without timeout issue.

#### Story 1

The user environment is under the circumstances that the remote backup store has lots of backup volumes and the latency between the longhorn manager 
to the remote backup store is high. Then, the user can list all backup volumes on the GUI.

#### Story 2

The user environment is under the circumstances that the remote backup store has lots of volume snapshot backups and the latency between the longhorn 
manager to the remote backup store is high. Then, the user can list all volume snapshot backups on the GUI.

### User Experience In Detail

None.

### API changes

1. For [longhorn/backupstore](https://github.com/longhorn/backupstore)

   The current [longhorn/backupstore](https://github.com/longhorn/backupstore) list and inspect command behavior are:
   - `backup ls --volume-only`: List all backup volumes and read it's metadata (`volume.cfg`). For example:
     ```shell
     $ backup ls s3://backupbucket@minio/ --volume-only
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
         "Name": "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
         "Size": "2147483648",
         "Labels": {},
         "Created": "2021-05-12T00:52:01Z",
         "LastBackupName": "backup-c5f548b7e86b4b56",
         "LastBackupAt": "2021-05-17T05:31:01Z",
         "DataStored": "121634816",
         "Messages": {}
       },
       "pvc-7a8ded68-862d-4abb-a08c-8cf9664dab10": {
         "Name": "pvc-7a8ded68-862d-4abb-a08c-8cf9664dab10",
         "Size": "10737418240",
         "Labels": {},
         "Created": "2021-05-10T02:43:02Z",
         "LastBackupName": "backup-432f4d6afa31481f",
         "LastBackupAt": "2021-05-10T06:04:02Z",
         "DataStored": "140509184",
         "Messages": {}
       }
     }
     ```
   - `backup ls --volume <volume-name>`: List all volume snapshot backups and read it's metadata (`backup_backup_<backup-hash>.cfg`). For example:
     ```shell
     $ backup ls s3://backupbucket@minio/ --volume pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
         "Name": "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
         "Size": "2147483648",
         "Labels": {}
         "Created": "2021-05-12T00:52:01Z",
         "LastBackupName": "backup-c5f548b7e86b4b56",
         "LastBackupAt": "2021-05-17T05:31:01Z",
         "DataStored": "121634816",
         "Messages": {},
         "Backups": {
           "s3://backupbucket@minio/?backup=backup-02224cb26b794e73\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
             "Name": "backup-02224cb26b794e73",
             "URL": "s3://backupbucket@minio/?backup=backup-02224cb26b794e73\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
             "SnapshotName": "backup-23c4fd9a",
             "SnapshotCreated": "2021-05-17T05:23:01Z",
             "Created": "2021-05-17T05:23:04Z",
             "Size": "115343360",
             "Labels": {},
             "IsIncremental": true,
             "Messages": null
            },
           ...
           "s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
             "Name": "backup-fa78d89827664840",
             "URL": "s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
             "SnapshotName": "backup-ac364071",
             "SnapshotCreated": "2021-05-17T04:42:01Z",
             "Created": "2021-05-17T04:42:03Z",
             "Size": "115343360",
             "Labels": {},
             "IsIncremental": true,
             "Messages": null
           }
         }
       }
     }
     ```
   - `backup inspect <backup>`: Read a single volume's backup metadata (`backup_backup_<backup-hash>.cfg`). For example:
     ```shell
     $ backup inspect s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "Name": "backup-fa78d89827664840",
       "URL": "s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
       "SnapshotName": "backup-ac364071",
       "SnapshotCreated": "2021-05-17T04:42:01Z",
       "Created": "2021-05-17T04:42:03Z",
       "Size": "115343360",
       "Labels": {},
       "IsIncremental": true,
       "VolumeName": "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
       "VolumeSize": "2147483648",
       "VolumeCreated": "2021-05-12T00:52:01Z",
       "Messages": null
     }
     ```

   After this enhancement, the [longhorn/backupstore](https://github.com/longhorn/backupstore) list and inspect command behavior are:
   - `backup ls --volume-only`: List all backup volume names. For example:
     ```shell
     $ backup ls s3://backupbucket@minio/ --volume-only
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {},
       "pvc-7a8ded68-862d-4abb-a08c-8cf9664dab10": {}
     }
     ```
   - `backup ls --volume <volume-name>`: List all volume snapshot backup names. For example:
     ```shell
     $ backup ls s3://backupbucket@minio/ --volume pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07": {
         "Backups": {
           "backup-02224cb26b794e73": {},
           ...
           "backup-fa78d89827664840": {}
         }
       }
     }
     ```
   - `backup inspect-volume <volume>`: Read a single backup volume metadata (`volume.cfg`). For example:
     ```shell
     $ backup inspect-volume s3://backupbucket@minio/?volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "Name": "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
       "Size": "2147483648",
       "Labels": {},
       "Created": "2021-05-12T00:52:01Z",
       "LastBackupName": "backup-c5f548b7e86b4b56",
       "LastBackupAt": "2021-05-17T05:31:01Z",
       "DataStored": "121634816",
       "Messages": {}
     }
     ```
   - `backup inspect <backup>`: Read a single volume's backup metadata (`backup_backup_<backup-hash>.cfg`). For example:
     ```shell
     $ backup inspect s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07
     {
       "Name": "backup-fa78d89827664840",
       "URL": "s3://backupbucket@minio/?backup=backup-fa78d89827664840\u0026volume=pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
       "SnapshotName": "backup-ac364071",
       "SnapshotCreated": "2021-05-17T04:42:01Z",
       "Created": "2021-05-17T04:42:03Z",
       "Size": "115343360",
       "Labels": {},
       "IsIncremental": true,
       "VolumeName": "pvc-004d8edb-3a8c-4596-a659-3d00122d3f07",
       "VolumeSize": "2147483648",
       "VolumeCreated": "2021-05-12T00:52:01Z",
       "Messages": null
     }
     ```

   Generally speaking, we want to separate the **list** and **read** command.

2. The Longhorn manager HTTP endpoints.

   The below HTTP endpoints current behavior are:
   - **GET** `/v1/backupvolumes`: read all backup volumes from the remote backup store.
   - **GET** `/v1/backupvolumes/{volName}`: read a backup volume from the remote backup store.
   - **DELETE** `/v1/backupvolumes/{volName}`: delete a backup volume from the remote backup store.
   - **POST** `/v1/volumes/{volName}?action=snapshotBackup`: create a volume snapshot backup to the remote backup store.
   - **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a list of volume snapshot backups from the remote backup store.
   - **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a volume snapshot backup from the remote backup store.
   - **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`: delete a volume snapshot backup from the remote backup store.
  
   After this enhancement, below HTTP endpoints behavior are:
   - **GET** `/v1/backupvolumes`: read all the BackupVolume CRs.
   - **GET** `/v1/backupvolumes/{volName}`: read a BackupVolume CR with the given volume name.
   - **DELETE** `/v1/backupvolumes/{volName}`: delete the remote backup store data and then delete the BackupVolume CR.
   - **POST** `/v1/volumes/{volName}?action=snapshotBackup`: create a new Backup CR.
   - **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a list of Backup CRs with the label filter `volume=<backup-volume-name>`.
   - **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a Backup CR with the label filter `volume=<backup-volume-name>`.
   - **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`: delete the remote backup store data and then delete the Backup CR.

## Design

### Implementation Overview

1. Create a new BackupTarget CRD `backuptargets.longhorn.io`.
   
   - `metadata.name`: the hashed name of backup target URL.
   - `spec.backupTargetURL`: the backup target URL.
   - `spec.credentialSecret`: the backup target credential secret.
   - `spec.pollInterval`: the backup target poll interval.
   - `status.lastSyncedTimestamp`: records the last time the backup target was running the reconcile process.

2. Create a new BackupVolume CRD `backupvolumes.longhorn.io`.
   - `metadata.name`: the backup volume name.
   - `metadata.ownerReferences`: the owning object.
   - `status.size`: the backup volume size.
   - `status.labels`: the backup volume labels.
   - `status.createTimestamp`: the backup volume creation timestamp.
   - `status.lastBackupName`: the latest volume backup name.
   - `status.lastBackupTimestamp`: the latest volume backup timestamp.
   - `status.dataStored`: the backup volume block count.
   - `status.messages`: the error messages when call longhorn engine on list or inspect backup volumes.
   - `status.lastSyncedTimestamp`: records the last time the backup volume was synced into the cluster.

3. Create a new Backup CRD `backups.longhorn.io`.

   - `metadata.name`: the volume snapshot backup name.
   - `metadata.labels`:
     - `volume=<backup-volume-name>`: this label indicates which backup volume the volume snapshot backup belongs to.
     - `snapshotBackup=true`: this label indicates the user creates snapshotBackup in the local cluster.
   - `metadata.ownerReferences`: the owning object.
   - `spec.snapshotName`: the volume snapshot name.
   - `spec.labels`: the labels of volume snapshot backup.
   - `status.url`: the volume snapshot backup url.
   - `status.snapshotName`: the volume snapshot name.
   - `status.snapshotCreateTimestamp`: the volume snapshot creation timestamp.
   - `status.backupCreateTimestamp`: the volume snapshot backup creation timestamp.
   - `status.size`: the volume snapshot size.
   - `status.labels`: the labels of volume snapshot backup.
   - `status.messages`: the error messages when call longhorn engine on list or inspect volume snapshot backups.
   - `status.lastSyncedTimestamp`: records the last time the volume snapshot backup was synced into the cluster.

4. At the existed `setting_controller`.
   
   Watches the changes of Setting CR `settings.longorn.io` field `backup-target`, `backup-target-credential-secret`, and `backupstore-poll-interval`. The setting controller is responsible to create/update/delete BackupTarget CR metadata and spec.

5. Create a new `backup_target_controller`.
   
   Watches the change of BackupTarget CR `backuptargets.longhorn.io`. The backup target controller is responsible creating/deleting BackupVolume CR metadata. The reconcile loop steps are:
   - **AddFunc**/**UpdateFunc**: Check if `the current timestamp - BackupTarget CR status.lastSyncedTimestamp >= spec.pollInterval`.
     - If no, abort the current reconcile process.
     - If yes:
      1. Call the longhorn engine to list all the backup volumes `backup ls --volume-only` from the remote backup store `backupStoreBackups`.
      2. List in cluster BackupVolume CRs `clusterBackupVolumes`.
      3. Find the difference backup volumes `backupVolumesToPull = backupStoreBackups - clusterBackupVolumes` and create BackupVolume CR `metadata.name` and `metadata.ownerReferences.Kind=BackupTarget`.
      4. Find the difference backup volumes `backupVolumesToDelete = clusterBackupVolumes - backupStoreBackups` and delete BackupVolume CR.
      5. Updates the BackupTarget CR `status.lastSyncedTimestamp`.
   - **DeleteFunc**: We don't need the DeleteFunc function because if the user changes/removes the backup target URL, the BackupTarget CR be deleted. Then all the related BackupVolume CR will be deleted because the BackupVolume CR owner reference is the BackupTarget CR.

6. For the Longhorn manager HTTP endpoints:
   - **DELETE** `/v1/backupvolumes/{volName}`:
     1. Add the finalizer to the BackupVolume CR with the given volume name.
     2. Delete a BackupVolume CR with the given volume name.

7. Create a new controller `backup_volume_controller`.
   
   Watches the change of BackupVolume CR `backupvolumes.longhorn.io`. The backup volume controller is responsible to update BackupVolume CR status field, and creating/deleting Backup CR metadata. The reconcile loop steps are:
   - **AddFunc**: Check if `the current timestamp - BackupTarget CR status.lastSyncedTimestamp >= spec.pollInterval`.
     - If no, abort this step.
     - If yes, list in cluster BackupVolume CRs `backupvolumes.longhorn.io` to get the volume name `volume-name` from the metadata.name field. For each backup volume `volume-name`:
        1. Call the longhorn engine to read all backup volumes' metadata `backup inspect-volume <volume-name>`.
        2. Updates the BackupVolume CR status field according to the backup volumes' metadata, and also updates the BackupVolume CR `status.lastSyncedTimestamp`.
        3. Call the longhorn engine to list all the volume snapshot backups `backup ls --volume <volume-name>` from the remote backup store `backupStoreVolumeSnapshotBackups`.
        4. List in cluster Backup CRs `clusterVolumeSnapshotBackups`.
        5. Find the difference volume snapshot backups `volumeSnapshotBackupsToPull = backupStoreVolumeSnapshotBackups - clusterVolumeSnapshotBackups` and create Backup CR `metadata.name` + `metadata.ownerReferences.Kind=BackupVolume`.
        6. Find the difference volume snapshot backups `volumeSnapshotBackupsToDelete = clusterVolumeSnapshotBackups - backupStoreVolumeSnapshotBackups` and delete Backup CR.
   - **DeleteFunc**: If the finalizer has been set, delete the backup volume from the remote backup store `backup rm --volume <volume-name> <url>`. After that, remove the finalizer.

8. For the Longhorn manager HTTP endpoints:
   - **POST** `/v1/volumes/{volName}?action=snapshotBackup`:
     1. Generate the backup name.
     2. Create a new Backup CR with
        - `metadata.name`
        - `metadata.ownerReferences.Kind=BackupVolume`
        - `metadata.Labels["snapshotBackup"]=true`
        - `spec.snapshotName`
        - `spec.labels`
   - **DELETE** `/v1/backupvolumes/{volName}?action=backupDelete`:
     1. Add the finalizer to the Backup CR with the given backup name.
     2. Delete a Backup CR with the given backup name.

9.  Create a new controller `backup_controller`.

   Watches the change of Backup CR `backups.longhorn.io`. The backup controller is responsible to update the Backup CR status field and create/delete backup to/from the remote backup store. The reconcile loop steps are:
   - **AddFunc**:
     1. List the Backup CR with label filter `snapshotBackup=true`. For each Backup, call the longhorn engine to perform snapshot backup to the remote backup store. After that, removes the label `snapshotBackup=true`, and updates the Backup CR status.
     2. Check if the `current timestamp - BackupVolume CR status.lastSyncedTimestamp >= spec.pollInterval`.
        - If no, abort the current snapshot backup process.
        - If yes, list all Backup CRs `backups.longhorn.io`, for each backup `<backup-name`:
          - If the Backup CR `status.lastSyncedTimestamp` is set, skip it.
          - If the Backup CR `status.lastSyncedTimestamp` is empty, call the longhorn engine to read the volume snapshot backup metadata `backup inspect <backup-url>` and updates the Backup CR status field according to the volume snapshot backup metadata. After that, updates the Backup CR `status.lastSyncedTimestamp`.
   - **DeleteFunc**: If the finalizer has been set, delete the backup from the remote backup store `backup rm <backup-url>`. After that, remove the finalizer.

11. For the Longhorn manager HTTP endpoints:
   - **GET** `/v1/backupvolumes`: read all the BackupVolume CRs.
   - **GET** `/v1/backupvolumes/{volName}`: read a BackupVolume CR with the given volume name.
   - **GET** `/v1/backupvolumes/{volName}?action=backupList`: read a list of Backup CRs with the label filter `volume=<backup-volume-name>`.
   - **GET** `/v1/backupvolumes/{volName}?action=backupGet`: read a Backup CR with the label filter `volume=<backup-volume-name>`.

### Test plan

With over 1k backup volumes and over 1k volume snapshot backups under pretty high network latency (700-800ms per operation)
from longhorn manager to the remote backup store:
- Create a single cluster with 2 volumes (vol-A and vol-B).
   1. The user configures the remote backup store S3 (s3://backupbucket@us-east-1/).
   2. The user creates two volume snapshots on vol-A and vol-B.
   3. The user can see the backup volume for vol-A and vol-B in backups GUI.
   4. The user can see the two volume snapshot backups under vol-A and vol-B in backups GUI.
   5. When the user deletes one of the volume snapshot backup of vol-A on the Longhorn GUI, the delete one will be deleted after the remote backup store backup be deleted.
   6. When the user deletes backup volume vol-A on the Longhorn GUI, the backup volume will be deleted after the remote backup store backup volume be deleted, and the volume snaphost backup of vol-A will be delete also.
   7. The user can see the backup volume for vol-B in backups GUI.
   8. The user can see two volume snapshot backups under vol-B in backups GUI.
   9. The user changes the remote backup store to another S3 (s3://backupbucket@us-east-2/), the user can't see the backup volume and volume snapshot backup of vol-B in backups GUI.
   10. The user configures the backstore poll interval to 1 mins.
   11. The user changes the remote backup store to original one S3 (s3://backupbucket@us-east-1/), after 1 mins later, the user can see the backup volume and volume snapshot backup of vol-B.
- Create two clusters (cluster-A and cluster-B) both points to the same remote backup store.
   1. At cluster A, create a volume and run a recurring backup to the remote backup store.
   2. At cluster B, after `backupstore-poll-interval` seconds, the user can list backup volumes or list volume backups on the Longhorn GUI.
   3. At cluster B, create a DR volume from the backup volume.
   4. At cluster B, check the DR volume `status.LastBackup` and `status.LastBackupAt` be updated periodically.
   5. At cluster A, delete the backup volume on the GUI.
   6. At cluster B, after `backupstore-poll-interval` seconds, the deleted backup volume does not exist on the Longhorn GUI.
   7. At cluster B, the DR volume `status.LastBackup` and `status.LastBackupAt` won't be updated anymore.

### Upgrade strategy

Before this enhancement, the user might set `backupstore-poll-interval` to `0` to reduce the request to the remote backup store.
This is majorly for the user who wants to reduce the cost on access AWS S3.

After this enhancement, if the user ever configured the setting `backupstore-poll-interval` to `0`, then the user can't see the backup volumes
on the Longhorn GUI. We'll add a warning message on the Longhorn GUI to notify the user to configure the `backupstore-poll-interval` greater 
than `0`, so the backup volume controller can synchronous the backup volumes from the remote backup store.

## Note

- With this enhancement, the CR _backupvolumes.longhorn.io_ and _backups.longhorn.io_ are updated when the controller resync timer trigger and `the current time - status.lastSyncedTime >= pollInterval`. The user might want to trigger it immediately. However currently, we haven't found a good way to let the user force trigger it now.

- If the user clicks volume backup on a new volume, the Backup CR will be create after backup controller finished volume snapshot backup. However, the BackupVolume CR will delay created after backup volume controller pull the new BackupVolume from the remote backup store. This is because we haven't found a way to from the backup controller to notify the backup volume controller to perform pull the remote backup store to update the BackupVolume status immediately.
